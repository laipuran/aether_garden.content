---
slug: derp-deploy
title: 自建 Tailscale DERP 中继服务器
date: 2026-07-20
excerpt: 在阿里云 ECS 上用 Docker 部署 Tailscale DERP 中继服务器，解决海外中继延迟高的问题。
tags:
  - tailscale
  - derp
  - docker
  - network
status: published
updatedAt: 2026-07-20
---

## 背景

Tailscale 的官方 DERP 中继服务器都在海外，国内延迟很高。自建一个国内的 DERP 节点可以显著改善连接质量。

## 环境

- 云服务器：阿里云 ECS，8.162.3.201，Ubuntu 22.04
- 已有服务：Nginx 托管 duckran.top（占用 80/443），已配置 Certbot Let's Encrypt 证书
- Tailscale：已在服务器上安装并认证
- 镜像源：`swr.cn-north-4.myhuaweicloud.com/ddn-k8s/ghcr.io/yangchuansheng/ip_derper:latest`

## 踩坑记录

### docker-compose.yml 初版

从网上找到的配置是这样的：

```yaml
environment:
  - DERP_DOMAIN=derp.duckran.top
  - DERP_CERT_MODE=letsencrypt
```

但这两个环境变量对这个镜像**完全无效**。看了 Dockerfile 才明白：

```dockerfile
CMD bash /app/build_cert.sh $DERP_HOST $DERP_CERTS /app/san.conf && \
    /app/derper --hostname=$DERP_HOST \
    --certmode=manual \
    ...
```

镜像实际用的变量是 `DERP_HOST`（不是 `DERP_DOMAIN`），且 `--certmode=manual` 是**硬编码**的，写 `letsencrypt` 也不会生效。

### build_cert.sh 的 bug

镜像里的 `build_cert.sh` 生成自签名证书时，把域名写到了 SAN 的 `IP.1` 字段：

```bash
[alt_names]
IP.1 = $CERT_HOST
```

OpenSSL 拒绝将域名填入 IP 字段，启动就崩溃：

```
140284822005056:error:220A4076:X509 V3 routines:a2i_GENERAL_NAME:bad ip address
```

修复方式是把 `IP.1` 改成 `DNS.1`。

### 自签名证书不被 Tailscale 信任

即使自签名证书生成成功，Tailscale 客户端在健康检查中报错：

```
Tailscale could not establish an encrypted connection with "derp.duckran.top":
likely intercepted connection; certificate is self-signed
```

解决方案是改用 Let's Encrypt 证书。

## 最终方案

因为服务器上已经有 Nginx + Certbot，直接用 webroot 模式签发证书最省事：

```bash
sudo certbot certonly --webroot \
  -w /var/www/html \
  -d derp.duckran.top
```

### 端口规划

80/443 被 Nginx 占用，所以 DERP 改用非标准端口：

| 端口 | 协议 | 用途 |
|------|------|------|
| 33443 | TCP | DERP relay |
| 3478 | UDP | STUN |

### 最终 docker-compose.yml

```yaml
services:
  derper:
    image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/ghcr.io/yangchuansheng/ip_derper:latest
    container_name: derper
    restart: always
    ports:
      - "33443:33443"
      - "3478:3478/udp"
    volumes:
      - /etc/letsencrypt/live/derp.duckran.top/fullchain.pem:/app/certs/derp.duckran.top.crt
      - /etc/letsencrypt/live/derp.duckran.top/privkey.pem:/app/certs/derp.duckran.top.key
      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock
    environment:
      - DERP_HOST=derp.duckran.top
      - DERP_ADDR=:33443
      - DERP_HTTP_PORT=0
      - DERP_CERTS=/app/certs/
      - DERP_STUN=true
      - DERP_VERIFY_CLIENTS=true
    command:
      - /app/derper
      - --hostname=derp.duckran.top
      - --certmode=manual
      - --certdir=/app/certs/
      - --stun=true
      - --a=:33443
      - --http-port=0
      - --verify-clients=true
```

关键点：
- `DERP_ADDR=:33443` 配合 `command` 中的 `--a=:33443` 指定非标准端口
- 直接挂载 LE 证书到 derper 期望的路径（`/app/certs/{hostname}.crt`）
- 用 `command` 覆盖镜像默认的 `CMD`，跳过坏的 `build_cert.sh`
- `--verify-clients=true` 依赖本地 `tailscaled.sock`，服务器必须先加入 tailnet

### Tailscale ACL 配置

```json
"Nodes": [
  {
    "Name": "1a",
    "RegionID": 901,
    "RegionCode": "my-derp",
    "HostName": "derp.duckran.top",
    "DERPPort": 33443,
    "STUNPort": 3478
  }
]
```

### 证书续签

Certbot 自动续签，续签后需要重启 DERP 容器加载新证书。在 deploy hook 中配置：

```bash
sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy

cat > /tmp/derper-restart.sh << 'EOF'
#!/bin/bash
docker compose -f /home/duckran/derper/docker-compose.yml restart derper
EOF

sudo cp /tmp/derper-restart.sh /etc/letsencrypt/renewal-hooks/deploy/derper-restart.sh
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/derper-restart.sh
```

## 验证

```bash
# 从公网验证 TLS 证书
openssl s_client -connect derp.duckran.top:33443 -servername derp.duckran.top

# DERP 页面
curl https://derp.duckran.top:33443/
```

## 总结

- `ip_derper` 镜像的文档和环境变量名与实际行为不一致，建议直接读 Dockerfile
- 非标准端口配合 Let's Encrypt 是最省心的方案，不需要动已有的 Nginx
- `--verify-clients` 依赖本地 `tailscaled`，服务器要先 `tailscale up`
