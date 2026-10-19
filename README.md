# VPS 科学上网全流程教程：MTProto + Xray(Reality) + Shadowsocks + Hysteria2

从 SSH 连接一台海外 VPS 开始，一步步搭建一套「Telegram 专用 + 通用代理 + 抗丢包」的完整方案。

本教程最终会在服务器上跑齐以下服务：

| 服务 | 协议 | 端口 | 用途 | 抗封锁/抗丢包 |
|---|---|---|---|---|
| **MTProto** | mtproto (Fake TLS) | 443/tcp | Telegram 专用 | 强（伪装 TLS） |
| **Xray** | VLESS + Reality | 8443/tcp | 通用代理 | 强（真实证书伪装） |
| **Xray** | Shadowsocks | 8388/tcp | 兼容老客户端 | 一般 |
| **Hysteria2** | hysteria2 (QUIC/UDP) | 443/udp | 通用代理，**抗丢包** | 最强（无视丢包） |
| **订阅服务** | nginx (http) | 8080/tcp | 一键导入 Clash | — |

> 说明：教程中所有敏感信息（IP、密码、密钥、UUID）均用占位符 `<...>` 表示，请替换成你自己的。
> 请妥善保管密钥，不要外泄；订阅地址也不要公开分享（否则别人能白嫖你的节点）。

---

## 一、参考项目

- MTProxy 一键脚本 / Docker 镜像（本项目）：<https://github.com/ellermister/mtproxy>
- MTProto 源项目 mtg（Golang 版）：<https://github.com/9seconds/mtg>
- Xray-core 官方：<https://github.com/XTLS/Xray-core>
- Hysteria2 官方：<https://github.com/apernet/hysteria>
- Clash Meta for Android（安卓客户端）：<https://github.com/MetaCubeX/ClashMetaForAndroid>
- Clash Verge Rev（桌面客户端）：<https://github.com/clash-verge-rev/clash-verge-rev>

---

## 二、准备工作

- 一台**海外** VPS（推荐 Ubuntu 22.04 / 24.04，最低 1 核 / 512MB 内存）
  - 关键：服务器必须能**直连 Telegram 与境外网络**。中国大陆境内的机器做不了这些代理（无法访问 Telegram）。
  - 线路建议：普通美西线路便宜但晚高峰丢包高（抖动大）；预算够可选 CN2 GIA / 联通 9929 / 移动 CMIN2 等优化线路。
- SSH 客户端：Windows 用 PowerShell / Git Bash / Xshell；macOS / Linux 用终端。
- 服务器信息（下单后商家提供）：
  - IP 地址：`<服务器IP>`
  - SSH 端口：`<SSH端口>`（默认 22，部分商家不同）
  - 用户名：`root`，密码：`<root密码>`

> 大部分海外商家（如 RackNerd）默认不封锁端口，无需额外配置防火墙。
> 若商家有安全组 / 防火墙面板，记得放行用到的端口（TCP 443、8443、8388、8080；UDP 443）。

---

## 三、连接服务器

```bash
ssh root@<服务器IP> -p <SSH端口>
```

首次连接提示 `Are you sure you want to continue connecting?` 输入 `yes`，再输入密码。

登录后确认系统信息：

```bash
hostname
cat /etc/os-release | head -3
uname -m            # 应显示 x86_64
nproc               # CPU 核心数
free -m | head -2   # 内存
df -h /             # 磁盘
```

### 验证能否访问 Telegram（很重要）

```bash
# 返回 200 表示可访问
curl -s -o /dev/null -w '%{http_code}\n' https://core.telegram.org/getProxySecret

# DC IP 连通性（返回 OK 即可）
for ip in 149.154.167.51 149.154.175.50 91.108.56.130; do
  echo -n "$ip "
  timeout 6 bash -c "echo > /dev/tcp/$ip/443" 2>/dev/null && echo OK || echo FAIL
done
```

若这里不通，说明机器访问不到 Telegram，请换一台**海外**机器。

---

## 四、安装 Docker

```bash
curl -fsSL https://get.docker.com -o /tmp/get-docker.sh && sh /tmp/get-docker.sh
systemctl enable --now docker
docker --version
```

> **国内加速（可选）**：拉取镜像慢时配置镜像加速：
>
> ```bash
> mkdir -p /etc/docker
> cat > /etc/docker/daemon.json <<'EOF'
> {
>   "registry-mirrors": [
>     "https://docker.m.daocloud.io",
>     "https://docker.1panel.live"
>   ],
>   "log-driver": "json-file",
>   "log-opts": { "max-size": "10m", "max-file": "3" }
> }
> EOF
> systemctl restart docker
> ```

---

## 五、搭建 Telegram MTProto 代理（Fake TLS）

使用 `ellermister/mtproxy` 镜像，一条命令部署。

### 1. 启动容器

```bash
docker run -d \
  --name mtproxy \
  --restart=always \
  -e domain="cloudflare.com" \
  -e ip_white_list="OFF" \
  -e provider=2 \
  -p 443:443 \
  -p 80:80 \
  ellermister/mtproxy
```

参数说明：

| 参数 | 说明 |
|---|---|
| `domain` | Fake TLS 伪装域名，默认 `cloudflare.com` |
| `ip_white_list` | `OFF` 关闭白名单；`IPSEG` 只放行登记过的 /24 网段；`IP` 只放行单个 IP |
| `provider` | `1` 官方 MTProxy C 程序；**`2` mtg（推荐，支持多平台）**；`3` mtprotoproxy |
| `secret` | 32 位十六进制密钥，不填自动生成（从日志看） |
| `-p 443:443` | 宿主机 443 → 容器 443 |

### 2. 查看代理链接

```bash
docker logs mtproxy 2>&1 | tail -20
```

日志会打印 **Secret** 和 `tg://` / `https://t.me/proxy` 一键链接。

> **端口映射很重要**：容器内部 MTProxy 实际监听 `8443`，对外端口取决于 `-p` 写法。
> - `-p 443:443` → 客户端用 **443**
> - `-p 8443:443` → 客户端用 **8443**
>
> 日志显示的是容器内端口 8443，请以你映射出来的**宿主机端口**为准。

### 3. 使用

把链接粘贴到 Telegram → 设置 → 数据与存储 → 代理 → 添加代理。

---

## 六、搭建 Xray（VLESS + Reality 与 Shadowsocks 双入站）

Reality 不需要域名和证书，探测时会看到真实网站证书，抗封锁强；
再加一个 Shadowsocks 入站，兼容不支持 VLESS 的老客户端。

### 1. 拉取镜像并生成密钥

```bash
docker pull ghcr.io/xtls/xray-core:latest

# UUID（VLESS 需要）
cat /proc/sys/kernel/random/uuid

# Reality 密钥对（记住 PrivateKey 和 PublicKey）
docker run --rm ghcr.io/xtls/xray-core:latest x25519

# shortId（十六进制 8 字节）
openssl rand -hex 8

# Shadowsocks 密码
openssl rand -hex 16
```

### 2. 写配置文件 `/etc/xray/config.json`

```bash
mkdir -p /etc/xray
cat > /etc/xray/config.json <<'EOF'
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 8443,
      "protocol": "vless",
      "settings": {
        "clients": [
          { "id": "<你的UUID>", "flow": "xtls-rprx-vision" }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.microsoft.com:443",
          "xver": 0,
          "serverNames": ["www.microsoft.com"],
          "privateKey": "<你的PrivateKey>",
          "shortIds": ["<你的shortId>"]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    },
    {
      "listen": "0.0.0.0",
      "port": 8388,
      "protocol": "shadowsocks",
      "settings": {
        "method": "chacha20-ietf-poly1305",
        "password": "<你的SS密码>",
        "network": "tcp,udp"
      }
    }
  ],
  "outbounds": [
    { "protocol": "freedom", "tag": "direct" },
    { "protocol": "blackhole", "tag": "block" }
  ],
  "routing": { "domainStrategy": "AsIs", "rules": [] }
}
EOF
```

> `dest` / `serverNames` 是伪装目标，需为支持 TLS1.3 的知名站点，如 `www.microsoft.com`、`www.cloudflare.com`。

### 3. 启动容器

```bash
docker run -d \
  --name xray \
  --restart=always \
  -v /etc/xray/config.json:/etc/xray/config.json:ro \
  -p 8443:8443 \
  -p 8388:8388 \
  ghcr.io/xtls/xray-core:latest run -config /etc/xray/config.json
```

### 4. 验证

```bash
docker ps
docker logs xray 2>&1 | tail -10

# 从本机验证 Reality（应返回伪装的真实网站证书）
echo | openssl s_client -connect <服务器IP>:8443 -servername www.microsoft.com 2>&1 \
  | grep -iE "subject=|issuer=|Verify return code"
```

> 日志可能出现 `REALITY: Listening on non-443 ports may get your IP blocked by the GFW` 与
> `Shadowsocks ... deprecated` 的警告，属正常提示，不影响使用。

---

## 七、搭建 Hysteria2（抗丢包神器）

Hysteria2 基于 QUIC/UDP，自带拥塞控制，**在丢包线路上抖动远小于 TCP 协议**，非常适合国内到美西这种高丢包线路。

### 1. 生成自签证书与密码

```bash
mkdir -p /etc/hysteria
openssl req -x509 -newkey rsa:2048 \
  -keyout /etc/hysteria/server.key -out /etc/hysteria/server.crt \
  -days 3650 -nodes -subj '/CN=www.microsoft.com'

openssl rand -hex 12    # 认证密码
openssl rand -hex 12    # obfs 混淆密码
```

### 2. 写配置 `/etc/hysteria/config.yaml`

```bash
cat > /etc/hysteria/config.yaml <<'EOF'
listen: :443

tls:
  cert: /etc/hysteria/server.crt
  key: /etc/hysteria/server.key

auth:
  type: password
  password: "<你的认证密码>"

obfs:
  type: salamander
  salamander:
    password: "<你的obfs密码>"

masquerade:
  type: proxy
  proxy:
    url: https://www.microsoft.com
    rewriteHost: true
EOF
```

### 3. 启动容器（注意是 UDP）

```bash
docker run -d \
  --name hysteria \
  --restart=always \
  -v /etc/hysteria:/etc/hysteria \
  -p 443:443/udp \
  tobyxdd/hysteria:latest server -c /etc/hysteria/config.yaml

docker logs hysteria 2>&1 | tail -5   # 应看到 server up and running
```

> MTProto 占用 TCP 443，Hysteria2 占用 **UDP 443**，两者不冲突，可共存。

---

## 八、优化：开启 BBR

高丢包线路建议开启 BBR，提升 TCP 协议（Reality / SS）的稳定性：

```bash
modprobe tcp_bbr
echo 'bbr' > /etc/modules-load.d/bbr.conf
printf 'net.core.default_qdisc=fq\nnet.ipv4.tcp_congestion_control=bbr\n' > /etc/sysctl.d/99-bbr.conf
sysctl --system

# 确认已生效
sysctl net.ipv4.tcp_congestion_control      # 应为 bbr
sysctl net.ipv4.tcp_available_congestion_control
```

---

## 九、搭建 http 订阅服务（可选，方便客户端一键导入）

用 nginx 容器托管 Clash 配置文件，客户端用 URL 直接导入。

### 1. 准备配置

```bash
mkdir -p /etc/clashsite
# /etc/clashsite/clash.yaml  → 完整 Clash 配置（含所有节点、策略组、规则）
# /etc/clashsite/clash-ss.yaml → 仅 Shadowsocks（老客户端用）
```

一个可直接用的 `clash.yaml` 示例（把占位符换成你的值）：

```yaml
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info
ipv6: false

dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver: [223.5.5.5, 119.29.29.29]

proxies:
  - name: "Hysteria2"
    type: hysteria2
    server: <服务器IP>
    port: 443
    password: "<Hysteria2密码>"
    obfs: salamander
    obfs-password: "<obfs密码>"
    sni: www.microsoft.com
    skip-cert-verify: true
    alpn: [h3]
    up: "100 Mbps"
    down: "500 Mbps"
  - name: "Reality"
    type: vless
    server: <服务器IP>
    port: 8443
    uuid: <你的UUID>
    network: tcp
    tls: true
    udp: true
    flow: xtls-rprx-vision
    servername: www.microsoft.com
    client-fingerprint: chrome
    reality-opts:
      public-key: <你的PublicKey>
      short-id: "<你的shortId>"
  - name: "Shadowsocks"
    type: ss
    server: <服务器IP>
    port: 8388
    cipher: chacha20-ietf-poly1305
    password: "<SS密码>"
    udp: true

proxy-groups:
  - name: PROXY
    type: select
    proxies: [Hysteria2, Reality, Shadowsocks, DIRECT]

rules:
  - GEOIP,LAN,DIRECT,no-resolve
  - GEOIP,CN,DIRECT
  - MATCH,PROXY
```

### 2. 启动订阅服务

```bash
docker run -d \
  --name sub \
  --restart=always \
  -p 8080:80 \
  -v /etc/clashsite:/usr/share/nginx/html:ro \
  nginx:alpine
```

### 3. 访问地址

```
http://<服务器IP>:8080/clash.yaml      # 完整（Meta 内核）
http://<服务器IP>:8080/clash-ss.yaml   # 仅 SS（老内核）
```

> 安全提醒：该地址是公开的，知道的人都能拉到你的节点。
> 建议改成**随机路径**或加 **http basic 认证**。

---

## 十、客户端使用

### 客户端选择（关键）

关键是**内核**，不是壳子：只有 **Mihomo / Clash.Meta 内核**才支持 Reality 和 Hysteria2。

| 客户端 | 平台 | 内核 | 支持 Reality / Hysteria2 |
|---|---|---|---|
| **Clash Meta for Android** | 安卓 | Mihomo | ✅ |
| **FlClash** | 安卓 / Win / macOS / Linux | Mihomo | ✅ |
| **Clash Verge Rev** | Win / macOS / Linux | Mihomo | ✅ |
| **Mihomo Party** | Win / macOS / Linux | Mihomo | ✅ |
| **ClashX Meta** | macOS | Mihomo | ✅ |
| Clash for Windows（CFW，已停更） | Win | Premium | ❌ 仅 SS/VMess/Trojan |
| Clash for Android（kr328 原版） | 安卓 | Premium | ❌ 仅 SS/VMess/Trojan |

> 报错 `unsupport proxy type: vless` 说明内核不支持 VLESS，请换 Mihomo/Meta 内核客户端。
> 确认方法：App / 软件「关于 / 版本 / 内核」显示 `Clash.Meta` / `Mihomo` 即支持，
> 显示 `Premium` 则不支持。

### 软件下载（Mihomo / Clash.Meta 内核）

**电脑版（Windows / macOS / Linux）**
- **Clash Verge Rev**（全平台，桌面首选）
  <https://github.com/clash-verge-rev/clash-verge-rev/releases>
- **Mihomo Party**（界面现代，上手简单，全平台）
  <https://github.com/mihomo-party-org/mihomo-party/releases>
- **Clash Nyanpasu**（老牌稳定，全平台）
  <https://github.com/libnyanpasu/clash-nyanpasu/releases>
- **ClashX Meta**（仅 macOS）
  <https://github.com/MetaCubeX/ClashX.Meta/releases>

**安卓**
- **Clash Meta for Android**（推荐）
  <https://github.com/MetaCubeX/ClashMetaForAndroid/releases>
  选 `arm64-v8a` 的 APK（近几年手机基本都是 arm64）。
- **FlClash**（安卓 / 桌面通用，界面现代）
  <https://github.com/chen08209/FlClash/releases>

> 文件名怎么选：
> - 安卓：`arm64-v8a` 最通用
> - Windows：`x64`（或 `amd64`）
> - macOS：Intel 选 `x64`/`amd64`，Apple 芯片（M 系列）选 `arm64`/`aarch64`
> - Linux：`amd64`/`x86_64` 或 AppImage
>
> 打不开 GitHub 时，可用客户端的「镜像站 / 加速」或搜索引擎找对应安装包。

> **为什么电脑上别用 Clash for Windows**：CFW 已停更下架，内核是闭源 **Premium**，
> 不支持 VLESS / Reality / Hysteria2，只能用 Shadowsocks。建议换上面的 Mihomo 客户端。

### 导入方式

1. **订阅导入**（推荐）：新建配置 → 从 URL 导入 → 粘贴 `http://<服务器IP>:8080/clash.yaml`
2. **手动添加**：按上面的 `proxies` 字段逐个填

选策略组 `PROXY` → 优先选 **Hysteria2**。

### Telegram 怎么走

- 直接用 `tg://` 的 **MTProto 代理**（设置 → 数据与存储 → 代理）
- 或让 Clash 全局/规则模式为 Telegram 分流，走 Hysteria2 节点

---

## 十一、延迟抖动大怎么办

**现象**：延迟一会儿 500ms、一会儿 800ms、一会儿 1~2 秒。

**排查**：

```bash
# 从国内机器 ping，看真实延迟与丢包
ping -c 50 -i 0.2 <服务器IP>
```

- 如果**原始 ping 稳定**（如 145ms，波动几毫秒）但**丢包高** → 抖动来自 **TCP 重传**（丢一个包要等 200/400/800ms… 重传）。
- 根因：**线路拥堵**（普通美西线路晚高峰丢包可达 12~40%），不是服务器配置问题。

**解决（按性价比）**：

| 方案 | 效果 | 代价 |
|---|---|---|
| 开 BBR（见第八节） | 缓解 TCP 拥塞 | 免费 |
| **换 Hysteria2（UDP）** | **最好**，QUIC 抗丢包 | 免费 |
| 加中转（国内→香港/日本→美西） | 明显改善，通用 | 多一台机器 |
| 换 CN2 GIA / 9929 / CMIN2 优化线路 | 根治 | 较贵 |

---

## 十二、日常维护

```bash
# 查看所有容器
docker ps -a

# 日志
docker logs -f mtproxy
docker logs -f xray
docker logs -f hysteria
docker logs -f sub

# 重启 / 停止 / 删除
docker restart xray
docker stop hysteria
docker rm -f mtproxy
```

- 所有容器都设了 `--restart=always`，服务器重启自动拉起；Docker 也已开机自启。
- 记录好 **Secret / UUID / PublicKey / shortId / 各类密码**，客户端配置需要用到。

---

## 十三、常见问题

**1. `docker pull` 超时**
配置国内镜像加速（见第四节），或换时间段重试。

**2. MTProto 链接显示 8443 但连不上**
8443 是容器内端口，客户端要用 `-p` 映射出的宿主机端口（如 `-p 443:443` 就用 443）。

**3. 客户端导入报 `unsupport proxy type: vless`**
内核不支持 VLESS。换 **Clash Meta for Android / Clash Verge Rev**（Mihomo 内核），或改用仅 SS 的订阅 `clash-ss.yaml`。

**4. 服务器访问不到 Telegram**
`core.telegram.org` 返回非 200、DC IP 全 FAIL → 机器在墙内或线路被限，**必须换海外机器**。

**5. 移动网络连不上**
部分机房（如 ColoCrossing 上游）被中国移动拉黑，移动宽带用户无法使用，电信/联通正常。

**6. 延迟抖动大**
见第十一节，优先上 Hysteria2。

**7. 想更安全 / 防白嫖**
MTProto 设 `ip_white_list=IPSEG`；订阅地址改随机路径或加 basic 认证；Xray/Hysteria2 用随机高位端口。

---

## 十四、参考链接

- MTProxy 一键脚本 / Docker 镜像：<https://github.com/ellermister/mtproxy>
- MTProto 源项目 mtg：<https://github.com/9seconds/mtg>
- Xray-core：<https://github.com/XTLS/Xray-core>
- Hysteria2：<https://github.com/apernet/hysteria>
- Clash Meta for Android：<https://github.com/MetaCubeX/ClashMetaForAndroid>
- FlClash：<https://github.com/chen08209/FlClash>
- Clash Verge Rev：<https://github.com/clash-verge-rev/clash-verge-rev>
- Mihomo Party：<https://github.com/mihomo-party-org/mihomo-party>
- Clash Nyanpasu：<https://github.com/libnyanpasu/clash-nyanpasu>
- ClashX Meta（macOS）：<https://github.com/MetaCubeX/ClashX.Meta>
