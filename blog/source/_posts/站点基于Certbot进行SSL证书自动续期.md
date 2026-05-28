---
title: 基于certbot为站点自动续期SSL证书
date: 2026/5/24 22:17:06
page_cover: https://img1.baidu.com/it/u=2523242416,2388336867&fm=253&fmt=auto&app=138&f=JPEG
---

# 基于certbot为站点自动续期SSL证书

> 由于公司官网是基于Docker进行部署的，所以此次部署的文档都是基于Docker进行的

## 一、创建对应的文件夹

在宿主机的`/`目录下创建以下文件夹

```shell
mkdir -p /data/certbot/conf
mkdir -p /data/certbot/www
mkdir -p /daya/certbot/nginx-certbot
```

在`/data/certbot/nginx-certbot`文件夹下创建`docker-compose.yml`文件和`nginx.conf`文件

```yaml
version: "3"

services:
  nginx:
    image: registry.cn-shanghai.aliyuncs.com/jellybeans_company/jellybeans_portal:latest
    container_name: jellybeans_portal
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - /data/certbot/conf:/etc/letsencrypt
      - /data/certbot/www:/var/www/certbot
    restart: always

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - /data/certbot/conf:/etc/letsencrypt
      - /data/certbot/www:/var/www/certbot
    entrypoint: >
      sh -c "while true; do
        certbot renew --webroot -w /var/www/certbot;
        sleep 12h;
      done"
```

```nginx
# 未生成证书前先保留此部分
# HTTP（用于验证 + 跳转 HTTPS）
server {
    listen 80;
    server_name jellybeans.com.cn;

    # 给 :contentReference[oaicite:0]{index=0} 验证用
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # 所有请求跳转到 HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# 此部分在执行完证书生成步骤后再加上
# HTTPS（真正访问）
server {
    listen 443 ssl;
    server_name jellybeans.com.cn;

    # 证书路径（Certbot 生成）
    ssl_certificate /etc/letsencrypt/live/jellybeans.com.cn/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/jellybeans.com.cn/privkey.pem;

    # 推荐安全配置（加上更规范）
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # 你的网站
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    # 静态资源缓存（优化用）
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 30d;
    }
}

```

## 二、申请证书

> 新申请证书

```shell
docker run --rm
	-v /data/certbot/conf:/etc/letsencrypt
	-v /data/certbot/www:/var/www/certbot certbot/certbot
	certonly
    --webroot
    --webroot-path=/var/www/certbot
    -d jellybeans.com.cn
    --email xueyinghao@jellybeans.com.cn
    --agree-tos --no-eff-email
```

> 追加申请证书

```shell
docker run --rm
	-v /data/certbot/conf:/etc/letsencrypt
	-v /data/certbot/www:/var/www/certbot
	certbot/certbot certonly
	--webroot
	--webroot-path=/var/www/certbot
	-d jellybeans.com.cn
	-d www.jellybeans.com.cn
	--email xueyinghao@jellybeans.com.cn
	--agree-tos
	--no-eff-email
	--expand
```

> **证书生成的位置：**

`/data/certbot/conf/live/jellybeans.com.cn/`文件夹下，里面包含：

- `fullchain.pem`
- `privkey.pem`

## 三、自动续期

再ECS中执行以下`shell`命令：

```shell
crontab -e
```

再`crontab`中填写如下命令：

```shell
0 3 * * * docker run --rm -v /data/certbot/conf:/etc/letsencrypt -v /data/certbot/www:/var/www/certbot certbot/certbot renew --quiet --deploy-hook "docker exec my-nginx nginx -s reload"
```

## 四、修改完Nginx配置文件重启

```shell
docker exec jellybeans_portal nginx -s reload
```
