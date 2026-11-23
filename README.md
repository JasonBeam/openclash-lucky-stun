# OpenClash + Lucky STUN 内网穿透配置指南

一套面向普通 OpenWrt / iStoreOS 用户的 **IPv4 TCP STUN** 内网穿透方案：路由器跑 OpenClash 透明代理，NAS 上跑 Lucky 打洞，再通过 UPnP（或手动端口转发）把服务暴露到公网。

包含**手动配置教程**、**AI Agent 辅助教程**，以及排查思路与常见问题。

> 适用前提：路由器使用 **Firewall4 / nftables**，OpenClash 为 **Fake-IP + REDIRECT（非 TUN）** 模式，Lucky 支持 IPv4-TCP 穿透。
> 这不是跨运营商、跨设备都成立的通用方案。换环境请先按第 4 节核对，不要直接复制地址和端口。

---

## 1. 这个项目解决什么问题

很多家庭网络的出口没有可直接使用的公网 IPv4（或处于运营商 NAT 之后），但又有从外部访问 NAS 服务的需求，例如：

- 在外面看 NAS 上的 Emby / Jellyfin；
- 访问 NAS 上的下载器、网盘、管理面板；
- 给朋友临时开一个服务入口。

同时路由器上还跑着 OpenClash 做透明代理。这类组合最常见的故障是：

```text
Lucky 面板显示“已获取公网地址和端口”  →  但从外网怎么都连不上
关掉 OpenClash 就正常                  →  一开就断
换端口、加 DMZ、放行防火墙都没用
```

本项目给出的方案是：**让 Lucky 指定源端口的 TCP 流量在防火墙层绕过 OpenClash 的接管链**，再让 UPnP 把公网映射落到 Lucky 的监听端口上。

---

## 2. 典型使用场景

```text
外网客户端
   │  IPv4 TCP
   ▼
运营商 NAT（公网地址:动态端口）
   │
   ▼
主路由 OpenWrt / iStoreOS  ── OpenClash（Fake-IP、REDIRECT、非 TUN）
   │  UPnP 端口映射
   ▼
NAS（Lucky，host 网络，监听 35080）
   │  Lucky 内置转发
   ▼
后端服务（如 Emby 127.0.0.1:8096）
```

两条路径都要通：

1. **出站**：NAS 的 Lucky 向 STUN 服务器探测，从运营商 NAT 拿到一个公网映射；
2. **入站**：外部访问这个公网地址和端口，经运营商 NAT → 路由器 UPnP 映射 → NAS 监听端口 → 后端服务。

任何一段断掉，表现都是“连不上”，所以排查必须按顺序来（见第 7 节）。

---

## 3. 核心原理：DIRECT 不等于绕过防火墙

很多人第一反应是“在 Clash 里把这个域名设成 DIRECT 就行了”。实测下来**这样不行**，原因有三层：

1. **接管发生在策略之前。** 流量先被防火墙规则重定向进 Clash 内核，DIRECT 只是“进了内核之后选择直连”，不代表这条连接没被接管。
2. **内核会用自己的 socket 重新拨号。** 源端口、目标地址都可能变化，STUN 依赖的 NAT 会话（五元组）就和探测时对不上了。
3. **SNI 嗅探会改写目标。** 本环境开启了 override-destination / force-dns-mapping，内核按域名重新解析后拨号，客户端指定的 IP 会被丢弃。

因此有效做法是：**在 OpenClash 的 nftables 接管链中，对「NAS 源 IP + TCP 源端口」提前 return**，让这部分流量根本不进入内核。

> 机制分析详见 [技术说明](docs/reference.md)。如果你只想知道怎么做，直接看 [手动配置教程](docs/manual-setup.md)。

---

## 4. 环境要求

| 组件 | 要求 | 怎么确认 |
| --- | --- | --- |
| 路由器系统 | OpenWrt / iStoreOS，**Firewall4 + nftables** | 路由器上执行 `nft -v`，能列出 `inet fw4` 表 |
| OpenClash | **Fake-IP 模式、REDIRECT 接管、非 TUN** | 路由器上执行 `nft -a list chain inet fw4 openclash`，能看到 redirect 到内核端口 |
| 自定义脚本 | 能写入 OpenClash 的防火墙自定义脚本 | 路由器上确认 `/etc/openclash/custom/openclash_custom_firewall_rules.sh` 存在 |
| Lucky | 支持 STUN 的 IPv4-TCP 规则；Docker 场景需 **host 网络** | NAS 上 `docker ps` 看 Lucky 是否使用 host 网络 |
| 后端服务 | 在 NAS 本机可访问 | NAS 上 `curl -I http://127.0.0.1:8096/` |
| 网络条件 | 满足 Lucky 所要求的 **NAT1** 条件，并具备端口转发能力（UPnP 或手动转发） | 以 Lucky 实际探测结果为准 |

**关于 NAT1**：使用 Lucky STUN 前，需要确认网络满足 Lucky 所要求的 NAT1 条件，同时具备必要的端口转发能力。符合 NAT1 条件不代表一定能够成功穿透，还需要实际验证。

> 不同 OpenClash 版本、不同接管模式（TUN / TPROXY）的链名和规则位置不同，本方案不保证适用。

---

## 5. 快速开始（最短路径）

以下是最短操作顺序，细节见 [手动配置教程](docs/manual-setup.md)。

| 步骤 | 在哪台设备 | 做什么 |
| --- | --- | --- |
| 1 | 路由器 | 确认 Firewall4 / nftables 与 OpenClash 的 REDIRECT 接管链存在 |
| 2 | NAS | 固定局域网 IP，确认 Emby 的 8096 端口本机可访问 |
| 3 | NAS（Lucky 面板） | 新建 **IPv4-TCP** 穿透规则：监听 35080，目标 127.0.0.1:8096 |
| 4 | NAS（Lucky 面板） | 保持内置转发开启，确认 NAT-PMP / UPnP 设置与部署方式匹配 |
| 5 | 路由器 | 确认 UPnP 已为 NAS 建立 35080 的映射 |
| 6 | 路由器 | 在 OpenClash 自定义防火墙脚本中，对 NAS IP + tcp sport 35080 提前 return |
| 7 | 外部网络 | 用手机蜂窝网络访问 Lucky 显示的公网地址与端口 |

绕过规则的完整写法（**在路由器上执行**，先备份）：

```sh
# 路由器
cp /etc/openclash/custom/openclash_custom_firewall_rules.sh \
   /etc/openclash/custom/openclash_custom_firewall_rules.sh.bak
```

```sh
# 路由器：把下面这段加到脚本里已有的 exit 0 之前
if ! nft list chain inet fw4 openclash 2>/dev/null | grep -q 'lucky_stun_35080'; then
    nft insert rule inet fw4 openclash \
        ip saddr 192.168.10.10 tcp sport 35080 \
        counter return comment 'lucky_stun_35080'
fi
```

> `sport` 是 **NAS 出站时使用的本地端口**（即 Lucky 监听端口），不是公网映射端口，也不是 STUN 服务器的端口。写成 `dport` 一定无效。

---

## 6. 两种使用方式

### 6.1 手动配置

适合想搞清楚每一步在做什么的用户：[《从零开始：OpenClash + Lucky STUN 手动配置教程》](docs/manual-setup.md)

包含：条件检查 → NAS/Emby 准备 → Lucky 建规则 → 路由器 UPnP → 检查接管链 → 加绕过规则 → 分步验证 → 回退。

### 6.2 AI Agent 辅助

适合已经把路由器、NAS 的 SSH 配好，想让 Agent 帮忙检测和落配置的用户：[《使用 Codex / DSH Agent 辅助配置 OpenClash 与 Lucky STUN》](docs/agent-guide.md)

包含四段可直接复制的 Prompt：**检测 → 配置 → 验收 → 回退**。所有写操作都要求先备份、经用户确认，并且没有真实外网测试时必须标“未验证”。

---

## 7. 配置验证

按顺序检查，每一步只回答一个问题，**不要跳步**。

| # | 检查项 | 设备 | 方法 | 通过说明什么 | 不能证明什么 |
| --- | --- | --- | --- | --- | --- |
| 1 | 后端活着 | NAS | `curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8096/` | Emby 正常响应 | 外部可达 |
| 2 | Lucky 拿到映射 | NAS（面板） | 查看规则的“当前公网地址”与“映射端口” | 出站打洞成功 | 入站放行 |
| 3 | 入站映射存在 | 路由器 | `nft list chain inet fw4 upnp_forward`（输出中找 35080） | 有指向 NAS 的转发 | 运营商放行 |
| 4 | 绕过规则位置正确 | 路由器 | `nft -a list chain inet fw4 openclash`（输出中找 lucky_stun） | return 序号在 REDIRECT 之前 | 流量真的命中 |
| 5 | 规则被命中 | 路由器 | 重新触发 Lucky 后看 counter 是否增长 | 有流量走了绕过分支 | 外网成功 |
| 6 | 外部可达 | 手机蜂窝网络 / 第三方检测 | `curl -4 -m 10 -v 'http://<公网IP>:<映射端口>/'` | 运营商放行、链路通 | 业务功能与长期稳定 |
| 7 | 到达正确后端 | 外部 + NAS | 页面内容/标题是目标服务；服务端日志或 `conntrack` 有连接 | 映射与后端地址正确 | 登录、播放、并发都正常 |

**常见误判**：

- counter 增长 ≠ 穿透成功，只说明有流量命中这条规则；
- 端口能连上 ≠ 服务可用，可能只是 Lucky 或反代返回了错误页；
- 单次成功 ≠ 长期可用，公网地址和端口会变（见 [技术说明](docs/reference.md)）。

---

## 8. 常见问题

**Q：在 Clash 里给这个域名/端口设了 DIRECT，为什么还是不行？**
因为接管发生在策略之前，内核还会用自己的 socket 重新拨号。需要在防火墙链里提前 return。

**Q：Lucky 显示“已获取公网地址”，是不是就通了？**
不是。那只说明出站打洞成功；入站还需要 UPnP / 端口转发，并且运营商要放行该映射端口。

**Q：为什么 443 端口做 STUN 容易失败？**
OpenClash 开启 `disable_udp_quic` 时会在 nftables 里 reject UDP 443；同时 Clash 规则里常见的 `DST-PORT,443` 会把流量送进代理组。STUN 探测包可能到不了目标。本方案只讨论 TCP 穿透，避开这个问题。

**Q：能不能用 DMZ 一劳永逸？**
不建议。DMZ 只是把 WAN 入站全部 DNAT 给 NAS，解决不了“运营商不放行固定端口”的问题，还会把整台 NAS 暴露到公网。

**Q：公网端口老是变怎么办？**
这是 STUN 方案的固有特性，普通 DNS A 记录也不携带端口。可以缩短 Lucky 的重测周期，或改用支持端口的入口方案（如隧道服务）。详见 [技术说明](docs/reference.md)。

更多问题见 [技术说明与常见问题](docs/reference.md)。

---

## 9. 安全须知

- **面板与服务分别鉴权**：Lucky 面板、Emby、下载器等都要有独立认证；对外入口建议 HTTPS。
- **UPnP 是信任边界**：任何获得权限的设备都能请求端口映射，定期核对 `upnp_forward` 的实际内容，只保留必要端口。
- **不要长期开放 DMZ**：等于把整台 NAS 暴露在 WAN 上。
- **发布截图和日志前脱敏**：隐藏公网 IP、端口、域名、账号、Token、订阅链接。
- **升级后复核**：路由器固件、OpenClash、Lucky 升级后，nftables 链名、规则顺序、脚本调用方式都可能变化。

---

## 10. 详细文档导航

| 文档 | 内容 |
| --- | --- |
| [手动配置教程](docs/manual-setup.md) | 从零开始的 8 个步骤，含完整案例、命令与回退 |
| [AI Agent 辅助教程](docs/agent-guide.md) | 四段 Prompt：检测、配置、验收、回退 |
| [技术说明与常见问题](docs/reference.md) | DIRECT 与 bypass、Fake-IP/REDIRECT、验证方法与失败条件 |
| [证据附录](docs/evidence.md) | 脱敏后的原始命令与输出（可选阅读） |

---

## 11. 许可证

[MIT](LICENSE)
