# Clash Verge Rev

适用于 Clash Verge Rev 2.x（mihomo 内核，菜单名称以 2.5 为准）。所有修改都通过订阅的「编辑节点 / 编辑代理组 / 编辑规则 / 扩展覆写配置」完成。这些修改保存在订阅文件之外，更新订阅后不会丢失。

入口：**订阅** 页面 → 右键当前使用的订阅。

各项配置的原理和注意事项见 [mihomo.md](mihomo.md)，本文只讲在 Clash Verge Rev 里怎么操作。

> 编辑器里的「添加前置代理节点」「添加前置代理组」「添加前置规则」，指的是插入到列表开头，与链式代理里的前置节点无关。

## 按顺序操作

mihomo 加载配置时会检查所有引用。引用了不存在的组、节点或规则集合，整份配置都会加载失败。所以必须按下面的顺序操作，保证每一步保存后配置都是完整的：

| 步骤 | 位置 | 内容 | 依赖 |
|---|---|---|---|
| 1 | 扩展覆写配置 | 规则集合 `ai-chain` | — |
| 2 | 编辑代理组 | 前置组 `AI-Inbound` | 订阅节点 |
| 3 | 编辑节点 | 落地 `AI-Outbound` | `AI-Inbound` |
| 4 | 编辑代理组 | 出口组 `AI-Chain` | `AI-Outbound` |
| 5 | 编辑规则 | AI 规则 | `ai-chain`、`AI-Chain` |

## 1. 规则集合（扩展覆写配置）

右键订阅 → **扩展覆写配置**，加入下面的内容。如果文件里已经有 `rule-providers`，把 `ai-chain` 合并进去：

```yaml
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
```

`+.` 匹配域名本身及所有子域名，等价于 v2rayN 规则中的 `domain:`。

## 2. 前置组 AI-Inbound（编辑代理组）

右键订阅 → **编辑代理组**。可以用表单填写：

| 字段 | 值 |
|---|---|
| 代理组类型 | `fallback` |
| 引入所有出站代理、代理集合 | 开启 |
| 过滤节点 | `美国\|🇺🇸\|\bUS\b\|United States` |
| 排除节点 | `^AI-` |

也可以切换到 **高级** 模式，直接写入：

```yaml
append:
  - name: AI-Inbound
    type: fallback
    include-all: true
    filter: '美国|🇺🇸|\bUS\b|United States'
    exclude-filter: '^AI-'
    url: https://www.gstatic.com/generate_204
    interval: 300
```

排除节点不能省，正则要用单引号，原因见 [mihomo.md](mihomo.md#排除节点)。

## 3. 落地 B（编辑节点）

右键订阅 → **编辑节点**。把下面的链接粘贴到输入框，点 **添加前置代理节点**：

```
socks5://your-username:your-password@1.2.3.4:1080#AI-Outbound
```

链接里无法表达链式关系，所以还要切到 **高级** 模式，给这个节点补上 `udp` 和 `dialer-proxy` 两行。补完后应该是这样：

```yaml
prepend:
  - name: AI-Outbound
    type: socks5
    server: 1.2.3.4
    port: 1080
    username: your-username
    password: your-password
    udp: false
    dialer-proxy: AI-Inbound
```

`dialer-proxy: AI-Inbound` 表示通过 AI-Inbound 去连接这个节点，这就是 mihomo 的链式写法。

## 4. 出口组 AI-Chain（编辑代理组）

再次打开 **编辑代理组**，追加：

```yaml
  - name: AI-Chain
    type: select
    url: https://www.anthropic.com
    proxies:
      - AI-Outbound
```

为什么要多包一层组：「编辑规则」的「代理策略」下拉框只列出代理组，不列单个节点。包一层之后，表单里就能直接选；在「代理」页面也能看到它，并测试整条链的延迟。

`url` 用于测延迟，必须是 B 白名单内的 AI 域名，见 [mihomo.md](mihomo.md#测试地址)。

## 5. AI 规则（编辑规则）

右键订阅 → **编辑规则** → **高级**：

```yaml
prepend:
  - AND,((RULE-SET,ai-chain),(NETWORK,UDP)),REJECT
  - RULE-SET,ai-chain,AI-Chain
```

第二条也可以用表单添加：规则类型选「引用规则集合 (RULE-SET)」，规则内容选 `ai-chain`，代理策略选 `AI-Chain`，然后点 **添加前置规则**。第一条是 AND 组合规则，只能在高级模式里写。

第一条不能省：没有它，AI 域名的 QUIC 流量会绕过链路，从机场节点出去，详见 [mihomo.md](mihomo.md#udp-拒绝规则不能省)。

## 6. 应用并检查

1. 确认这个订阅处于「使用」状态。
2. 打开 **代理** 页面：
   - `AI-Inbound` 里应该列出美国节点。如果只有 `COMPATIBLE`，说明过滤条件没匹配到节点，B 会被本机直接连接，见 [mihomo.md](mihomo.md#前置组必须筛到节点)。
   - `AI-Chain` 组里应该是 `AI-Outbound`。对它测延迟，测的是 A → B 整条链路，测试地址是第 4 步设置的 `url`。
3. 打开 ChatGPT 或 Claude，在 **连接** 页面查看对应域名：「规则」一列应为 `RuleSet(ai-chain)`，「链路」一列应包含 `AI-Chain` 和 `AI-Outbound`。
4. 按 [verify.md](verify.md) 确认出口 IP。

## 可选：TUN 模式下开启嗅探

使用虚拟网卡（TUN）模式，并且浏览器开启了自定义 DoH 时，需要开启嗅探。把 [mihomo.md](mihomo.md#tun-模式下开启嗅探) 中的 `sniffer` 配置加到 **扩展覆写配置** 里即可。

## 终端与 CLI 工具

Claude Code、Codex CLI 等命令行工具不读取系统代理。可以开启虚拟网卡模式（需要先安装服务模式），也可以在终端设置环境变量：

```bash
export https_proxy=http://127.0.0.1:7897 http_proxy=http://127.0.0.1:7897 all_proxy=socks5://127.0.0.1:7897
```

端口以 **设置 → 端口设置** 中的混合代理端口为准。
