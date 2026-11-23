# 证据附录（已脱敏）

本文件保留 [技术说明](reference.md) 中标注【实测】的原始命令与输出片段。真实地址、端口、主机名、域名、Token 已替换为示例值；结构、顺序、数量级保持不变。

## 1. OpenClash 接管链与绕过规则

```sh
# 路由器
nft -a list chain inet fw4 openclash
```

```text
# 关键片段（省略无关规则）
... tcp dport { 80, 443 } redirect to :7892 comment "openclash tcp redirect" ...
... ip saddr 192.168.10.10 tcp sport 35080-35099 counter packets 657 bytes 34164 return comment "lucky_stun_range" # handle 6178
```

- 绕过规则在 redirect 规则**之前**，因此先命中 `return`；
- counter 随时间增长，说明 NAS 该源端口段确实有出站流量经过此链。

## 2. 外部可达性（check-host）

```text
GET https://check-host.net/check-tcp?host=<公网IP>:<映射端口>&max_nodes=6
（Accept: application/json）

结果（略去节点名）：
  OK 0.20 s / OK 0.23 s / OK 0.26 s / OK 0.29 s / OK 0.31 s / OK 0.35 s
```

同一映射端口用第三方阅读器抓 `/web/index.html`，返回页面标题 `MyNAS`。

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

（`AddrRecord` 为空表示该时刻未获取到映射；公网 IP 与端口都发生过变化，记录时间仅用于说明变化频率。）

## 6. 443 端口 UDP 被 reject

```sh
# 路由器
nft list chain inet fw4 input | grep -A2 'udp dport 443'
```

```text
udp dport 443 ip daddr != @china_ip_route counter packets 554 bytes 44200
    reject with icmp type port-unreachable
```

发送 3 个 STUN 请求到 `:443` 后：

```text
counter packets 554 → 557
同一时间 openclash 日志、clash /connections：无任何 UDP 443 记录
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
# 两次结果一致：拨号目标不是 --resolve 指定的 IP
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
time_starttransfer: 0.284452 s   ← 真实指标
```

---

相关文档：[技术说明与常见问题](reference.md) · [手动配置教程](manual-setup.md)
