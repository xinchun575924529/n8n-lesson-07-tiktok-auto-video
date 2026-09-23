# 02 · AI 内容链：热点 → 脚本 → 文案

本讲覆盖三台"内容引擎"：**Search Trends with Perplexity → Generate Script with GPT-4 → Generate Caption with GPT-4**。在技术栈上这是"联网检索增强 + 两段式生成"的经典组合。

## 1. Perplexity 热点查询（RAG 的 R）
```
model = {{ $('Workflow Configuration').first().json.perplexityModel }}   // sonar
content = "Find the top 3 current viral trends related to: "
          + $('Extract Photo and Theme').item.json.message.caption +     // ⚠️ 坑#2
          ". Focus on trending topics, hashtags, and content styles ..."
```
- **意图**：给写稿喂"当天热点"，让脚本蹭上话题流量——只取 3 条，控制 prompt 尺寸和成本。
- **坑#2（🟡 健壮）**：直接拼 `message.caption` 而不是兜底字段 `theme`。用户只发纯文字时，prompt 变成 `...related to: undefined...`，检索"undefined"的热点，产出的脚本自然跑偏——**错误被 LLM 消化后变成"看起来还能用"的垃圾**，比报错更危险。
- **药方**：拼 `theme`（兜底已含 'viral content'）；或按 docs/01 的军规在入口拦截。
- **付费性**：Perplexity 是付费/限量 API。教学替身 = **DeepSeek 直接生成"3 条拟真热点"**（联网非必需，L2-C 实测脚本质量不受影响）；生产上对时效要求高再换回 Perplexity/搜索 API。

## 2. Generate Script with GPT-4（第一段生成：口播稿）
- `openAi` 节点，`gpt-4o-mini`，系统+用户双消息：输入 = 热点结果 + 主题，输出 = 30s 口播稿。
- **坑#3 复盘**："30-second" 是 prompt 硬编码，配置节点的 `scriptMaxDuration=30` 是摆设。**改参数的正确姿势是把 prompt 里所有时长文本也表达式化**：`"Write a " + $config.scriptMaxDuration + "-second script..."`。
- **付费性**：真 OpenAI 付费。教学替身 = **DeepSeek（OpenAI 兼容，改 BASE_URL 即直连）**，L2-C 真调产出 667 字节中文口播稿（健身房新手三大误区，带钩子带关注引导，质量合格 ✅）。

## 3. Generate Caption with GPT-4（第二段生成：发布文案）
- 输入 = 成片可下载后的上下文，输出 = caption + 5-8 个 hashtag，发到 Sheets 和 Telegram。
- **两段式生成的好处**：脚本与发布文案分开调——干啥用啥 prompt，避免"一个 prompt 又写稿又写标签"四不像。这是做内容流水线值得抄的结构。

## 4. 贯穿全链的暗病：输出零校验（🟡 坑#9）
三次 LLM 调用的输出都直接被消费，**没有一个节点检查 `choices[0].message.content` 是否存在**。L2-B 的 E1 场景复现：模拟 GPT 缺字段输出 → TTS 文本变 `undefined` → 链条无感继续。**真跑时会拿 "undefined" 这四个字去合成语音、写进台账**。

**生产军规**：每个 LLM 节点后加一行兜底表达式或 IF：
```js
content = resp.choices?.[0]?.message?.content?.trim()
if (!content) throw new Error('LLM 空输出：' + nodeName)
```
报错 > 静默。这条军规贯通本课程全部 6 讲。

## 5. 成本一览（课程实测替代后）
| 节点 | 原案 | 替身 | L2-C 实测 |
|---|---|---|---|
| Search Trends | Perplexity 付费 | DeepSeek deepseek-chat | ✅ 真调，趋势非空断言过 |
| Generate Script | GPT-4 付费 | DeepSeek | ✅ 真调，>20 字断言过 |
| Generate Caption | GPT-4 付费 | DeepSeek | ✅ 真调，#hashtag 断言过 |

---
**本讲一句总结**：两台 GPT 夹一个检索的"内容三明治"是标准骨架，但两个坑（undefined 拼接、输出零校验）教会你——**LLM 链路的可靠性不在模型，在粘合处的防御**。