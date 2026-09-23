# 练习与参考答案（10000 · 共 6 题）

> 做题顺序建议按讲次推进。答案附在每题之后（先自己想）。

---

## 练习 1（对应 00 讲）：实拉照妖

**题目**：用 `https://api.n8n.io/api/workflows/templates/10000` 拉取模板，数出功能节点数与便签数，并把它分成「内容主线」与「分发/台账支线」两组。

**参考答案**：
- 实拉 **35 节点 = 28 功能 + 7 便签**（Setup Guide / Step1-5 / How It Works）
- **内容主线（约 14 个）**：Telegram Trigger → Workflow Configuration → Extract Photo and Theme → Get Photo File → Build Public Image URL → Perplexity → GPT Script → ElevenLabs → mpga→mp3 Code → Upload Audio → FAL.ai → Wait → Download → GPT Caption
- **分发/台账支线（约 14 个）**：Save Sheets → Send a video → Upload BLOTATO → Blotato ×9 → Merge1 → Update Status
- 教训：**Blotato 的 10 个节点会把"节点数"撑到 2 倍**，画 mock 时别当成主线节点数。

## 练习 2（对应 01 讲）：亲手复现兜底孤岛

**题目**：搭一个对照实验：Set 节点输出 `theme="viral content"`、`caption=undefined` 两种情况，分别观察 Perplexity prompt 与台账 IDEA 列拿到什么。

**参考答案**：
| 输入 | prompt 拼接 | 台账 IDEA |
|---|---|---|
| 只发文字（caption 缺失） | `related to: undefined` | `undefined` |
| 带 caption | 正常主题 | 正常主题 |
- 关键认知：`theme` 有三级兜底，但 **prompt 和台账都没指向它**——兜底造了绳子没绑脚。
- 修复：prompt 与台账统一取 `$('Extract Photo and Theme').item.json.theme`。
- 这就是 L2-B S2 场景复现并断言的 **坑#2 + 坑#6**。

## 练习 3（对应 02 讲）：LLM 链路的防线

**题目**：写一个"空值断言"表达式/配置，放在任意 LLM 节点后面，让 GPT 返回结构缺 `choices[0].message.content` 时显式报错而不是静默继续。

**参考答案**：
```js
// Code 节点（runOnceForEachItem）
const c = $json.choices?.[0]?.message?.content?.trim();
if (!c) throw new Error('LLM 空输出 @ Generate Script with GPT-4');
return { json: { ...$json, content: c } };
```
- 验证：用 mock 发 `{choices:[]}` → 必须抛错；发正常结构 → 透传。
- 军规：**LLM 输出永远带问号链 + 显式报错**，null/undefined 静默传播是本课最恶的坑型。

## 练习 4（对应 03 讲）：异步渲染加固改造

**题目**：把"Wait 10 分钟 → Download"改成带失败保护的轮询环：要求渲染 FAILED 或超过 15 分钟未完成时打到 Error 分支并在台账写 FAILED。

**参考答案**（节点结构）：
```
[FAL 提交] → [Wait 1min] → [Download]
   ↓
[IF status]
 ├ COMPLETED 且有 video → 继续主线
 ├ FAILED → [Set STATUS=FAILED] → [Error 分支：告警 + 台账回写]
 └ 其他(排队/进行中) → [计数器++] → 超 15 次 → FAILED 分支；否则回 Wait
```
- 关键认知：**Wait 死等 = 异步工作流的第一杀手**；改轮询环是生产异步任务的标配。
- 练习产出要求：在 `workflows/` 提交改造版 JSON，并在 docs/05 登记新断言。

## 练习 5（对应 04 讲）：Merge1 假成功探针

**题目**：搭一个三平台 mock 扇出：Tiktok 成功、Youtube 失败（抛错但 continueOnFail）、Pinterest 成功，观察 Merge1(chooseBranch) 的输出与台账状态，并写出修复方案。

**参考答案**：
- 现象：Merge1 拿到 Tiktok 分支数据（first arrived 非空），台账标 DONE——**Youtube 挂了没人知道**（坑#10 行为事实）。
- 修复 A（细粒度台账）：每个平台节点后各加一条 Sheets update，标该平台 per-platform 状态。
- 修复 B（汇总合流前）：`Summarize` 节点统计非空分支数，不足 9 写入 `PARTIAL_FAIL`。
- 军规：**分发成功的定义是"每个平台都成功"，不是"至少有一路到达 Merge"**。

## 练习 6（对应 05 讲）：二进制手术试炼

**题目**：写一个 Code 节点，把上游 httpRequest 产生的 `data_httpRequest` 二进制属性重命名为 `audio`，mime 改为 `audio/mpeg`，文件名 `.mpga`→`.mp3`，并给出 5 条断言：改名成功/mime 正确/原字段保留/无二进制透传/默认文件名生效。

**参考答案**：
```js
const b = items[0].binary['data_httpRequest'];
if (!b) throw new Error('上游无二进制');
return items.map(item => ({
  json: item.json,
  binary: {
    audio: {
      data: b.data,
      fileName: (b.fileName || 'output.mpga').replace(/\.mpga$/, '.mp3') || 'output.mp3',
      mimeType: 'audio/mpeg',
    }
  }
}));
```
- 断言示例：`binary.audio.mimeType==='audio/mpeg'` / `binary.audio.fileName.endsWith('.mp3')` / `json.script` 原样保留 / 输入无二进制时抛错 / `fileName` 为空时兜底 `output.mp3`。
- 坑点：`$binary` 在 runOnceForAllItems 模式不可用；`binary['data_httpRequest']` 的 key 名依赖上游节点类型。

---
> **附加挑战**：把 docs/05 的 10 坑整理成你团队的「n8n 流水线 Code Review Checklist」（5~8 条），并让每条都能在本包某条断言里找到"验证现场"。