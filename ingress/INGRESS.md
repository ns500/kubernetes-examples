Kubernetes Ingress For ElasticSearch and Kibana
----------------------------------------------------

# Ingress Controller
```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ingress-lb
spec:
  template:
    metadata:
      labels:
        name: nginx-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60      
      containers:
      - image: nginx-ingress-controller:0.5
        name: nginx-ingress-lb
        imagePullPolicy: Always
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10249
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        # use downward API
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 4444
        args:
        - /nginx-ingress-controller
        - --default-backend-service=default/default-http-backend
```

```Dockfile
# Copyright 2015 The Kubernetes Authors. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

FROM gcr.io/google_containers/nginx-slim:0.5

RUN apt-get update && apt-get install -y \
  diffutils procps net-tools \
  --no-install-recommends \
  && rm -rf /var/lib/apt/lists/*

COPY nginx-ingress-controller /
COPY nginx.tmpl /
COPY default.conf /etc/nginx/nginx.conf

COPY lua /etc/nginx/lua/

WORKDIR /

CMD ["/nginx-ingress-controller"]
```

```bash
docker build -t nginx-ingress-controller:0.5 .
```

```tmpl
{{ $cfg := .cfg }}
daemon off;

worker_processes {{ $cfg.workerProcesses }};

pid /run/nginx.pid;

worker_rlimit_nofile 131072;

pcre_jit on;

events {
    multi_accept        on;
    worker_connections  {{ $cfg.maxWorkerConnections }};
    use                 epoll; 
}

http {
    {{ if $cfg.enableVtsStatus}}vhost_traffic_status_zone shared:vhost_traffic_status:{{ $cfg.vtsStatusZoneSize }};{{ end }}

    # lus sectrion to return proper error codes when custom pages are used
    lua_package_path '.?.lua;./etc/nginx/lua/?.lua;/etc/nginx/lua/vendor/lua-resty-http/lib/?.lua;';
    init_by_lua_block {        
        require("error_page")
    }

    sendfile            on;
    aio                 threads;
    tcp_nopush          on;
    tcp_nodelay         on;
    
    log_subrequest      on;

    reset_timedout_connection on;

    keepalive_timeout {{ $cfg.keepAlive }}s;

    types_hash_max_size 2048;
    server_names_hash_max_size {{ $cfg.serverNameHashMaxSize }};
    server_names_hash_bucket_size {{ $cfg.serverNameHashBucketSize }};

    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    {{ if $cfg.useGzip }}
    gzip on;
    gzip_comp_level 5;
    gzip_http_version 1.1;
    gzip_min_length 256;
    gzip_types {{ $cfg.gzipTypes }};    
    gzip_proxied any;
    gzip_vary on;
    {{ end }}

    client_max_body_size "{{ $cfg.bodySize }}";

    {{ if $cfg.useProxyProtocol }}
    set_real_ip_from    {{ $cfg.proxyRealIpCidr }};
    real_ip_header      proxy_protocol;
    {{ end }}

    log_format upstreaminfo '{{ if $cfg.useProxyProtocol }}$proxy_protocol_addr{{ else }}$remote_addr{{ end }} - '
        '[$proxy_add_x_forwarded_for] - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" '
        '$request_length $request_time $upstream_addr $upstream_response_length $upstream_response_time $upstream_status';

    access_log /var/log/nginx/access.log upstreaminfo;
    error_log  /var/log/nginx/error.log {{ $cfg.errorLogLevel }};

    {{ if not (empty .defResolver) }}# Custom dns resolver.
    resolver {{ .defResolver }} valid=30s;
    {{ end }}

    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }


    # Map a response error watching the header Content-Type
    map $http_accept $httpAccept {
        default          html;
        application/json json;
        application/xml  xml;
        text/plain       text;
    }

    map $httpAccept $httpReturnType {
        default          text/html;
        json             application/json;
        xml              application/xml;
        text             text/plain;
    }

    server_name_in_redirect off;
    port_in_redirect off;

    ssl_protocols {{ $cfg.sslProtocols }};

    # turn on session caching to drastically improve performance
    {{ if $cfg.sslSessionCache }}
    ssl_session_cache builtin:1000 shared:SSL:{{ $cfg.sslSessionCacheSize }};
    ssl_session_timeout {{ $cfg.sslSessionTimeout }};
    {{ end }}

    # allow configuring ssl session tickets
    ssl_session_tickets {{ if $cfg.sslSessionTickets }}on{{ else }}off{{ end }};

    # slightly reduce the time-to-first-byte
    ssl_buffer_size {{ $cfg.sslBufferSize }};

    {{ if not (empty $cfg.sslCiphers) }}
    # allow configuring custom ssl ciphers
    ssl_ciphers '{{ $cfg.sslCiphers }}';
    ssl_prefer_server_ciphers on;
    {{ end }}

    {{ if not (empty .sslDHParam) }}
    # allow custom DH file http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_dhparam
    ssl_dhparam {{ .sslDHParam }};
    {{ end }}

    # Custom error pages
    proxy_intercept_errors on;

    error_page 403 = @custom_403;
    error_page 404 = @custom_404;
    error_page 405 = @custom_405;
    error_page 408 = @custom_408;
    error_page 413 = @custom_413;
    error_page 501 = @custom_501;
    error_page 502 = @custom_502;
    error_page 503 = @custom_503;
    error_page 504 = @custom_504;

    # In case of errors try the next upstream server before returning an error
    proxy_next_upstream                     error timeout invalid_header http_502 http_503 http_504 {{ if $cfg.retryNonIdempotent }}non_idempotent{{ end }};

    server {
        listen 80 default_server{{ if $cfg.useProxyProtocol }} proxy_protocol{{ end }};

        location / {
            return 200;
        }

        location /nginx_status {
            allow 127.0.0.1;
            deny all;
            
            access_log off;
            stub_status on;
        }

        {{ template "CUSTOM_ERRORS" $cfg }}
    }

    {{range $name, $upstream := .upstreams}}
    upstream {{$upstream.Name}} {
        least_conn;
        {{range $server := $upstream.Backends}}server {{$server.Address}}:{{$server.Port}};
        {{end}}
    }
    {{end}}

    {{ range $server := .servers }}
    server {
        listen 80;
        {{ if $server.SSL }}listen 443 ssl http2;
        ssl_certificate {{ $server.SSLCertificate }};
        ssl_certificate_key {{ $server.SSLCertificateKey }};{{ end }}
        {{ if $cfg.enableVtsStatus }}
        vhost_traffic_status_filter_by_set_key {{ $server.Name }} application::*;
        {{ end }}

        server_name {{ $server.Name }};

        {{ range $location := $server.Locations }}
        location {{ $location.Path }} {
            proxy_set_header Host                   $host;

            # Pass Real IP
            proxy_set_header X-Real-IP              $remote_addr;

            # Allow websocket connections
            proxy_set_header                        Upgrade           $http_upgrade;
            proxy_set_header                        Connection        $connection_upgrade;

            proxy_set_header X-Forwarded-For        $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Host       $host;
            proxy_set_header X-Forwarded-Server     $host;

            proxy_connect_timeout                   {{ $cfg.proxyConnectTimeout }}s;
            proxy_send_timeout                      {{ $cfg.proxySendTimeout }}s;
            proxy_read_timeout                      {{ $cfg.proxyReadTimeout }}s;

            proxy_redirect                          off;
            proxy_buffering                         off;

            proxy_http_version                      1.1;

            proxy_pass http://{{ $location.Upstream.Name }};
        }
        {{ end }}
        {{ template "CUSTOM_ERRORS" $cfg }}
    }
    {{ end }}

    # default server, including healthcheck
    server {
        listen 8080 default_server{{ if $cfg.useProxyProtocol }} proxy_protocol{{ end }} reuseport;

        location /healthz {
            access_log off;
            return 200;
        }
       
        location /health-check {
            access_log off;
            proxy_pass http://127.0.0.1:10249/healthz;
        }

        location /nginx_status {
            {{ if $cfg.enableVtsStatus }}
            vhost_traffic_status_display;
            vhost_traffic_status_display_format html;
            {{ else }}
            access_log off;
            stub_status on;
            {{ end }}
        }

        location / {
            proxy_pass             http://upstream-default-backend;
        }
        {{ template "CUSTOM_ERRORS" $cfg }}
    }

    # default server for services without endpoints
    server {
        listen 8181;

        location / {
            content_by_lua_block {
                openURL(503)
            }
        }        
    }    
}


stream {

# TCP services
{{ range $i, $tcpServer := .tcpUpstreams }}
    upstream tcp-{{ $tcpServer.Upstream.Name }} {
        {{ range $server := $tcpServer.Upstream.Backends }}server {{ $server.Address }}:{{ $server.Port }};
        {{ end }}
    }

    server {
        listen {{ $tcpServer.Path }};
        proxy_connect_timeout  {{ $cfg.proxyConnectTimeout }};
        proxy_timeout          {{ $cfg.proxyReadTimeout }};
        proxy_pass             tcp-{{ $tcpServer.Upstream.Name }};
    }
{{ end }}

# UDP services
{{ range $i, $udpServer := .udpUpstreams }}
    upstream udp-{{ $udpServer.Upstream.Name }} {
        {{ range $server := $udpServer.Upstream.Backends }}server {{ $server.Address }}:{{ $server.Port }};
        {{ end }}
    }

    server {
        listen {{ $udpServer.Path }} udp;
        proxy_timeout          10s;
        proxy_responses        1;
        proxy_pass             udp-{{ $udpServer.Upstream.Name }};
    }
{{ end }}

}

{{/* definition of templates to avoid repetitions */}}
{{ define "CUSTOM_ERRORS" }}
        location @custom_403 {
            internal;
            content_by_lua_block {
                openURL(403)
            }
        }

        location @custom_404 {
            internal;
            content_by_lua_block {
                openURL(404)
            }
        }

        location @custom_405 {
            internal;
            content_by_lua_block {
                openURL(405)
            }
        }

        location @custom_408 {
            internal;
            content_by_lua_block {
                openURL(408)
            }
        }

        location @custom_413 {
            internal;
            content_by_lua_block {
                openURL(413)
            }
        }

        location @custom_502 {
            internal;
            content_by_lua_block {
                openURL(502)
            }
        }

        location @custom_503 {
            internal;
            content_by_lua_block {
                openURL(503)
            }
        }

        location @custom_504 {
            internal;
            content_by_lua_block {
                openURL(504)
            }
        }
{{ end }}
```

```yaml
# An Ingress with 2 hosts and 3 endpoints
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echomap
spec:
  backend:
    serviceName:
    servicePort: 80
  rules:
  - host: caas.one
    http:
      paths:
      - path: /kibana
        backend:
          serviceName: kibana
          servicePort: 5601
  - host: jingru.io
    http:
      paths:
      - path: /es
        backend:
          serviceName: elasticsearch
          servicePort: 9200
```