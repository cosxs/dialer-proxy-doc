# Clash Party

适用于 Clash Party 2.x（mihomo 内核，菜单名称以 2.0.3 为准）。Clash Party 的「覆写」可以把整套配置写成一个 YAML 文件。覆写独立于订阅保存，更新订阅后不会丢失。

各项配置的原理和注意事项见 [mihomo.md](mihomo.md)，本文只讲在 Clash Party 里怎么操作。

## 1. 新建 YAML 覆写

侧边栏 **覆写** → 点 **+** → **新建 YAML**。双击新建的卡片（或右键 → **编辑文件**），把下面的内容整段粘贴进去，点 **确认**：

```yaml
proxies+:
  - name: AI-Outbound
    type: socks5
    server: 1.2.3.4
    port: 1080
    username: your-username
    password: your-password
    udp: false
    dialer-proxy: AI-Inbound

proxy-groups+:
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

+rules:
  - AND,((RULE-SET,ai-chain),(NETWORK,UDP)),REJECT
  - RULE-SET,ai-chain,AI-Chain
```

键名前后的 `+` 决定合并方式：

| 写法 | 效果 |
|---|---|
| `+rules` | 插到订阅原有规则的**前面** |
| `proxies+`、`proxy-groups+` | 追加到订阅原有列表的后面 |
| `rule-providers`（不带符号） | 与订阅的规则集合合并，原有的保留 |

不要写成不带符号的 `rules:`、`proxies:`。列表不带符号时，会**整体替换**订阅原有的内容。

## 2. 应用到订阅

二选一：

- **只对某个订阅生效**：侧边栏 **订阅管理** → 订阅卡片右键 → **编辑信息** → 在「覆写」一栏点 **+** → 选择刚建的覆写 → **保存**。
- **对所有订阅生效**：在覆写卡片上打开 **全局启用**。

保存后会自动重新加载配置。

## 3. 检查

1. 在侧边栏的订阅卡片上点 **查看运行时配置**，确认其中包含 [mihomo.md](mihomo.md#最终生成的配置) 列出的全部内容，并且两条 AI 规则排在最前面。
2. 打开 **代理组与节点** 页面：
   - `AI-Inbound` 里应该列出美国节点，不能只有 `COMPATIBLE`。
   - 对 `AI-Chain` 测延迟，测的是 A → B 整条链路。
3. 打开 ChatGPT 或 Claude，在 **连接** 页面点开对应连接，查看「连接详情」：规则为 `RuleSet(ai-chain)`，代理链为 `AI-Chain>>AI-Outbound`。
4. 按 [verify.md](verify.md) 确认出口 IP。

## Clash Party 特有的注意事项

- **「延迟测试 URL」要留空。** 在 **应用设置** 中填了全局「延迟测试 URL」后，组上的 `url` 会被它覆盖，测 `AI-Chain` 时就会被 B 的白名单拒绝。
- **Smart 内核会破坏链路。** 同时开启「启用 Smart 内核」和「使用自动 Smart 规则覆写」后，会出现两个问题：
  - 它会把 `url-test` 组改名，但不会同步修改 `dialer-proxy` 里的组名，引用会断掉。所以 `AI-Inbound` 要保持 `fallback`，不要改成 `url-test`。
  - 配置里没有 `url-test` / `load-balance` 组时，它会把所有规则的目标改成 Smart 组，`RULE-SET,ai-chain,AI-Chain` 也会被改掉，AI 流量就绕过了链路。

  使用本方案时，建议关闭「使用自动 Smart 规则覆写」。
- **不要用「编辑订阅」改配置。** 这里的修改在更新订阅后会还原。
- **「编辑规则」里添加的规则排得更靠前。** 订阅卡片上的「编辑规则」添加的前置规则，会排在覆写的 `+rules` 之前。如果其中有能匹配 AI 域名的规则，会先于 AI 规则生效。
- **覆写出错会阻止订阅更新。** 更新订阅时，Clash Party 会连同覆写一起校验。覆写有错时，会继续使用旧的订阅内容。
- **出站模式必须是「规则」。** 全局或直连模式下，AI 规则不会生效。

## 终端与 CLI 工具

Claude Code、Codex CLI 等命令行工具不读取系统代理。可以开启 TUN 模式，也可以在终端设置环境变量：

```bash
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890
```

端口以 **内核设置 → 混合端口** 为准。
