# Surge (Mac)

适用于 Surge Mac 6.x。Surge 的链式写法是给落地节点设置 `underlying-proxy`（界面上叫「跳板策略」），规则集用配置文件里的内联规则集 `[Ruleset AI]`，不依赖外部链接。

下面以文本方式修改配置：用「在文本编辑器中编辑」打开当前配置文件。

> **托管配置是只读的。** 如果你用的是机场提供的托管配置，需要先新建一个本地配置，再通过 `policy-path` 引用机场订阅，见第 2 步。

## 1. 落地 B

在 `[Proxy]` 段加入：

```ini
[Proxy]
AI-Outbound = socks5, 1.2.3.4, 1080, your-username, your-password, underlying-proxy=AI-Inbound, test-url=https://www.anthropic.com
```

如果用图形界面添加：在 **策略** 页面新建一个 SOCKS5 代理，填好服务器、端口、用户名和密码，然后把「跳板策略」选为 `AI-Inbound`。

`test-url` 用于测延迟。B 的出口只放行 AI 域名，用全局默认的测试地址测会被 B 拒绝，所以这里单独指定一个白名单内的 AI 域名。

## 2. 前置组和出口组

在 `[Proxy Group]` 段加入：

```ini
[Proxy Group]
AI-Inbound = fallback, include-all-proxies=true, policy-regex-filter=美国|🇺🇸|\bUS\b|United States
AI-Chain = select, AI-Outbound
```

`AI-Inbound` 的节点来源取决于机场节点放在哪里，选择对应的写法：

| 机场节点在哪 | 写法 |
|---|---|
| 直接写在本配置的 `[Proxy]` 段 | `include-all-proxies=true` |
| 外部订阅链接 | `policy-path=https://example.com/your-subscription` |
| 已有的某个策略组 | `include-other-group="机场组名"` |

三种写法都可以和 `policy-regex-filter` 一起使用。注意以下几点：

- `underlying-proxy` 可以指向策略组，所以前置节点故障时会自动切换，出口 IP 不变。
- `include-all-proxies` 也会把 `AI-Outbound` 纳入候选。落地节点的名字**不要包含**能匹配过滤条件的词，比如 `US`，否则会形成拨号环。
- 正则不要用不区分大小写的 `US`，否则会匹配到 Russia、Australia 这类节点名。

## 3. 规则集

在 `[Rule]` 段的**最前面**加入引用，并在文件中任意位置新增 `[Ruleset AI]` 段：

```ini
[Rule]
RULE-SET,AI,AI-Chain
# …原有规则…

[Ruleset AI]
DOMAIN-SUFFIX,anthropic.com
DOMAIN-SUFFIX,claude.ai
DOMAIN-SUFFIX,claude.com
DOMAIN-SUFFIX,openai.com
DOMAIN-SUFFIX,chatgpt.com
DOMAIN-SUFFIX,sora.com
DOMAIN-SUFFIX,oaistatic.com
DOMAIN-SUFFIX,oaistatsig.com
DOMAIN-SUFFIX,oaiusercontent.com
DOMAIN-SUFFIX,openaiapi-site.azureedge.net
```

Surge 解析 `RULE-SET` 后面的名字时，依次查找内置规则集（`SYSTEM`、`LAN`）、内联规则集、外部资源。所以内联规则集不要取这两个保留名。`DOMAIN-SUFFIX` 同时匹配域名本身和子域名，等价于 v2rayN 规则中的 `domain:`。

## 4. UDP 与 QUIC

Surge 6 的默认行为已经能防止 UDP 泄漏，一般无需额外配置：

- **代理策略默认阻止 QUIC**：不写 `block-quic` 时，代理策略默认会阻止 QUIC，浏览器会回落到 TCP。想明确一点，可以在 `AI-Outbound` 那一行加上 `block-quic=on`。
- **SOCKS5 默认不转发 UDP**（`udp-relay` 默认值为 `false`）。
- UDP 流量命中不支持 UDP 的策略时，Surge Mac 6.0 起默认**拒绝**（`udp-policy-not-supported-behaviour` 默认值为 `REJECT`）。

只需检查自己的 `[General]` 段**没有**把 `udp-policy-not-supported-behaviour` 设为 `DIRECT`。设为 `DIRECT` 的话，命中 AI 规则的 UDP 流量会直接连接。

## 5. 校验与检查

保存前可以先离线校验配置文件。引用不存在的策略会直接报错，例如 `Proxy AI-Outbound used an unknown underlying proxy`：

```bash
surge-cli --check ~/path/to/your.conf
```

加载配置后，用 surge-cli 检查：

```bash
surge-cli rule explain chatgpt.com
```

`rule explain` 会列出命中的规则和每一层策略组的选择结果，包括跳板链路。应该能看到命中 `RULE-SET,AI`，`AI-Chain` 解析到 `AI-Outbound`，跳板为 `AI-Inbound`。

```bash
surge-cli http probe https://www.anthropic.com AI-Chain
```

`http probe` 会通过 `AI-Chain` 发送一个请求，并显示状态码、耗时和所用策略。能正常返回，说明 A → B 整条链路是通的。

不要用 `test-policy-external-ip` 查出口 IP。它访问的是非 AI 的 IP 查询服务，会被 B 的白名单拒绝。出口 IP 请按 [verify.md](verify.md) 用 AI 域名的 trace 地址确认。

在图形界面中，也可以打开 **活动 / 请求**，查看 ChatGPT 或 Claude 的请求所用的策略。

## 终端与 CLI 工具

Claude Code、Codex CLI 等命令行工具不读取系统代理。可以开启增强模式，也可以在终端设置环境变量：

```bash
export https_proxy=http://127.0.0.1:6152 http_proxy=http://127.0.0.1:6152 all_proxy=socks5://127.0.0.1:6153
```

端口以 Surge 设置中的 HTTP / SOCKS5 代理端口为准。
