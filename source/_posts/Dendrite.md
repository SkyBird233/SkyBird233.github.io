---
title: 部署 Matrix 服务器（Dendrite）
date: 2023-01-26 16:53:55
tags: matrix
---
> [Dendrite](https://matrix.org/docs/projects/server/dendrite) is a second-generation [Matrix](https://matrix.org) homeserver written in Go. It intends to provide an efficient, reliable and scalable alternative to [Synapse](https://matrix.org/docs/projects/server/synapse)

挺早了解了 Matrix，但听说 Synapse 占用较高，一般服务器不太跑得动，于是对 Matrix 敬而远之 x。最近与另一位先生交流了才知道 Dendrite 占用已经可以非常低了，于是也试着部署了一个。

我的部署方式是 Docker compose，Monolith mode + PostgrSQL，关于 mode 区别和数据库选择请参考[官方说明](https://matrix-org.github.io/dendrite/installation/planning) 。以及官方的配置要求看着挺高，但小规模使用的话没啥参考价值 x，自己的 Dendrite 占用才 60MB+（PostgreSQL 也大约 60MB+）。

# 配置
[主配置文件 dendrite-sample.monolith](https://github.com/matrix-org/dendrite/blob/main/dendrite-sample.monolith.yaml) 
[Docker 相关文件](https://github.com/matrix-org/dendrite/tree/main/build/docker)

文件结构（需要设置的部分，并非最终结果）
```
dendrite
├── config
│   ├── dendrite.yaml
│   ├── matrix_key.pem
│   ├── server.crt
│   └── server.key
├── docker-compose.yml
└── postgres
    └── create_db.sh
```
下文中主域名就用 `example.com` 代替了。

## dendrite.yaml
- `global:servername`
	- `example.com`
- `global:database:connection_string`
	- `username`、`password` 和 docker-compose.yaml 里的保持相同
	- `hostname` 换成 `postgres` （Docker 下不额外配置的话主机名好像是跟着容器名…？）
- `global:well-known-*-name`
	- `example.com`
- `media_api:basic_path`
	- 好像默认的和 docker-compose.yml 中的不同，我把这里的改成了 `/var/dendrite/media`（对应 docker-compose.yml 第 35 行）

## docker-compose.yml
- `services:postgres:volums`
	- 参考注释修改第二条，把 `./path_to/postgresql` 改改（~~其实不改也不是不行~~）
- `services:postgres:environment`
	- 用户名密码，和 dendrite.yaml 中的保持相同

## matrix_key.pem
参考官方给的命令，在 config 目录下执行：
```bash
docker run --rm --entrypoint="" \
  -v $(pwd):/mnt \
  matrixdotorg/dendrite-monolith:latest \
  /usr/bin/generate-keys \
  -private-key /mnt/matrix_key.pem \
```
这里只需要生成 `matrix_key.pem`。以及可以随时执行，Docker 会自动拉取镜像。

## server. crt & server.key
我使用 [acme.sh](https://acme.sh) 来自动配置：
1. 默认使用 Let's Encrypt（可选）：`acme.sh --set-default-ca --server letsencrypt`
2. 申请证书：`acme.sh --issue -d matrix.example.com` （根据实际情况不同，可能需要临时开个 80 口，具体请看 acme.sh 的说明并选择申请方式）
3. 安装证书：`acme.sh --install-cert -d matrix.example.com --key-file path_to_dendrite/config/server.key --fullchain-file path_to_dendrite/config/server.crt --reloadcmd "docker restart dendrite-monolith-1"` （根据实际替换 `path_to_dendrite` 和 `dendrite-monolith-1`）
之后续期啥的留给 acme.sh 就行了。

## postgres/create_db.sh
https://github.com/matrix-org/dendrite/blob/main/build/docker/postgres/create_db.sh
（不需要修改）

## Delegation
> https://matrix-org.github.io/dendrite/installation/domainname#delegation

简单说大致就是使用 `matrix.example.com` 定位实际服务器、与服务器交流，但用户名等部分仍然使用 `example.com`。比较建议这种方式，既保持了用户名的简洁也方便服务器迁移啥的。
具体设置部分需要注意的就是（Well-known delegation）
1. 在 `dendrite.yaml` 的 `server_name` 部分使用 `example.com`
2. 证书（`server.crt` 和 `server.key`）用 `matrix.example.com`
3. 访问 `example.com/.well-known/matrix/server` 和 `example.com/.well-known/matrix/client` 时返回特定 json 指向 `matrix.example.com`
> 第 3 步可以通过 SRV 记录实现（DNS SRV delegation），不过官方说不太推荐。似乎也只能用于服务器见的发现。

### Cloudflare Workers 实现 Well-known delegation
要实现 Delegation 实际上额外需求只有在客户端请求 `example.com/.well-known/matrix/*` 时返回简单 json。官方文档中是使用 Caddy/Nginx，但不考虑反代的话单独为了这个架个 Web 服务器感觉没啥必要 x，于是就丢给 Cf Workers 了。

#### Workers 参考代码：
> 参考~~（主要抄自）~~： https://helderferreira.io/matrix-well-known-with-cloudflare/ （有所修改）
```js
addEventListener("fetch", (event) => {
  event.respondWith(
    handleRequest(event.request).catch(
      (err) => new Response(err.stack, { status: 500 })
    )
  );
});


export async function handleRequest(request){
  
  const url = new URL(request.url)

  const headers = {
    headers: {
      "content-type": "application/json;charset=UTF-8",
      'Access-Control-Allow-Origin': '*',  //解决可能会遇到跨域问题
    }
  }

  const serverJson = {
    "m.server": "matrix.example.com:8448" 
  }

  const clientJson = {
      "m.homeserver": {
          "base_url": "https://matrix.example.com:8448"
      },
  }

  var msg

  if (url.pathname.endsWith("server")) {
    msg  = JSON.stringify(serverJson)
  }

  if (url.pathname.endsWith("client")) {
    msg = JSON.stringify(clientJson)
  }

  if (msg) {
    return new Response( msg , headers);
  } else {
    return new Response('Not Found.', { status: 404 })
  }
}
```

#### Route 设置
Route：`example.com/.well-known/matrix/*`

# 测试
可以使用 [Federation Tester](https://federationtester.matrix.org/) 测试 federation 情况。客户端的话直接用客户端测试即可（记得开注册）。以及 Element Desktop 调试快捷键是 `Ctrl + Shift + I`。

# 帐号管理
自己由于不开放注册，也没几个号，于是临时开放注册然后手动关闭 x。具体请参考[官方说明](https://matrix-org.github.io/dendrite/administration)。

# 与 XMPP 对比
其实都没有很深入的了解，XMPP 服务端（[Prosody](https://prosody.im)）的部署时间也比较早了，应该有所遗忘。这里只是简单比较一下个人的感受。
- 服务端
	Prosody 和 Dendrite 对比的话，感觉 Dendrite 部署难度上要低于 Prosody，不过占用略高于 Prosody（假如 PostgreSQL 也算上的话内存占用至少是 Prosody 的两倍了）。我的 Prosody 部署的比较早，当时连实现上传文件这一功能都需要专门加个模组，不过较新的 0.12 好像内置了。
- 客户端
	Matrix 基础功能感觉比 XMPP 完善多了，客户端完成度也高于 XMPP。虽然 Element 客户端基于 Electron，但至少能保证多平台都有相似的用户体验，也有很多其他客户端可以选择。XMPP 客户端体验感觉参差不齐，也还要看对 XEP 的支持程度。虽然基础功能肯定有所保证，但其他功能得客户端全都支持才能真正用上。
