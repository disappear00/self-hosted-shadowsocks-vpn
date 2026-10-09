# 自建 Shadowsocks 中转服务器

这是一个面向 Ubuntu/Debian 云服务器的 Shadowsocks-libev 部署与排障指南，适用于将 Azure VM 配置为可供 Clash Verge 等客户端使用的 Shadowsocks 中转节点。

文档重点是使用可验证、可回滚的命令完成安装、配置、启动、网络放行和客户端接入，避免把“软件已安装”误判为“服务已可用”。

## 适用场景

- 在 Ubuntu/Debian 云服务器上部署 Shadowsocks-libev
- 在 Azure VM 上配置 TCP/UDP 代理端口
- 使用 Clash Verge 导入并启用自建 Shadowsocks 节点
- 检查 systemd 服务、监听端口和连接日志
- 排查配置路径、JSON、权限、防火墙和网络安全组问题

本项目只适用于管理者拥有或已获授权的服务器与网络，不用于绕过访问控制、攻击第三方或隐藏违法活动。

## 快速开始

### 1. 安装 Shadowsocks-libev

```bash
cat /etc/os-release
dpkg --print-architecture
sudo apt update
sudo apt install shadowsocks-libev
```

检查已安装版本：

```bash
dpkg-query -W -f='${Version}\n' shadowsocks-libev
```

### 2. 创建服务端配置

先确认 systemd 模板实际读取的配置路径：

```bash
systemctl cat shadowsocks-libev-server@.service
```

创建配置文件：

```bash
sudo install -d -m 755 /etc/shadowsocks
sudo nano /etc/shadowsocks/config.json
```

最小配置示例：

```json
{
    "server": "0.0.0.0",
    "server_port": 8388,
    "password": "REPLACE_WITH_PASSWORD",
    "method": "chacha20-ietf-poly1305",
    "mode": "tcp_and_udp",
    "fast_open": false
}
```

校验格式并限制权限：

```bash
sudo python3 -m json.tool /etc/shadowsocks/config.json
sudo chown root:root /etc/shadowsocks/config.json
sudo chmod 600 /etc/shadowsocks/config.json
```

### 3. 启动服务

```bash
sudo systemctl enable --now shadowsocks-libev-server@config
sudo systemctl status shadowsocks-libev-server@config --no-pager -l
sudo ss -lntup | grep ':8388'
```

成功标准是服务状态显示 `Active: active (running)`，并且端口 `8388` 正在监听。

### 4. 放行网络端口

在 Azure 网络安全组中放行服务器端口 `8388` 的 TCP 和 UDP 流量。正式使用时，应将来源限制为可信客户端公网 IP，而不是长期使用 `Any`。

如果服务器启用了 UFW：

```bash
sudo ufw allow 8388/tcp
sudo ufw allow 8388/udp
sudo ufw status verbose
```

### 5. 配置 Clash Verge

对于单个自建节点，建议创建或导入本地 YAML，而不是将节点当作 Remote 订阅 URL：

```yaml
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info
ipv6: false

proxies:
  - name: Azure-SS
    type: ss
    server: SERVER_PUBLIC_IP
    port: 8388
    cipher: chacha20-ietf-poly1305
    password: "REPLACE_WITH_PASSWORD"
    udp: true

proxy-groups:
  - name: 节点选择
    type: select
    proxies:
      - Azure-SS
      - DIRECT

rules:
  - MATCH,节点选择
```

在 Clash Verge 中加载配置后，进入“代理”，在“节点选择”中选中 `Azure-SS`，再开启系统代理。排查基础链路时，不建议一开始启用 TUN。

## 验证与排障

### 服务端检查

```bash
sudo systemctl is-active shadowsocks-libev-server@config
sudo ss -lntup | grep ':8388'
sudo journalctl -u shadowsocks-libev-server@config -n 30 --no-pager
```

查看最近的完整错误：

```bash
sudo journalctl -u shadowsocks-libev-server@config.service -n 50 --no-pager
```

也可以绕过 systemd 临时前台运行：

```bash
sudo /usr/bin/ss-server -c /etc/shadowsocks/config.json -v
```

常见问题：

- `status=255/EXCEPTION`：检查配置路径、JSON 格式、文件权限、密码和加密方式。
- `Address already in use`：检查 `8388` 是否已被其他 `ss-server` 进程占用。
- 找不到 `shadowsocks-libev.service`：使用 `shadowsocks-libev-server@config` 实例服务。
- 服务反复重启：先查看 journal，不要盲目重复执行 `restart`。
- Clash 节点不显示：重载配置或重启 Clash Verge。

### 客户端检查

1. Clash 配置已成功加载。
2. “代理”页面存在 `Azure-SS`。
3. “节点选择”已选中 `Azure-SS`，而不是 `DIRECT`。
4. 系统代理已开启。
5. 访问 `https://ifconfig.me` 或 `https://ipinfo.io` 检查出口 IP。
6. 同时观察服务端 journal，确认出现连接记录。

## 生成 Shadowsocks URL

不要手工拼接 Base64，可以使用以下命令生成单行 URL：

```bash
python3 -c "import base64; ip='SERVER_PUBLIC_IP'; print('ss://' + base64.urlsafe_b64encode(b'chacha20-ietf-poly1305:REPLACE_WITH_PASSWORD').decode().rstrip('=') + '@' + ip + ':8388#Azure-SS')"
```

如果客户端拒绝 URL：

- 确认整行没有真实换行或多余空格。
- 确认包含 `ss://` 前缀。
- 去掉 `#Azure-SS` 后重试。
- 直接使用 Clash 本地 YAML 或手动填写节点字段。

## 安全建议

- 生产环境不要使用弱密码，建议使用 `openssl rand -base64 24` 生成随机长密码。
- 修改服务端密码后，同时更新 Clash 配置并重启服务。
- Azure NSG 的来源范围应限制为可信公网 IP。
- 配置文件应保持 `chmod 600`，避免密码泄露。
- 不要在聊天、截图或公共文档中暴露真实公网 IP、密码或完整订阅 URL。
- Shadowsocks 是代理中转，不等同于 WireGuard/OpenVPN 的完整三层 VPN。若需要整机流量、内网互通或设备级 VPN，应评估 WireGuard 等方案。

## 文件说明

- `SKILL.md`：供 AI 助手使用的详细部署、配置、排障和安全工作流。
- `README.md`：面向用户的项目概览和快速操作指南。

## 许可证

当前仓库未声明许可证。如需公开发布或被其他项目引用，请先补充合适的许可证文件。
