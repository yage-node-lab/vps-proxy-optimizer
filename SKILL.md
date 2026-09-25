---
name: vps-proxy-optimizer
description: >-
  VPS搭建技能（从零搭建、深度优化与故障排查）：专用于全新空白 VPS 的全自动一键搭建自建节点，
  或者对已有 VPS 节点进行深度优化与排障。支持一键部署 VLESS-Reality、Hysteria 2、
  Cloudflare 官方 WARP SOCKS5 智能分流（Netflix/AI 走纯净 IP，YouTube/日常 100% 原生高速直连），
  全套实施内核级深层 TCP/UDP 64MB BDP 吞吐调优、BBR 加速、TFO 双向握手优化，并交付防污染客户端订阅与二维码。
---

# 🌐 VPS搭建技能 (全新机一键搭建 · 深度优化 · 智能分流 · 故障排障)

本技能专为 **全新空白 VPS 从零全自动搭建** 以及 **已有 VPS 节点深度体检与优化** 设计。遵循工业级实战经验，杜绝网页卡顿、杜绝 IPv6 假死超时、完美兼顾 **“Netflix 与 AI 解锁”** 与 **“YouTube 4K 与日常网页原生极速秒开”**。

---

## 核心交付架构

无论新机搭建还是旧机优化，统一交付以下高可用、抗封锁、极速网络架构：

| 协议 / 服务 | 端口 / 传输 | 核心用途与优势 |
| :--- | :--- | :--- |
| **Hysteria 2** | `UDP 38443` | **抗丢包主力**（基于 QUIC/UDP，带 Salamander 混淆，晚高峰极速狂飙） |
| **VLESS-Reality** | `TCP 443` | **稳定伪装主力**（原生 TLS 1.3 借用 Mozilla 证书伪装，免自己维护域名） |
| **Cloudflare WARP** | 本地 `127.0.0.1:40000` | **SOCKS5 纯净代理出口**（仅供 Netflix/AI 分流出站，绝不污染全局系统路由） |
| **TCP/UDP BDP 调优** | 系统内核级（64MB） | 解锁长肥网络（LFN）吞吐上限，彻底释放跨洋长距离延迟下的单线程极速 |
| **BBR + TFO 加速** | 系统内核级 | 优化拥塞控制，双向 TCP Fast Open 减少握手 RTT，关闭空闲慢启动 |
| **单栈 IPv6 禁用** | 系统内核级 | 单 IPv4 机型彻底关闭 IPv6，杜绝 AAAA 寻址造成的 5~15s 超时卡顿 |

---

## 选型与环境铁律 (Pre-flight Rules)

1. **操作系统镜像**：
   * **强烈首选**：`Debian 12 (Bookworm) 64-bit`（生态兼容性最稳定，内存占用仅 60MB）。
   * **禁止选择**：`Debian 13 (Trixie)` 或测试版（Cloudflare WARP 官方 apt 源未适配，会导致安装 404）；避免 `CentOS 7`（已淘汰）与 `Alpine`（musl libc 易引发预编译二进制不兼容）。
2. **流量模式**：
   * 优先选择 `1Gbps` 高速突发带宽模式；避免低速无限流模式（如 4Mbps，限速只有 500KB/s，无法流畅观看 4K）。
3. **远程交互**：
   * 在 Windows / 脚本中执行非交互式 SSH 时，**必须使用 `ssh -n`**，防止因 stdin 挂起导致会话无限阻塞。

---

## 模式一：全新空白 VPS 一键从零搭建 (Clean Setup)

当用户提供了一台崭新的空白服务器（如 Debian 12 / Ubuntu 22.04 LTS 等）时，按以下流水线全自动执行：

### 步骤 1：系统底层初始化与深层 TCP/UDP 协议栈调优

```bash
# 1. 自动等待新机首次开机的后台系统升级锁释放
while pgrep -x 'apt-get' >/dev/null || pgrep -x 'dpkg' >/dev/null; do
    echo "Waiting for background package manager lock..."
    sleep 3
done

# 2. 更新系统软件包与基础工具
export DEBIAN_FRONTEND=noninteractive
apt-get update -qq && apt-get install -y curl wget git jq ufw openssl lsb-release gnupg socat

# 3. 写入工业级深层 TCP / 网络协议栈调优参数
cat << 'EOF' > /etc/sysctl.d/99-vps-optimizer.conf
# 拥塞控制与队列算法
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# BDP 长肥管道吞吐优化（TCP & UDP 缓冲区提升至 64MB）
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864

# 突发连接队列扩容
net.core.somaxconn = 8192
net.ipv4.tcp_max_syn_backlog = 8192
net.core.netdev_max_backlog = 10000

# 连接快速回收与复用
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_keepalive_probes = 5

# 握手加速与窗口行为（双向 TFO + 关闭空闲慢启动）
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_slow_start_after_idle = 0

# 单 IPv4 机型禁用 IPv6（若无原生 IPv6 路由时开启）
# net.ipv6.conf.all.disable_ipv6 = 1
# net.ipv6.conf.default.disable_ipv6 = 1

# 文件描述符上限
fs.file-max = 1000000
EOF

sysctl -p /etc/sysctl.d/99-vps-optimizer.conf

# 4. 提升系统句柄限制
cat << 'EOF' > /etc/security/limits.d/99-nofile.conf
* soft nofile 1000000
* hard nofile 1000000
root soft nofile 1000000
root hard nofile 1000000
EOF

# 5. 配置防火墙放行
ufw allow 22/tcp || true
ufw allow 443/tcp || true
ufw allow 38443/udp || true
```

### 步骤 2：安装配置 Cloudflare 官方 WARP（SOCKS5 局部代理）

```bash
# 1. 接入 Cloudflare 官方 apt 源并安装
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | gpg --yes --dearmor --output /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/cloudflare-client.list
apt-get update -qq && apt-get install -y cloudflare-warp

# 2. 注册并设为 SOCKS5 代理模式（端口 40000）
warp-cli --accept-tos registration new 2>/dev/null || warp-cli --accept-tos register 2>/dev/null || true
warp-cli --accept-tos mode proxy
warp-cli --accept-tos proxy port 40000
warp-cli --accept-tos connect

# 3. 验证出口 IP 是否为 Cloudflare 清洁 IP
curl -s -m 5 --socks5 127.0.0.1:40000 https://api.ipify.org
```

### 步骤 3：部署 Hysteria 2（带严格 ACL 分流）

```bash
# 1. 一键拉取官方最新 Hysteria 2 二进制
bash <(curl -fsSL https://get.hy2.sh/)

# 2. 生成自签名证书
mkdir -p /etc/hysteria
openssl req -x509 -nodes -newkey ec:<(openssl ecparam -name prime256v1) -keyout /etc/hysteria/server.key -out /etc/hysteria/server.crt -subj "/CN=bing.com" -days 36500

# 3. 生成随机密码与混淆密钥并写入 /etc/hysteria/config.yaml
HY_PASS=$(openssl rand -hex 16)
HY_OBFS=$(openssl rand -hex 12)

cat << EOF > /etc/hysteria/config.yaml
listen: :38443

tls:
  cert: /etc/hysteria/server.crt
  key: /etc/hysteria/server.key

auth:
  type: password
  password: "${HY_PASS}"

obfs:
  type: salamander
  salamander:
    password: "${HY_OBFS}"

outbounds:
  - name: direct
    type: direct
  - name: warp
    type: socks5
    socks5:
      addr: 127.0.0.1:40000

acl:
  inline:
    - warp(domain-suffix(netflix.com))
    - warp(domain-suffix(nflxvideo.net))
    - warp(domain-suffix(nflxext.com))
    - warp(domain-suffix(nflximg.com))
    - warp(domain-suffix(nflxso.net))
    - warp(domain-suffix(fast.com))
    - warp(domain-suffix(openai.com))
    - warp(domain-suffix(chatgpt.com))
    - warp(domain-suffix(oaistatic.com))
    - warp(domain-suffix(oaiusercontent.com))
    - warp(domain-suffix(anthropic.com))
    - warp(domain-suffix(claude.ai))
    - warp(domain-suffix(challenges.cloudflare.com))
    - direct(all)
EOF

chown -R hysteria:hysteria /etc/hysteria
chmod 600 /etc/hysteria/server.key /etc/hysteria/config.yaml
systemctl enable --now hysteria-server
```

### 步骤 4：部署 Xray (VLESS-Reality)

* **SNI 伪装域名选择铁律**：严禁使用 `gateway.icloud.com` 等 Apple 域名（Xray 会报被 GFW 深度探测拦截的警告）。**强烈推荐采用 `addons.mozilla.org` 或 `www.yahoo.com`**（支持原生 TLS 1.3 / HTTP/2，无任何安全警告）。

```bash
# 1. 官方脚本安装 Xray-core
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install

# 2. 生成密钥对
UUID=$(xray uuid)
RAW_KEY=$(xray x25519)
PRIVKEY=$(echo "$RAW_KEY" | grep "Private" | cut -d: -f2 | tr -d ' ')
PUBKEY=$(echo "$RAW_KEY" | grep "Public" | cut -d: -f2 | tr -d ' ')
SHORTID=$(openssl rand -hex 8)

cat << EOF > /usr/local/etc/xray/config.json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "${UUID}",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "addons.mozilla.org:443",
          "xver": 0,
          "serverNames": [
            "addons.mozilla.org"
          ],
          "privateKey": "${PRIVKEY}",
          "shortIds": [
            "${SHORTID}"
          ]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      }
    }
  ],
  "outbounds": [
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {
        "domainStrategy": "AsIs"
      }
    },
    {
      "tag": "warp",
      "protocol": "socks",
      "settings": {
        "servers": [
          {
            "address": "127.0.0.1",
            "port": 40000
          }
        ]
      }
    },
    {
      "tag": "block",
      "protocol": "blackhole",
      "settings": {}
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "outboundTag": "warp",
        "domain": [
          "geosite:netflix",
          "geosite:openai",
          "geosite:anthropic"
        ]
      },
      {
        "type": "field",
        "outboundTag": "direct",
        "network": "tcp,udp"
      }
    ]
  }
}
EOF

systemctl enable --now xray
```

---

### 步骤 5：交付客户端链接、二维码与防污染配置（防坑核心）

#### 1. 链接与二维码（手机便捷导入）
1. **Hysteria 2 链接**：
   `hysteria2://${HY_PASS}@${SERVER_IP}:38443/?sni=bing.com&insecure=1&obfs=salamander&obfs-password=${HY_OBFS}#My-VPS-Hy2`
2. **VLESS-Reality 链接**：
   `vless://${UUID}@${SERVER_IP}:443?security=reality&encryption=none&pbk=${PUBKEY}&headerType=none&fp=chrome&type=tcp&flow=xtls-rprx-vision&sni=addons.mozilla.org&sid=${SHORTID}#My-VPS-Reality`
3. **二维码生成**：使用 Python `qrcode` 生成 PNG 图片保存在产物目录，直接展示二维码供用户手机相机/代理工具扫描。

#### 2. Clash / FlClash 配置文件交付铁律（避免全盘断网事故）
* **【致命踩坑防范】**：**严禁交付只有单节点和简单规则的空白配置文件**！若配置中缺少境外 Fallback DNS，国内 DNS 解析 Google 会被污染；若缺少 `MATCH` 规则，开启 TUN 时 Clash 会把所有外网流量当作 `DIRECT` 国内直连，导致用户出现“节点延迟很低，但连不上谷歌，甚至原有节点也瘫痪”的假死故障！
* **交付要求**：
  1. **完备 DNS 链**：必须包含 `enhanced-mode: fake-ip`、国内 DoH (`dns.alidns.com`) 以及境外防污染 Fallback (`1.1.1.1` / `8.8.8.8`)。
  2. **完备白名单分流**：国内常见应用（微信、支付宝、淘宝、B站、银行、政府）必须标明 `DIRECT`。
  3. **必须包含 `MATCH` 规则**：未匹配流量统一投递到代理策略组。
  4. **老配置融合优先**：若用户客户端已有主力配置（如 `Hybrid-Optimal`），应优先将新节点直接注入已有配置的 `proxies` 和 `proxy-groups` 中，保证多节点互为备份。

---

## 终验 Checklist

- [ ] **原生网速**：`curl -I https://www.youtube.com` 返回 200，绝不走 WARP。
- [ ] **流媒体解锁**：WARP 出口访问 `netflix.com/title/80018499` 成功解锁。
- [ ] **AI 纯净度**：`chatgpt.com` 正常访问，人机验证无阻塞。
- [ ] **TCP BDP 状态**：`sysctl net.core.rmem_max` 返回 `67108864`（64MB）。
- [ ] **BBR 算法**：`sysctl net.ipv4.tcp_congestion_control` 确认输出 `bbr`。
- [ ] **TFO 与空闲慢启动**：`tcp_fastopen = 3`，`tcp_slow_start_after_idle = 0`。
- [ ] **客户端全绿验证**：通过本地核心代理测试 Google、YouTube、Cloudflare 204 全部返回 200/204。
