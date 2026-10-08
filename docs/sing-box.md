# sing-box

适用于 sing-box 1.14.x。文中配置已在 sing-box 1.14.2 上通过 `sing-box check` 校验，并用模拟的 A、B 实际运行验证过。

sing-box 的链式写法是给 socks 出站设置 `detour`。

> **sing-box 不能从订阅合并节点。** 1.14 没有类似 `proxy-providers` 的功能，机场的美国节点必须作为出站写在同一份配置里。通常的做法是：拿到机场提供的 sing-box 配置，把下面的片段合并进去，再作为**本地**配置导入客户端。远程配置在官方客户端里是只读的，无法修改。

## 1. 出站

在 `outbounds` 数组的**末尾**加入下面三个出站。如果加在开头，又没有设置 `route.final`，sing-box 会把第一个出站当作默认出站。

```json
{
  "type": "urltest",
  "tag": "AI-Inbound",
  "outbounds": ["US-01", "US-02"],
  "url": "https://www.anthropic.com",
  "interval": "3m",
  "interrupt_exist_connections": false
},
{
  "type": "socks",
  "tag": "AI-Outbound",
  "server": "1.2.3.4",
  "server_port": 1080,
  "version": "5",
  "username": "your-username",
  "password": "your-password",
  "network": "tcp",
  "detour": "AI-Inbound"
},
{
  "type": "selector",
  "tag": "AI-Chain",
  "outbounds": ["AI-Outbound"]
}
```

各字段的要点：

- **`US-01`、`US-02`**：换成配置里机场美国节点的 tag。sing-box 不支持用正则筛选节点，需要逐个列出。
- **`detour`**：表示通过 `AI-Inbound` 去连接 B。它可以指向单个出站，也可以指向组。
- **`network: "tcp"`**：B 不支持 UDP。不写的话，UDP 流量会尝试经 A 向 B 发起 UDP 转发，结果是失败。
- **`urltest`**：sing-box 没有 mihomo 那样的 `fallback`（故障转移）类型，`urltest` 会自动选延迟最低的节点。`interrupt_exist_connections: false` 能保证切换节点时，正在进行的 AI 流式响应不会被中断。

## 2. 规则集与路由规则

在 `route.rule_set` 中加入内联规则集：

```json
{
  "type": "inline",
  "tag": "ai-domains",
  "rules": [
    {
      "domain_suffix": [
        "anthropic.com",
        "claude.ai",
        "claude.com",
        "openai.com",
        "chatgpt.com",
        "sora.com",
        "oaistatic.com",
        "oaistatsig.com",
        "oaiusercontent.com",
        "openaiapi-site.azureedge.net"
      ]
    }
  ]
}
```

`domain_suffix` 会同时匹配域名本身和所有子域名，等价于 v2rayN 规则中的 `domain:`。

在 `route.rules` 中加入两条规则，放在原有的 `sniff`、`hijack-dns` 规则**之后**、其他所有规则**之前**：

```json
{ "rule_set": "ai-domains", "network": "udp", "action": "reject" },
{ "rule_set": "ai-domains", "action": "route", "outbound": "AI-Chain" }
```

各规则的要点：

- **使用 TUN 时，前面必须有 `{ "action": "sniff" }`。** 没有它的话，TUN 收到的连接只有 IP，匹配不到域名。
- **第一条用来拒绝 UDP。** sing-box 不会像 mihomo 那样让 UDP 落到后面的规则：没有这一条时，UDP 流量命中 `AI-Chain` 后会直接失败。加上这一条，浏览器的 QUIC 会被立即拒绝，然后马上回落到 TCP。实测日志：`match[2] network=udp rule_set=ai-domains => reject`。
- **AI 规则要放在所有 `resolve` 规则之前。** `resolve` 会先把域名解析成 IP，排在它后面的话，B 收到的就是 IP 而不是域名。

## 3. TUN 模式：让 B 收到域名

使用系统代理（mixed / http / socks 入站）时，应用会把域名交给 sing-box，B 收到的也是域名。实测确认 B 收到的地址类型为域名。

使用 **TUN 并且 DNS 返回真实 IP** 时，情况不同：嗅探到的域名只用于规则匹配，sing-box 1.13 起已经移除了 `sniff_override_destination`，所以 B 收到的是 **IP**。如果 B 的家宽网关是按 SOCKS 请求里的域名做白名单判断，这些请求会被拒绝。

解决办法是让 AI 域名走 FakeIP，sing-box 会把假 IP 还原成域名再发给 B：

```json
"dns": {
  "servers": [
    { "type": "local", "tag": "local" },
    { "type": "fakeip", "tag": "fakeip", "inet4_range": "198.18.0.0/15", "inet6_range": "fc00::/18" }
  ],
  "rules": [
    { "rule_set": "ai-domains", "query_type": ["A", "AAAA"], "action": "route", "server": "fakeip" }
  ],
  "final": "local"
},
"route": {
  "default_domain_resolver": "local"
},
"experimental": {
  "cache_file": { "enabled": true, "store_fakeip": true }
}
```

需要注意：

- `local` 换成你配置里原有的普通 DNS 服务器 tag，`servers` 和 `rules` 与原有内容合并。
- 配置了多个 DNS 服务器时，sing-box 1.14 **必须**设置 `route.default_domain_resolver`，否则启动时报错。它要指向普通 DNS 服务器，不能指向 `fakeip`。

## 4. 校验与检查

**`sing-box check` 只检查格式，不检查引用。** 节点 tag 写错、`detour` 指向不存在的出站，`check` 都能通过，运行时才报错。实测：引用一个不存在的 `US-03`，`check` 返回成功，`run` 时才报 `dependency[US-03] not found`。

所以校验完格式后，要再实际走一遍链路：

```bash
sing-box check -c config.json
```

```bash
sing-box tools fetch -c config.json -o AI-Chain https://www.anthropic.com
```

`tools fetch` 只启动出站，不启动入站，所以不影响正在运行的 sing-box。它能正常返回网页内容，说明 A → B 整条链路是通的。

把日志级别设为 `debug` 后，访问 AI 网站时应该能看到类似下面的日志：

```
router: match[3] rule_set=ai-domains => route(AI-Chain)
outbound/socks[AI-Outbound]: outbound connection to claude.ai:443
outbound/shadowsocks[US-01]: outbound connection to 1.2.3.4:1080
```

`match[3]` 中的序号取决于规则的位置；最后一行的 `shadowsocks` 取决于机场节点的协议类型。

关于延迟测试：

- 在图形客户端里测 `AI-Chain` 时，用的是默认测试地址（`gstatic.com`），会被 B 的白名单拒绝，`AI-Outbound` 会显示失败。这只影响显示，不影响 AI 流量走链路。
- `AI-Inbound` 测的是 A 本身，使用组上设置的 `url`。
- 用 Clash API 面板测延迟时，把测试地址设为 `https://` 开头的 AI 域名。

最后按 [verify.md](verify.md) 确认出口 IP。

## 5. 在官方客户端中导入

**macOS / iOS**：

1. **仪表** → **配置** → **新建配置**。
2. 「类型」选 **本地**，「文件」选 **导入**（或 **新建** 后粘贴），点 **创建**。
3. 要修改时：**编辑** → **编辑配置** → **编辑内容** → **保存**。

**Android**：

1. **仪表** → **配置** → **添加配置文件** → **从文件导入** 或 **手动创建**。
2. 要修改时：**编辑** → **编辑配置** → **JSON 编辑器**。

远程配置只能查看，不能修改。要改的话，需要先改托管在服务器上的配置文件。

## 终端与 CLI 工具

Claude Code、Codex CLI 等命令行工具不读取系统代理。可以开启 TUN，也可以把环境变量指向配置里 mixed 入站的端口：

```bash
export https_proxy=http://127.0.0.1:2080 http_proxy=http://127.0.0.1:2080 all_proxy=socks5://127.0.0.1:2080
```

`2080` 换成你配置里 mixed 入站的 `listen_port`。
