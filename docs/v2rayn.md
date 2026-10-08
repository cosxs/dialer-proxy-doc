# v2rayN

适用于 v2rayN 7.15 及以上版本（菜单名称以 7.25 为准）。整体思路是建三个配置：落地 `AI-Outbound`、前置策略组 `AI-Inbound`、链式代理 `AI-Chain`，再把规则集导入当前路由。主界面的活动节点保持原来的机场节点不变。

## 1. 添加落地 B（SOCKS5）

先在配置列表上方的分组标签中切到「所有」，再添加节点。手动添加的节点会归属当前分组，如果放在订阅分组里，删除该订阅时会被一并删除。

**方式一：导入链接**（推荐）

复制下面的链接（替换占位符），然后选择 **配置项 → 从剪贴板导入分享链接**：

```
socks5://your-username:your-password@1.2.3.4:1080#AI-Outbound
```

`#` 后面的部分就是别名。

**方式二：手动填写**

选择 **配置项 → 添加 [SOCKS]**：

| 字段 | 值 |
|---|---|
| 别名 (remarks) | `AI-Outbound` |
| 地址 (address) | `1.2.3.4` |
| 端口 (port) | `1080` |
| 用户名 (可选) | `your-username` |
| 密码 (可选) | `your-password` |

不要把 `AI-Outbound` 设为活动节点，它只供链式代理引用。

## 2. 建前置 A（策略组）

选择 **配置项 → 添加策略组**：

| 字段 | 值 |
|---|---|
| 别名 | `AI-Inbound` |
| 策略组类型 | 故障转移 |

在「子配置项一，从订阅分组中自动添加」标签页中：

| 字段 | 值 |
|---|---|
| 订阅分组 | 选择机场订阅 |
| 别名正则过滤 | `美国\|🇺🇸\|\bUS\b\|United States` |

切到「子配置项预览」，确认列出的只有美国节点。

> 不要在链式代理里直接选择某个订阅节点。更新订阅时，v2rayN 会删除旧节点并重新导入，直接选择的节点会失效。用正则从订阅分组筛选，每次生成配置时都会重新匹配，不受订阅更新影响。
>
> 正则不要写成不区分大小写的 `US`，否则会匹配到 Russia、Australia 这类节点名。

## 3. 建链式代理 AI-Chain

选择 **配置项 → 添加链式代理**：

- **别名**：填 `AI-Chain`。必须与第 4 步规则中的 `outboundTag` 完全一致，并且全局唯一。
- 在「子配置项二，从自建中选择添加」标签页，点 **添加子配置**，依次选择 `AI-Inbound` 和 `AI-Outbound`。

子配置的顺序就是流量的走向：

```
1. AI-Inbound   ← 前置，本机先连它
2. AI-Outbound    ← 落地，最终出口
```

顺序不对可以用 **上移 / 下移** 调整。可以在「子配置项预览」中再确认一次。

对 `AI-Chain` 做「测试真连接延迟」会显示失败，这是正常的。测试访问的是 **设置 → 参数设置** 中的「真连接测试地址」，默认不是 AI 域名，会被 B 的白名单拒绝。这个地址是全局设置，所有节点和策略组的健康检查都会用到，不建议为此修改。链路是否可用，请按 [verify.md](verify.md) 用 AI 域名确认。

## 4. 导入规则集

1. 复制下面的规则 JSON。`domain:` 表示匹配该域名本身及所有子域名。

   ```json
   [
     {
       "remarks": "AI 域名经 A 转发至美国住宅代理 B",
       "outboundTag": "AI-Chain",
       "domain": [
         "domain:anthropic.com",
         "domain:claude.ai",
         "domain:claude.com",
         "domain:openai.com",
         "domain:chatgpt.com",
         "domain:sora.com",
         "domain:oaistatic.com",
         "domain:oaistatsig.com",
         "domain:oaiusercontent.com",
         "domain:openaiapi-site.azureedge.net"
       ],
       "enabled": true
     }
   ]
   ```

2. 选择 **设置 → 路由设置**，在「预定义规则集列表」中双击**当前活动的规则集**（主界面状态栏「路由」下拉框中选中的那个），打开「规则集设置」。
3. 点击 **从剪贴板中导入规则**。弹出「是否追加规则？」时选 **是**；选「否」会替换掉全部原有规则。
4. 新规则会追加在「规则列表」的末尾。右键选择 **上移至顶**。
5. 依次点 **确定** 保存。

规则按从上到下的顺序匹配。默认规则集的最后一条通常是兜底规则，追加在它后面的规则永远不会被匹配到，所以必须把 AI 规则放到顶部。

也可以把 JSON 保存为文件，再用 **从文件中导入规则** 导入。

### 可选：拒绝 AI 域名的 QUIC

住宅 SOCKS5 通常不支持 UDP。浏览器访问 ChatGPT 时会先尝试 QUIC，失败后才回落到 TCP，首次打开会慢几秒。可以再导入一条规则直接拒绝这部分流量，**放在 AI-Chain 规则之上**：

```json
[
  {
    "remarks": "AI 域名 QUIC 拒绝，回落 TCP",
    "outboundTag": "block",
    "network": "udp",
    "port": "443",
    "domain": [
      "domain:anthropic.com",
      "domain:claude.ai",
      "domain:claude.com",
      "domain:openai.com",
      "domain:chatgpt.com",
      "domain:sora.com",
      "domain:oaistatic.com",
      "domain:oaistatsig.com",
      "domain:oaiusercontent.com",
      "domain:openaiapi-site.azureedge.net"
    ],
    "enabled": true
  }
]
```

## 5. 检查别名是否生效

如果规则里的 `outboundTag` 找不到同名配置，v2rayN **不会中断启动**，只会弹出一条短暂的提示，并写入主界面的信息面板：

```
路由规则 … 的出站节点 AI-Chain 未找到，已回退为仅使用代理节点。
```

此时 AI 流量会改走当前活动的机场节点。网站照样能打开，但出口 IP 已经不是 B，很难察觉。改完配置后看一眼信息面板，再按 [verify.md](verify.md) 检查出口 IP。

常见原因：

- 别名有多余空格，或大小写不一致
- 存在两个同名为 `AI-Chain` 的配置
- 修改的不是当前活动的规则集

## 终端与 CLI 工具

Claude Code、Codex CLI 等命令行工具不读取系统代理。可以开启 Tun 模式，也可以在托盘（菜单栏）图标菜单中选择 **复制终端代理命令至剪贴板**，拿到 `export` 命令后在终端里执行。
