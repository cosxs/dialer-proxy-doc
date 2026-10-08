# 验证与排错

## 确认出口 IP

用浏览器打开以下地址。它们由 Cloudflare 提供，会显示访问者的 IP 和所在地区：

| 地址 | 用途 |
|---|---|
| https://chatgpt.com/cdn-cgi/trace | ChatGPT 网页 |
| https://claude.ai/cdn-cgi/trace | Claude 网页 |
| https://api.openai.com/cdn-cgi/trace | OpenAI API |
| https://api.anthropic.com/cdn-cgi/trace | Anthropic API |
| https://www.cloudflare.com/cdn-cgi/trace | 对照组：非 AI 域名 |

预期结果：

- AI 域名的 `ip=` 是 B 的出口 IP，`loc=US`。
- 对照组的 `ip=` 是你平时的出口，可能是机场节点，也可能是本地 IP，取决于原有规则。它与 AI 域名的 IP 不同，才说明分流生效。

住宅代理的网关地址和实际出口 IP 不一定相同，出口 IP 以服务商面板显示的为准。

在终端里检查（需要已开启 TUN，或已设置代理环境变量）：

```bash
curl -s https://api.anthropic.com/cdn-cgi/trace | grep -E '^(ip|loc)='
```

## 出口不是 B

按出现频率从高到低排查：

1. **规则没有放在最前面。** 在客户端的连接 / 请求列表中查看 AI 域名命中的规则。如果命中的是 GEOIP、兜底规则或其他规则，说明 AI 规则排得太靠后。
2. **v2rayN 别名不一致。** 规则的 `outboundTag` 找不到同名配置时，会静默回退到当前活动节点，详见 [v2rayn.md](v2rayn.md#5-检查别名是否生效)。
3. **mihomo 的 UDP 泄漏。** 漏写 `AND,((RULE-SET,ai-chain),(NETWORK,UDP)),REJECT` 时，QUIC 会落到后面的规则，详见 [mihomo.md](mihomo.md#udp-拒绝规则不能省)。
4. **脚本没有生效。** Bettbox 的脚本出错时会静默回退到原始订阅；FlClash 的脚本出错时会推送空配置。在客户端里查看运行时配置或预览确认。
5. **TUN 模式下浏览器用了自定义 DoH。** 浏览器拿到的是真实 IP，连接里没有域名。可以开启嗅探，或关闭浏览器的「安全 DNS」。
6. **改的不是正在使用的配置。** 例如 v2rayN 改了非活动的规则集，或 Clash Verge Rev、Clash Party 改了非当前使用的订阅。
7. **出站模式不是「规则」。** mihomo 系客户端处于全局或直连模式时，AI 规则不会生效。

## 连不上或超时

先单独测前置：在 `AI-Inbound` 里选一个节点，直接访问普通网站。确认 A 正常后，再测 `AI-Chain`。

测 `AI-Chain` 时，测试地址必须是 B 白名单内的 AI 域名，例如 `https://www.anthropic.com`，否则会被 B 拒绝、显示超时。各客户端的情况：

- **mihomo 系客户端、Surge**：文档里已经把 `AI-Chain` 的测试地址设为这个域名。Clash Party 要把全局「延迟测试 URL」留空，Bettbox 要关闭「覆盖配置」，否则组上的设置会被覆盖。
- **v2rayN、sing-box 图形客户端**：测试地址是全局的，测 `AI-Chain` 会显示失败。直接用上面的 trace 地址检查，sing-box 也可以用 `sing-box tools fetch`。

| 现象 | 可能原因 | 处理 |
|---|---|---|
| A 正常，链路超时 | 机场限制了出站端口，不允许连接 B 的端口 | 换一个机场的节点做前置，或请 B 的服务商改用常见端口 |
| 链路立即失败 | B 的用户名或密码错误 | 核对凭据；密码含特殊字符时，导入链接前需要做 URL 编码 |
| 时好时坏 | B 使用 IP 白名单认证 | 链式代理下，B 看到的来源是 A 的出口 IP，而且 A 故障切换后会变。改用用户名密码认证 |
| 网页能开，首次加载慢 | 浏览器先尝试 QUIC，超时后才回落 TCP | 按各客户端文档拒绝 AI 域名的 UDP 443 |
| 主站正常，部分功能失败（如上传、语音） | 该功能用到的域名在规则集里，但不在 B 的白名单内 | 在连接 / 请求列表找到失败的域名，加入 B 的白名单 |
| mihomo 系客户端中 `AI-Inbound` 只有 `COMPATIBLE` | 过滤条件没匹配到任何节点，B 会被本机直接连接 | 按机场的节点命名调整过滤条件，见 [mihomo.md](mihomo.md#前置组必须筛到节点) |
| sing-box 在 TUN 模式下 AI 请求被 B 拒绝 | DNS 返回真实 IP，B 收到的是 IP 而不是域名 | AI 域名改走 FakeIP，见 [sing-box.md](sing-box.md#3-tun-模式让-b-收到域名) |
| sing-box 能启动但 AI 网站打不开 | 出站 tag 写错，`check` 查不出来 | 用 `sing-box tools fetch -o AI-Chain` 实测，看报错 |

## 速度慢

链式代理的延迟是「本机 → A」加上「A → B」。

- A 选择离 B 近的节点。B 在美国就选美国节点，避免选择绕路的中转节点。
- 不要拿低倍率、高负载的节点当前置，它的拥塞会直接拖慢整条链路。

## 规则集遗漏

使用 AI 服务时，打开连接 / 请求列表，筛选同一时间段内**没有**走 `AI-Chain` 的请求，看是否有相关的第三方域名，例如：

- 第三方登录：`accounts.google.com`、`appleid.apple.com`、`login.microsoftonline.com`
- 人机验证：`challenges.cloudflare.com`

是否把它们加入规则集，取决于你是否希望同一会话的所有请求都从 B 出去。需要注意，这些是通用域名，其他网站也会用到。加入后，那些网站的对应请求同样会经过 B，这与「只走 AI 流量、保证 IP 纯净」的初衷相冲突，请按需取舍。

确定要加时，按以下顺序操作：

1. 先把域名加入 B 的家宽网关白名单。只加规则不加白名单，请求会被 B 拒绝。
2. 再同步修改各客户端：
   - v2rayN：重新导入规则
   - Clash Verge Rev、Clash Party：修改规则集合的 `payload`
   - FlClash、Bettbox：修改脚本里的 `AI_DOMAINS`
   - sing-box：修改 `ai-domains` 规则集的 `domain_suffix`
   - Surge：修改 `[Ruleset AI]`

## 安全提示

- SOCKS5 的认证是明文的。A 的运营方能看到你连接 B 时使用的用户名和密码，也能看到你访问的 AI 域名（TLS SNI）。HTTPS 内容本身仍然是加密的。
- 截图或分享配置前，先把节点地址和凭据换成占位符。
