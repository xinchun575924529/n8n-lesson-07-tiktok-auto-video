# 01 · 输入与配置：入口三件套

本讲覆盖前 3 个节点：**Telegram Trigger → Workflow Configuration → Extract Photo and Theme**。这是整条生产线的"收单台"，决定了后面每一步拿什么干活。

## 1. Telegram Trigger（入口）
- `updates = ["message"]`：只接收 message 更新。
- 用户的使用姿势：**发一张照片，配一句 caption 作为主题**（而不是发两条消息）。这个"照片+caption 单消息"的约定贯穿全模板——也是坑的多发地（见下）。
- 免费替代：BotFather 建测试 bot 零成本；不进 TG 生态的教学场景可换飞书触发（我们工厂的 FeishuBridge 就是这么干的）。

## 2. Workflow Configuration（集中配置块）
```
elevenLabsApiKey   = YOUR_ELEVENLABS_API_KEY
elevenLabsVoiceId  = YOUR_VOICE_ID
falApiKey          = YOUR_FAL_API_KEY
scriptMaxDuration  = 30          ← ⚠️ 坑#3：这个 30 配了也白配
perplexityModel    = sonar
includeOtherFields = true        ← 关键：让上游 message 对象穿透到下游
```
- **设计意图值得学**：把所有 API Key 和可调参数集中在一个 Set 节点，下游用 `$('Workflow Configuration').first().json.xxx` 引用——改配置只动一处。
- **但它名不副实**（🔍 坑#3）：下游 GPT 节点的 prompt 里 `"Write a 30-second script"` 是**硬编码文本**，`scriptMaxDuration` 没有任何节点真正引用它参与生成。你把这个数改成 60，产出的还是 30s 脚本。**配置 ≠ 生效，要看有没有被表达式消费**。L2-A 的 U5/U9 断言把这个"死配置"钉死坐实。

## 3. Extract Photo and Theme（提取与兜底）
两个表达式是全模板的地基，也是 L2-A 的 U1 单元（7 断言）：
```js
photoUrl = {{ $json.message.photo
             ? $json.message.photo[$json.message.photo.length - 1].file_id
             : '' }}
theme    = {{ $json.message.caption || $json.message.text || 'viral content' }}
```
### 值得学的好设计
- **取最大尺寸**：Telegram 一张照片给多个尺寸，模板取数组最后一项 = 最大分辨率——小而正确的细节。
- **theme 三级兜底**：`caption → text → 'viral content'`，用户怎么输入都不至于空选题。

### 但兜底是"孤岛"（🔍 坑#2 / 坑#6 的引信）
`theme` 这个兜底字段**只有主题输入这一条路在用**；下游其余地方（Perplexity 的 prompt、Sheets 的 IDEA 列）全部直接拼 `message.caption`——**caption 为空时不走兜底，直接拼出字面 `undefined`**。等于模板作者造了安全绳却只绑了一只脚。L2-B 的 S2 场景（只发文字）复现：Perplexity prompt 出现 `undefined`、台账 IDEA 列写入 `undefined`，链条照样"正常"走完——**带毒交付，最恶心的坏 bug 类型**。

## 输入校验结论（与坑#7 呼应）
模板对 「只发文字不发照片」没有任何拦截：`photoUrl=''` 会一路传到 FAL 的 `image_url`（🟥 坑#7，docs/03 详解）。**生产用法的第一条军规：在 Extract 之后加一个 IF**——`photoUrl != ''` 才放行，否则回复用户"请带照片一起发"。这是所有"收单台"类入口的标准补强。

---
**本讲一句总结**：入口三件套教会你"配置集中化、兜底多级化"两个好设计，和"兜底要全链一致、输入要先过闸"两条军规。