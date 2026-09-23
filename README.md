# AI 数字人短视频自动工厂：发张照片就长出一条口播视频（官方模板 10000 拆解）
> 出处：https://n8n.io/workflows/10000/（Auto-create TikTok videos with VEED.io AI avatars, ElevenLabs & GPT-4）

这个模板讲的是一个极其上头的玩法：给 Telegram 机器人**发一张照片 + 一句主题**，它自动查 TikTok 热点、写 30s 口播稿、AI 配音、把照片变成**对口型说话的数字人**、写好文案和话题标签、存台账、一键多发 TikTok/YouTube 等 9 个平台，最后把成片发回给你。一条 UGC/带货号的量产流水线，全程不用人。

我们把模板 35 个节点（28 功能节点 + 7 张便签）完整拉到本地做了 L2 三级验证——**A 单元 47/47、B mock 全链 33/33、C 免费替身真跑 12/12，三关全绿共 92 断言**，中途抓出 **10 个模板坑**（其中 3 个是"真跑必出事"级别：渲染失败全链静默死亡、无输入校验空提交、undefined 带病入库）。C 关用**零付费替身链真实产出了一条 46 秒 1920×1080 的口播成片**，证明：除了 VEED 数字人渲染，整条工厂链可以一分钱不花复刻。

## 你将学到（6 讲每讲配可跑工作流）
1. **00 总览**：35 节点骨架地图；一条「照片 → 选题 → 写稿 → 配音 → 数字人 → 台账 → 9 平台分发」主轴
2. **01 输入与配置**：Telegram 收图收题 / Workflow Configuration 集中放 Key / **theme 三级兜底为何形同虚设**
3. **02 AI 内容链**：Perplexity 查热点 → GPT-4 写稿写文案；**undefined 病毒怎么走完全链**；DeepSeek 平替方案
4. **03 声音与数字人**：ElevenLabs TTS → mpga→mp3 二进制手术 → tmpfiles 免费公网中转（正则陷阱）→ FAL.ai VEED Fabric 数字人渲染 → **写死 10 分钟的 Wait**
5. **04 台账与分发**：Google Sheets 五列台账 / Blotato 1 上传 + 9 平台扇出 / Merge1 九路合流的隐晦行为
6. **05 排障图谱**：10 坑照妖镜合集 + 全部断言固化的验证现场

## 10 个坑先睹为快（剧透式目录，正文有药方）
| # | 坑 | 级别 | 一句话后果 |
|---|---|---|---|
| 1 | tmpfiles.org URL 改写正则只认 `http://` 且路径必须数字开头 | 🟡 脆弱 | 上游服务返回一变（https 或已含 /dl/），改写**静默不生效**，FAL 拿到不可下载的 URL |
| 2 | Perplexity 提示词直接拼 `message.caption`，不走 theme 兜底 | 🟡 健壮 | 只发文字不发 caption 时，prompt 拼出字面 **`undefined`** |
| 3 | `scriptMaxDuration=30` 是死配置：prompt 里 "30-second" 是硬编码文本 | 🟡 误导 | 改配置节点**毫无作用**，白调 |
| 4 | FAL Authorization 发**裸 key**（官方要求 `Key <apikey>`） | 🔴 疑似 bug | 真跑鉴权可能被拒 |
| 5 | fabric-1.0 分辨率 **480p 硬编码**，参数不可配 | 🟡 兼容 | 想高清只能改节点表达式 |
| 6 | Sheets 的 IDEA 列也取 `message.caption` 而非 theme | 🟡 数据 | text-only 输入把 **`undefined` 写进台账** |
| 7 | **零输入校验**：无照片时 FAL `image_url=''` 照样提交渲染 | 🔴 必炸 | 真跑必失败，流程却假装走完 |
| 8 | **渲染失败 = 全链静默死亡**：Wait 死等 10 分钟，无 Error Trigger / 无重试 / 无告警 | 🔴 必炸 | 失败后台账、发布、回写**全部静默断链**，没人知道 |
| 9 | GPT 输出缺 `message.content` 无 schema 校验，直传 TTS | 🟡 健壮 | TTS 拿 `undefined` 合成语音 |
| 10 | Merge1(chooseBranch, 9 输入)：任一平台失败无人察觉 | 🟡 隐晦 | 8 个平台挂了，台账照样写 DONE |

修复与加固方法全写在 `docs/05`，且**每一个坑都有本地断言固化**（A 6 个 + B 4 个）。

## 免费替代链（成本直减 99%+，L2C 实测真链）
| 模板角色 | 原案（付费性） | 课程实测替身 | 成本 |
|---|---|---|---|
| 热点查询 | Perplexity（付费/限量） | **DeepSeek deepseek-chat** | 分毛/次 |
| 写稿 + 写文案 | OpenAI GPT-4 ×2（付费） | **DeepSeek**（真调） | 分毛/次 |
| TTS 配音 | ElevenLabs（付费） | **edge-tts 晓晓 +8%**（本地） | 0 |
| 数字人渲染 | FAL.ai VEED Fabric（**付费且无替身**） | **ffmpeg AI图+音频合成**（降级出片，真数字人列为付费升级项） | 0 |
| 公网文件中转 | tmpfiles.org（免费但脆） | 本地路径直给 / 本地 http.server | 0 |
| 台账 | Google Sheets（OAuth） | **本地 JSON 台账**（飞书多维表也可） | 0 |
| 多平台分发 | Blotato 社区节点 ×10（付费） | 本厂**抖音/B站浏览器发布通道**（课程内 mock） | 0 |
| 消息入口 | Telegram bot（免费） | 保留（也可换飞书触发） | 0 |

**结论性产品卖点**：除真数字人渲染外，整条「AI 口播视频工厂」可零成本上线；想要对口型数字人，再按月付 FAL.ai —— 一门课同时教会你"省钱版"和"顶配版"。

## 现场实测证据（不是复述官方话术）
- **L2-A 单元单测**：9 单元 47 断言，CLI 首次执行 **47/47 全过**（移植 Extract/Code/tmpfiles 正则/FAL 请求体等全部表达式级逻辑）
- **L2-B mock 全链**：模板全节点名全连线保留 ×5 场景（正链/无 caption/无照片/渲染失败/GPT 缺字段），**33/33 断言全过**，坑 2/6/7/8/9 逐个复现坐实
- **L2-C 免费真跑**：DeepSeek 真调 ×3 + edge-tts 真出 267KB 配音 + ffmpeg 真出 **1.82MB/46.4s/1920×1080 成片** + 本地 JSON 台账，**12/12 断言全过**
- 完整证据与测试工作流见 `workflows/`，复现手记见 `docs/`

## 仓库地图
```
.
├── README.md                     本文件
├── LEGEND.md                     图例与代号（L1-L5 / A-C 关 / 🔴-⚪ / ✅）
├── docs/
│   ├── 00-overview.md            节点地图与一条主轴
│   ├── 01-input-and-config.md    TG 入口 / 配置节点 / 提取表达式
│   ├── 02-ai-content-chain.md    热点查询与 GPT 写稿写文案
│   ├── 03-voice-and-video.md     TTS / 二进制手术 / tmpfiles / VEED 数字人
│   ├── 04-ledger-and-publish.md  Sheets 台账 / Blotato 9 平台 / Merge1
│   └── 05-pitfalls-defense.md    10 坑排障图谱与药方
├── exercises/exercises.md        6 个逐级动手练习（附参考答案）
├── script/short-video.md         60s 获客口播稿 + 分镜清单
├── workflows/
│   ├── README.md                 四个 JSON 的用法与安全红线
│   ├── 10000-original-template.json   原模板 35 节点（只读参考）
│   ├── L2A-unit-test.json             单元验证流（9 单元 47 断言）
│   ├── L2B-integration-mock-s1.json   1:1 拓扑 mock 正链场景（18 断言）
│   └── L2C-free-replacement.json      免费替身链（真能跑出 mp4 成片）
└── site/                         GitHub Pages 静态站（push 自动部署）
```

**License**: MIT（教程内容可自由转载，模板版权归 n8n 官方）

---
*本项目由「n8n 工厂店」教研车间产出 · L2 三级验证完成于 2026-09-14 · L3 教案封装 2026-09-24*