# Docker OpenVPN 批量快速部署指南

本文档说明如何利用 `docker commit` 将一台已初始化的 OpenVPN 容器打包成镜像，再分发到多台服务器实现**一次初始化、批量部署**，无需每台服务器单独执行 PKI 生成流程。

---

## 前提条件

- 所有目标服务器均已安装 Docker
- 防火墙已开放 VPN 端口（默认 TCP 1194）
- 拥有一台可访问所有目标服务器的"主控机"（用于分发镜像包）

---

## 步骤 1：在第一台服务器上完成初始化

启动容器，让其自动生成 PKI、服务器证书、TLS 密钥及 `client.ovpn`：

```bash
docker run \
    --name openvpn \
    --restart=always \
    -p 1194:1194/tcp \
    -d --cap-add=NET_ADMIN \
    --device=/dev/net/tun \
    --sysctl net.ipv4.ip_forward=1 \
    --sysctl net.ipv6.conf.all.forwarding=1 \
    hwdsl2/openvpn-server
```

等待初始化完成（通常约 30 秒），通过日志确认：

```bash
docker logs -f openvpn
# 看到类似 "OpenVPN started successfully" 的输出即可按 Ctrl+C 退出
```

---

## 步骤 2：验证初始化结果

确认 `client.ovpn` 已生成：

```bash
docker exec openvpn ls /etc/openvpn/clients/
# 应输出：client.ovpn
```

---

## 步骤 3：将容器打包为新镜像

```bash
# 将运行中的容器提交为新镜像
docker commit openvpn my-openvpn:v1

# 确认镜像已创建
docker images my-openvpn:v1
```

---

## 步骤 4：导出镜像为文件

```bash
docker save my-openvpn:v1 | gzip > openvpn-v1.tar.gz

# 确认文件大小（通常约 20–30 MB）
ls -lh openvpn-v1.tar.gz
```

---

## 步骤 5：提取客户端配置文件

```bash
docker cp openvpn:/etc/openvpn/clients/client.ovpn ./client.ovpn
```

此文件将分发给 VPN 用户。连接不同服务器时，需修改其中的 `remote` 行（见步骤 8）。

---

## 步骤 6：将镜像包分发到所有目标服务器

将 `openvpn-v1.tar.gz` 上传到每台目标服务器：

```bash
# 示例：使用 scp 分发（对每台服务器重复执行）
scp openvpn-v1.tar.gz user@<目标服务器IP>:/root/
```

如果服务器数量较多，可使用批量工具（如 `pscp`、Ansible、`parallel-ssh`）并行上传：

```bash
# 示例：使用 parallel-ssh + pscp 批量上传
# 先将所有目标服务器 IP 保存到 hosts.txt（每行一个 IP）
pscp -h hosts.txt -l root openvpn-v1.tar.gz /root/
```

---

## 步骤 7：在每台目标服务器上加载并启动容器

登录到每台目标服务器后执行：

```bash
# 加载镜像
docker load < /root/openvpn-v1.tar.gz

# 启动容器
docker run \
    --name openvpn \
    --restart=always \
    -p 1194:1194/tcp \
    -d --cap-add=NET_ADMIN \
    --device=/dev/net/tun \
    --sysctl net.ipv4.ip_forward=1 \
    --sysctl net.ipv6.conf.all.forwarding=1 \
    my-openvpn:v1

# 确认容器运行正常
docker ps | grep openvpn
```

或使用批量工具在所有服务器上并行执行：

```bash
# 示例：使用 parallel-ssh 批量启动
parallel-ssh -h hosts.txt -l root \
  "docker load < /root/openvpn-v1.tar.gz && docker run --name openvpn --restart=always -p 1194:1194/tcp -d --cap-add=NET_ADMIN --device=/dev/net/tun --sysctl net.ipv4.ip_forward=1 --sysctl net.ipv6.conf.all.forwarding=1 my-openvpn:v1"
```

---

## 步骤 8：为每台服务器生成对应的客户端配置文件

所有服务器共用同一套证书，只需修改 `client.ovpn` 中的 `remote` 地址：

```bash
# 为服务器 IP 192.168.1.100 生成配置
sed 's/^remote .*/remote 192.168.1.100 1194/' client.ovpn > client-192.168.1.100.ovpn
```

批量生成脚本示例：

```bash
#!/bin/bash
# gen_clients.sh
# 用法：将所有服务器 IP 保存到 hosts.txt，然后运行此脚本

while read -r IP; do
    sed "s/^remote .*/remote ${IP} 1194/" client.ovpn > "client-${IP}.ovpn"
    echo "已生成：client-${IP}.ovpn"
done < hosts.txt
```

```bash
chmod +x gen_clients.sh
./gen_clients.sh
```

---

## 步骤 9：将客户端配置分发给用户

将对应的 `.ovpn` 文件发送给需要连接该服务器的用户。用户使用 OpenVPN 客户端导入即可连接。

---

## 注意事项

| 事项 | 说明 |
|---|---|
| **证书共享** | 所有服务器使用相同的 CA 和服务器证书，同一份 `client.ovpn`（修改 IP 后）可连接任意服务器 |
| **安全性** | 由于共享密钥，一台服务器被攻破将影响所有服务器。如有高安全需求，请为每台服务器单独初始化 |
| **端口冲突** | 如宿主机 1194 端口已被占用，可将 `-p 1194:1194/tcp` 改为 `-p <其他端口>:1194/tcp` |
| **防火墙** | 确保每台服务器的防火墙规则允许 TCP 1194 入站流量 |
| **更新镜像** | 需要更新时，重新在一台服务器上 `docker pull hwdsl2/openvpn-server`，完成初始化后重新走步骤 3–8 |

---

## 快速参考命令汇总

```bash
# ===== 初始化服务器（执行一次）=====
docker run --name openvpn --restart=always -p 1194:1194/tcp -d \
  --cap-add=NET_ADMIN --device=/dev/net/tun \
  --sysctl net.ipv4.ip_forward=1 --sysctl net.ipv6.conf.all.forwarding=1 \
  hwdsl2/openvpn-server

# ===== 打包镜像 =====
docker commit openvpn my-openvpn:v1
docker save my-openvpn:v1 | gzip > openvpn-v1.tar.gz
docker cp openvpn:/etc/openvpn/clients/client.ovpn .

# ===== 目标服务器部署（每台执行）=====
docker load < openvpn-v1.tar.gz
docker run --name openvpn --restart=always -p 1194:1194/tcp -d \
  --cap-add=NET_ADMIN --device=/dev/net/tun \
  --sysctl net.ipv4.ip_forward=1 --sysctl net.ipv6.conf.all.forwarding=1 \
  my-openvpn:v1

# ===== 生成指定 IP 的客户端配置 =====
sed 's/^remote .*/remote <服务器IP> 1194/' client.ovpn > client-<服务器IP>.ovpn
```
