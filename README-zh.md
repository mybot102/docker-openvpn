[简体中文](README-zh.md) | [繁體中文](README-zh-Hant.md)

# 在 Docker 上运行 OpenVPN 服务器

[![Build Status](https://github.com/mybot102/docker-openvpn/actions/workflows/main.yml/badge.svg)](https://github.com/mybot102/docker-openvpn/actions/workflows/main.yml) &nbsp;[![License: MIT](docs/images/license.svg)](https://opensource.org/licenses/MIT)

一个用于运行 OpenVPN 服务器的 Docker 镜像。基于 Alpine Linux，集成 OpenVPN 和 EasyRSA，设计目标是简单、现代且易于维护。

- 首次启动时自动生成 PKI、服务器证书以及客户端配置
- 使用辅助脚本（`ovpn_manage`）进行客户端管理
- 现代加密套件：AES-256-GCM、SHA256、tls-crypt
- 服务器有公网 IPv6 地址时支持 IPv6（参见[要求](#ipv6-支持)）
- 使用 Docker 卷实现数据持久化
- 多架构支持：`linux/amd64`、`linux/arm64`、`linux/arm/v7`

**另提供：** [WireGuard](https://github.com/hwdsl2/docker-wireguard/blob/main/README-zh.md)、[IPsec VPN](https://github.com/hwdsl2/docker-ipsec-vpn-server/blob/master/README-zh.md) 和 [Headscale](https://github.com/hwdsl2/docker-headscale/blob/main/README-zh.md) 的 Docker 镜像。

## 快速开始

**步骤 1.** 启动 OpenVPN 服务器：

```bash
docker run \
    --name openvpn \
    --restart=always \
    -v openvpn-data:/etc/openvpn \
    -p 1194:1194/tcp \
    -d --cap-add=NET_ADMIN \
    --device=/dev/net/tun \
    --sysctl net.ipv4.ip_forward=1 \
    --sysctl net.ipv6.conf.all.forwarding=1 \
    ghcr.io/mybot102/openvpn-server
```

首次启动时，服务器将自动生成 PKI、服务器证书、TLS 加密密钥以及名为 `client.ovpn` 的客户端配置文件。

**步骤 2.** 将客户端配置文件复制到本机：

```bash
docker cp openvpn:/etc/openvpn/clients/client.ovpn .
```

将 `client.ovpn` 导入到 OpenVPN 客户端即可连接。

另外，你也可以在不使用 Docker 的情况下[安装 OpenVPN](https://github.com/hwdsl2/openvpn-install/blob/master/README-zh.md)。要了解更多有关如何使用本镜像的信息，请继续阅读以下部分。

## 系统要求

- 具有公网 IP 地址或 DNS 名称的 Linux 服务器
- 已安装 Docker
- 防火墙中已开放 VPN 端口（默认 TCP 1194，或自定义端口/协议）

## 下载

从 [GitHub Container Registry](https://ghcr.io/mybot102/openvpn-server) 获取镜像：

```bash
docker pull ghcr.io/mybot102/openvpn-server
```

支持平台：`linux/amd64`、`linux/arm64` 和 `linux/arm/v7`。

## 环境变量

所有变量均为可选项。未设置时将自动使用安全默认值。

此 Docker 镜像使用以下变量，可以在 `env` 文件中声明（参见[示例](vpn.env.example)）：

| 变量 | 说明 | 默认值 |
|---|---|---|
| `VPN_DNS_NAME` | 服务器的完全限定域名 (FQDN) | 自动检测公网 IP |
| `VPN_PUBLIC_IP` | 服务器的公网 IPv4 地址 | 自动检测 |
| `VPN_PUBLIC_IP6` | 服务器的公网 IPv6 地址 | 自动检测 |
| `VPN_PROTO` | VPN 协议：`udp` 或 `tcp` | `tcp` |
| `VPN_PORT` | VPN 端口（1–65535） | `1194` |
| `VPN_CLIENT_NAME` | 生成的第一个客户端配置名称 | `client` |
| `VPN_EXTRA_CLIENTS` | 首次启动时额外预置的客户端名称列表（逗号分隔） | 无 |
| `VPN_DNS_SRV1` | 推送给客户端的主 DNS 服务器 | `8.8.8.8` |
| `VPN_DNS_SRV2` | 推送给客户端的备用 DNS 服务器 | `8.8.4.4` |

**注：** 在 `env` 文件中，不要在值周围添加 `""` 或 `''`，也不要在 `=` 周围添加空格。如果修改了 `VPN_PORT` 或 `VPN_PROTO`，请相应更新 `docker run` 命令中的 `-p` 参数。

使用 `env` 文件的示例：

```bash
docker run \
    --name openvpn \
    --env-file ./vpn.env \
    --restart=always \
    -v openvpn-data:/etc/openvpn \
    -p 1194:1194/tcp \
    -d --cap-add=NET_ADMIN \
    --device=/dev/net/tun \
    --sysctl net.ipv4.ip_forward=1 \
    --sysctl net.ipv6.conf.all.forwarding=1 \
    ghcr.io/mybot102/openvpn-server
```

## 客户端管理

使用 `docker exec` 配合 `ovpn_manage` 辅助脚本管理客户端。

**添加新客户端：**

```bash
docker exec openvpn ovpn_manage --addclient alice
docker cp openvpn:/etc/openvpn/clients/alice.ovpn .
```

**导出客户端配置**（输出到标准输出）：

```bash
docker exec openvpn ovpn_manage --exportclient alice > alice.ovpn
```

**列出客户端：**

```bash
docker exec openvpn ovpn_manage --listclients
```

**吊销客户端**（将提示确认）：

```bash
docker exec -it openvpn ovpn_manage --revokeclient alice
# 或不提示确认直接吊销：
docker exec openvpn ovpn_manage --revokeclient alice -y
```

## 预置多个用户

### 方法一：通过环境变量批量创建（推荐）

在 `vpn.env` 中设置 `VPN_EXTRA_CLIENTS`，首次启动时自动创建所有用户：

```
VPN_CLIENT_NAME=alice
VPN_EXTRA_CLIENTS=bob,charlie,dave
```

启动后批量下载所有配置文件：

```bash
docker compose up -d
for name in alice bob charlie dave; do
  docker cp openvpn:/etc/openvpn/clients/${name}.ovpn .
done
```

### 方法二：将配置打包进镜像（无需恢复数据卷）

如果你希望将服务器和用户配置直接打包进 Docker 镜像，以便在任意主机上部署时无需挂载或恢复数据卷，可以使用 `docker commit`：

**步骤 1.** 正常启动容器，让首次初始化完成（含所有预置用户）：

```bash
VPN_DNS_NAME=vpn.example.com  # 必须设置，客户端配置里会写入此地址
VPN_CLIENT_NAME=alice
VPN_EXTRA_CLIENTS=bob,charlie,dave
# 在 vpn.env 中设置好上述变量，然后：
docker compose up -d
# 等待初始化完成
docker logs openvpn
```

**步骤 2.** 将运行中的容器状态提交为新镜像：

```bash
docker commit openvpn my-openvpn:v1
```

**步骤 3.** 将镜像推送到你的镜像仓库或导出为文件：

```bash
# 推送到镜像仓库
docker push my-openvpn:v1

# 或导出为 tar 文件
docker save my-openvpn:v1 | gzip > my-openvpn-v1.tar.gz
```

**步骤 4.** 在新主机上直接使用该镜像，无需挂载数据卷：

```bash
# 加载镜像（如果是 tar 文件）
docker load < my-openvpn-v1.tar.gz

# 直接运行，无需 -v 挂载卷
docker run \
    --name openvpn \
    --restart=always \
    -p 1194:1194/tcp \
    -d --cap-add=NET_ADMIN \
    --device=/dev/net/tun \
    --sysctl net.ipv4.ip_forward=1 \
    --sysctl net.ipv6.conf.all.forwarding=1 \
    my-openvpn:v1
```

> **注意：** 打包进镜像的客户端配置中包含 `VPN_DNS_NAME`（或 `VPN_PUBLIC_IP`）。如果在新主机上 IP/域名不同，客户端需要手动修改 `.ovpn` 文件中的 `remote` 行，或重新生成配置。私钥会随镜像分发，请妥善管理镜像访问权限。

## 持久化数据

所有服务器和客户端数据均存储在 Docker 卷（容器内的 `/etc/openvpn`）中：

```
/etc/openvpn/
├── server/
│   ├── server.conf         # OpenVPN 服务器配置
│   ├── ca.crt              # CA 证书
│   ├── server.crt/key      # 服务器证书和密钥
│   ├── tc.key              # TLS 加密密钥
│   ├── dh.pem              # DH 参数
│   ├── crl.pem             # 证书吊销列表
│   ├── client-common.txt   # 客户端配置模板
│   ├── ipp.txt             # IP 池持久化
│   └── easy-rsa/pki/       # 完整 PKI 目录
└── clients/
    ├── client.ovpn         # 第一个客户端配置
    └── alice.ovpn          # 其他客户端
```

备份 Docker 卷以保存所有密钥和客户端配置。

## IPv6 支持

如果 Docker 宿主机拥有公共（全局单播）IPv6 地址并且满足以下要求，IPv6 支持将在容器启动时自动启用，无需手动配置。

**要求：**
- Docker 宿主机必须拥有可路由的全局单播 IPv6 地址（以 `2` 或 `3` 开头）。链路本地地址（`fe80::/10`）不满足要求。
- 必须为 Docker 容器启用 IPv6。参见[在 Docker 中启用 IPv6 支持](https://docs.docker.com/engine/daemon/ipv6/)。

要为 Docker 容器启用 IPv6，首先在 Docker 宿主机上将以下内容添加到 `/etc/docker/daemon.json`，然后重启 Docker：

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "fddd:1::/64"
}
```

然后重新创建容器。要验证 IPv6 是否正常工作，请连接到 VPN，然后检查你的 IPv6 地址，例如使用 [test-ipv6.com](https://test-ipv6.com)。

## 使用 docker-compose

```bash
cp vpn.env.example vpn.env
# 如需修改，请编辑 vpn.env，然后：
docker compose up -d
docker cp openvpn:/etc/openvpn/clients/client.ovpn .
```

## 更新 Docker 镜像

要更新 Docker 镜像和容器，首先[下载](#下载)最新版本：

```bash
docker pull ghcr.io/mybot102/openvpn-server
```

如果 Docker 镜像已是最新版本，将显示：

```
Status: Image is up to date for ghcr.io/mybot102/openvpn-server:latest
```

否则将下载最新版本。按照[快速开始](#快速开始)中的说明删除并重新创建容器。数据保存在 `openvpn-data` 卷中。

## 技术细节

- 基础镜像：`alpine:3.23`
- OpenVPN：来自 Alpine 软件包的最新版本
- EasyRSA：3.2.6（构建时内置）
- 加密算法：AES-256-GCM
- 认证：SHA256
- 密钥交换：tls-crypt（HMAC + 加密）
- DH 参数：预定义 ffdhe2048 组（RFC 7919）
- 客户端证书：10 年有效期
- VPN 子网：`10.8.0.0/24`
- IPv6 VPN 子网：`fddd:1194:1194:1194::/64`（服务器有 IPv6 时启用）

## 授权协议

**注：** 预构建镜像中的软件组件（如 OpenVPN 和 EasyRSA）遵循各自版权持有者所选择的相应许可证。对于任何预构建镜像的使用，镜像用户有责任确保其使用符合镜像中所包含的所有软件的相关许可证。

Copyright (C) 2026 Lin Song   
本作品依据 [MIT 许可证](https://opensource.org/licenses/MIT)授权。

本项目部分基于 [Nyr 和贡献者](https://github.com/Nyr/openvpn-install)的工作，遵循 [MIT 许可证](https://github.com/Nyr/openvpn-install/blob/master/LICENSE)。