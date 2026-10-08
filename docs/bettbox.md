# Bettbox

适用于 Bettbox 1.19.x（菜单名称以 1.19.4 为准）。Bettbox 是基于 FlClash 开发的 mihomo 客户端，支持 Windows、macOS、Linux 和 Android，没有 iOS 版。

和 FlClash 一样，只有 **脚本** 能添加带 `dialer-proxy` 的节点和规则集合。订阅的「覆写」只能改规则，不适用。

各项配置的原理和注意事项见 [mihomo.md](mihomo.md)，本文只讲在 Bettbox 里怎么操作。

## 1. 添加脚本

**配置** 页面右上角的 **脚本** 图标（也可以从 **更多 → 查看 → 脚本** 进入）→ 点 **+** → **通过代码导入**，粘贴下面的脚本后保存：

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

保存后，**打开这个脚本卡片上的开关**。

Bettbox **同一时间只能启用一个脚本**。如果你已经在用别的脚本（比如分流脚本），需要把这段代码合并进去，不能同时启用两个。

正则在 JS 字符串里要写成 `\\b`，原因见 [mihomo.md](mihomo.md#正则写法)。

## 2. 确认订阅使用全局脚本

在脚本页面点右上角的齿轮图标（设置）→「使用全局脚本覆写」，确认当前订阅的开关是打开的。这个开关默认打开。

## 3. 让 AI-Chain 的测试地址生效

Bettbox 的「覆盖配置」默认开启，会把**所有**组的测试地址改成应用设置里的「测试链接」。这样一来，`AI-Chain` 上设的 `https://www.anthropic.com` 就失效了，测延迟会被 B 的白名单拒绝。

到 **更多 → 内核配置 → 常规 → 测试链接**，关闭 **覆盖配置**。这是全局设置，关闭后各组都会使用自己设置的 `url`。

如果不想改这个设置，也可以不关。这只影响延迟显示，不影响 AI 流量走链路。

## 4. 检查

1. 在 **配置** 页面**长按**订阅卡片 → **运行时配置**，确认其中包含 [mihomo.md](mihomo.md#最终生成的配置) 列出的全部内容，并且两条 AI 规则排在最前面。注意：「覆盖配置」对测试地址的改写不会显示在这里。
2. 打开 **代理** 页面：
   - `AI-Inbound` 里应该列出美国节点，不能只有 `COMPATIBLE`。
   - 对 `AI-Chain` 测延迟，测的是 A → B 整条链路。
3. 打开 ChatGPT 或 Claude，在 **连接** 或 **请求** 页面点开对应条目：规则为 `RuleSet(ai-chain)`，代理链包含 `AI-Outbound` 和 `AI-Chain`。另外会多出一条发往 `1.2.3.4:1080` 的连接，这是 A 连接 B 的那一段，属于正常现象。
4. 按 [verify.md](verify.md) 确认出口 IP。

## Bettbox 特有的注意事项

- **脚本出错时会静默回退。** 脚本出错或返回值不对时，Bettbox 会直接使用**原始订阅**。这时 AI 流量走的是机场的规则，网页照样能打开，很难察觉。修改脚本后务必看一眼运行时配置。
- **「节点排除」的正则不能匹配 `AI-Outbound`。** 位置在 **代理 → ⋮ → 进阶设置 → 节点排除**。如果匹配上了，Bettbox 会把 `AI-Outbound` 从 `AI-Chain` 中移除并换成 **DIRECT**，AI 流量会从本机直接连接。
- **被拒绝的 UDP 在连接列表里看不到。** Bettbox 的内核在记录连接之前就拒绝了这些 UDP 流量，所以它们不会出现在「连接 / 请求」中。要查看的话，需要把日志级别调到 info，在 **日志** 里看。
- **启用脚本后，覆写规则不生效。** 订阅「覆写」里的规则在脚本模式下不会生效。
- **出站模式必须是「规则」。**
- **「禁用QUIC」不冲突。** 开启后，它会在最前面再插入一条拒绝 UDP 443 的规则，与本方案不冲突。

## 终端与 CLI 工具

Claude Code、Codex CLI 等命令行工具不读取系统代理。可以开启 TUN 模式，也可以在终端设置环境变量：

```bash
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890
```

端口以 **更多 → 内核配置 → 常规 → 端口 → 混合端口** 为准。
