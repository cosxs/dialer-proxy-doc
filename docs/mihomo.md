# mihomo 内核通用说明

适用于 Clash Verge Rev、Clash Party、FlClash、Bettbox。这四个客户端都用 mihomo 内核，最终生成的配置完全相同，区别只在于从哪里把这些内容加到订阅上：

| 客户端 | 添加方式 | 文档 |
|---|---|---|
| Clash Verge Rev | 订阅的编辑节点 / 编辑代理组 / 编辑规则 / 扩展覆写配置 | [clash-verge-rev.md](clash-verge-rev.md) |
| Clash Party | YAML 覆写 | [clash-party.md](clash-party.md) |
| FlClash | 覆写脚本 | [flclash.md](flclash.md) |
| Bettbox | 全局脚本 | [bettbox.md](bettbox.md) |

## 最终生成的配置

不管用哪个客户端，合并后的配置里都应该有下面这些内容。各客户端都有查看运行时配置的入口，可以对照检查。

```yaml
proxies:
  - name: AI-Outbound
    type: socks5
    server: 1.2.3.4
    port: 1080
    username: your-username
    password: your-password
    udp: false
    dialer-proxy: AI-Inbound

proxy-groups:
  - name: AI-Inbound
    type: fallback
    include-all: true
    filter: '美国|🇺🇸|\bUS\b|United States'
    exclude-filter: '^AI-'
    url: https://www.gstatic.com/generate_204
    interval: 300
  - name: AI-Chain
    type: select
    url: https://www.anthropic.com
    proxies:
      - AI-Outbound

rule-providers:
  ai-chain:
    type: inline
    behavior: domain
    payload:
      - '+.anthropic.com'
      - '+.claude.ai'
      - '+.claude.com'
      - '+.openai.com'
      - '+.chatgpt.com'
      - '+.sora.com'
      - '+.oaistatic.com'
      - '+.oaistatsig.com'
      - '+.oaiusercontent.com'
      - '+.openaiapi-site.azureedge.net'

rules:
  - AND,((RULE-SET,ai-chain),(NETWORK,UDP)),REJECT
  - RULE-SET,ai-chain,AI-Chain
  # …订阅原有的规则…
```

## 要点

### dialer-proxy

`dialer-proxy: AI-Inbound` 表示 `AI-Outbound` 要通过 `AI-Inbound` 去连接，这就是 mihomo 的链式写法。它可以指向单个节点，也可以指向代理组。

### 前置组必须筛到节点

如果 `AI-Inbound` 的过滤条件**一个节点都没匹配上**，mihomo 不会报错，而是把这个组填成 `COMPATIBLE`（直连）。这时 `AI-Outbound` 会被**本机直接连接**，正是我们要避免的情况。

实测：订阅里只有香港节点时，`AI-Inbound` 的成员为 `['COMPATIBLE']`，B 收到的连接来自本机。

配置完成后，务必在「代理」页面确认 `AI-Inbound` 里列出的是美国节点。机场改了节点命名时，也要回来检查一次。

### 排除节点

`include-all` 会把所有节点纳入候选，包括 `AI-Outbound` 自己。`exclude-filter: '^AI-'` 把它排除掉，避免形成「自己经过自己」的拨号环。

反过来也要注意：订阅里如果有使用 `include-all` 的组，同样会把 `AI-Outbound` 纳入。不要在这些组里选中它，否则所有流量都会发往 B，而 B 的白名单会拒绝非 AI 流量。

### UDP 拒绝规则不能省

`AI-Outbound` 不支持 UDP。如果 UDP 连接命中的规则指向一个不支持 UDP 的节点，mihomo **不会拒绝这个连接，而是跳过这条规则、继续往下匹配**。结果是：浏览器访问 ChatGPT 发起的 QUIC（UDP 443）会落到后面的规则，从机场节点出去，出口 IP 和 TCP 连接不一致。

实测：去掉第一条规则后，发往 `chatgpt.com:443` 的 UDP 连接日志为 `match Match using Proxy[DIRECT]`；加上之后为 `match AND(...) using REJECT`。显式拒绝后，浏览器会自动回落到 TCP。

### 测试地址

B 的出口只放行 AI 域名，用默认地址（如 `gstatic.com`）测 `AI-Chain` 会被 B 拒绝，显示超时。所以 `AI-Chain` 的 `url` 必须是白名单内的 AI 域名。mihomo 测延迟发的是 HEAD 请求，收到任何 HTTP 响应都算成功。`https://www.anthropic.com` 会直接返回 200，用来测试最直观。

`AI-Inbound` 测的是 A 本身，不经过 B，用默认测试地址即可。

部分客户端的全局设置会覆盖组上的 `url`，各客户端文档里有说明。

### 正则写法

- 不要用不区分大小写的 `US`，否则会匹配到 Russia、Australia 这类节点名。
- YAML 中正则要用**单引号**。双引号里的 `\b` 会被解析成退格字符。
- JavaScript 字符串中要写成 `\\b`，原因相同。

### TUN 模式下开启嗅探

使用 TUN 模式时，如果浏览器开启了自定义的「安全 DNS」（DoH），浏览器拿到的是真实 IP，连接里没有域名，AI 规则就匹配不到。可以加上 SNI 嗅探来补救：

```yaml
sniffer:
  enable: true
  parse-pure-ip: true
  sniff:
    HTTP:
      ports: [80, 8080-8880]
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]
```

只用系统代理模式时，浏览器会把域名直接交给代理，不需要嗅探。部分客户端在应用设置里自带嗅探开关，例如 Clash Party 的「嗅探覆写」默认开启，这种情况以客户端设置为准。
