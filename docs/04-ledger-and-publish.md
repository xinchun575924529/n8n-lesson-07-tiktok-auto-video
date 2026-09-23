# 04 · 台账与分发：收尾的暗坑比主菜多

本讲覆盖生产线尾巴：**Google Sheets 台账（写+回写）→ Telegram 回传 → Blotato 9 平台扇出 → Merge1 合流**。看起来是"录个表、发个视频"的杂活，坑的密度却仅次于数字人环节。

## 1. Google Sheets 台账（五列）
```
IDEA = message.caption        ← ⚠️ 坑#6：不走 theme 兜底
SCRIPT / VIDEO_URL / CAPTION / STATUS
```
- **坑#6（🟡 数据）**：IDEA 列直接取 `message.caption`——text-only 输入时入表 `undefined`（S2 场景复现 ✅）。生产修复：统一取 `$('Extract Photo and Theme').item.json.theme`。
- sovereignty 优势：Sheets 的 **documentId / sheetId 在模板里是占位**——导入必断，这是 n8n 模板的常态；替身 = **本地 JSON 台账（L2-C 实测真写盘 ✅）** 或飞书多维表。
- 推荐状态机：`PENDING → RENDERING → DONE / FAILED`——配合 docs/03 坑#8 的药方，不让"假 DONE"骗过你。

## 2. Telegram 回传成片（闭环的爽点）
`sendVideo(chatId=发件人, video=成片URL或二进制)`——"发张照片，几分钟后收到成品视频"——是这套演示最出片的闭环。
- 注意 `chatId` 要从 Trigger 的 message 对象传递，不要硬编码（这是所有机器人回信的通用军规）。

## 3. Blotato 社区节点 ×10（分发扇出）
```
[Upload Video to BLOTATO] → [Tiktok][Youtube][Instagram][Facebook][Linkedin][Twitter(X)][Threads][Bluesky][Pinterest]
```
- **形态特征**：1 个上传节点 + 9 个平台节点直连扇出，全是 **社区节点（@blotato/n8n-nodes-blotato）**——**本机没装的话导入即缺节点**（B 阶段处置 = mock 节点原名占位；教学处置 = 出海发布线 **TikTok（Creator Fund）+ YouTube** 双平台（老板 2026-09-24 拍板，生产通道待建））。
- **付费性**：Blotato 订阅制；它的价值是"一个 API 管 9 平台"，值不值看你的发布频率。
- **Bluesky/Threads/Pinterest** 的名字值得记住——多数分发工具不覆盖这三个。

## 4. Merge1（chooseBranch, 9 输入）——最隐晦的节点（🟡 坑#10）
9 个平台节点全连进 Merge1，配置 `chooseBranch`：**返回第一个到达且非空的分支数据**（L2-B S1 顺带验证坐实：9 平台全成功时台账拿到的只是 Tiktok 分支的数据）。
- **表面行为**：合流后统一回写 DONE，逻辑正确——诺 9 平台全成功。
- **实际上**：**任何单个平台失败，其他 8 个照常跑完，台账照样标 DONE，没有任何人知道那个平台没发。** 8/9 都挂了只要第一个分支有数据就无感。
- **药方**：① 每个平台节点后各自回写该行该平台的状态列（细粒度台账）；② 或 Merge 前加 IF 汇总各平台结果，失败计数字段进台账。
  这是所有"扇出分发"工作流的通用加固模式——**分发成功的定义是"每个平台都成功"，不是"至少有数据合流"**。

## 5. 便签 ×7（Setup Guide / Step1-5 / How It Works）
- 模板作者写了 7 张大便签做逐步 setup 教程，**法文混杂**（"ta propriété binaire actuelle"）加 LinkedIn/YouTube 引流链接——教学内容可忽略，但**别删**：staging 复刻时便签是理解原作者意图的第一手线索。

---
**本讲一句总结**：录表的取错字段，发视频的"假成功"，合流的"装作都发了"——**收尾环节不修，前面 30 个节点白干**。