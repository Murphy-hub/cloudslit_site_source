---
title: "ZTALAB Best Practices"
date: 2022-05-07T14:19:28+08:00
draft: false
tags:
    - ZTA
    - ZTALAB
    - Practice
---

## ZTALAB Practice

This paper mainly records the rapid implementation and deployment of the main projects of ztalab, as well as the necessary configuration of each project, so as to realize the zero trust network security platform as soon as possible.

### Get Start

Our zero-trust platform initialization relies on the basic components of Mysql, ZACA and ZAManager. The following is the deployment method we have prepared for you for your reference. You can customize and modify it and use it better.

First of all, we need to create the container network we need:

```shell
$ docker network create zta
```

We all use docker-compose deployment of these components. The following is the docker-compose reference, but this docker-compose cannot be used immediately. Some initialization database, initialization configuration and other information need to be mapped, which will also be displayed next:

Finally, the new deployment directory folder is as follows:

```
$ tree ./
├── docker-compose.yaml
├── init.sql
├── nginx.conf
└── config.json
```

docker.compose.yml Example：

```yaml
version: '3.3'
services:
  mysql:
    image: mysql:5.7
    restart: always
    ports:
      - 3306:3306
    command: --init-file /data/application/init.sql
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - zta
    volumes:
      - ./init.sql:/data/application/init.sql
      - mysql-data:/var/lib/mysql

  root-ca-tls:
    image: zhangshuainbb/zaca:0.0.2
    command: ["./zaca", "tls"]
    volumes:
      - "./config.json:/etc/zaca/config.json"
    environment:
      IS_ENV: 'test'
      IS_HTTP_CA_LISTEN: '0.0.0.0:8083'
      IS_SINGLECA_CONFIG_PATH: '/etc/zaca/config.json'
      IS_INFLUXDB_ENABLED: 'false'
      IS_KEYMANAGER_SELF_SIGN: 'true'
      IS_MYSQL_DSN: 'root:root@tcp(mysql:3306)/root_cap?charset=utf8mb4&parseTime=True&loc=Local'
    depends_on:
      - mysql
    networks:
      - zta
    restart: always

  zaca-tls:
    image: zhangshuainbb/zaca:0.0.2
    command: ["./zaca", "tls"]
    ports:
      - "8081:8081"
    volumes:
      - "./config.json:/etc/zaca/config.json"
    environment:
      IS_ENV: 'test'
      IS_SINGLECA_CONFIG_PATH: '/etc/zaca/config.json'
      IS_KEYMANAGER_UPPER_CA: "https://root-ca-tls:8083"
      IS_MYSQL_DSN: 'root:root@tcp(mysql:3306)/cap?charset=utf8mb4&parseTime=True&loc=Local'
      IS_OCSP_HOST: 'http://zaca-ocsp:8082'
    depends_on:
      - mysql
      - root-ca-tls
    networks:
      - zta
    restart: always

  zaca-ocsp:
    image: zhangshuainbb/zaca:0.0.2
    command: ["./zaca", "ocsp"]
    ports:
      - "8082:8082"
    volumes:
      - "./ca_config.json:/etc/zaca/config.json"
    environment:
      IS_ENV: 'test'
      IS_SINGLECA_CONFIG_PATH: '/etc/zaca/config.json'
      IS_KEYMANAGER_UPPER_CA: "https://root-ca-tls:8083"
      IS_MYSQL_DSN: 'root:root@tcp(mysql:3306)/cap?charset=utf8mb4&parseTime=True&loc=Local'
    depends_on:
      - mysql
      - root-ca-tls
    networks:
      - zta
    restart: always

  zaca-api:
    image: zhangshuainbb/zaca:0.0.2
    command: ["./zaca", "api"]
    ports:
      - "8080:8080"
    volumes:
      - "./ca_config.json:/etc/zaca/config.json"
    environment:
      IS_ENV: 'test'
      IS_SINGLECA_CONFIG_PATH: '/etc/zaca/config.json'
      IS_KEYMANAGER_UPPER_CA: "https://root-ca-tls:8083"
      IS_MYSQL_DSN: 'root:root@tcp(mysql:3306)/cap?charset=utf8mb4&parseTime=True&loc=Local'
    depends_on:
      - mysql
      - root-ca-tls
    networks:
      - zta
    restart: always


  zta-portal:
    image: rovast/za-portal:v0.0.2
    restart: always
    ports:
      - '80:80'
      - '443:443'
    networks:
      - zta
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

  zta-backend:
    image: taosheng205054/zamanager:0.0.4
    restart: always
    environment:
      ZTA_MYSQL_HOST: 'mysql'
      ZTA_MYSQL_USER: 'root'
      ZTA_MYSQL_PASSWORD: 'root'
      ZTA_CA_SIGN_URL: 'https://root-ca-tls:8083'
      ZTA_CA_AUTH_KEY: '0739a645a7d6601d9d45f6b237c4edeadad904f2fce53625dfdd541ec4fc8134'
    depends_on:
      - mysql
    networks:
      - zta

networks:
  zta:
    driver: bridge

volumes:
  mysql-data:

```

Init.sql Example: Initialize database information

```sql
CREATE DATABASE IF NOT EXISTS root_cap;
CREATE DATABASE IF NOT EXISTS cap;
CREATE DATABASE IF NOT EXISTS zta;
```

nginx.conf Example: The front end of the zero-trust network platform needs the support of nginx，You need to modify the domain name to facilitate your access.

```shell
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  10240;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    keepalive_timeout 65;

    access_log off;
    error_log off;

    include mime.types;
    default_type application/octet-stream;

    gzip  on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/html text/richtext text/plain text/css text/x-script text/x-component text/x-java-source text/x-markdown application/javascript application/x-javascript text/javascript text/js image/x-icon image/vnd.microsoft.icon application/x-perl application/x-httpd-cgi text/xml application/xml application/xml+rss application/vnd.api+json  application/x-protobuf  application/json multipart/bag multipart/mixed application/xhtml+xml font/ttf font/otf font/x-woff image/svg+xml application/vnd.ms-fontobject application/ttf application/x-ttf application/otf application/x-otf application/truetype application/opentype application/x-opentype application/font-woff application/eot application/font application/font-sfnt application/wasm application/javascript-binast  application/manifest+json  application/ld+json;

    brotli on;
    brotli_static on;
    brotli_comp_level 6;
    brotli_types text/html text/richtext text/plain text/css text/x-script text/x-component text/x-java-source text/x-markdown application/javascript application/x-javascript text/javascript text/js image/x-icon image/vnd.microsoft.icon application/x-perl application/x-httpd-cgi text/xml application/xml application/xml+rss application/vnd.api+json  application/x-protobuf  application/json multipart/bag multipart/mixed application/xhtml+xml font/ttf font/otf font/x-woff image/svg+xml application/vnd.ms-fontobject application/ttf application/x-ttf application/otf application/x-otf application/truetype application/opentype application/x-opentype application/font-woff application/eot application/font application/font-sfnt application/wasm application/javascript-binast  application/manifest+json  application/ld+json;

    server {
        listen 80 default_server;

        server_name your_domain;  #here is your domain

        root /usr/share/nginx/html;
        index index.html;

        location ^~ /api/ {
            proxy_pass   http://zta-backend:8080;
        }

        location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|flv|mp4|ico)$ {
          expires 30d;
          access_log off;
        }

        location ~ .*\.(js|css)?$ {
          expires 7d;
          access_log off;
        }

        location / {
          try_files $uri $uri/ /index.html;
        }
    }
}

```

config.json Example: Zero-trust security certificate needs to configure some certificate issuance information.

```json
{
  "auth_keys": {
    "intermediate": {
      "type": "standard",
      "key": "52abb3ac91971bb72bce17e7a289cd04476490b19e0d8eb7810dc42d4ac16c41"
    },
    "default": {
      "type": "standard",
      "key": "0739a645a7d6601d9d45f6b237c4edeadad904f2fce53625dfdd541ec4fc8134"
    }
  },
  "signing": {
    "profiles": {
      "default": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "1440h",
        "copy_extensions": true,
        "auth_key": "default"
      },
      "intermediate": {
        "usages": [
          "digital signature",
          "cert sign",
          "crl sign"
        ],
        "expiry": "17520h",
        "copy_extensions": true,
        "auth_key": "intermediate",
        "ca_constraint": {
          "is_ca": true
        }
      }
    }
  }
}
```



### [Control Center Platform](https://github.com/ztalab/ZAManager)

When you deploy successfully, you can open the zero-trust platform address you configure. After opening it:

#### ZAManager Front-end page

![image-20220506184355830](https://user-images.githubusercontent.com/52234994/167241385-d0855ea9-729f-4820-b6d4-3e41837fa45f.png)

#### Resource management

![image-20220506184628646](https://user-images.githubusercontent.com/52234994/167241390-4b446ba7-f771-4b80-8fdf-4142431d83e8.png)

#### Service management

![image-20220506184957141](https://user-images.githubusercontent.com/52234994/167241392-516eab72-5bcf-41cc-997e-a9b816336269.png)

#### Client management

![image-20220506185220731](https://user-images.githubusercontent.com/52234994/167241398-4d103fb4-6d7e-461c-966f-2b30884cf1cc.png)



### **[ZTANetwork security construction](https://github.com/ztalab/ZASentinel)**

The minimum closed loop of ZASentinel requires two service components, the client and server. We use the simplest way to run this project, which only requires server and client certificates to run:

In the ZAManager service, add resources, servers and clients (here you should understand the basic common sense and components of zero-trust projects, and I will not repeat them here)

After adding server and client resources, download the corresponding certificate as the project certificate configuration of ZA Sentinel (which can also be configured in the configuration file or environment variable configuration).

The same way the server and client startup rely on the corresponding certificate.

#### Server

Start the server in the private network and ensure that the server can access private resources:

```
$ git clone git@github.com:ztalab/ZASentinel.git
$ cd ZASentinel && mkdir cert
$ cd cert
$ vim ca.pem
....
$ vim cert.pem
....
```

Start the project. Here we configure the server according to the certificate, and the port is 5091:

```
$ go run cmd/main.go -c configs/config.toml
```

<img src="https://user-images.githubusercontent.com/52234994/167241288-36a4d0ad-68cd-449f-841c-070488cc2b13.png" alt="image-20220506171751457" style="width: 80%;" />

The port started by the server is not directly accessed by the client. It is necessary to hang the server in the nginx agent, provide the ssl certificate service, and provide the server with public network services. Refer to the nginx configuration file:

The external port here defaults to port 443, and the client will also request the domain name and port configured by this nginx.

You need to modify the domain name you configure, which needs to be consistent with your new server or trunk information on the ZTA platform.

```shell
server {
    listen 443 ssl; 							# The port is open here
    server_name server.demo.xyz;	# Here is the domain name of your server or relay
  
    ssl_certificate  cert/fullchain.crt;
    ssl_certificate_key cert/private.pem;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    location / {
      proxy_set_header X-Forwarded-For $remote_addr;
      proxy_set_header Host            $http_host;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_ssl_name "server.demo.xyz"; # Here is the domain name of your server or relay
      proxy_ssl_server_name on;
      proxy_pass http://127.0.0.1:5091; # Here is the listening port of your server or relay
    }
}
server {
    listen 80;										  # The port is open here
    server_name server.demo.xyz;		# Here is the domain name of your server or relay
    rewrite ^(.*)$ https://$host$1 permanent;
}
```

#### Relay

The relay is a non-essential service. The client can directly connect to the server or add multiple repeaters, which mainly depends on whether you need a relay.

The deployment of the trunk can change geographically. His role is more inclined to accelerate the network. As with the deployment of the server, as a basic service, it is necessary to use its own services with nginx prox agents and provide ssl certificate services.

Secondly, Relay can be hung on the CDN network to better prevent DDOS attacks.

#### Client

Start the client locally, and the startup method is the same as that of the server. Here, the startup client is configured according to the certificate, and the port is 3091:

![image-20220506173134287](https://user-images.githubusercontent.com/52234994/167241289-6ccabaef-67dc-4631-a4bf-9b356fec9695.png)

#### Test

The resource I just configured in ZAManager is mysql, the responding server and client, and we also configure mysql, so we directly connect to the client, and the client will proxy our connection:

```shell
$ mysql -h 127.0.0.1 -uroot -P3091 -p
```

![image-20220506174223718](https://user-images.githubusercontent.com/52234994/167241292-1a28a026-7fc9-46d0-8354-28dfcae5b019.png)