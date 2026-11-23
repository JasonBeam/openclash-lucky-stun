# 证据附录（已脱敏）

本文件保留 [技术说明](reference.md) 中标注【实测】的脱敏的历史命令和输出摘要。地址、端口等可能替换为示例值；未能确认的原始细节不应视为完整原始日志。

## 1. OpenClash 接管链与绕过规则

```sh
# 路由器
nft -a list chain inet fw4 openclash
```

原附录展示的两个规则片段顺序与“绕过在 REDIRECT 前”的文字结论冲突，且现有脱敏记录无法可靠还原当时的完整链输出；因此不将该片段作为排序证据，也不重新编造输出。

- 历史记录显示曾配置 NAS 指定 TCP 源端口绕过，并观察到 counter 增长；这只能证明有流量命中。
- 判断当前顺序时，应读取完整链，以实际输出的先后为准；handle 是标识，不是排序编号。

## 2. 外部可达性（check-host）

```text
GET https://check-host.net/check-tcp?host=<公网IP>:<映射端口>&max_nodes=6
（Accept: application/json）

结果（略去节点名）：
  OK 0.20 s / OK 0.23 s / OK 0.26 s / OK 0.29 s / OK 0.31 s / OK 0.35 s
```

同一映射端口用第三方阅读器抓 `/web/index.html`，返回页面标题 `MyNAS`。

另一轮针对同一业务的检测中，5 个外部检测点收到 HTTP 302；配置完成后用户在真实使用中确认入口可用。302 与单次成功都只说明服务有响应，不等于已验证登录、播放或长期稳定。

## 3. DNAT 路径（conntrack）

```text
tcp 6 431999 ESTABLISHED src=198.51.100.24 dst=203.0.113.10 sport=52104 dport=35080
    src=192.168.10.10 dst=198.51.100.24 sport=35080 dport=52104 [ASSURED]
```

## 4. 固定端口无入站（对照实验）

```sh
# 路由器 WAN 口抓包，同时用外部检测点打固定端口（面板 16601）
tcpdump -ni <WAN> 'tcp and (dst host <公网IP> or src host <公网IP>)' -w /tmp/w.pcap
```

```text
抓到的 3023 个包全部是本机连接的正常回包；目标端口 16601 的入站 SYN：0 个。
同一时段对映射端口做同样探测：可达（见第 2 节）。
```

UDP 对照同样为 0：

```text
tcpdump -ni <WAN> 'udp and port <port>' → 0 packets
```

## 5. 公网地址与端口变化（Lucky API）

```json
// GET /666/api/stunrulelist → list[0].PublicAddrHistroy（节选）
[
  {"AddrRecord": "203.0.113.10:50180", "UpdateTime": "<时间1>"},
  {"AddrRecord": "",                  "UpdateTime": "<时间2>"},
  {"AddrRecord": "203.0.113.10:50180", "UpdateTime": "<时间3>"},
  {"AddrRecord": "198.51.100.77:24871", "UpdateTime": "<时间4>"},
  {"AddrRecord": "",                  "UpdateTime": "<时间5>"}
]
```

（历史脱敏摘录含空值和不同地址；时间未保留，不能据此判断变化频率。）

## 6. 443 端口 UDP 被 reject

```sh
# 路由器
nft list chain inet fw4 input | grep -A2 'udp dport 443'
```

```text
udp dport 443 ip daddr != @china_ip_route counter packets 554 bytes 44200
    reject with icmp type port-unreachable
```

历史记录称发送 3 个 UDP STUN 请求到 `:443` 后：

```text
counter packets 554 → 557
同一时间相关日志未显示 UDP 443 记录；这不足以说明所有 TCP STUN 穿透失败
```

配置来源：`/etc/config/openclash` 中 `option disable_udp_quic '1'`；Clash yaml 另有 `- DST-PORT,443,Others`。

## 7. DIRECT 无法保留客户端目标 IP

```sh
curl --resolve tracker.example.com:443:1.1.1.1 -o /dev/null -w '%{http_code} %{time_total}\n' https://tracker.example.com/
curl --resolve tracker.example.com:443:203.0.113.7 -o /dev/null -w '%{http_code} %{time_total}\n' https://tracker.example.com/
```

```text
404 0.72
404 0.75
# 两次状态码接近，本结果不足以单独证明实际拨号目标；需结合抓包或连接记录
```

对应 sniffer 配置：`override-destination: true`、`parse-pure-ip: true`、`force-dns-mapping: true`。

## 8. Fake-IP 与测速失真

```text
# NAS 上
getent hosts www.example.com      → 198.18.x.x        （fake-ip）
getent hosts api.github.com       → <真实 IP>          （在 fake-ip-filter 中排除）

# 开启 OpenClash 时的 curl 计时
time_connect: 0.000612 s   ← 本机内核立即完成，无参考价值
time_appconnect: 0.282967 s
time_starttransfer: 0.284452 s   ← 包含代理与服务端处理，不等于纯 RTT
```

---

相关文档：[技术说明与常见问题](reference.md) · [手动配置教程](manual-setup.md)
