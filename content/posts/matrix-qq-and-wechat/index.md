---
########## 文章标题
title: "Matrix QQ and Wechat"
########## 文章副标题
subtitle: ""
########## 页面图片, 用于 Open Graph 和 Twitter Cards
images: []
########## 用在主页预览的文章特色图片
featuredImagePreview: ""
################# 特色照片
featuredImage: "../matrix-qq-and-wechat/img/logo.png"
########## 标签
tags: [Matrix,QQ,微信]
########## 分类
categories: [笔记]

# 这篇文章创建的日期时间. 它通常是从文章的前置参数中的 date 字段获取的, 但是也可以在 网站配置 中设置
date: 2023-01-13T00:21:09+08:00
# 上次修改内容的日期时间
lastmod: 2023-01-13T00:21:09+08:00
# 如果设为 true, 除非 hugo 命令使用了 --buildDrafts/-D 参数, 这篇文章不会被渲染
draft: false
# 文章作者
author: ""
# 文章作者的链接
authorLink: ""
# 文章内容的描述
description: ""
# 这篇文章特殊的许可
license: ""
# 是否在主页隐藏一篇文章
hiddenFromHomePage: false
# 是否在搜索结果中隐藏一篇文章
hiddenFromSearch: false
# 是否使用 twemoji
twemoji: false
# 是否使用 lightgallery
lightgallery: true
# 是否使用 ruby 扩展语法
ruby: true
# 是否使用 fraction 扩展语法
fraction: true
# 是否使用 fontawesome 扩展语法
fontawesome: true
# 是否在文章页面显示原始 Markdown 文档链接
linkToMarkdown: true
# 是否在 RSS 中显示全文内容
rssFullText: false

toc:
  enable: true
  auto: true
code:
  copy: true
  maxShownLines: 168
math:
  enable: false
  # ...
mapbox:
  # ...
share:
  enable: true
  # ...
comment:
  enable: true
  # ...
library:
  css:
    # someCSS = "some.css"
    # located in "assets/"
    # Or
    # someCSS = "https://cdn.example.com/some.css"
  js:
    # someJS = "some.js"
    # located in "assets/"
    # Or
    # someJS = "https://cdn.example.com/some.js"
seo:
  images: []
  # ...
---

<!--more-->

## 起因

平常不同的朋友分散在不同的的社交APP，有些人吧，你微信叫她一百遍都不会理你，但是你QQ上哪怕是发个句号，她就会立马回复你。。

所以一直想搞一个聊天的聚合，可以在一个APP上处理多平台社交信息，本着白嫖及隐私自主的想法，找了一些试用了下，多多少少都有一些不如意。

经过一番折腾，最后还是选择了 **Matrix** ，开源免费，可以部署在家中也可以部署在云服务器，隐私自主安全，最合我意的是主力App **Element** 多平台上的应用感受都不错，手机上的也支持安全验证，可以整合多平台社交信息，奈斯！

然后不知道是不是政策的问题，相关的 **Matrix** 文章不多，参考官方文档部署足够，折腾了好几天，多多少少总有些问题，直到发现了这位大佬的帖子 [Matrix, QQ and Wechat](https://duo.github.io/posts/matrix-qq-wechat) 按照教程捣鼓了下，搞定了之前的问题。在这里感谢这位大佬的贡献，有需要可以直接去大佬的博客看下原始教程，这篇笔记只是自己按照大佬教程自己捣鼓后做下备忘笔记。

**此篇笔记用到一些大佬的链接：**

[GitHub - duo/matrix-qq: A Matrix-QQ puppeting bridge](https://github.com/duo/matrix-qq)

[GitHub - duo/matrix-wechat: A Matrix-WeChat puppeting bridge](https://github.com/duo/matrix-wechat)

[GitHub - duo/matrix-wechat-agent: An agent for matrix-wechat](https://github.com/duo/matrix-wechat-agent)

## 插曲

一开始部署在家里的NAS，无奈家里无法启用80个443端口，虽然其他端口可以正常访问使用，但是多少感觉差点什么，正好腾讯云有台闲置的轻量云，索性部署在轻量云上。轻量云上本来就没有部署任何web服务，全是Docker应用，所以正好按照大佬的教程用上了。

## 部署

Matrix 服务端  [Synapse](https://matrix.org/docs/projects/server/synapse)  代理服务端  [Caddy2](https://caddyserver.com/v2)  通过  [Portainer](https://www.portainer.io/)  部署。

以下教程以服务器名  matrix.ccccc.com 部署，通过 [Delegation](https://github.com/matrix-org/synapse/blob/develop/docs/delegate.md) 机制，用户和房间的 ID 都为 *:ccccc.com 的形式。

目录结构：

```目录结构
.
├── caddy
│   ├── Caddyfile
│   ├── config
│   └── data
├── coturn
│   ├── my.conf
│   └── turn.ccccc.com
│       ├── certificate.pem
│       └── private.pem
├── postgres
│   ├── create_db.sh
│   └── data
├── redis
└── synapse
    ├── element
    ├── logs
    ├── matrix-qq
    │   ├── config.yaml
    │   └── registration.yaml
    ├── matrix-wechat
    │   ├── config.yaml
    │   └── registration.yaml
    ├── matrix-wechat-agent
    │   ├── log
    │   ├── matrix-wechat-agent.exe
    │   ├── SWeChatRobot.dll
    │   ├── wxDriver.dll
    │   └── wxDriver64.dll
    ├── media_store
    ├── uploads
    ├── homeserver.yaml
    ├── xxxxx.com.log.config
    ├── xxxxx.com.signing.key
    ├── qq-registration.yaml
    ├── shared_secret_authenticator.py    
    └── wechat-registration.yaml
```

### Postgres

建立 Postgres 数据库相应的目录

```bash
mkdir -p postgres/data
touch postgres/create_db.sh
```

然后修改数据库初始sh文件

```create_db.sh
#!/bin/sh

createdb -U synapse_user -O synapse_user --encoding=UTF8 --locale=C --template=template0 synapse
createdb -U synapse_user -O synapse_user matrix_qq
createdb -U synapse_user -O synapse_user matrix_wechat
```

在 **portainer** 的 **Stacks** 里新建一个 **Stacks** ：

```yaml
version: '3.7'
services:

# Postgres 数据库
  postgres:
    hostname: postgres
    container_name: postgres
    image: postgres:14
    restart: unless-stopped
    environment:
      TZ: Asia/Shanghai
      POSTGRES_USER: synapse_user
      POSTGRES_PASSWORD: 123456789
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U synapse_user" ]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - /postgres/create_db.sh:/docker-entrypoint-initdb.d/20-create_db.sh
      - /postgres/data:/var/lib/postgresql/data
    ports:
      - 5555:5432
    networks:
      - matrix-net

networks:
  matrix-net:
    attachable: true
```

{{< admonition >}}

修改`environment:`  ➔  `POSTGRES_PASSWORD:`  用户密码。
{{< /admonition >}}

点击 **Deploy the stacl** 部署运行看看。也可以进入到sh里面手动输入初始文件内的命令进行创建数据库。创建好数据库后开始配置 **Synapse**

### Synapse

建立 **Synapse** 目录

```bash
mkdir -p synapse/logs
mkdir synapse/media_store
mkdir synapse/uploads
```

然后通过ssh生成初始化目录

```bash
docker run -it --rm \
    -v /synapse:/data \
    -e SYNAPSE_SERVER_NAME=ccccc.com \
    -e SYNAPSE_REPORT_STATS=no \
    -e SYNAPSE_LOG_LEVEL=WARNING \
    matrixdotorg/synapse:latest generate
```

将对应目录的属主改成991

```sh
sudo chown 991:991 synapse/logs synapse/media_store synapse/uploads
```

然后修改配置文件。

**synapse/example.com.log.config**

```yaml
version: 1

formatters:
  precise:
    format: '%(asctime)s - %(name)s - %(lineno)d - %(levelname)s - %(request)s - %(message)s'

handlers:
  console:
    class: logging.StreamHandler
    formatter: precise
  file:
    class: logging.handlers.TimedRotatingFileHandler
    formatter: precise
    filename: /data/logs/homeserver.log
    when: midnight
    backupCount: 7
    encoding: utf8
  buffer:
    class: synapse.logging.handlers.PeriodicallyFlushingMemoryHandler
    target: file
    capacity: 10
    flushLevel: 30
    period: 5

loggers:
  synapse.storage.SQL:
    level: INFO

root:
  level: INFO
  handlers: [buffer]

disable_existing_loggers: false
```

**synapse/homeserver.yaml**

```yaml
server_name: "ccccc.com"
public_baseurl: "https://matrix.ccccc.com"
serve_server_wellknown: true
pid_file: /data/homeserver.pid
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    resources:
      - names: [client, federation]
        compress: false

#psycopg数据库
#注释掉SQLite数据库
#database:
#  name: sqlite3
#  args:
#    database: /data/homeserver.db

database:
  name: psycopg2
  txn_limit: 10000
  args:
    user: synapse_user
    password: 123456789
    database: synapse
    host: postgres
    port: 5432
    cp_min: 5
    cp_max: 10

#url预览    
url_preview_enabled: true
url_preview_ip_range_blacklist:
  - '127.0.0.0/8'
  - '10.0.0.0/8'
  - '172.16.0.0/12'
  - '192.168.0.0/16'
  - '100.64.0.0/10'
  - '192.0.0.0/24'
  - '169.254.0.0/16'
  - '192.88.99.0/24'
  - '198.18.0.0/15'
  - '192.0.2.0/24'
  - '198.51.100.0/24'
  - '203.0.113.0/24'
  - '224.0.0.0/4'
  - '::1/128'
  - 'fe80::/10'
  - 'fc00::/7'
  - '2001:db8::/32'
  - 'ff00::/8'
  - 'fec0::/10'

url_preview_url_blacklist:
  - scheme: 'http'
  - netloc: '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$'
```

在 **portainer** 的 **Stacks** 里编辑刚创建的 **Stacks** 下添加内容：

```yaml
# Synapse Matrix服务端
  synapse:
    hostname: synapse
    container_name: synapse
    image: matrixdotorg/synapse:latest
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      TZ: Asia/Shanghai
    healthcheck:
      test: [ "CMD", "curl", "-fSs", "http://localhost:8008/health" ]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 5s
    volumes:
      - /synapse:/data
    ports:
      - 8999:8008
      - 8444:8448
      - 8555:20002
    networks:
      - matrix-net
```

{{< admonition >}}
后期可能会更换其他代理，所以这里我表明了端口。按照本教程部署的话可以表明置端口。
{{< /admonition >}}
然后更新 **Stacks** 运行。
如果没没问，现在服务器已经运行了，下面部署 **caddy** 代理。

{{< admonition type=warning >}}
请保管好 ccccc.com.signing.key 这个文件
{{< /admonition >}}

### Caddy

建立对应的目录和文件

```bash
mkdir -p caddy/config
mkdir caddy/data
touch caddy/Caddyfile
```

修改 **Caddyfile** 文件

**caddy/Caddyfile**

```caddy
{
    email admin@ccccc.com
}

ccccc.com {
    header /.well-known/matrix/* Content-Type application/json
    header /.well-known/matrix/* Access-Control-Allow-Origin *

    respond /.well-known/matrix/server `{"m.server": "matrix.ccccc.com:443"}`
    respond /.well-known/matrix/client `{"m.homeserver":{"base_url":"https://matrix.ccccc.com"}}`
}

matrix.ccccc.com {
    log {
        level INFO
        output file /data/access.log {
            roll_size 100MB
            roll_local_time
            roll_keep 10
        }
    }

    reverse_proxy /_matrix/* synapse:8008
    reverse_proxy /_synapse/client/* synapse:8008
}
```

在 **portainer** 的 **Stacks** 里编辑 **Stacks** 新添加内容：

```yaml
# Caddy 代理服务端      
  caddy:
    hostname: caddy
    container_name: caddy
    image: caddy:2
    restart: unless-stopped
    environment:
      TZ: Asia/Shanghai
    volumes:
      - /caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - /caddy/config:/config
      - /caddy/data:/data
    ports:
      - 80:80
      - 443:443
      - 8443:8443
    networks:
      - matrix-net
```

然后更新 **Stacks** 运行。

然后访问 https://matrix.ccccc.com/_matrix/federation/v1/version 看是否有返回。

然后到 [https://federationtester.matrix.org/](https://federationtester.matrix.org/) 输入服务器名 (比如 ccccc.com) 看看 federation 是否正常, 绿色就可以.

一切正常的话，服务已经正常运行了，下面注册个账号。

```bash
docker exec -it synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml
```

通过 [https://app.element.io/](https://app.element.io/) 登录看看, 服务器选择 matrix.ccccc.com 或者 ccccc.com , 用新建的用户登录.一切正常的话下面就开始折腾QQ和微信的桥接了。

### Shared Secret Authenticator

先把 [Shared Secret Authenticator](https://github.com/devture/matrix-synapse-shared-secret-auth) 给启用了, 这样 [double puppeting](https://docs.mau.fi/bridges/general/double-puppeting.html) 就不需要手动绑定 Matrix 帐号

```
wget https://raw.githubusercontent.com/devture/matrix-synapse-shared-secret-auth/master/shared_secret_authenticator.py -O synapse/shared_secret_authenticator.py
sudo chown 991:991 synapse/shared_secret_authenticator.py
```

在 **portainer** 的 **Stacks** 里编辑 **Stacks** ，在 synapse 的 volumes 加入

```yaml
  - /synapse/shared_secret_authenticator.py:/usr/local/lib/python3.9/site-packages/shared_secret_authenticator.py
```

**synapse/homeserver.yaml** 添加以下内容 (这里的 Key 后续在桥接配置的时候要用到)

```yaml
modules:
  - module: shared_secret_authenticator.SharedSecretAuthProvider
    config:
      shared_secret: "设置一个key"
      m_login_password_support_enabled: true
```

然后更新 **Stacks** 运行。

### Matrix-QQ

执行 `mkdir matrix-qq` 创建目录

运行以下命令生成初始的 **matrix-qq/config.yaml** 配置文件

```sh
docker run --rm -v /synapse/matrix-qq:/data:z lxduo/matrix-qq:latest
```

然后这个文件主要修改的内容是

```yaml
homeserver:
    address: http://synapse:8008
    domain: ccccc.com
appservice:
    address: http://matrix-qq:17777

    database:
        uri: postgres://synapse_user:123456789@postgres/matrix_qq?sslmode=disable
bridge:
    double_puppet_server_map:
        ccccc.com: https://matrix.ccccc.com

    login_shared_secret_map:
        ccccc.com: "你设置的key"

    permissions:
        "ccccc.com": user
        "@admin:ccccc.com": admin
```

再运行一次生成注册文件

```shell
docker run --rm -v /synapse/matrix-qq:/data:z lxduo/matrix-qq:latest
```

把注册文件拷到 synpase 目录

```sh
cp matrix-qq/registration.yaml synapse/qq-registration.yaml
sudo chown 991:991 synapse/qq-registration.yaml
```

编辑 **synapse/homeserver.yaml**，加入

```yaml
app_service_config_files:
  - /data/qq-registration.yaml
```

在 **portainer** 的 **Stacks** 里编辑 **Stacks** 新添加内容：

```yaml
# Matrix-Qq QQ桥
  matrix-qq:
    hostname: matrix-qq
    container_name: matrix-qq
    image: lxduo/matrix-qq:latest
    restart: unless-stopped
    depends_on:
      synapse:
        condition: service_healthy
    volumes:
      - /synapse/matrix-qq:/data
    networks:
      - matrix-net
```

然后更新 **Stacks** 运行。

新建和 @qqbot:ccccc.com 的聊天, 输入 help 查看使用帮助
{{< admonition type=warning >}}
当密码登录不工作的时候, 可以考虑扫码登录, 或者将 [go-cqhttp](https://github.com/Mrs4s/go-cqhttp) 或者 [MiraiGo-Template](https://github.com/Logiase/MiraiGo-Template) 生成的 device.json 和 session.token 各做个 base64 然后用 token 方式登录...

参见: https://github.com/Mrs4s/go-cqhttp/issues/1469
{{< /admonition >}}

### Matrix-Wechat

执行 `mkdir matrix-wechat` 创建目录

运行以下命令生成初始的 **matrix-wechat/config.yaml**

```sh
docker run --rm -v /synapse/matrix-wechat:/data:z lxduo/matrix-wechat:latest
```

然后这个文件主要修改的内容是

```yaml
homeserver:
    address: http://synapse:8008
    domain: ccccc.com
appservice:
    address: http://matrix-wechat:17778

    database:
        uri: postgres://synapse_user:123456789@postgres/matrix_wechat?sslmode=disable
bridge:
    listen_secret: "你设置的key"

    double_puppet_server_map:
        ccccc.com: https://matrix.ccccc.com

    login_shared_secret_map:
        ccccc.com: "你设置的key"

    permissions:
        "ccccc.com": user
        "@admin:ccccc.com": admin
```

再运行一次生成注册文件

```shell
docker run --rm -v /synapse/matrix-wechat:/data:z lxduo/matrix-wechat:latest
```

查看生成的registration文件，看看是否需要编辑 **matrix-wechat/registration.yaml** 里面的正则

```yaml
namespaces:
    users:
        - regex: ^@wechatbot:ccccc\.com$
          exclusive: true
        - regex: ^@_wechat_.+:ccccc\.com$
          exclusive: true
```

把注册文件拷到 **synpase** 目录, 同时修改属主

```sh
cp matrix-wechat/registration.yaml synapse/wechat-registration.yaml
sudo chown 991:991 synapse/wechat-registration.yaml
```

编辑 **synapse/homeserver.yaml**, 在 **app_service_config_files** 里加入

```yaml
 - /data/wechat-registration.yaml
```

在 **portainer** 的 **Stacks** 里编辑 **Stacks** 新添加内容：

```yaml
# Matrix-Wechat 微信桥
  matrix-wechat:
    hostname: matrix-wechat
    container_name: matrix-wechat
    image: lxduo/matrix-wechat:latest
    restart: unless-stopped
    depends_on:
      synapse:
        condition: service_healthy
    volumes:
      - /synapse/matrix-wechat:/data
    networks:
      - matrix-net
```

编辑 **caddy/Caddyfile**, 加入

```caddy
    reverse_proxy /_wechat/* matrix-wechat:20002
```

更新 **Stacks** 运行。

### Matrix-Wechat-Agent

可以通过 Docoker 部署, 不过毕竟是 Wine 来运行的, 稳定性一般, 也可以用 Windows 的主机来运行

#### Linux 主机

执行 `mkdir matrix-wechat-agent` 创建目录

在 **portainer** 的 **Stacks** 里编辑 **Stacks** 新添加内容：

```yaml
# Matrix-Wechat-Agent 微信代理
  matrix-wechat-agent:
    hostname: matrix-wechat-agent
    container_name: matrix-wechat-agent
    image: lxduo/matrix-wechat-docker:latest
    restart: unless-stopped
    #security_opt:
    #  - seccomp:unconfined
    depends_on:
      - matrix-wechat
    environment:
      TZ: Asia/Shanghai
      WECHAT_HOST: ws://matrix-wechat:20002
      WECHAT_SECRET: 你设置的key
    #shm_size: 1gb
    #devices:
    #  - /dev/dri:/dev/dri
    volumes:
      - /synapse/matrix-wechat-agent:/home/user/matrix-wechat-agent
    #ports:
    #  - 15905:5905
    networks:
      - matrix-net
```

{{< admonition type=tip >}}
如果遇到 Docker Engine 的兼容性问题, 打开下面的配置。

```yaml
    security_opt:
      - seccomp:unconfined
```

如果微信运行的不稳定, 把注释的 shm_size 打开看看

如果需要更新 Agent, 将目录里的 exe 或 dll 删除后重启该 docker 即可
{{< /admonition >}}

#### Windows 主机

从 [duo/matrix-wechat-agent](https://github.com/duo/matrix-wechat-agent/releases) 下载得到 matrix-wechat-agent.exe

从 [ljc545w/ComWeChatRobot](https://github.com/ljc545w/ComWeChatRobot/releases) 下载解压得到 SWeChatRobot.dll 和 wxDriver.dll

将 matrix-wechat-agent.exe 和 SWeChatRobot.dll 以及 wxDriver.dll 置于同一目录

安装 Visual C++ Redistributable ([Latest supported Visual C++ Redistributable downloads | Microsoft Learn](https://docs.microsoft.com/en-US/cpp/windows/latest-supported-vc-redist?view=msvc-170))

最后命令行执行

```
matrix-wechat-agent.exe -h wss://matrix.ccccc.com/_wechat/ -s 你设置的key
```

提示 Appservice websocket connected 就 ok 了

新建和 @wechatbot:ccccc.com 的聊天，输入 help 查看使用帮助

### 电报

TG我只需求查看群内信息，所以选择了 [T2bot.io](https://t2bot.io/telegram/) 

打开官方按照教程几步就能进行通信了。

### 其他 （可选）

#### Element & Synapse-admin & Redis & Turn

看个人需求，因为 matrix.org 被墙，官方备用Turn服务器 （turn.matrix.org）无法链接，如果无法正常使用语音和视频，最好部署 Turn 服务器来穿透。

#### Coturn

创建目录和配置文件

```shell
mkdir -p coturn/turn.ccccc.com
touch coturn/my.conf
```

{{< admonition type=tip >}}

解析到服务器一个域名，比如：turn.ccccc.com

申请一个证书，配置文件里指定下证书位置。
{{< /admonition >}}

编辑配置文件 **my.conf**

```conf
log-binding
log-file=/etc/turn.log
no-multicast-peers
no-rfc5780
no-sslv3
no-stun-backward-compatibility
no-tcp-relay
no-tlsv1
no-tlsv1_1
no-tlsv1_2
cert=/coturn/turn.ccccc.com/certificate.pem
pkey=/coturn/turn.ccccc.com/private.pem
realm=turn.ccccc.com
response-origin-only-with-rfc5780
server-name=turn.ccccc.com
static-auth-secret={设置一个key}
total-quota=1200
use-auth-secret
verbose
```

编辑 **homeserver.yaml** 文件加入

```yaml
turn_uris: [ "turns:turn.ccccc.com?transport=udp", "turns:turn.ccccc.com?transport=tcp", "turn:turn.ccccc.com?transport=udp", "turn:turn.ccccc.com?transport=tcp" ]
turn_shared_secret: "{你设置的key}"
turn_user_lifetime: 86400000
turn_allow_guests: true
```

在 **portainer** 的 **Stacks** 里编辑 **Stacks** 新添加内容：

```yaml
# Coturn 穿透服务
  coturn:
    hostname: coturn
    container_name: coturn
    restart: unless-stopped
    image: coturn/coturn:latest
    volumes:
      - /coturn/my.conf:/etc/coturn/turnserver.conf
    network_mode: host
```

更新 **Stacks** 运行。

如果运行在云服务器上，记得打开相应的端口。
`3478` 和 `5479` =  `TCP&UDP`
`49152-65535` = `UDP`
官方配置文档 https://matrix-org.github.io/synapse/v1.41/turn-howto.html

#### Element & Synapse-admin & Redis

新建目录和配置文件

```shell
mkdir synapse/element
touch synapse/element/config.json
```

然后编辑 config.json 填入以下内容

```json
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://ccccc.com",
            "server_name": "ccccc"
        },
        "m.identity_server": {
            "base_url": "https://vector.im"
        }
    },
    "disable_custom_urls": false,
    "disable_guests": false,
    "disable_login_language_selector": false,
    "disable_3pid_login": false,
    "brand": "Element",
    "integrations_ui_url": "https://scalar.vector.im/",
    "integrations_rest_url": "https://scalar.vector.im/api",
    "integrations_widgets_urls": [
        "https://scalar.vector.im/_matrix/integrations/v1",
        "https://scalar.vector.im/api",
        "https://scalar-staging.vector.im/_matrix/integrations/v1",
        "https://scalar-staging.vector.im/api",
        "https://scalar-staging.riot.im/scalar/api"
    ],
    "bug_report_endpoint_url": "https://element.io/bugreports/submit",
    "uisi_autorageshake_app": "element-auto-uisi",
    "default_country_code": "GB",
    "show_labs_settings": false,
    "features": {},
    "default_federate": true,
    "default_theme": "light",
    "room_directory": {
        "servers": ["matrix.org"]
    },
    "enable_presence_by_hs_url": {
        "https://matrix.org": false,
        "https://matrix-client.matrix.org": false
    },
    "setting_defaults": {
        "breadcrumbs": true
    },
    "jitsi": {
        "preferred_domain": "meet.element.io"
    },
    "element_call": {
        "url": "https://call.element.io",
        "participant_limit": 8,
        "brand": "Element Call"
    },
    "features": {
        "feature_you_want_to_turn_on": true,
        "feature_you_want_to_keep_off": true
    },
    "map_style_url": "https://api.maptiler.com/maps/streets/style.json?key=fU3vlMsMn4Jb6dnEIFsx"
}
```

编辑 **homeserver.yaml** 文件加入 **redis**

```yaml
  redis:
  enabled: true
  host: redis
  port: 6379
```

在 **portainer** 的 **Stacks** 里编辑 **Stacks** 新添加内容：

```yaml
# Redis 缓存
  redis:
    hostname: redis
    container_name: redis
    image: redis:6.0-alpine
    restart: unless-stopped    
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
    volumes:
      - /redis:/data
    networks:
      - matrix-net

# Synapse-Admin 管理
  synapse-admin:
    hostname: synapse-admin
    container_name: synapse-admin
    restart: unless-stopped
    build:
      context: https://github.com/Awesome-Technologies/synapse-admin.git
      # args:
      #   - NODE_OPTIONS="--max_old_space_size=1024"
    ports:
      - 8333:80
    networks:
      - matrix-net

# Element Web客户端        
  element:
    hostname: element
    container_name: element
    image: vectorim/element-web:latest
    restart: unless-stopped
    volumes:
      - /synapse/element/config.json:/app/config.json
    ports:
      -  8222:80
    networks:
      - matrix-net
```

更新 **Stacks** 运行。

{{< admonition type=tip >}}
**synapse-admin** 如果登录后无法管理，可以在 **Caddy** 内新加一个解析用来管理。

```caddy
synapse-admin.ccccc.com {
    reverse_proxy synapse-admin:80
}
```

{{< /admonition >}}

此刻基本部署完毕。再次感谢大佬的贡献。

### 我的完整Docker配置

```yaml
version: '3.7'
services:
# Postgres 数据库
  postgres:
    hostname: postgres
    container_name: postgres
    image: postgres:14
    restart: unless-stopped
    environment:
      TZ: Asia/Shanghai
      POSTGRES_USER: synapse_user
      POSTGRES_PASSWORD: 123456789
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U synapse_user" ]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - /postgres/create_db.sh:/docker-entrypoint-initdb.d/20-create_db.sh
      - /postgres/data:/var/lib/postgresql/data
    ports:
      - 5555:5432
    networks:
      - matrix-net

# Synapse Matrix服务
  synapse:
    hostname: synapse
    container_name: synapse
    image: matrixdotorg/synapse:latest
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      TZ: Asia/Shanghai
    healthcheck:
      test: [ "CMD", "curl", "-fSs", "http://localhost:8008/health" ]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 5s
    volumes:
      - /synapse:/data
      - /synapse/shared_secret_authenticator.py:/usr/local/lib/python3.9/site-packages/shared_secret_authenticator.py
    ports:
      - 8999:8008
      - 8444:8448
      - 8555:20002
    networks:
      - matrix-net

# Caddy 代理服务     
  caddy:
    hostname: caddy
    container_name: caddy
    image: caddy:2
    restart: unless-stopped
    environment:
      TZ: Asia/Shanghai
    volumes:
      - /caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - /caddy/config:/config
      - /caddy/data:/data
    ports:
      - 80:80
      - 443:443
      - 8443:8443
    networks:
      - matrix-net

# Matrix-Qq QQ桥
  matrix-qq:
    hostname: matrix-qq
    container_name: matrix-qq
    image: lxduo/matrix-qq:latest
    restart: unless-stopped
    depends_on:
      synapse:
        condition: service_healthy
    volumes:
      - /synapse/matrix-qq:/data
    networks:
      - matrix-net

# Matrix-Wechat 微信桥
  matrix-wechat:
    hostname: matrix-wechat
    container_name: matrix-wechat
    image: lxduo/matrix-wechat:latest
    restart: unless-stopped
    depends_on:
      synapse:
        condition: service_healthy
    volumes:
      - /synapse/matrix-wechat:/data
    networks:
      - matrix-net

# Matrix-Wechat-Agent 微信代理
  matrix-wechat-agent:
    hostname: matrix-wechat-agent
    container_name: matrix-wechat-agent
    image: lxduo/matrix-wechat-docker:latest
    restart: unless-stopped
    #security_opt:
    #  - seccomp:unconfined
    depends_on:
      - matrix-wechat
    environment:
      TZ: Asia/Shanghai
      WECHAT_HOST: ws://matrix-wechat:20002
      WECHAT_SECRET: 你设置的key
    shm_size: 1gb
    #devices:
    #  - /dev/dri:/dev/dri
    volumes:
      - /synapse/matrix-wechat-agent:/home/user/matrix-wechat-agent
    #ports:
    #  - 15905:5905
    networks:
      - matrix-net

# Redis 缓存
  redis:
    hostname: redis
    container_name: redis
    image: redis:6.0-alpine
    restart: unless-stopped    
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
    volumes:
      - /redis:/data
    networks:
      - matrix-net

# Synapse-Admin 管理
  synapse-admin:
    hostname: synapse-admin
    container_name: synapse-admin
    restart: unless-stopped
    build:
      context: https://github.com/Awesome-Technologies/synapse-admin.git
      # args:
      #   - NODE_OPTIONS="--max_old_space_size=1024"
    ports:
      - 8333:80
    networks:
      - matrix-net

# Element Web客户端        
  element:
    hostname: element
    container_name: element
    image: vectorim/element-web:latest
    restart: unless-stopped
    volumes:
      - /synapse/element/config.json:/app/config.json
    ports:
      -  8222:80
    networks:
      - matrix-net

# Coturn 穿透服务
  coturn:
    hostname: coturn
    container_name: coturn
    restart: unless-stopped
    image: coturn/coturn:latest
    volumes:
      - /coturn/my.conf:/etc/coturn/turnserver.conf
    network_mode: host

networks:
  matrix-net:
    attachable: true
```
