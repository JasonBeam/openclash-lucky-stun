# 使用 Codex / DSH Agent 辅助配置 OpenClash 与 Lucky STUN

本教程教你怎么让一个**具备终端和 SSH 能力**的 AI Agent（Codex、DSH 等）辅助你完成检测、配置、验收和回退。

它不会替你开发新的 Agent 应用，也不要求 Agent 自动改设备——所有写操作都以**你明确授权**为前提。

> 核心原则：**Agent 负责查和改，人负责授权和兜底。** 让 Agent 先只读检测，确认适用后再动手；每一步都要有备份和回退路径。

---

## 第一部分：准备

### 1.1 本地电脑

准备能运行 Agent 的电脑（macOS / Linux / Windows 均可），并确认：

- Agent 支持执行 shell 命令（能跑 `ssh`、`nft`、`curl`）；
- 你可以随时看到它执行的命令，并在需要时打断。

### 1.2 SSH 连接路由器

```sh
# 本地电脑
ssh root@192.168.10.1 'uname -a; nft -v'
```

建议先配好免密（密钥），否则 Agent 会在密码提示处卡住。给 Agent 的账号应当是**能执行 nft 的管理员账号**，但不要给它不必要的权限。

### 1.3 SSH 连接 NAS（如果支持）

```sh
# 本地电脑
ssh <用户>@192.168.10.10 'uname -a; docker ps | grep -i lucky'
```

有些 NAS（如 QNAP）默认只允许特定账号 SSH，先确认可用。Lucky 的面板操作通常仍需你自己点，Agent 一般只能读日志/进程。

### 1.4 检查依赖命令

```sh
# 路由器
command -v nft conntrack curl ss tcpdump
# NAS
command -v curl ss docker
```

缺 `conntrack` 或 `tcpdump` 不影响主流程，但会降低排障能力。

### 1.5 权限控制与脱敏

- 不要让 Agent 读取或回显：订阅链接、Token、面板密码、私钥、公网 IP 的完整值（报告里可打码）；
- 不要给 Agent 无限制的 sudo / root 常驻会话，按需授权；
- 分享 Agent 报告前，先检查是否包含内网地址、公网地址、账号；
- 让 Agent 把「执行的命令」和「输出」分开列，便于你复核。

---

## 第二部分：先检测（只读 Prompt）

把下面整段发给 Agent。它只做只读采集，最后给出一份判断报告。

```text
你是我的网络排障助手。目标：判断我家是否适用「OpenClash + Lucky IPv4-TCP STUN」方案。
现在只做只读检查，不要修改任何文件、规则或服务。

环境：
- 路由器：root@192.168.10.1（OpenWrt / iStoreOS）
- NAS：192.168.10.10（可选，若能 SSH）
- Lucky 面板：http://192.168.10.10:<端口>

请在路由器上执行并记录输出：
1. uname -a; cat /etc/openwrt_release 2>/dev/null | head -5
2. nft -v; nft list tables | grep fw4
3. nft -a list chain inet fw4 openclash
4. ls -l /etc/openclash/custom/openclash_custom_firewall_rules.sh
5. grep -n 'openclash_custom_firewall_rules' /etc/init.d/openclash /usr/share/openclash/*.sh 2>/dev/null | head
6. ip -4 addr show | grep -E 'inet |^[0-9]+:'; ip route
7. nft list chain inet fw4 upnp_forward 2>/dev/null | head -20

请在 NAS 上执行（若可 SSH）：
1. ss -lntp | grep -E ':(8096|35080)'
2. curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8096/
3. docker ps --format '{{.Names}} {{.Image}} {{.Ports}}' | grep -i lucky

输出要求：
- 逐条列出命令与关键输出（脱敏内网/公网地址后可保留网段）；
- 明确回答：Firewall4？有 openclash 链？接管方式是 REDIRECT 还是 TUN/TPROXY？自定义脚本是否会被调用？
- 给出结论：适用 / 不适用 / 需要补充信息，并说明依据；
- 不要给修改建议，不要执行任何写操作。
```

---

## 第三部分：再配置（授权后最小改动）

确认检测报告为「适用」后，再发这段。Agent 必须先备份、先给出改动内容，得到你确认后才写入。

```text
基于刚才的检测结果，帮我添加 OpenClash 绕过规则。严格按下面约束执行：

参数（如与实际不符请先问我）：
- NAS 局域网 IP：192.168.10.10
- Lucky 监听端口：35080（TCP）
- 规则注释标记：lucky_stun_35080

流程：
1. 备份 /etc/openclash/custom/openclash_custom_firewall_rules.sh 到带时间戳的 .bak 文件，并告诉我备份路径；
2. 读取现有脚本，找到 exit 0 的位置；
3. 生成要插入的片段（不要直接写文件），先贴给我看：
   - 检查 inet fw4 openclash 链是否存在；
   - 用 grep 注释标记防止重复添加；
   - nft insert rule inet fw4 openclash ip saddr 192.168.10.10 tcp sport 35080 counter return comment 'lucky_stun_35080'；
   - 片段必须在 exit 0 之前；
4. 等我明确回复“确认写入”后，再修改文件；
5. 写入后不要重启设备，先只读检查 nft 链里规则顺序是否正确；
6. 如果链不存在、脚本里没有 exit 0、或已有同类规则，立刻停止并报告，不要自行猜测。

禁止：清空任何 nft 链、覆盖整个脚本、改动其他服务、修改 OpenClash 主配置、重启路由器。
```

---

## 第四部分：自动验收

```text
现在验收刚才的配置，只读为主，按顺序检查并逐项给结论：

1. nft -a list chain inet fw4 openclash | grep -E 'lucky_stun|redirect'
   → 绕过规则是否在 REDIRECT 之前？贴出 handle 与顺序。
2. nft list chain inet fw4 upnp_forward 2>/dev/null | grep 35080
   nft list chain inet fw4 upnp_prerouting 2>/dev/null | grep 35080
   → 入站映射是否存在且指向 NAS？
3. 查看 counter：nft -a list chain inet fw4 openclash | grep lucky_stun
   → 记录当前包数；让我触发一次测试后再看是否增长。
4. 让我执行一次 OpenClash 重启，然后复查规则是否仍在。
5. 外网验证：如果你无法直接访问外部网络，明确写“未验证”，不要根据内网结果推断成功。

输出要求：
- 用表格列出：检查项 / 命令 / 结果 / 结论（通过/失败/未验证）；
- 任何一项“未验证”都要说明原因；
- 不要为了给出“成功”结论而省略失败项。
```

---

## 第五部分：出问题如何恢复

```text
现在回退 OpenClash 绕过规则，按最小影响处理：

1. 先在 nft -a list chain inet fw4 openclash 里找到 comment 为 lucky_stun_35080 的规则，记录当前 handle；
2. 用该 handle 删除单条规则：nft delete rule inet fw4 openclash handle <handle>；
3. 从 /etc/openclash/custom/openclash_custom_firewall_rules.sh 中删除对应的 if...fi 片段；
4. 复查：nft list chain inet fw4 openclash | grep -c lucky_stun 应为 0；
5. 确认上网与代理正常；
6. 报告：删了哪条规则、改了哪几行、备份文件在哪、如何再次恢复。

禁止：清空整条 openclash 链、恢复整个 OpenClash 配置、重启设备。
```

---

## Agent 的工作流程

```text
检测 → 分析 → 备份 → 用户授权 → 最小配置 → 验证 → 回退方案 → 输出报告
```

| 阶段 | Agent 做什么 | 人做什么 |
| --- | --- | --- |
| 检测 | 只读采集环境，判断是否适用 | 提供正确的地址与访问方式 |
| 分析 | 说明为什么适用/不适用，指出风险 | 复核结论，决定是否继续 |
| 备份 | 复制脚本并记录路径 | 确认备份存在 |
| 授权 | 给出要写入的片段，等待确认 | 明确回复“确认写入” |
| 最小配置 | 只加一条规则，不改其他 | 观察是否断网 |
| 验证 | 逐项检查并标注未验证项 | 用真实外网环境测试 |
| 回退 | 给出删除步骤并确认生效 | 确认服务恢复 |
| 报告 | 汇总改了什麼、验证了什么、没验证什么 | 归档 |

## 注意事项

- **不要让 Agent 在没有备份的情况下改防火墙**；
- **不要把“规则存在”当成“穿透成功”**；
- **没有外网测试能力时，必须写“未验证”**，而不是“应该可以”；
- Agent 的输出可能包含内网拓扑，分享前先脱敏。

---

相关文档：[手动配置教程](manual-setup.md) · [技术说明与常见问题](reference.md)
