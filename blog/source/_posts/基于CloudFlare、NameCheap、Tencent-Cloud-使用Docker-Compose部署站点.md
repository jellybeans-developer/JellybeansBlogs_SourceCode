---
title: 基于CloudFlare、NameCheap、Tencent Cloud 使用Docker Compose部署站点
date: 2026-05-29 17:36:47
tags: CloudFlare NameCheap
---

# 在Tencent Cloud 安装Docker Git



## Docker简介与操作配置



### Docker是什么

Docker是容器运行时，安装Docker后就可执行以下操作：
- 创建容器
- 启动容器
- 停止容器
- 删除容器
- 拉取镜像
等一些列操作。

> Docker一系列操作演示
```shell
# 安装 Docker
yum install -y docker

# 启动 Docker 服务
systemctl start docker

# 拉取 nginx 镜像
docker pull nginx

# 运行 nginx
docker run -d \
  --name nginx \
  -p 80:80 \
  nginx
```



### Docker Compose是什么

Dokcer Compose 是Docker的编排工具。如果开发者的项目中有多个容器，例如：
```shell
博客项目
├── nginx
├── hexo
├── mysql
├── redis
```
如果不用Compose 那么就要这样启动这些容器：
```shell
docker run ...
docker run ...
docker run ...
docker run ...
```
而Compose可以将这些配置到一个文档中
```yaml
services:

  nginx:
    image: nginx
    ports:
      - "80:80"

  redis:
    image: redis

  mysql:
    image: mysql
```
然后执行shell命令，全部启动
```shell
docker compose up -d
```
全部停止
```shell
docker compose down
```

重启
```shell
docker compose restart
```



### 在Centos中安装Docker

> 安装Docker
```shell
yum install -y yum-utils

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install -y docker-ce docker-ce-cli containerd.io
```

> 启动
```bash
systemctl enable docker
systemctl start docker
```

> 查看
```shell
docker version
```


### 安装Docker Compose

在之前需要单独安装，但是现在在前面安装完Docker之后Compose就已经安装好了

```shell
# 以前安装Docker Compose
docker-compose

# 现在已经作为Docker插件集成单独安装Dokcer Compose
yum install -y docker-compose-plugin
```
> 验证
```shell
docker compose version
```
> 安装成功输出
```shell
Docker Compose version v2.39.1
```



## Git的配置操作



### 在Centos中安装Git



#### 查看系统版本

```bash
car /etc/os-release
```
> 系统版本输出
```shell
CentOS Linux 7
#或者是
Centos Stream 8
```

#### Centos 7 安装Git
> 更新仓库
```shell
yum makecache
```

> 安装Git

```shell
yum install -y git
```

> 查看Git版本

```bash
git --version

# 安装成功显示版本信息
git version 1.8.3.1
```

#### 配置Git

> 配置Git

```bash
git config --global user.name "这里输入开发者用户名"
git config --global user.email "这里输入开发者邮箱地址"
```

> 查看配置信息

```bash
git config --global --list

# 配置信息列表
user.name=xxx
user.email=xxx@gmail.com
```



# 基于yaml文件部署newapi站点



## 当前项目的文件目录

```shell
/data/new-api
├── ssl
	├── cert.pem
	├── key.pem
├── docker-compose.yml
├── nginx.conf
```



## docker-compose.yml

```yaml
version: '3.4' # For compatibility with older Docker versions

services:
  new-api:
    image: calciumion/new-api:latest
    container_name: new-api
    restart: always
    command: --log-dir /app/logs
    ports:
      - "3000:3000" 
    volumes:
      - ./data:/data
      - ./logs:/app/logs
    environment:
      - SQL_DSN=postgresql://root:xxx@postgres:15432/new-api # ⚠️ IMPORTANT: Change the password in production!
#      - SQL_DSN=root:xxx@tcp(mysql:3306)/new-api  # Point to the mysql service, uncomment if using MySQL
      - REDIS_CONN_STRING=redis://:xxx@redis:36379 # ⚠️ IMPORTANT: Change the password in production!
      - TZ=Asia/Shanghai
      - ERROR_LOG_ENABLED=true # 是否启用错误日志记录 (Whether to enable error log recording)
      - BATCH_UPDATE_ENABLED=true  # 是否启用批量更新 (Whether to enable batch update)
      - NODE_NAME=new-api-node-1  # 节点名称，用于审计日志中标识节点身份；多节点/容器部署时建议设置 (Node name used in audit logs; recommended when running multiple instances or in containers)
#      - STREAMING_TIMEOUT=300  # 流模式无响应超时时间，单位秒，默认120秒，如果出现空补全可以尝试改为更大值 （Streaming timeout in seconds, default is 120s. Increase if experiencing empty completions）
#      - SESSION_SECRET=random_string  # 多机部署时设置，必须修改这个随机字符串！！ （multi-node deployment, set this to a random string!!!!!!!）
#      - SYNC_FREQUENCY=60  # Uncomment if regular database syncing is needed
#      - GOOGLE_ANALYTICS_ID=G-XXXXXXXXXX  # Google Analytics 的测量 ID (Google Analytics Measurement ID)
#      - UMAMI_WEBSITE_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  # Umami 网站 ID (Umami Website ID)
#      - UMAMI_SCRIPT_URL=https://analytics.umami.is/script.js  # Umami 脚本 URL，默认为官方地址 (Umami Script URL, defaults to official URL)

    depends_on:
      - redis
      - postgres
#      - mysql  # Uncomment if using MySQL
    networks:
      - new-api-network
    healthcheck:
      test: ["CMD-SHELL", "wget -q -O - http://localhost:3000/api/status | grep -o '\"success\":\\s*true' || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:stable-alpine
    container_name: newapi_nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - new-api 
    networks:
      - new-api-network

  redis:
    image: redis:latest
    container_name: redis
    restart: always
    command: ["redis-server", "--requirepass", "xxx"]  # ⚠️ IMPORTANT: Change this password in production!
    networks:
      - new-api-network

  postgres:
    image: postgres:15
    container_name: postgres
    restart: always
    environment:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: xxx  # ⚠️ IMPORTANT: Change this password in production!
      POSTGRES_DB: new-api
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - new-api-network
#    ports:
#      - "5432:5432"  # Uncomment if you need to access PostgreSQL from outside Docker

#  mysql:
#    image: mysql:8.2
#    container_name: mysql
#    restart: always
#    environment:
#      MYSQL_ROOT_PASSWORD: 123456  # ⚠️ IMPORTANT: Change this password in production!
#      MYSQL_DATABASE: new-api
#    volumes:
#      - mysql_data:/var/lib/mysql
#    networks:
#      - new-api-network
#    ports:
#      - "3306:3306"  # Uncomment if you need to access MySQL from outside Docker

volumes:
  pg_data:
#  mysql_data:

networks:
  new-api-network:
    driver: bridge

```

 在 `new-api`中将宿主机的容器的3000端口映射到宿主机的3000端口

同时需要在yaml文件中添加

```yaml
nginx:
    image: nginx:stable-alpine
    container_name: newapi_nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - new-api 
    networks:
      - new-api-network
```

这一段来启动一个nginx的容器 为后续的站点开始https使用。

> 需要注意

-  需要在`ports`部分添加 `- "443:443"` 。让nginx的容器的443端口映射到宿主机的443端口。

- 在`volumes`部分添加宿主机的nginx.conf文件挂载到容器的/etc/nginx/conf.d/default.conf

- `./ssl:/etc/nginx/ssl`这里需要将宿主机在cloudflare生成的证书的目录挂载到容器的`/etc/nginx/ssl`目录

-  `networks:  - new-api-network` 必须将nginx容器也加入到创建的网络中，即整个yaml文件中的容器都需要处在同一个网络中，否则在后面创建的`nginx.conf`文件中直接访问`new-api`容器的写法是访问不到`new-api`容器的。 



## nginx.conf

```ngi
server {
    listen 80;
    server_name 你的域名;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name 你的域名;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://new-api:3000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

-  ssl_certificate /etc/nginx/ssl/cert.pem;
- ssl_certificate_key /etc/nginx/ssl/key.pem;

**这里`nginx.conf`文件中关于ssl证书的位置是要配置为容器中的证书存放的位置。**因为在前面的`yaml`文件中我们是这样配置证书文件夹的挂载的` - ./ssl:/etc/nginx/ssl` 所以证书的位置需要配置如上。



# Cloudflare中的配置操作



## 登录Cloudflare配置Nameserver

### 第一步：添加域名并添加解析记录

登录 Cloudflare：

```
Add a Domain
```

输入你的域名：

```
xxx.com
```

然后选择：

```
Free Plan
```

即可。

<img src="https://jellybeans-developer.github.io/picx-images-hosting/cloudflare配置/image.lwdd3af2o.webp" style="zoom:50%;" />

<img src="https://jellybeans-developer.github.io/picx-images-hosting/cloudflare配置/image.7lkmuzoohp.webp" style="zoom:50%;" />

<img src="https://jellybeans-developer.github.io/picx-images-hosting/cloudflare配置/image.96adugmnny.webp" style="zoom:50%;" />

<img src="https://jellybeans-developer.github.io/picx-images-hosting/cloudflare配置/image.46boicpip.webp" style="zoom:50%;" />

<img src="https://jellybeans-developer.github.io/picx-images-hosting/cloudflare配置/image.6ikxk3zn5y.webp" style="zoom:50%;" />

**注意 ：在这里进行添加解析记录时 直接将Proxy Status打开**



### 第二步：修改域名 NS

Cloudflare 会给你两个 Nameserver：

例如：

```
alice.ns.cloudflare.com
bob.ns.cloudflare.com
```

然后到你的域名注册商后台：

如果你在 Namecheap 买的：

```
Domain List
↓
Manage
↓
Nameservers
↓
Custom DNS
```

把默认 NS 改成 Cloudflare 提供的两个 NS。

保存。

<img src="https://jellybeans-developer.github.io/picx-images-hosting/cloudflare配置/image.5q822dm0u6.webp" style="zoom:50%;" />

<img src="https://jellybeans-developer.github.io/picx-images-hosting/cloudflare配置/image.51esicz7dx.webp" style="zoom:50%;" />