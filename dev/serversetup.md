# Setup Notes for the OmniOSce Repo Server

## Setting Up the Repo Servers

```bash
port=10000
for release in r151022 bloody; do
    for arch in core extra staging; do
        if [ $arch = staging  -a $release = bloody ]; then
           continue
        fi
        ((port++))
        pub=omnios
        collection=core
        if [ $arch = extra ]; then
           pub=omnios-extra
           collection=supplemental
        fi
	pkgrepo create /pkg/$release/$arch
	pkgrepo set -s /pkg/$release/$arch publisher/prefix=$pub
	pkgrepo set -s /pkg/$release/$arch -p $pub repository/collection_type=core
	pkgrepo set -s /pkg/$release/$arch -p $pub repository/description="IPS Packages for OmniOS $release $arch"
	pkgrepo set -s /pkg/$release/$arch -p $pub repository/name="OmniOS $release $arch"
	svccfg -s pkg/server add ${release}_$arch
	svccfg -s pkg/server:${release}_$arch addpg pkg application
	svccfg -s pkg/server:${release}_$arch setprop pkg/inst_root = /pkg/$release/$arch
	svccfg -s pkg/server:${release}_$arch setprop pkg/content_root = /pkg/content_root
	svccfg -s pkg/server:${release}_$arch setprop pkg/port = $port
	svccfg -s pkg/server:${release}_$arch setprop pkg/proxy_base = https://pkg.omniosce.org/$release/$arch
	svccfg -s pkg/server:${release}_$arch setprop pkg/address = 127.0.0.1
	svcadm enable  pkg/server:${release}_$arch
	svcadm restart  pkg/server:${release}_$arch
    done
done
```

## Nginx Config

```
worker_processes  4;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 10000;
    types_hash_max_size 2048;
    server_tokens off;
    client_max_body_size 200M;

        ##
    # SSL Settings
    ##

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS:!DES-CBC3-SHA';
    ssl_dhparam /etc/ssl/private/dhparam.pem;
    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    add_header Strict-Transport-Security max-age=15768000;
    ssl_stapling on; ssl_stapling_verify on;

    server {
        listen 80 ;
        listen [::]:80 ;
        server_name _;
        location ^~ /.well-known {
            allow all;
            alias /var/www/html/.well-known;
        }
        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl;
        server_name pkg.omniosce.org;
        
        ssl_certificate /etc/ssl/certs/acme-omniosce.org.pem;
        ssl_certificate_key /etc/ssl/private/acme-omniosce.org.key;
        # ssl_client_certificate /etc/ssl/certs/omniosce-client-ca.pem
        # ssl_verify_client optional
        set $auth_and_method "$ssl_client_verify$request_method";

        location /r151022/core {
            if ($auth_and_method ~ "FAILED.+(POST|PUT|DELETE)"){
		return 404;
	    }
            proxy_pass http://127.0.0.1:10001/;
        }

        location /r151022/extra {
            if ($auth_and_method ~ "FAILED.+(POST|PUT|DELETE)"){
		return 404;
	    }
            proxy_pass http://127.0.0.1:10002/;
        }

                
        location /r151022/staging {
            if ($auth_and_method ~ "FAILED.+(POST|PUT|DELETE)"){
                return 404;
            }
            proxy_pass http://127.0.0.1:10003/;
        }

        location /bloody/core {
            if ($auth_and_method ~ "FAILED.+(POST|PUT|DELETE)"){
                return 404;
            }
            proxy_pass http://127.0.0.1:10004/;
        }
        
        location /bloody/extra {
            if ($auth_and_method ~ "FAILED.+(POST|PUT|DELETE)"){
                return 404;
            }
            proxy_pass http://127.0.0.1:10005/;
        }
        location /r151022 {
            return 301 https://$host/r151022/core;
        }
        location /bloody {
            return 301 https://$host/bloody/core;
        }
        location / {
            return 301 https://$host/r151022/core;
        }
    }
    server {
        listen       443 ssl;
        server_name mirrors.omniosce.org;

        ssl_certificate /etc/ssl/certs/acme-omniosce.org.pem;
        ssl_certificate_key /etc/ssl/private/acme-omniosce.org.key;
        location / {
            root /mirrors;
            autoindex on;
        }
    }
   server {
        listen       443 ssl;
        server_name downloads.omniosce.org;

        ssl_certificate /etc/ssl/certs/acme-omniosce.org.pem;
        ssl_certificate_key /etc/ssl/private/acme-omniosce.org.key;
        location / {
            root /downloads;
            autoindex on;
        }
    }

    server {
        listen       443 ssl;
        server_name crl.omniosce.org;

        ssl_certificate /etc/ssl/certs/acme-omniosce.org.pem;
        ssl_certificate_key /etc/ssl/private/acme-omniosce.org.key;
        location / {
            root /downloads/crl;
        }
    }
}
```