# OpenClash 与 Lucky STUN 共存指南（IPv4 TCP 内网穿透）

> **一句话结论**：OpenClash 里把流量设成 `DIRECT`，**不等于**这条连接在防火墙层没被接管。
> 需要让 NAS 指定源端口的 TCP 流量在进入 OpenClash 的接管链之前就 `return`，再让 UPnP 把公网映射落到 Lucky 的监听端口上。

这是一个**单案例实录 + 可复用步骤**，不是跨运营商的通用真理。文中把「已实测」「机制推断」「未验证的可能性」严格分开标注：

| 标记 | 含义 |
| --- | --- |
| 【实测】 | 在本环境中做过对照实验，有计数器 / 抓包 / 外部探测等证据 |
| 【推断】 | 与实测现象一致，但没有逐包逐层证明，可能是主因也可能是并发因素 |
| 【未验证】 | 只是可能性或社区说法，本文没有在本环境证实 |

- 首次验证：2026-11-14（链路打通并外部可达）
- 最近复核：2026-11-23（绕过规则仍在生效，Lucky 侧 STUN 规则已被本人停用）

---

## 1. 适用与不适用

**适用（本文实测过的组合）**

- 路由器：iStoreOS / OpenWrt，**Firewall4 + nftables**，存在 `inet fw4 openclash` 链。
- 代理：OpenClash，**Fake-IP 模式、非 TUN**，TCP 透明接管走 **REDIRECT**（转发到内核的 redir 端口）。
- 服务端：NAS 上 Lucky 以 **host 网络**运行，后端服务（如 Emby）与 Lucky 同网络命名空间。
- 需求：没有公网 IPv4 直连能力，通过 **Lucky IPv4 TCP STUN** 打洞 + 路由器 UPnP 映射对外提供服务。

**不适用 / 不要照搬**

- Firewall3 / iptables 的路由器（链名、表结构完全不同）。
- OpenClash 的 TUN 模式、TPROXY 模式（接管点不同，绕过位置要重新找）。
- 没有 `inet fw4 openclash` 链，或接管规则不在该链内。
- 纯 IPv6 方案：本文完全没覆盖 IPv6。

> 判断依据只有一个：**你自己路由器上实际的链和规则顺序**。先看，再改。

---

## 2. 三条结论

1. **【实测】只在 Clash 里加 `DIRECT` 解决不了。** 流量在防火墙层已经被重定向进内核；内核拿到连接后还会按 SNI 重新解析和拨号，客户端原本的目标 IP / 源端口关系可能被改写，STUN 拿到的映射与实际出站不一致。
2. **【实测】按「NAS 源端口」在 `openclash` 链最前面 `return`，链路恢复。** 规则形如 `ip saddr <NAS> tcp sport 35080-35099 counter return`，位置必须在 TCP REDIRECT 之前。
3. **【实测】外网能否进来，取决于运营商 NAT 是否放行"它当前分配给这个 socket 的端口"。** 固定端口（如管理面板 16601）从外网抓不到任何入站 SYN；而 STUN 打出来的映射端口可达。**DMZ 在这种环境下基本无收益。**

---

## 3. 先判断：你遇到的是不是同一类问题

典型现象：

- Lucky 面板显示已获取公网地址和端口，但外网连不上 / 时通时不通。
- 关闭 OpenClash 就好，开启就坏（或反过来只有某些端口坏）。
- 换端口、改 DMZ、加防火墙放行都没用。
- 管理面板（固定端口）外网永远进不来，但 STUN 打洞的端口偶尔能进。

快速三连（全部只读）：

```sh
# 1) 是否存在 OpenClash 的 nftables 链，接管规则长什么样
nft -a list chain inet fw4 openclash

# 2) Lucky 实际监听与后端是否同命名空间
#    NAS 上执行
ss -lntp | grep -E ':(35080|8096)'
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:35080/
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8096/

# 3) 公网入站到底有没有到路由器（在路由器 WAN 口抓）
tcpdump -ni <WAN接口> 'tcp and port <STUN映射端口>' -c 20
```

- 第 1 条能看到 `redirect to :7892` 之类的规则 → 属于本文场景。
- 第 3 条**抓到入站 SYN** → 继续按本文绕过；**一个包都没有** → 先解决运营商侧（见第 6 节），改配置没用。

---

## 4. 已验证的事实（附证据）

> 下列地址、端口、IP 已脱敏为示例值；证据来自本机实际输出，仅做格式整理。

| # | 事实 | 证据 |
| --- | --- | --- |
| 1 | 存在 OpenClash 的 nft 接管链，且自定义脚本的规则能插进该链并计数 | `nft -a list chain inet fw4 openclash` 可见 `... tcp sport 35080-35099 counter packets 657 bytes 34164 return comment "codex_lucky_stun_range"` |
| 2 | 绕过规则确实被命中（有真实流量经过它） | 同上 counter 持续增长（657 → 310 包/16 KB，跨重启重建） |
| 3 | 外网 TCP 能到达该服务 | check-host 6 个外部节点对 `<公网IP>:<映射端口>` 全部 TCP 成功，0.20–0.35 s |
| 4 | 外部拿到的确实是 NAS 上的服务 | 用第三方阅读器抓 `http://<公网IP>:<映射端口>/web/index.html`，页面标题为 `BeamJasonNAS` |
| 5 | DNAT 路径正确 | conntrack：`34.34.253.19 → 100.72.239.176:35080` DNAT 到 `192.168.10.10:35080`，状态 ESTABLISHED |
| 6 | 运营商不给固定端口入站 | 同一时段内，对**固定端口**（面板 16601）做外部探测，WAN 抓包 **0 个入站 SYN**；而映射端口可达 |
| 7 | 公网端口会变 | Lucky 的 `PublicAddrHistroy` 记录过多个不同端口（如 50180 / 51510 / 52566 / 53424） |
| 8 | 公网 IPv4 也会变 | 同一条历史记录里，公网地址从 `14.216.5.26` 变为 `14.218.35.219`（间隔数天） |
| 9 | DMZ 在这种环境无效 | 配了 `dnat ip to <NAS>` 全端口 DMZ，期间外部入站抓包仍为 0 |
| 10 | 443 端口的 STUN 一定失败 | 发 3 个 STUN 包到 `:443`，`chain input` 的 `udp dport 443 ... reject with icmp port-unreachable` 计数 554 → 557；同期 clash 日志与连接表无任何记录 |
| 11 | `DIRECT` 之后客户端目标 IP 会被丢弃 | `curl --resolve <域名>:443:1.1.1.1` 与 `:203.0.113.7` 结果完全相同（都正常返回），说明拨号目标不是指定的 IP |
| 12 | Fake-IP 生效时本地解析是假地址 | NAS `getent hosts www.google.com` → `198.18.x.x`；被规则集排除的域名才拿到真实 IP |
| 13 | OpenClash 开启时 `TCP connect` 计时不可信 | `curl --resolve ...:443:104.16.91.215` 的 `time_connect=0.0006s`（内核本地立即完成握手），`time_starttransfer` 才是真实值（0.28 s） |

关于第 10 条的配置来源：`/etc/config/openclash` 中 `option disable_udp_quic '1'`，配合 nft 的 443 reject；同时 Clash 配置里存在 `- DST-PORT,443,Others` 规则，即便放行也会被送进代理组。

---

## 5. 机制：为什么 DIRECT 不够，为什么 bypass 能修

**【推断】** 下面这条链路能解释所有实测现象，但没有做逐包证明。

```text
普通流量：
  应用 socket → 路由器 forward/mangle → 命中 openclash 链
              → REDIRECT 到内核 redir 端口 → Clash 判定 DIRECT/代理
              → 内核用自己的 socket 重新拨号（可能重解析域名、换源端口）
              → 运营商 NAT 看到的是"内核的会话"，不是应用的会话

绕过流量：
  应用 socket → 命中 openclash 链的第一条 return
              → 不进入内核 → 直接按原 socket 从 WAN 发出
              → 运营商 NAT 看到的就是 STUN 探测时的那个会话
```

三个要点：

1. **接管发生在策略之前。** `DIRECT` 是"进了内核之后选择直连"，不是"没进内核"。
2. **STUN 依赖 5 元组一致性。** 打洞成功后，运营商 NAT 的映射绑定在「内网 IP:端口 → 公网 IP:端口」这条会话上；只要出口 socket 变了（源端口被内核改写、或域名被重新解析到另一个地址），映射就对不上。
3. **SNI 嗅探会改写目标。** 本环境 sniffer 开启 `override-destination` 与 `force-dns-mapping`，内核按域名重解析后拨号，客户端 `--resolve` 钉住的 IP 直接失效（对应第 11 条）。

**为什么 UPnP 也要在**：STUN 只负责"打出一个公网映射"，入站方向还需要一条静态转发把 `WAN:端口` 送到 NAS 的监听端口。没有 UPnP（或手动端口转发），出站映射成功、入站照样进不来。

---

## 6. 未验证的部分（不要当成结论）

以下都是**可能性**，本文没有在本环境证明，引用时请保留不确定性：

1. **运营商 NAT 类型**（EIM / EIF / 对称型 / 地址受限）。工具常把"同一 socket 对两个 STUN 服务器得到同一映射"误报成 Full Cone；本环境只验证了"当前分配端口可入站、固定端口不可"，没做完整 NAT 类型测试。
2. **手动端口转发能否替代 UPnP**。理论上可以（把固定端口转发到 NAS 监听端口），但本环境没有实际测试过固定端口转发是否会被运营商 NAT 放行。
3. **关闭 `disable_udp_quic` 后 443 STUN 是否可用**。没关过，因此不知道是"只差这一条"还是"还有别的原因"。
4. **TUN / TPROXY 模式下的正确绕过位置**。本文只覆盖 REDIRECT。
5. **绕过规则对同端口段其他应用的影响**。规则按"NAS + 源端口段"匹配，不区分进程；理论上同段端口的其他出站也会被绕过，本文没有逐项回归。
6. **长期稳定性**。观察窗口约 6 天，期间公网端口和公网 IPv4 都变过；Lucky 侧规则后来被停用，没有做跨月观察。
7. **不同 OpenClash 版本的链名与脚本路径**（`/etc/openclash/custom/openclash_custom_firewall_rules.sh` 是否被调用、何时重建链）。版本差异较大，务必自行核对。
8. **IPv6 入站**。DMZ 只处理 IPv4；NAS 的 IPv6 侧另有流量，但未系统验证。
9. **运营商为何只放行"当前映射端口"**。可能是 NAT 会话跟踪、也可能有额外策略，没有证据。本文只描述现象。

---

## 7. 配置步骤

### 7.1 准备

1. 给 NAS 固定局域网 IPv4（DHCP 静态绑定）。
2. 确认后端服务本机可达：`curl -I http://127.0.0.1:8096/`。
3. Lucky 新建 **IPv4 TCP** STUN 规则：监听未占用端口（示例 `35080`），后端 `127.0.0.1:8096`。
   - 后端与 Lucky 不同网络命名空间时，`127.0.0.1` 必须改成实际地址。
4. 确认 Lucky 拿到公网地址与端口（面板显示），记下当前端口——**这个端口是动态的**。

### 7.2 确认链与接管规则

```sh
nft -a list chain inet fw4 openclash
```

找到把 TCP 重定向到内核的那几条（通常是 `redirect to :<redir-port>`）。绕过规则必须插在它们**前面**。

### 7.3 写入绕过规则

先备份 OpenClash 的自定义防火墙脚本，再在原有 `exit 0` **之前**追加：

```sh
# 备份
cp /etc/openclash/custom/openclash_custom_firewall_rules.sh \
   /etc/openclash/custom/openclash_custom_firewall_rules.sh.bak.$(date +%Y%m%d-%H%M%S)
```

```sh
# 在 exit 0 之前插入；标签保持唯一，便于识别与回退
if ! nft list chain inet fw4 openclash 2>/dev/null | grep -q 'lucky_stun_range'; then
    nft insert rule inet fw4 openclash \
        ip saddr 192.168.10.10 tcp sport 35080-35099 \
        counter return comment 'lucky_stun_range'
fi
```

注意事项：

- 匹配的是 **TCP 源端口 `sport`**（NAS 出站时用的本地端口），不是 STUN 服务器的目标端口，也不是面板显示的公网端口。
- 端口段只保留必要范围；Lucky 监听端口变了就同步改。
- `return` 若排在 REDIRECT 之后，等于没写——务必看 `nft -a` 的顺序号。
- 只跑一次脚本不等于持久化，要按你的 OpenClash 版本确认"重载/重启后是否重放"，并在重启后复查。

### 7.4 打通入站

- 路由器开启 UPnP，允许 NAS 建立映射；或手动把 `WAN:<端口>` 转发到 `NAS:<监听端口>`。
- 检查实际映射：`nft list chain inet fw4 upnp_forward` / `upnp_prerouting` 应出现指向 NAS 的对应端口。
- 不要在没搞清原理时用 DMZ 兜底（本环境实测无收益，且把整台 NAS 暴露到 WAN）。

---

## 8. 验证（按顺序，缺一不可）

```sh
# 1) 规则在、顺序对、能计数
nft -a list chain inet fw4 openclash | grep -E 'lucky_stun|redirect'

# 2) 映射存在
nft list chain inet fw4 upnp_forward 2>/dev/null | grep 35080

# 3) 本地服务活着
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:35080/

# 4) 外部可达（用手机蜂窝网络 / 第三方检测，强制 IPv4）
curl -4 -m 10 -v 'http://<当前公网IP>:<当前映射端口>/'

# 5) 连接确实落到 NAS
conntrack -L 2>/dev/null | grep 35080
```

判读规则：

- **counter 增长** = 有流量命中绕过规则，**但不等于**外网成功。
- **端口可达** = 运营商放行了这个映射，**但不等于**服务端登录/播放正常。
- **单次成功** = 只在当下成立；端口和公网 IP 都会变，要观察一段时间。
- 任何一项失败，按第 9 节顺序往回查，不要同时改多处。

---

## 9. 排错决策树

```text
外网连不上
├─ WAN 抓包有入站 SYN？
│  ├─ 没有 → 运营商侧：映射端口失效/变化，或该端口未被放行
│  │        → 回 Lucky 看当前 PublicAddr；DMZ/固定端口放行通常无效
│  └─ 有 → 继续
├─ conntrack 里有 DNAT 到 NAS？
│  ├─ 没有 → UPnP/端口转发没生效（看 upnp_forward/upnp_prerouting）
│  └─ 有 → 继续
├─ NAS 本地 curl 监听端口有响应？
│  ├─ 没有 → Lucky 规则没启用/监听没起来/后端地址错
│  └─ 有 → 继续
├─ openclash 链里 bypass 计数增长？
│  ├─ 没有 → 规则位置在 REDIRECT 之后，或 saddr/sport 写错
│  └─ 有 → 仍不通时对比"开/关 OpenClash"，确认是否被 sniffer 改写目标
```

---

## 10. 回退

1. 从自定义脚本里删除本次新增的 `if ... fi` 段，保留原有规则和 `exit 0`。
2. 找到运行时规则并删单条：

```sh
nft -a list chain inet fw4 openclash | grep lucky_stun_range
nft delete rule inet fw4 openclash handle <实际handle>
```

3. 不要清空整条 `openclash` 链，也不要复制旧 handle（规则重建后 handle 会变）。
4. 回退后重跑第 8 节验证，确认外网行为随之改变。

---

## 11. 安全与运维

- **面板与服务分开鉴权**：Lucky 面板、后端服务（Emby 等）都要独立认证；对外入口建议 HTTPS。
- **UPnP 是信任边界**：任何拿到权限的设备都能请求映射，定期核对 `upnp_forward` 实际内容，只留必要端口。
- **公网端口/地址会变**：普通 DNS A 记录只存 IP，不携带端口；靠域名访问需要额外方案（DDNS 只能解决 IP 变化）。
- **发布截图/日志前**：隐藏公网 IP、端口、域名、账号、令牌、订阅链接。
- **升级后复核**：路由器固件、OpenClash、Lucky 升级，或切换接管模式后，链名与规则顺序都可能变。

---

## 12. 常见误区（FAQ）

**Q1：我在 Clash 里给这个域名/端口设了 DIRECT，为什么还不行？**
因为 DIRECT 是"进内核之后的策略"。接管在防火墙层已经发生，且内核可能按 SNI 重解析、换源端口。

**Q2：Lucky 显示"已获取公网地址"，是不是就通了？**
不是。那只证明出站打洞成功，入站还需要一条映射（UPnP/手动转发），且运营商要放行该端口。

**Q3：开了 DMZ 不就全通了吗？**
本环境实测无效：外部入站抓包为 0。DMZ 只是把 DNAT 指向 NAS，运营商不放行的端口照样进不来，还把整台设备暴露在 WAN。同理，固定端口（如管理面板）也进不来。

**Q4：为什么 443 端口做 STUN 必挂？**
本环境 OpenClash 开了 `disable_udp_quic`，nft 在 input 链对 `udp dport 443` 直接 reject；同时 Clash 配置里有 `DST-PORT,443` 规则会把流量送进代理组。两条叠加，443 的 UDP 根本到不了目标。

**Q5：我在 NAS 的 hosts 里钉了 IP，为什么没生效？**
Fake-IP + sniffer 的 `override-destination` 会让内核按域名重新解析拨号；客户端钉的 IP 对经代理的连接无效。实测 `--resolve` 到任意 IP 结果都相同。

**Q6：测速脚本显示延迟 0.2 ms，是不是节点很快？**
不是。OpenClash 开启时 TCP 握手在本地内核完成，`time_connect` 恒为毫秒级；要看 `time_starttransfer` / TTFB。

**Q7：换端口能解决问题吗？**
能改变"运营商当前分配哪个端口"，但不能改变"固定端口不被放行"这个现象。端口和公网 IP 都会变，不要把某次成功当成永久配置。

**Q8：能不能只让 Lucky 绕过，不影响其他应用？**
本方案按「源地址 + 源端口段」匹配，不区分进程。要更精细就需要 per-process 匹配（nft `meta skuid`/cgroup）或把 Lucky 放进独立网络命名空间——本文未验证。

---

## 13. 参考资料

- [Lucky 官方：STUN 内网穿透](https://www.lucky666.cn/docs/modules/stun/)
- [OpenClash #4808：开启后无法使用 Lucky STUN](https://github.com/vernesong/OpenClash/issues/4808)
- [OpenClash #5003：Lucky 与 OpenClash 同时运行](https://github.com/vernesong/OpenClash/issues/5003)

## 14. 复现信息

- 路由器：iStoreOS（OpenWrt，防火墙4 / nftables），OpenClash + Meta 内核，Fake-IP，非 TUN。
- WAN：PPPoE，内网侧拿到 `100.64.0.0/10` 地址，实际对外是运营商 NAT 后的公网 IPv4。
- NAS：QNAP QTS，Lucky 在 Docker 的 host 网络，后端 Emby 映射到宿主 `8096`。
- 验证日期：2026-11-14（打通），2026-11-23（复核）。

> 本文只记录一个环境的真实排障过程。换环境请按第 3 节重新确认，不要直接复制 IP、端口和规则位置。
