# PonyGate

A bypass gateway (旁路由) running on a **QNAP x86 NAS**: one Docker container with the mihomo core turns your NAS into a whole-LAN split-routing gateway. Blocked sites go through proxies, everything else goes direct — zero configuration on client devices.

跑在 **QNAP x86 NAS** 上的旁路由：一个 Docker 容器 + mihomo 内核，把 NAS 变成全屋的分流网关。被墙的走代理，其余直连，设备零配置。

---

## Architecture | 架构

```
LAN devices → Router (DHCP: gateway + DNS = NAS IP)
            → mihomo container on NAS (TUN transparent proxy)
               ├─ Rule-matched traffic (blocked sites / custom domains) → proxy nodes
               └─ Everything else → direct

局域网设备 → 路由器（DHCP 下发 网关+DNS = NAS IP）
           → NAS 上的 mihomo 容器（TUN 透明代理）
              ├─ 命中规则（被墙站点 / 自定义域名）→ 代理节点
              └─ 其余流量 → 直连
```

- Core | 内核：[MetaCubeX/mihomo](https://github.com/MetaCubeX/mihomo) (Clash Meta family — same core as ClashX / FlyingBird | 与 ClashX / FlyingBird 同款内核)
- Dashboard | 面板：[metacubexd](https://github.com/MetaCubeX/metacubexd) — switch nodes and inspect connections in your browser | 浏览器里切节点、看连接
- Subscriptions | 订阅：standard Clash subscription URLs via `proxy-providers`, auto-updated daily | 标准 Clash 订阅链接直接可用，每天自动更新

## Proxy Group Design | 代理分组设计

Three groups — the core of this setup | 配置里有三个组，是这套方案的核心：

| Group | Type | Purpose |
|---|---|---|
| `PROXY` | select | Main egress. Points to `AUTO` by default; pin a specific node in the dashboard anytime. 总出口，默认指向 `AUTO`，也可手动钉节点 |
| `AUTO` | url-test | Latency test every 5 min, always uses the fastest node. 每 5 分钟测速，自动用延迟最低的节点 |
| `AI` | select | **Dedicated lane for overseas AI services**: US/Singapore nodes + self-hosted VPS only. **海外 AI 服务专用通道**：只含美国/新加坡节点 + 自建 VPS |

### Why a dedicated AI group | 为什么 AI 要单独分组

ChatGPT / Grok / Gemini / Claude have two special requirements | 这类服务有两个特殊要求：

1. **Frequent egress-IP changes trigger risk control** — `AUTO` may hop nodes every few minutes; fine for normal sites, but AI services treat IP churn as suspicious logins, which can lead to bans. `AI` is a manual-select group: pinned until you change it.
   **忌讳频繁换出口 IP**——`AUTO` 每几分钟可能跳节点，AI 服务会把 IP 抖动视为异常登录，触发风控甚至封号。所以 `AI` 组是纯手动选择，选定就固定。
2. **Region requirements** — Hong Kong nodes are often restricted or degraded by AI providers, so the `AI` group uses a `filter` regex to keep only US/Singapore nodes, plus a self-hosted VPS as fallback.
   **对节点地区有要求**——香港节点常被 AI 服务限制或降质，所以 `AI` 组用 `filter` 正则只保留美国/新加坡节点，再加自建 VPS 兜底。

Rules point AI domains to this group. Note: Gemini lives under Google domains, so it **must precede `GEOSITE,google`** | 规则上把 AI 域名指向这个组（Gemini 是 Google 域名，**必须排在 `GEOSITE,google` 之前**）：

```yaml
- GEOSITE,openai,AI
- DOMAIN-SUFFIX,x.ai,AI
- DOMAIN-SUFFIX,grok.com,AI
- DOMAIN-SUFFIX,gemini.google.com,AI      # must precede GEOSITE,google
- DOMAIN-SUFFIX,generativelanguage.googleapis.com,AI
- GEOSITE,anthropic,AI
```

Also recommended: use `exclude-filter` at the provider level to drop **plaintext (non-TLS) nodes** and **MITM nodes** | 另外建议在订阅层面用 `exclude-filter` 剔除**无 TLS 的明文传输节点**和**名字带 MITM 的节点**。

## QNAP x86 Pitfalls (all solved in this config) | QNAP x86 特有的坑（都已在本配置中解决）

Running mihomo TUN on QTS hits problems you won't see on generic Linux — this config's main value is here | 在 QTS 上跑 mihomo TUN 会遇到几个通用 Linux 上碰不到的问题，这份配置的价值主要在这：

1. **QTS kernel has IPv6 disabled** → mihomo TUN fails with `configure tun interface: permission denied`. Fix: explicitly empty `inet6-address: []` and `inet6-route-address: []`.
   **QTS 内核禁用 IPv6** → TUN 报权限错误。解法：显式留空 v6 地址与路由。
2. **mihomo's own DNS queries get hijacked by its own TUN (infinite loop)** → subscription fetch fails with `dns resolve failed`. Fix: use DoH (TCP 443) for all upstreams, bypassing the port-53 hijack; put subscription API and node domains in `fake-ip-filter`.
   **mihomo 自身 DNS 被自己的 TUN 劫持形成死循环** → 订阅拉取失败。解法：上游全部 DoH，订阅/节点域名加 `fake-ip-filter`。
3. **Client DNS must land on mihomo's port 53** → QTS dnsmasq occupies 53 on loopback/bridge addresses. Fix: bind `listen` to the NAS LAN IP (e.g. `192.168.1.2:53`).
   **客户端 DNS 必须落在 mihomo 的 53 端口** → QTS dnsmasq 占用回环/网桥 53。解法：`listen` 绑定 NAS 局域网 IP。
4. **Docker Hub / GitHub blocked** → pull images via a registry mirror (e.g. `docker.1ms.run`); import GEO databases and dashboard via a GitHub proxy mirror; point `geox-url` at the mirror for daily auto-updates.
   **Docker Hub / GitHub 直连被墙** → 镜像站拉镜像、GEO 数据库离线导入、`geox-url` 指向镜像实现每日自动更新。
5. **Container needs `privileged: true`** — `NET_ADMIN` alone is not enough on QNAP's Docker.
   **容器需要 `privileged: true`**——QNAP 的 Docker 只给 `NET_ADMIN` 不够。

Server-side hardening | 服务端加固：`sniffer` (SNI sniffing, recovers domains from devices using their own DoH | SNI 嗅探，应对设备私自 DoH) + `DST-PORT,853,REJECT` (block DoT, force local DNS | 封 DoT，强制本地 DNS)。

## Deployment | 部署

```bash
# 1. Download GEO databases (if GitHub is unreachable from the NAS, fetch on another machine and upload)
#    geoip.metadb / geosite.dat → config/
# 2. Copy the example config and fill in your subscription URLs
cp config/config.example.yaml config/config.yaml
# 3. Edit config.yaml: subscription URLs ×2, secret, and the NAS LAN IP in dns.listen
# 4. Start
docker compose up -d
```

Then in your router's DHCP settings, set **gateway and DNS** to the NAS IP (do NOT put an ISP DNS as secondary; also disable router IPv6 to stop RA from advertising ISP v6 DNS, which would bypass the gateway).

然后在路由器 DHCP 设置里把**网关和 DNS** 都改为 NAS IP（备用 DNS 不要填运营商 DNS；建议同时关闭路由器 IPv6，避免 RA 下发运营商 v6 DNS 形成旁路）。

Dashboard | 管理面板：`http://NAS_IP:9090/ui` (password = `secret` in config | 密码为配置里的 `secret`)。

## Files | 文件说明

- `docker-compose.yml` — container definition (host network + privileged + /dev/net/tun | 容器定义)
- `config/config.example.yaml` — fully commented config template; solutions to all QNAP pitfalls are in the comments | 完整注释的配置模板，QNAP 坑位的解法都写在注释里

## Disclaimer | 免责

For network research and learning only. Make sure your proxy usage complies with local laws and service terms.

本项目仅供网络技术学习研究。请确保你使用的代理服务符合当地法律法规和服务条款。
