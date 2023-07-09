---
title: Traefik 的基础使用
date: 2023-07-09
tags:
---
前阵子迁移服务器，想着 Docker 容器挺多的，就试着把反代从 Nginx 换成了 Traefik。虽然主要只用上了比较基础的部分，但感觉挺不错的。Traefik 的一大特点就是对于一堆 Docker 容器，容器配置在自己 label 那儿（简单点一两个 label 就够）。容器启动 Traefik 就能自动发现并应用配置，如果配置过自动的证书申请证书也一起申了，不用启动了服务再单独找个地方改配置、申证书

不过如果没接触过反代也许从 Nginx 之类入手更好，资料更多且不少文档给出的反代例子都是 Nginx 的，或者希望方便点的话可以看看 Nginx Proxy Manager / Caddy 那些。Traefik 可能更难理解一点，适合常用 Docker 的用户

以及…
- 本文纯乱折腾，可能有错误（如果您发现错误请联系我）
- 主要涉及（个人）比较常用的部分，毕竟不熟的同样翻文档
- 因为 Traefik 文档只有英文，为了避免混淆，本文中关键概念尽量用与文档相同的英文

# 配置
>  https://doc.traefik.io/traefik/routing/overview/

Traefik 的配置分为两类，一种静态一种动态。静态部分主要是一些启动后就固定的部分，比如 Traefik 自己的配置、Entrypoints 和一些证书相关的配置；动态部分则主要和每个服务有关，例如路由规则、用哪个端口之类。静态可以是文件或者命令行参数，动态常用的可能是 Docker 的 label 或者文件（Traefik 可以观察文件变化）

实际使用静态参数既可以当参数写进 Docker 的启动命令也可以单独提供文件；动态参数如果对象是 Docker 容器的话一般用 label，本地服务的话应该只能用文件了。

有个可能会混淆的点是有些部分名称是自定义的，但省事的话可能会和预定义的名称混起来，文中尽量用一些不太可能是关键字的来代替，例如 `my-router` 之类。以及（特别是 label 形式定义的）组件常常看起来缺少一个“定义”的过程，例如 `traefik.http.routers.www-router.rule=Host(xxx)`  中的 `www-router` 不需要提前定义，直接就能指定它的 Host。

## Static
命令行参数或文件（YAML / TOML）或环境变量
### Entrypoints
> https://doc.traefik.io/traefik/routing/entrypoints/

开 80 443 且 80 转 443 的例子（`web` 和 `websecure` 都是自定义的名字，不过在这边可能有点约定俗成）
```yaml
entryPoints:
    web:
        address: ":80"  # [host]:port[/tcp|/udp]
        http:
            redirections:
                entryPoint:
                    to: websecure
                    scheme: https
    websecure:
        address: ":443"
```

### Providers
主要看看 Docker provider： https://doc.traefik.io/traefik/routing/providers/docker/ （其他也都不熟 x）
#### 启用
文件：
```yaml
providers:
    docker: {}
```
参数里：`--providers.docker`

#### 使用
文中例子以文件形式的配置为主（更好看清结构），对于二者转换可以参考下面例子。大体上把 YAML 的结构改成用点连接即可，具体请参考文档
例如：
```yaml
# Docker Provider
version: "3"
    services:
        my-container:
            # ...
            labels:
                - traefik.http.routers.www-router.rule=Host(`example-a.com`)
                - traefik.http.routers.www-router.service=www-service
                - traefik.http.services.www-service.loadbalancer.server.port=8000
                - traefik.http.routers.admin-router.rule=Host(`example-b.com`)
                - traefik.http.routers.admin-router.service=admin-service
                - traefik.http.services.admin-service.loadbalancer.server.port=9000
```
```yaml
# File Provider (dynamic config)
http:
    routers:
        www-router:
            rule: "Host(`example-a.com`)"
            service: www-service
        admin-router:
            rule: "Host(`example-b.com`)"
            service: admin-service
    services:
        www-service:
            loadBalancer:
                servers:
                    - url: "http://127.0.0.1:8000" # For example
        admin-service:
            loadBalancer:
                server:
                    - url: "http://127.0.0.1:9000" # For example
```

注意这种情况（HTTP）下如果标签里只有 router，会自动生成一个 service。同时如果既有一个 router 也有一个 service 但 router 没有指定 service，则 service 会自动分配给那个 router

于是最简单的 labels 可以这样写（如果容器有开单端口甚至可以省略 port， Traefik 会自动用那个端口）
```yaml
labels:
    - "traefik.http.routers.myproxy.rule=Host(`example.net`)"
    - "traefik.http.services.myservice.loadbalancer.server.port=80"
```

## Dynamic
### Routers
> https://doc.traefik.io/traefik/routing/routers/

主要 http routers
Routers 默认监听所有 Entrypoints，一般主要关注规则部分

router 需要的 rule（什么情况下用这部分配置）、middleware（中间需要什么处理）、service（给谁）、其他要求（如 TLS）
例子：
```yaml
http:
    routers:
        Router-1:
            rule: "Host(`example.com`)" # 反引号或\"来框字符串（好像是 Go 的要求）
            middlewares: # 不一定需要
                - authentication
            service: "service-1" # 用 label 可以省略
            priority: 1 # 不一定需要
            tls:
                certResolver: leresolver # 见 tls
```

#### Rule
具体请看官方文档，这里给个简单例子
```yaml
rule = "Host(`example.com`) || (Host(`example.org`) && PathPrefix(`/traefik/`))"
```
（PathPrefix 匹配路径前缀）

#### Service
每个 router 都必须有一个 service。一般来说需要预先定义 service，但例如用 Docker 的 label 就可以省略（Traefik 会自动创建并分配）


### Services
每个 service 必有一个 loadbalancer（即使不需要负载均衡）

一般用到主要是主动指定容器内端口或者监听本地的 url。
主动指定端口：`traefik.http.services.my-service.loadbalancer.server.port=9000`（一行就够不依赖其他，或者说其他都自动生成了）

文件形式的例子：
```yaml
http:
    routers:
        my-router:
            rule: "Host(`example.com`)"
            service: my-service
    services:
        my-service:
            loadBalancer:
                servers:
                    - url: "http://host.docker.internal:8080"
```

### Middlewares
细节请看文档 https://doc.traefik.io/traefik/middlewares/overview 。能干挺多事，但先只说个 BasicAuth

#### BasicAuth
为一些没有认证功能的服务加个认证功能，例如 Traefik 的 Dashboard（见后文）
例子：
```yaml
labels:
  - "traefik.http.middlewares.test-auth.basicauth.users=username:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/,test2:$$apr1$$d9hr9HBB$$4HxwgUir3HP4EsggP/QNo0"
  - "traefik.http.routers.my-router.middlewares=test-auth"
```
格式是“用户名:处理过的密码”，处理过密码部分需要是实际密码 MD5 / SHA1 / BCrypt 后的结果，如果有 `$` 的话要换成 `$$`
密码生成可以用 htpasswd，或者熟悉 Python（Python 3）的话：
```bash
pip install bcrypt
python -c "import bcrypt;print(bcrypt.hashpw('YOUR_PASSWORD'.encode('UTF-8'),bcrypt.gensalt()).decode('UTF-8').replace('$','\$\$'))"
```
如果用 Windows PowerShell 需要把最后的 `\$\$` 中的 \\ 换成 \` 

## TLS
> https://doc.traefik.io/traefik/https/overview/

理论上 TLS 相关设置既有静态也有动态，但感觉还是放在一起看比较合适

TLS 证书可以手动指定，也可以让 Traefik 自动申请（是 Let's Encrypt 的）。Traefik 可以全自动完成证书申请和续期（主要看看这个，手动指定请参考文档）

首先要在静态配置中定义 certificatesResolvers，之后每个 router 通过 tls.certresolver 选一个用。Traefik 会自动通过对应 rule 找域名申请对应证书（也可以通过 tls.domains 指定）

例子（结合起来看）：
```yaml
#Static configuration 
certificatesResolvers:
    my-resolver:
        acme:
            email: "name@example.com" # Let's Encrypt 注册邮箱
            storage: "acme.json" # 容器外映射进来（xxx/acme.json:/acme.json），权限 600
            httpChallenge: # 不同申请方法见下文
                entryPoint: web
```
```yaml
# Dynamic configuration
labels:
- traefik.http.routers.my-router.rule=Host(`example.com`) && Path(`/example`)
# - traefik.http.routers.blog.tls=true
- traefik.http.routers.my-router.tls.certresolver=my-resolver
```

也可以针对 Entrypoints 配置默认的 certResolver
```yaml
entryPoints:
    websecure:
    address: ':443' 
    http: 
        tls: 
            certResolver: leresolver
```

###  ACME Challenges
证书申请需要 Challenges 验证域名真正指向了服务器，有几种可选的验证方式
#### tlsChallenge
443 开就行。但注意[套 Cloudflare 时这种方式无效](https://community.letsencrypt.org/t/cannot-negotiate-alpn-protocol/111996/2)（看 CDN 是否允许 non-HTTP ALPNs），可以暂时关掉 CDN 只保留 DNS 解析（并不推荐）或者换其他方法（套 Cloudflare 了可以考虑下文的 DNS challenge）
```yaml
certificatesResolvers:
    myresolver:
        acme:
            # ...
            tlsChallenge: {}
```

#### httpChallenge
需要开 80 口
```yaml
certificatesResolvers:
    myresolver:
        acme:
            # ...
            httpChallenge:
                entrypoint: web # 对应 80 口的那个
```

#### dnsChallenge
参考 https://doc.traefik.io/traefik/https/acme/#dnschallenge
只有这种方式能申请通配符证书，具体配置看所使用的 DNS 提供商

Cloudflare 的话建议去 My Profile → API Tokens → Create Token 创建一个低权限的 Token，根据[这个 issue](https://github.com/traefik/traefik/issues/5965)，需要的权限是“Zone - Zone - Read”和“Zon - DNS - Edit”，至于具体 Zone 的选择就看实际情况了
之后在 Docker 的环境变量中增加“CF_DNS_API_TOKEN=XXXXXXX”即可


# Extra

## Network 相关
Docker compose 默认会创建单独 network，如果一起用默认的 bridge 的话要加上 `network_mode: bridge`。或者其他方式但总之 Traefik 得和目标容器有在相同 network 中。

## Docker 内的 Traefik 访问 Docker 外的本地服务
虽然以 Docker 为主，但总得有这个能力 x。这里的方法不止适用于 Traefik，其他情况下容器内访问容器外本地也可以这么做
需要的设置：
- docker run 添加参数：`--add-host=host.docker.internal:host-gateway`（docker 版本 20+）
- docker compose 在对应 container 下面增加
    ```docker-compose
    extra_hosts:
        - "host.docker.internal:host-gateway"
    ```
例如 `localhost:1234` 的服务，设置后 Docker 内访问则用 `host.docker.internal:1234` 即可

参考： https://stackoverflow.com/questions/24319662

## Dashboard
https://doc.traefik.io/traefik/operations/dashboard/
不一定有必要，但 ~~看着好看~~ 有时 debug 相对方便
给 Traefik 自己上 label 就行
```yaml
# 改自官方的例子
# Dynamic Configuration
labels:
- "traefik.http.routers.dashboard.rule=Host(`traefik.example.com`)"
- "traefik.http.routers.dashboard.service=api@internal"
- "traefik.http.routers.dashboard.middlewares=auth"
- "traefik.http.middlewares.auth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/,test2:$$apr1$$d9hr9HBB$$4HxwgUir3HP4EsggP/QNo0"
```
（发现上文没写 middleware，可能有空加上）
其中 rule 部分要包含 `/api` 和 `/dashboard` （这里直接整个域名肯定包含了），service 是固定的 `api@internal`。虽然 dashboard 只能看看但最好还是上个验证，例如 `basicauth` `

之后访问 `https://traefik.example.com/dashboard/` 即可，注意默认情况下最后的那个 `/` 不可省略

## 关于端口
不少 web 服务都是直接 `-p xxxx:xxxx`，个人感觉不太建议，毕竟除非容器内服务配置了只监听部分 IP 或 IP 段，这样一般会监听所有 ip，可能会被扫出来。测试的话可以 `-p 127.0.0.1:xxxx:xxxx` 加 SSH 的本地端口转发或者其他方法，如果比较简单感觉 Traefik 直接用指定端口就行


# 参考
参考过的文章，也许它们比本文更合适
- [Traefik Proxy 2.x and Docker 101]( https://traefik.io/blog/traefik-2-0-docker-101-fc2893944b9d/ )
- [Understanding Traefik]( https://kwojcicki.github.io/blog/TRAEFIK )
- [Deploying Portainer behind Traefik Proxy](https://docs.portainer.io/advanced/reverse-proxy/traefik)
- https://doc.traefik.io/traefik/user-guides/docker-compose/basic-example/
