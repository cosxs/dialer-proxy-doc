# FlClash

适用于 FlClash 0.8.x（mihomo 内核，菜单名称以 0.8.99 为准）。

FlClash 每个订阅只能选一种「覆写模式」，其中只有 **脚本** 能添加带 `dialer-proxy` 的节点和规则集合，所以本文使用脚本。另外两种模式都不适用：

- **标准**：只能追加规则，不能引用规则集合，也不能添加节点和代理组。
- **自定义**：会**整体替换**订阅的代理组和规则列表，之后订阅更新时不会跟着变。

各项配置的原理和注意事项见 [mihomo.md](mihomo.md)，本文只讲在 FlClash 里怎么操作。

## 1. 新建脚本

**工具 → 进阶配置 → 脚本 → 添加 → 从零开始**，粘贴下面的脚本，点保存图标，输入脚本名称（例如 `AI-Chain`）：

```js
function main(config) {
  const AI_DOMAINS = [
    '+.anthropic.com',
    '+.claude.ai',
    '+.claude.com',
    '+.openai.com',
    '+.chatgpt.com',
    '+.sora.com',
    '+.oaistatic.com',
    '+.oaistatsig.com',
    '+.oaiusercontent.com',
    '+.openaiapi-site.azureedge.net',
  ];
  const OWN = ['AI-Outbound', 'AI-Inbound', 'AI-Chain'];
  const list = (v) => (Array.isArray(v) ? v : []);

  config.proxies = list(config.proxies).filter((p) => !OWN.includes(p.name));
  config.proxies.push({
    name: 'AI-Outbound',
    type: 'socks5',
    server: '1.2.3.4',
    port: 1080,
    username: 'your-username',
    password: 'your-password',
    udp: false,
    'dialer-proxy': 'AI-Inbound',
  });

  config['proxy-groups'] = list(config['proxy-groups']).filter((g) => !OWN.includes(g.name));
  config['proxy-groups'].push(
    {
      name: 'AI-Inbound',
      type: 'fallback',
      'include-all': true,
      filter: '美国|🇺🇸|\\bUS\\b|United States',
      'exclude-filter': '^AI-',
      url: 'https://www.gstatic.com/generate_204',
      interval: 300,
    },
    {
      name: 'AI-Chain',
      type: 'select',
      url: 'https://www.anthropic.com',
      proxies: ['AI-Outbound'],
    },
  );

  config['rule-providers'] = Object.assign({}, config['rule-providers'], {
    'ai-chain': { type: 'inline', behavior: 'domain', payload: AI_DOMAINS },
  });

  config.rules = [
    'AND,((RULE-SET,ai-chain),(NETWORK,UDP)),REJECT',
    'RULE-SET,ai-chain,AI-Chain',
    ...list(config.rules).filter((r) => !r.includes('RULE-SET,ai-chain')),
  ];
  return config;
}
```

脚本要点：

- 正则在 JS 字符串里要写成 `\\b`。写成 `\b` 会变成退格字符，过滤条件永远匹配不上。
- 订阅没有规则集合时，FlClash 传入的 `rule-providers` 是 `null`，脚本已做处理。
- 脚本会先删除同名的节点、组和规则，再重新添加，重复执行结果不变。

## 2. 应用到订阅

**配置** → 订阅卡片的 **⋮** → **更多** → **覆写**：

1. 「覆写模式」选择 **脚本**。
2. 在「覆写脚本」中选中刚建的脚本。
3. 点右上角 **预览**，确认合并后的配置包含 [mihomo.md](mihomo.md#最终生成的配置) 列出的全部内容，并且两条 AI 规则排在最前面。

离开页面后配置会自动重新应用。

## 3. 检查

1. 打开 **代理** 页面：
   - `AI-Inbound` 里应该列出美国节点，不能只有 `COMPATIBLE`。
   - 对 `AI-Chain` 测延迟，测的是 A → B 整条链路。FlClash 会使用组上的 `url`；组上没设时，才使用 **工具 → 常规 → 连接 → 测速链接**。
2. 打开 ChatGPT 或 Claude，在 **连接** 页面点开对应连接：规则为 `RuleSet(ai-chain)`，代理链为 `AI-Chain → AI-Outbound`。连接列表里还会多出一条发往 `1.2.3.4:1080` 的连接，没有对应规则。这是 A 连接 B 的那一段，属于正常现象。
3. 按 [verify.md](verify.md) 确认出口 IP。

## FlClash 特有的注意事项

- **脚本出错会影响所有流量。** 脚本抛出异常，或生成的配置被 mihomo 拒绝（例如名字重复、`dialer-proxy` 指向不存在的组）时，FlClash 会推送一份空配置，**所有流量**都会受影响，不只是 AI 流量。修改脚本后，先用「预览」确认。
- **出站模式必须是「规则」。** FlClash 会用应用里的「出站模式」覆盖订阅的设置。
- **附加规则不再生效。** 使用脚本模式后，「标准」模式下的附加规则（包括全局附加规则）都不会生效。
- **部分设置写在脚本里无效。** 端口、TUN、DNS 等由应用设置决定，脚本里改了也会被覆盖。

## 终端与 CLI 工具

Claude Code、Codex CLI 等命令行工具不读取系统代理。可以开启 TUN 模式，也可以在终端设置环境变量：

```bash
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890
```

端口以 **工具 → 常规 → 入站 → 端口 → 混合端口** 为准。
