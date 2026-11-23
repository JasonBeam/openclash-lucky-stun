# 从零开始：OpenClash + Lucky STUN 手动配置教程

本教程用一个完整案例，把「NAS 上的 Emby 通过 Lucky IPv4-TCP STUN 暴露到公网」的每一步走完。

**案例里的地址、端口、服务名都是演示值**，请替换成你自己的；不要把它们当成 Lucky 或 OpenClash 的默认值。

## 案例环境

| 项目 | 示例值 |
| --- | --- |
| 主路由 | OpenWrt / iStoreOS |
| 防火墙 | Firewall4 + nftables |
| OpenClash | Fake-IP、REDIRECT、非 TUN |
| NAS | `192.168.10.10` |
| Lucky | Docker，host 网络 |
| 后端服务 | Emby |
| Emby 端口 | `8096/TCP` |
| Lucky STUN 监听端口 | `35080/TCP` |
| 外网映射端口 | 由 STUN 动态获取 |

开始之前先记住一句话：**Lucky 显示拿到公网地址，只完成了一半**。入站还需要路由器映射和运营商放行，缺一不可。

---

## 第 1 步：检查使用条件

这一步的目标不是配置，而是判断「这套方案在你家是否成立」。任何一项不满足，先停下来。

### 1.1 网络是否满足 NAT1 使用要求

Lucky STUN 依赖网络具备较好的 NAT 行为。使用前请确认：

- 网络满足 Lucky 所要求的 **NAT1** 条件；
- 具备**端口转发能力**（路由器 UPnP，或可手动配置端口转发）。

判断方式以 Lucky 自带的检测/日志和官方说明为准。**符合 NAT1 条件不代表一定穿透成功**，还要看运营商是否放行映射端口、以及入站链路是否完整。

### 1.2 检查 OpenClash 的运行模式

在路由器上执行（只读）：

```sh
# 路由器
nft -a list chain inet fw4 openclash | grep -i redirect
```

如果能看到类似 `redirect to :7892` 的规则，说明 TCP 是被 **REDIRECT** 接管到 Clash 内核的，属于本教程覆盖的模式。

同时确认：

- OpenClash 使用 **Fake-IP** 模式；
- **没有开启 TUN 模式**（TUN 的接管点在设备层，本教程的绕过规则不适用）；
- 如果使用 **TPROXY** 接管，链名和绕过位置不同，请勿照搬。

### 1.3 检查 Firewall4 / nftables

```sh
# 路由器
nft -v
nft list tables | grep fw4
```

- 有 `nft` 命令、存在 `inet fw4` 表 → 继续；
- 只有 `iptables`（Firewall3） → **停止**，本文命令不适用；
- 存在 `inet fw4 openclash` 链 → 说明 OpenClash 接管在 nftables 里，符合预期。

### 1.4 确认 LAN、WAN、NAS 的连接关系

```sh
# 路由器：确认 WAN 与 LAN，以及 NAS 在哪个网段
ip -4 addr show | grep -E 'inet |^[0-9]+:'
ip route
```

预期：NAS 与路由器 LAN 同网段（本案例 `192.168.10.0/24`），路由器有 WAN 口（PPPoE 或 DHCP）。

如果 NAS 挂了旁路由、或者存在多层 NAT，先画清楚路径再继续——多一层 NAT 就多一次映射，STUN 的映射可能落在错误的那一层。

### 1.5 不满足条件时怎么停

出现下面任一情况，请停止配置，不要继续改防火墙：

| 情况 | 原因 | 建议 |
| --- | --- | --- |
| 只有 Firewall3 / iptables | 链名、表结构完全不同 | 换方案或自行改写规则 |
| OpenClash 用 TUN | 流量在设备层被接管 | 关闭 TUN 再试，或另找绕过点 |
| 没有 `inet fw4 openclash` 链 | OpenClash 未接管或版本差异 | 先确认代理是否真的生效 |
| NAT 条件不满足 / 运营商不放行 | STUN 映射无法建立或不可达 | 改用中转/隧道方案 |
| 没有端口转发能力 | 入站无法到达 NAS | 先解决 UPnP 或手动转发 |

---

## 第 2 步：配置 NAS 和 Emby

### 2.1 固定 NAS 的局域网地址

在路由器上给 NAS 做 **DHCP 静态绑定**（按 MAC 固定 IP），或在 NAS 上设置静态 IP。

为什么要固定：绕过规则按 **NAS 源 IP** 匹配，IP 一变规则就失效，而且 OpenClash 会把这个新 IP 当普通客户端接管。

### 2.2 检查 Emby 的 8096 端口

```sh
# NAS
ss -lntp | grep 8096
# 期望看到：0.0.0.0:8096 或 127.0.0.1:8096 处于 LISTEN
```

### 2.3 用 curl 验证后端正常

```sh
# NAS
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8096/
# 期望：200 / 302 之类的正常响应，而不是 000
```

返回 `000` 表示连不上；先解决 Emby，再谈穿透。

### 2.4 Docker Host 网络与 bridge 网络的差异

| 部署方式 | Lucky 能否用 `127.0.0.1` 访问后端 | 说明 |
| --- | --- | --- |
| `--network host` | 可以 | Lucky 与 NAS 共享网络命名空间，`127.0.0.1:8096` 就是 NAS 本机 |
| bridge（默认） | 不建议 | 容器内的 `127.0.0.1` 是容器自己，应填 NAS 的局域网 IP，或改用 host 网络 |
| Lucky 与 Emby 在不同容器 | 视网络而定 | 同一自定义网络可用容器名；否则用 NAS 局域网 IP |

**结论**：`127.0.0.1` 只有在 Lucky 与后端处于同一网络命名空间时才成立。不确定就写 NAS 的局域网地址。

---

## 第 3 步：在 Lucky 添加 STUN 穿透规则

打开 Lucky 面板，按下面顺序操作（界面文字可能随版本略有差异）：

1. 进入 **STUN 内网穿透** 模块。
2. 打开 **穿透规则列表**。
3. 点击 **添加穿透规则**。
4. 选择 **简易模式**。
5. 穿透类型选择 **IPv4-TCP**。
6. 设置监听端口 **35080**（选一个未被占用的端口）。
7. 按部署方式设置 **NAT-PMP / UPnP**：
   - 路由器支持 UPnP 且已开启 → 使用 UPnP 让路由器自动建映射；
   - 不支持或无权限 → 关闭自动映射，改在路由器上手动做端口转发；
   - 两者都配容易产生重复/冲突映射，选一种即可。
8. 保持 **Lucky 内置转发** 启用（由 Lucky 把监听端口的流量转到后端）。
9. 目标地址填写 **可访问 Emby 的地址**：本案例 Lucky 为 host 网络，填 `127.0.0.1`。
10. 目标端口填写 **8096**。
11. 保存并 **启用** 规则。
12. 查看规则 **日志、当前公网地址、映射端口**，确认拿到了映射。

> 公网地址和映射端口是**动态**的，会随运营商 NAT 变化。记录它们只用于当下测试，不要写死到任何配置里。

---

## 第 4 步：检查路由器 UPnP

### 4.1 开启 UPnP

在 LuCI 里通常是「网络 → UPnP」或「服务 → UPnP」，确认：

- UPnP 服务已启用；
- 允许内网设备（至少 NAS 所在网段）请求映射；
- 若使用 miniupnpd，确认它监听的是 **WAN 侧**接口。

### 4.2 检查动态端口映射

```sh
# 路由器
nft list chain inet fw4 upnp_forward 2>/dev/null | grep 35080
nft list chain inet fw4 upnp_prerouting 2>/dev/null | grep 35080
```

- `upnp_forward` 有规则 → 入站被允许转发；
- `upnp_prerouting` 有 DNAT 到 NAS → 目标地址被改写正确。

两条都存在，才说明「WAN 端口 → NAS 监听端口」这条映射是完整的。

### 4.3 WAN 端口与内网监听端口的区别

- **内网监听端口**：Lucky 在 NAS 上监听的端口（本案例 `35080`），固定不变；
- **WAN 端口**：运营商 NAT 分配给你的公网端口，由 STUN 决定，**每次可能不同**。

外部访问用的是「公网 IP + WAN 端口」，路由器再把它转回「NAS:35080」。所以规则里匹配的一定是 NAS 的 `sport 35080`，不是 WAN 端口。

### 4.4 不建议用 DMZ

DMZ 会把 WAN 入站全部 DNAT 给 NAS：

- 解决不了「运营商不放行固定端口」的问题；
- 等于把整台 NAS 的应用全部暴露到公网；
- 出问题时更难定位是哪一层。

只开放业务需要的端口即可。

---

## 第 5 步：检查 OpenClash 防火墙接管

```sh
# 路由器：只读
nft -a list chain inet fw4 openclash
```

看三件事：

1. **有没有 REDIRECT 规则**：类似 `tcp dport { 80, 443 } redirect to :7892`，说明 TCP 被送进内核；
2. **规则顺序**：绕过规则必须排在 REDIRECT **之前**才有效；
3. **有没有现成的 bypass**：如果已经有同类规则，先核对再决定是否新增，避免重复。

还要确认自定义脚本会被调用：

```sh
# 路由器
ls -l /etc/openclash/custom/openclash_custom_firewall_rules.sh
grep -n 'openclash_custom_firewall_rules' /etc/init.d/openclash /usr/share/openclash/*.sh 2>/dev/null | head
```

不同 OpenClash 版本的调用方式不同：有的在启动/重载时 source 该脚本，有的只在特定操作时执行。**以自己的版本为准**；如果脚本根本不会被调用，写进去也不会生效。

---

## 第 6 步：添加 OpenClash 绕过规则

### 6.1 备份原文件

```sh
# 路由器
cp /etc/openclash/custom/openclash_custom_firewall_rules.sh \
   /etc/openclash/custom/openclash_custom_firewall_rules.sh.bak.$(date +%Y%m%d-%H%M%S)
```

### 6.2 检查目标链是否存在

```sh
# 路由器
nft list chain inet fw4 openclash >/dev/null 2>&1 && echo "chain ok" || echo "chain missing"
```

`chain missing` 时不要继续：说明接管链不在预期位置，需要先按第 5 步重新确认。

### 6.3 写入规则（防重复、带 counter、唯一标记）

编辑脚本，把下面这段放在 **已有 `exit 0` 之前**（脚本里通常末尾有 `exit 0`，写在其后不会执行）：

```sh
# 路由器：OpenClash 自定义防火墙脚本片段
# 说明：让 NAS 从 35080 端口发出的 TCP 流量绕过 OpenClash 接管链
# 只匹配 NAS 源 IP + TCP 源端口；sport 是 NAS 出站的本地端口
if nft list chain inet fw4 openclash >/dev/null 2>&1; then
    if ! nft list chain inet fw4 openclash | grep -q 'lucky_stun_35080'; then
        nft insert rule inet fw4 openclash \
            ip saddr 192.168.10.10 tcp sport 35080 \
            counter return comment 'lucky_stun_35080'
    fi
fi
```

要点：

- **`sport` 不是 `dport`**：匹配的是 NAS 出站源端口；写成 `dport 35080` 匹配不到任何流量。
- **`insert` 而不是 `add`**：`insert` 默认插到链首，保证在 REDIRECT 之前；`add` 会追加到末尾。
- **`counter`**：便于判断是否命中（但不等于成功）。
- **唯一注释 `lucky_stun_35080`**：用于防重复和回退时定位。
- **端口范围尽量小**：只覆盖 Lucky 实际使用的端口；如果 Lucky 配置的是一个端口段（例如 `35080-35099`），这里就写 `tcp sport 35080-35099`，注释标记仍是唯一的。
- **注释标记只是示例**：换一个你自己的唯一标记也可以，但要和防重复、回退时用的字符串保持一致。

### 6.4 检查规则顺序

```sh
# 路由器
nft -a list chain inet fw4 openclash | grep -nE 'lucky_stun|redirect'
```

确认 `lucky_stun_35080` 的 handle 排在 redirect 规则**之前**。如果不在，说明脚本插入时机或链不对。

### 6.5 验证 OpenClash 重启后规则仍在

```sh
# 路由器
/etc/init.d/openclash restart
sleep 15
nft list chain inet fw4 openclash | grep lucky_stun_35080
```

重启后规则消失，说明脚本没有被调用或没有持久化，需要按你的版本调整调用方式，而不是反复手动插入。

---

## 第 7 步：验证穿透成功

按顺序做，每步只回答一个问题。

### 7.1 NAS 本地 Emby 是否可访问

```sh
# NAS
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8096/
```

返回 000 → 后端没起来，后面都不用看。

### 7.2 Lucky 是否取得当前公网映射

在 Lucky 面板看规则详情：有公网地址 + 映射端口，且日志没有持续报错。

只证明「出站探测成功」，**不证明入站可达**。

### 7.3 UPnP 映射是否正确

```sh
# 路由器
nft list chain inet fw4 upnp_forward 2>/dev/null | grep 35080
nft list chain inet fw4 upnp_prerouting 2>/dev/null | grep 35080
```

没有映射 → 检查 UPnP 是否开启、Lucky 是否有权限、是否指向了错误的网段。

### 7.4 绕过规则是否在 REDIRECT 之前

```sh
# 路由器
nft -a list chain inet fw4 openclash | grep -E 'lucky_stun|redirect'
```

顺序不对 → 回到 6.3，用 `insert` 并确认脚本在 `exit 0` 之前。

### 7.5 counter 是否持续增长

重新触发一次外部访问（或让 Lucky 重连），再看：

```sh
# 路由器
nft -a list chain inet fw4 openclash | grep lucky_stun
```

- 增长 → 有流量走了绕过分支，说明匹配生效；
- 不增长 → `saddr` / `sport` 写错，或规则位置不对。

**增长 ≠ 成功**，只说明规则被命中。

### 7.6 用外部 IPv4 网络实际连接

用手机蜂窝网络（关闭 Wi-Fi），或第三方在线检测，访问：

```sh
# 本地电脑（任意外网环境）
curl -4 -m 10 -v 'http://<当前公网IP>:<当前映射端口>/'
```

能连上说明运营商放行了这个映射端口、DNAT 正确。

### 7.7 检查是否到达正确后端

- 返回内容/标题应是目标服务（如 `BeamJasonNAS` 这类）；
- NAS 上 `conntrack -L | grep 35080` 或 Emby 日志里能看到对应连接。

如果返回的是 Lucky 的错误页、路由器页面或 502，说明映射到了错误的地方。

> 结果解读：第 1~5 步全过、第 6 步失败 → 问题在运营商或端口变化；第 6 步过、第 7 步错 → 映射到了错误的后端。**单次成功不代表长期稳定**。

---

## 第 8 步：保存、重启复测与回退

### 8.1 保存与复测

1. 保存脚本并确认 `exit 0` 仍在最后；
2. 重载或重启 OpenClash（会短暂断网）；
3. 确认规则仍在、counter 继续增长；
4. 再走一遍第 7 步的外部验证；
5. 有条件的话重启一次路由器，确认 UPnP 映射和绕过规则都能自动恢复。

### 8.2 回退：删除自定义规则

方式一：恢复备份

```sh
# 路由器
ls -t /etc/openclash/custom/openclash_custom_firewall_rules.sh.bak.* | head -1
cp /etc/openclash/custom/openclash_custom_firewall_rules.sh.bak.<时间戳> \
   /etc/openclash/custom/openclash_custom_firewall_rules.sh
```

方式二：只删运行时那一条（推荐，影响最小）

```sh
# 路由器：先看 handle
nft -a list chain inet fw4 openclash | grep lucky_stun_35080
# 再用实际 handle 删除，例如 handle 1234
nft delete rule inet fw4 openclash handle 1234
```

要点：

- handle 在链重建后会变，**每次都重新查**，不要复制旧值；
- **不要清空整条 `openclash` 链**（`nft flush chain ...`），会破坏 OpenClash 的接管规则；
- 同时删除脚本里的那段 `if ... fi`，否则下次重载又会出现。

### 8.3 回退后确认

```sh
# 路由器
nft list chain inet fw4 openclash | grep -c lucky_stun
```

期望 0；再确认代理与上网正常，外部访问按预期失效（说明之前的成功确实来自这条绕过规则）。

---

## 下一步

- 想让 AI Agent 帮你做检测和落配置：[AI Agent 辅助教程](agent-guide.md)
- 想理解每一步背后的机制与失败条件：[技术说明与常见问题](reference.md)
