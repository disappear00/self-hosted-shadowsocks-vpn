---
name: self-hosted-shadowsocks-vpn
description: This skill should be used when setting up, configuring, troubleshooting, or documenting a self-hosted Shadowsocks-libev relay server on Ubuntu/Debian, especially an Azure VM used with Clash Verge/Clash clients. It covers package installation, service configuration, systemd diagnosis, Azure NSG/UFW rules, Clash YAML configuration, activation, testing, and security hardening.
agent_created: true
---

# 自建 Shadowsocks 中转服务器

## Overview

用于在 Ubuntu/Debian 云服务器上部署 Shadowsocks-libev，并将其配置为 Clash Verge/Clash 客户端可用的代理中转节点。优先采用可验证、可回滚的命令；每一步都先确认实际路径、服务状态和监听端口，避免把“软件已安装”误判为“服务已可用”。

## 使用边界

- 仅用于用户拥有或获授权管理的服务器与网络。
- 不协助绕过访问控制、攻击第三方或隐藏违法活动。
- 生产环境禁止使用弱密码；用户若明确要求临时测试密码，可先照做，但必须标注公网暴露风险。
- 不假设发行版、服务路径或公网 IP；先读取命令输出。

## 工作流决策

### A. 安装失败或版本确认

1. 识别系统：

```bash
cat /etc/os-release
dpkg --print-architecture
```

2. 更新索引并安装：

```bash
sudo apt update
sudo apt install shadowsocks-libev
```

3. 不使用 `ss-server --version` 判断版本。该版本可能将 `--version` 视为未知参数，但仍打印版本和帮助。使用：

```bash
dpkg-query -W -f='${Version}\n' shadowsocks-libev
```

4. 若安装日志出现 `Setting up shadowsocks-libev`，视为安装成功；`needrestart` 的内核重启提示不是安装失败。

### B. 创建配置并确认实际服务路径

1. 先读取 systemd 模板，不能凭经验猜路径：

```bash
systemctl cat shadowsocks-libev-server@.service
```

2. 本次 Ubuntu 24.04 实例的实际 `ExecStart` 使用：

```text
/etc/shadowsocks/config.json
```

3. 创建目录与配置：

```bash
sudo install -d -m 755 /etc/shadowsocks
sudo nano /etc/shadowsocks/config.json
```

4. 最小配置模板：

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

5. 校验 JSON 和权限：

```bash
sudo python3 -m json.tool /etc/shadowsocks/config.json
sudo chown root:root /etc/shadowsocks/config.json
sudo chmod 600 /etc/shadowsocks/config.json
```

6. 配置目录 `/etc/shadowsocks-libev/` 不一定存在，不能据此判断安装失败。若已在该目录创建文件，需复制到服务实际读取路径：

```bash
sudo install -d -m 755 /etc/shadowsocks
sudo cp /etc/shadowsocks-libev/config.json /etc/shadowsocks/config.json
```

### C. 启动与诊断

1. 启用实例服务：

```bash
sudo systemctl enable --now shadowsocks-libev-server@config
```

2. 成功标准：

```bash
sudo systemctl status shadowsocks-libev-server@config --no-pager -l
```

必须看到 `Active: active (running)`。

3. 确认监听：

```bash
sudo ss -lntup | grep ':8388'
```

4. 如果失败，必须获取完整原因：

```bash
sudo journalctl -u shadowsocks-libev-server@config.service -n 50 --no-pager
```

5. 可直接前台验证：

```bash
sudo /usr/bin/ss-server -c /etc/shadowsocks/config.json -v
```

持续运行表示配置和进程基本正常；按 `Ctrl+C` 停止前台测试后再启动 systemd 服务。

6. 常见诊断：

- `status=255/EXCEPTION`：优先检查配置路径、JSON、权限和密码/加密字段。
- `Address already in use`：检查 `sudo ss -lntup | grep ':8388'`，清理重复 `ss-server` 进程或改端口。
- 找不到服务 `shadowsocks-libev.service`：使用 `shadowsocks-libev-server` 或 `shadowsocks-libev-server@config`。
- 服务反复重启：查看 journal，不要盲目重复 restart。

### D. Azure 网络与 Ubuntu 防火墙

1. 在 Azure 网络安全组放行服务器端口；正式使用将来源限制为客户端公网 IP。需要分别考虑 TCP/UDP `8388`。
2. 如果 UFW 已启用：

```bash
sudo ufw allow 8388/tcp
sudo ufw allow 8388/udp
sudo ufw status verbose
```

3. 获取出口公网 IP：

```bash
curl -4 https://ifconfig.me
```

使用 Azure 门户或该命令得到的当前公网 IP，不能使用 VM 内网 IP。建议为生产 VM 配置静态公网 IP。

### E. Clash Verge 本地 YAML

对于单个自建 Shadowsocks 节点，不要把节点当作“Remote 订阅 URL”填写。创建或导入本地 YAML：

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

约束：

- `server` 只写公网 IP 或域名，不写 `ss://`。
- `port` 只写数字。
- `cipher`、密码、端口必须和服务端完全一致。
- 使用空格缩进，不使用 Tab。
- 配置文件成功导入但节点不显示时，重载配置或重启 Clash Verge；本次实测重启后节点正常出现。
- 页面“订阅”显示本地配置不等于节点已激活；进入“代理”，在“节点选择”中选 `Azure-SS`。
- 选中节点后再开启系统代理；不要一开始就启用 TUN，以便先排查基础链路。

### F. SS URL 生成

不要手工拼接 Base64。使用程序生成单行 URL：

```bash
python3 -c "import base64; ip='SERVER_PUBLIC_IP'; print('ss://' + base64.urlsafe_b64encode(b'chacha20-ietf-poly1305:REPLACE_WITH_PASSWORD').decode().rstrip('=') + '@' + ip + ':8388#Azure-SS')"
```

如果客户端拒绝 URL：

1. 确认整行没有真实换行或空格；
2. 确认 `ss://` 前缀存在；
3. 去掉 `#Azure-SS` 再试；
4. 直接使用 Clash 本地 YAML 或手动填写节点字段，避免 URL 编码兼容性问题。

## 验证清单

### 服务端

```bash
sudo systemctl is-active shadowsocks-libev-server@config
sudo ss -lntup | grep ':8388'
sudo journalctl -u shadowsocks-libev-server@config -n 30 --no-pager
```

### 客户端

1. Clash 配置已加载。
2. “代理”页面存在 `Azure-SS`。
3. “节点选择”已选 `Azure-SS`，不是 `DIRECT`。
4. 系统代理已开启。
5. 访问 `https://ifconfig.me` 或 `https://ipinfo.io` 检查出口。
6. 同时观察服务端 journal，确认有连接记录。

## 安全加固

- `123.com` 仅可用于临时测试；正式使用立即更换随机长密码：

```bash
openssl rand -base64 24
```

- 修改服务端密码后同步修改 Clash YAML，并重启服务。
- Azure NSG 来源限制为可信公网 IP；不长期使用 `Any`。
- 配置文件使用 `chmod 600`，避免密码泄露。
- 不在聊天、截图或公共文档中暴露真实公网 IP、密码和完整订阅 URL。
- Shadowsocks 是代理中转，不等同于 WireGuard/OpenVPN 的完整三层 VPN。若需求是整机流量、内网互通或设备级 VPN，评估 WireGuard。
