---
title: 用Docker部署个人RSS服务（TTRSS+RSSHub）
key: 2019-06-03-用Docker部署个人RSS服务（TTRSS+RSSHub）
tags: RSS Docker Nginx

---

由于对不断刷新各种Timeline感到烦躁，深感对人生的浪费，尝试回归一下原始的方式，重新获得对信息流的绝对控制。

考察了一下，TTRSS+RSSHub似乎是个过渡的好方案，简单在服务器上搭一搭，看看能不能让信息获取变得更有效率一些。

<!--more-->

## 安装Docker
Docker实在是个方便的东西，所以采用其对服务进行部署，安装方式不再赘述，见Docker官方文档。  
> [Get Docker CE for Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

## 使用Docker Compose进行配置
首先安装Docker Compose  
```shell
$ sudo apt install docker-compose
```

再参照[Awesome-TTRSS](https://github.com/HenryQW/Awesome-TTRSS/blob/master/docker-compose.yml)及[RSSHub](https://github.com/DIYgod/RSSHub/blob/master/docker-compose.yml)写一个自己的docker-compose.yml。

```yaml docker-compose.yml
version: '3'

services:
  service.ttrss:
    image: wangqiru/ttrss:latest
    container_name: ttrss
    environment:
      - DB_HOST=db.postgres
      - DB_PORT=5432
      - DB_NAME=ttrss
      - DB_USER=postgres
      - ENABLE_PLUGINS=auth_internal,fever # auth_internal is required. Plugins enabled here will be enabled for all users as system plugins
    env_file:
      - ./ttrss/ttrss.env
    stdin_open: true
    tty: true
    restart: always
    command: sh -c 'sh /wait-for.sh db.postgres:5432 -- php /configure-db.php && exec s6-svscan /etc/s6/'

  service.mercury: # set Mercury Parser API endpoint to `service.mercury:3000` on TTRSS plugin setting page
    image: wangqiru/mercury-parser-api:latest
    container_name: mercury
    expose:
      - 3000
    restart: always

  service.opencc: # set OpenCC API endpoint to `service.opencc:3000` on TTRSS plugin setting page
    image: wangqiru/opencc-api-server:latest
    container_name: opencc
    environment:
      NODE_ENV: production
    expose:
      - 3000
    restart: always

  service.rsshub:
    image: diygod/rsshub
    container_name: rsshub
    restart: always
    expose: 
      - 1200
    environment:
      NODE_ENV: production
      CACHE_TYPE: redis
      REDIS_URL: 'redis://db.redis:6379/'
      PUPPETEER_WS_ENDPOINT: 'ws://service.browserless:3000'
    env_file:
      - ./rsshub/rsshub.env
    depends_on:
      - db.redis
      - service.browserless

  service.browserless:
    image: browserless/chrome
    container_name: browserless
    restart: always

  db.redis:
    image: redis
    container_name: redis
    restart: always
    volumes:
        - ./redis/data/:/data

  db.postgres:
    image: sameersbn/postgresql:latest
    container_name: postgres
    environment:
      - DB_NAME=ttrss
      - DB_EXTENSION=pg_trgm
    env_file:
      - ./postgres/postgres.env
    volumes:
      - ./postgres/data/:/var/lib/postgresql/
    restart: always
  
  server.nginx:
    image: nginx:stable
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./letsencrypt:/etc/letsencrypt
    depends_on:
      - service.rsshub
      - service.ttrss 
    restart: always
```

整体目录结构
```
├── docker-compose.yml
├── letsencrypt
├── nginx
│   ├── conf.d
│   │   └── example.org.conf
│   └── nginx.conf
├── postgres
│   └── postgres.env
├── rsshub
│   └── rsshub.env
└── ttrss
    └── ttrss.env
```

通过.env文件配置相关环境变量。

**postgres.env:**
``` env
PG_PASSWORD={YOUR_DB_PASSWORD} # replace this with your prefered password
```
更多可用环境变量见[sameersbn/postgresql](https://hub.docker.com/r/sameersbn/postgresql/)。

**ttrss.env:**
``` env
SELF_URL_PATH=https://rss.example.org/
DB_PASS={YOUR_DB_PASSWORD} # same as the password in postgres.env
```

**rsshub.env:**
``` env
HTTP_BASIC_AUTH_NAME={YOUR_NAME}
HTTP_BASIC_AUTH_PASS={YOUR_PASSWORD}
```

更多可用环境变量见[RSShub/配置](https://docs.rsshub.app/install/#配置)，以启用更多如Twitter、Bilibli、Pixiv订阅源。

注：使用env_file方式配置环境变量，无需使用各类引号。

## 使用Certbot获取Let's Encrypt证书
由于打算使用rss.example.org访问ttrss，rsshub.example.org来访问同一服务器的rsshub，直接获取域名的通配符证书。

获取通配符证书需要向DNS添加TXT记录，若域名托管在Cloudflare上，可使用certbot的cloudflare插件自动完成这一过程。

首先在``letsencrypt``目录下创建``cloudflare.ini``进行如下配置，并设置其权限为600：
``` ini
dns_cloudflare_email = {YOUR_CLOUDFLARE_EMAIL}
dns_cloudflare_api_key = {YOUR_CLOUDFLARE_GLOBAL_API_KEY}
```
API KEY可在[账户信息](https://www.cloudflare.com/a/account/my-account)页末尾获取。

使用certbot的docker镜像获取证书。
``` shell
$ docker run -it --rm --name certbot \
            -v "{ABSOLUTE_PATH_TO_DIRECTORY_letsencrypt}:/etc/letsencrypt" \
            certbot/dns-cloudflare certonly \
            --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
            --dns-cloudflare-propagation-seconds 60 \
            --server https://acme-v02.api.letsencrypt.org/directory \
            -d example.org -d '*.example.org'
```
回车后输入邮箱及同意相关协议，等待其设置完毕并获取证书，成功后letsencrypt目录结构如下：
```
letsencrypt
├── accounts
│   └── ...
├── archive
│   └── ...
├── cloudflare.ini
├── csr
│   └── ...
├── keys
│   └── ...
├── live
│   ├── example.org
│   │   ├── cert.pem
│   │   ├── chain.pem
│   │   ├── fullchain.pem
│   │   ├── privkey.pem
│   │   └── README
│   └── README
├── renewal
│   └── example.org.conf
└── renewal-hooks
    ├──...
```

Let's Encrypt证书有效期为90天，输入``crontab -e``来添加一条任务以自动更新证书并重启Nginx。
```
0 0 */15 * * docker run -t --rm -v {ABSOLUTE_PATH_TO_DIRECTORY_letsencrypt}:/etc/letsencrypt -v /var/log/letsencrypt:/var/log/letsencrypt certbot/dns-cloudflare renew --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini && docker kill -s HUP nginx >/dev/null 2>&1
```



## 配置Nginx
对``nginx/conf.d/example.org.conf``进行如下配置：
``` conf
server {
    listen      80;
    listen [::]:80;
    server_name rss.example.org rsshub.example.org;

    location / {
        rewrite ^ https://$host$request_uri? permanent;
    }
}

server {
    listen      443           ssl http2;
    listen [::]:443           ssl http2;
    server_name               rss.example.org rsshub.example.org;

    add_header                Strict-Transport-Security "max-age=31536000" always;

    ssl_session_cache         shared:SSL:20m;
    ssl_session_timeout       10m;

    ssl_protocols             TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers               "ECDH+AESGCM:ECDH+AES256:ECDH+AES128:!ADH:!AECDH:!MD5;";

    ssl_stapling              on;
    ssl_stapling_verify       on;
    resolver                  1.1.1.1 1.0.0.1;

    ssl_certificate           /etc/letsencrypt/live/example.org/fullchain.pem;
    ssl_certificate_key       /etc/letsencrypt/live/example.org/privkey.pem;
    ssl_trusted_certificate   /etc/letsencrypt/live/example.org/chain.pem;

    access_log                /dev/stdout main;
    error_log                 /dev/stderr info;

    if ($host ~* "^(.*?)\.example\.org$") {
                set $domain $1;
        }

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        if ($domain ~* "^rss$" ){
            proxy_pass http://service.ttrss;
        }
        if ($domain ~* "^rsshub$" ){
            proxy_pass http://service.rsshub:1200;
        }
    }

}
```
如此来将不同二级域名代理到不同服务，同时强制所有http请求重定向到https。

若同时使用Cloudflare的免费加速服务，须将站点设置Crypto中的SSL设为FULL，则http请求可正常跳转。

## 启动！

回到``docker-compose.yml``目录，输入：
``` shell
$ docker-compose up -d
```
启动所有服务。

TTRSS mercury及opencc插件启用不再赘述，见[Awesome TTRSS部署/插件](https://ttrss.henry.wang/zh/#插件)

## 最后
输入``https://rss.example.org``并访问，用默认用户admin及密码password登录就可以愉快地使用自己的RSS服务了。
