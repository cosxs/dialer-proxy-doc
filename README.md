# AI 流量链式代理配置指南

只让 AI 域名经「前置节点 A → 美国住宅 SOCKS5 落地 B」访问 OpenAI / Anthropic，其余流量照常分流。

```mermaid
flowchart LR
    U[本机客户端] -->|AI 域名| A[前置 A<br/>机场美国节点]
    A -->|SOCKS5| B[落地 B<br/>美国住宅 IP]
    B --> S[(OpenAI / Anthropic)]
    U -->|其他流量| P[原有分流规则]
```

## 为什么要链式

- **为什么要 B**：AI 服务对机房 IP、多人共用的机场 IP 风控严格；固定的住宅 IP 更稳定，也更少触发验证。
- **为什么要经过 A**：SOCKS5 没有加密，从本机直连 B 时，认证信息和目标域名都是明文，容易被识别和阻断。所以先经 A 加密出境，再由 A 连接 B。
- **为什么只走 AI 流量**：为了保证 IP 纯净、防止滥用。B 的家宽网关出口只放行 AI 相关域名，其他流量即使发到 B 也会被拒绝。因此客户端的 AI 规则集必须和 B 的白名单保持一致。

## 命名约定

各篇文档统一使用下面三个名字，AI 规则统一指向 `AI-Chain`：

| 名称 | 角色 | 说明 |
|---|---|---|
| `AI-Inbound` | 前置 A | 策略组，从订阅中筛选美国节点，故障自动切换 |
| `AI-Outbound` | 落地 B | 住宅 SOCKS5 节点，经 `AI-Inbound` 拨号 |
| `AI-Chain` | 规则出口 | AI 规则指向的策略，实际出口 IP 是 B |

前置切换不会改变出口 IP，所以 `AI-Inbound` 用故障转移即可，不必手动固定某个节点。

## 占位符

文档中的节点信息都是占位符，使用时替换为自己的值：

| 占位符 | 含义 |
|---|---|
| `1.2.3.4` | B 的服务器地址 |
| `1080` | B 的端口 |
| `your-username` / `your-password` | B 的认证信息 |

对应的链接格式为 `socks5://your-username:your-password@1.2.3.4:1080`。

## 客户端

| 客户端 | 内核 | 链式写法 | 添加方式 | 文档 |
|---|---|---|---|---|
| v2rayN 7.15+ | Xray / sing-box | 「链式代理」配置 | 图形界面 + 导入规则 | [docs/v2rayn.md](docs/v2rayn.md) |
| Clash Verge Rev 2.x | mihomo | `dialer-proxy` | 订阅的编辑器 | [docs/clash-verge-rev.md](docs/clash-verge-rev.md) |
| Clash Party 2.x | mihomo | `dialer-proxy` | YAML 覆写 | [docs/clash-party.md](docs/clash-party.md) |
| FlClash 0.8.x | mihomo | `dialer-proxy` | 覆写脚本 | [docs/flclash.md](docs/flclash.md) |
| Bettbox 1.19.x | mihomo | `dialer-proxy` | 全局脚本 | [docs/bettbox.md](docs/bettbox.md) |
| sing-box 1.14.x | sing-box | `detour` | 编辑 JSON 配置 | [docs/sing-box.md](docs/sing-box.md) |
| Surge Mac 6 | Surge | `underlying-proxy`（跳板策略） | 编辑配置文本 | [docs/surge.md](docs/surge.md) |

四个 mihomo 内核的客户端，最终生成的配置相同，原理和注意事项统一写在 [docs/mihomo.md](docs/mihomo.md)。

配置完成后，按 [docs/verify.md](docs/verify.md) 确认出口 IP 并排查问题。
