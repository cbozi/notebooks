Nginx https反向代理http

## 安装Nginx

```
brew install nginx

==> /usr/local/etc/nginx/nginx.conf
sudo nginx
sudo nginx -s reload
```

[MAC下安装nginx - 个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000016020328)

## self-signed certificate

```
# Become a Certificate Authority
######################

# Generate private key
openssl genrsa -des3 -out myCA.key 2048
# Generate root certificate
openssl req -x509 -new -nodes -key myCA.key -sha256 -days 825 -out myCA.pem

######################
# Create CA-signed certs
######################

NAME=mydomain.com # Use your own domain name
# Generate a private key
openssl genrsa -out $NAME.key 2048
# Create a certificate-signing request
openssl req -new -key $NAME.key -out $NAME.csr
# Create a config file for the extensions
>$NAME.ext cat <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = $NAME # Be sure to include the domain name here because Common Name is not so commonly honoured by itself
DNS.2 = bar.$NAME # Optionally, add additional domains (I've added a subdomain here)
IP.1 = 192.168.0.13 # Optionally, add an IP address (if the connection which you have planned requires it)
EOF
# Create the signed certificate
openssl x509 -req -in $NAME.csr -CA myCA.pem -CAkey myCA.key -CAcreateserial \
-out $NAME.crt -days 825 -sha256 -extfile $NAME.ext
```

[ssl - Getting Chrome to accept self-signed localhost certificate - Stack Overflow](https://stackoverflow.com/questions/7580508/getting-chrome-to-accept-self-signed-localhost-certificate)

### Nginx conf

```
#user  nobody;
worker_processes 1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;
events {
    worker_connections 1024;
}


http {
    include mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
    '$status $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log logs/access.log main;

    sendfile on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout 65;

    gzip on;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    upstream mobile {
        server localhost:3000;
    }

    upstream www {
        server localhost:3001;
    }

    # HTTPS server
    #
    server {
        listen 8081 ssl;
        server_name localhost a.yuanfudao.biz;

        ssl_certificate /Users/yuanjiebj/git/self-shared-ceritficate/a.yuanfudao.biz.crt;
        ssl_certificate_key /Users/yuanjiebj/git/self-shared-ceritficate/a.yuanfudao.biz.key;

        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout 5m;

        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;


        location / {
            proxy_pass http://mobile;
        }
    }
    include servers/*;
}
```