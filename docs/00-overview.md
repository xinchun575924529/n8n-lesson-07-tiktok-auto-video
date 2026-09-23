# 00 · 总览：35 节点的一条主生产线

模板页：https://n8n.io/workflows/10000/（Auto-create TikTok videos with VEED.io AI avatars, ElevenLabs & GPT-4）

## 一句话
Telegram 收到「一张照片 + 一句主题」→ 查 TikTok 热点（Perplexity）→ GPT-4 写 30s 口播稿 → ElevenLabs 配音 → VEED Fabric 把照片渲染成对口型数字人 → GPT-4 写文案标签 → 存 Google Sheets 台账 → Blotato 一键发 9 个平台 → 成片发回 Telegram。

## 节点地图（28 功能节点 + 7 张便签）
```
[Telegram Trigger] 收照片+主题
 → [Workflow Configuration] set     ← 集中放 5 个配置（elevenLabsKey/voiceId/falKey/时长/perplexityModel）
 → [Extract Photo and Theme] set    ← 取最大尺寸照片 file_id + theme 三级兜底
 → [Get Photo File from Telegram]   ← 按 file_id 下载照片二进制
 → [Build Public Image URL] http    ← 照片传 tmpfiles.org 得公网 URL（⚠️ 正则改写坑#1）
 → [Search Trends with Perplexity]  ← 查 3 条相关热点（⚠️ 坑#2 undefined）
 → [Generate Script with GPT-4]     ← 30s 口播稿（⚠️ 坑#3 死配置）
 → [ElevenLabs Voice Synthesis] http← TTS，返回二进制 mpga
 → [Convert .mpga to .mp3] code     ← 全模板唯一的 Code 节点：重命名二进制属性/文件名/mime
 → [Upload Audio to Public URL] http← 音频传 tmpfiles.org（同坑#1）
 → [FAL.ai Video Generation] http   ← 提交 queue.fal.run/veed/fabric-1.0（⚠️ 坑#4 裸key/坑#5 480p/坑#7 无校验）
 → [Wait for VEED] wait             ← ⚠️ 写死等 10 分钟（坑#8）
 → [Download VEED Video] http       ← 按 request_id 拉结果（⚠️ 渲染失败=静默死亡 坑#8）
 → [Generate Caption with GPT-4]    ← 文案 + 5-8 个 hashtag
 → [Save to Google Sheets] append   ← 台账五列（⚠️ 坑#6）
 → [Send a video] telegram          ← 成片发回用户
 → [Upload Video to BLOTATO]        ← 先上传 Blotato
 → [Tiktok][Youtube][Instagram][Facebook][Linkedin][Twitter(X)][Threads][Bluesky][Pinterest]  9 平台扇出
 → [Merge1] (chooseBranch, 9 输入)  ← 九路合流（⚠️ 坑#10）
 → [Update Status to "DONE"]        ← 台账状态回写
便签 ×7：Setup Guide / Step1-5 / How It Works（逐步教程，夹带作者 LinkedIn/YouTube 引流，可忽略）
```

## 你要安装的三个世界观
1. **主轴 = 一条内容生产线**：选题（热点）→ 文案（GPT）→ 声音（TTS）→ 画面（数字人）→ 分发（Blotato）→ 留痕（Sheets）。所有"AI 短视频工厂"类模板都是这条线的变形——lesson-04 的 YouTube Shorts 工厂是"AI 找素材剪视频"，本模板是"AI 数字人口播"。
2. **两次"公网化"是隐形依赖**：VEED Fabric 只吃公网 URL，所以音频和照片都要先传 tmpfiles.org 中转。这是个免费第三方小服务——**返回值格式一旦变化，正则改写静默失效，整条链断在 FAL 之前**。免费依赖 = 定时炸弹（坑#1）。
3. **异步渲染的"死等"模式**：FAL 提交后返回 `request_id`，模板用一个写死 10 分钟的 Wait 节点"赌"它渲染完，没有任何轮询、超时和错误网。**10 分钟没到就白等，渲染失败就全链静默死亡**（坑#8）——这是 35 节点里最不能接受的设计。

## 验证战绩速览（三级全过）
| 关 | 方法 | 断言/场景 | 结果 |
|---|---|---|---|
| L2-A 单元 | 全表达式移植 + 断言固化 | 9 单元 47 断言 | ✅ 47/47（首跑全过） |
| L2-B 集成 | 27 节点全名全连线 mock ×5 场景 | 33 断言 | ✅ 33/33（坑 2/6/7/8/9 逐个复现） |
| L2-C 免费替身 | DeepSeek×3 + edge-tts + ffmpeg 出片 + JSON 台账 | 12 断言 | ✅ 12/12，真出 1.82MB mp4 |

工作流与报告全文见 `workflows/`；10 坑与药方见 `docs/05`。