# workflows/ · 四级配演工作流导览

四个 JSON 四种用法：**一个用来看，两个用来测，一个用来跑**。

| 文件 | 级别 | 用法 | 会不会真花钱 |
|------|------|------|--------------|
| `10000-original-template.json` | 原版 | **只读参考**。官方 API 实拉解包 35 节点，10 个坑原样保留——别直接上生产 | 导出不跑=0；真跑=烧 Perplexity/OpenAI/ElevenLabs/FAL/Blotato |
| `L2A-unit-test.json` | A 级 | **单元验证**。9 单元 47 断言：移植 Extract/Code/tmpfiles 正则/FAL 请求体/Sheets 列映射/Config 语义等全部表达式级逻辑 | 0（离线断言，无外发） |
| `L2B-integration-mock-s1.json` | B 级 | **集成验证**。模板 27 功能节点全名全连线保留 + Mock TG 输入 + 18 断言，正链 S1 场景；另 4 场景（S2 无文字 / S3 无照片 / S4 渲染失败 / E1 GPT 缺字段）源码见 `n8n-lesson-crew` 仓库的 `build-l2-mock-10000.py` 生成器 | 0（全是假 mock；看断言学行为事实） |
| `L2C-free-replacement.json` | C 级 | **免费真跑**。DeepSeek 真调 ×3 + edge-tts 真出配音 + ffmpeg 真出成片 + JSON 台账 + mock 9 平台 | 分毛级（DeepSeek）+ 其他全 0 |

## 导入与触发

### 原版（读）
- 导入后对照 `docs/00-overview.md` 的节点地图读每一处配置。
- ⚠️ 先盯三个地方：`FAL.ai Video Generation` 的裸 Authorization（坑#4）、 `Wait for VEED` 写死 10 分钟（坑#8）、 `Save to Google Sheets` 的 `documentId/sheetId='='` 占位（导入必断）。

### L2A（测）
- 触发方式：CLI `n8n execute --id=<导入后的工作流ID>`。
- 期望：末节点断言汇总 **47 checks 全过**。
- 重点看 U3（tmpfiles 正则四种输入形态）与 U6（FAL 请求体+鉴权头）——这两个是"表达式名"和"链名"都易被忽略的验证现场。

### L2B（测）
- 包含"Mock TG Message In"+全部 27 功能节点原名原连线+ASSERT 断言。
- 期望：S1 正链 18 断言全过；想看坑复现请运行 `scripts/build-l2-mock-10000.py` 生成的 S2/S3/S4/E1 版本。

### L2C（跑）
- 前置（三件套）：
  1. **DeepSeek API key**（本机环境变量或凭证）；
  2. **edge-tts**：`pip install edge-tts`；
  3. **ffmpeg**（WinPut PATH 内有，Windows 自装包注意 PATH）。
- 关键坑：CLI 需要 `NODE_FUNCTION_ALLOW_BUILTIN=child_process` 白名单；含空格路径一律加双引号。
- 期望：12/12 断言全过 + 产出 `state/10000-c-data/` 四件套（script.txt / voice.mp3 / .mp4 / ledger.json）。

## 常见排错

| 症状 | 先看哪 |
|------|--------|
| FAL 401 | Authorization 加 `Key ` 前缀（坑#4） |
| 渲染 10 分钟后空气 | Wait 改轮询环 + Error Trigger（docs/03 坑#8） |
| 台账出现 undefined | prompt 与台账统一取 `theme` 兜底字段（坑#2/#6） |
| 9 平台挂了没人知道 | Merge1 前加汇总断言或台账 per-platform 状态列 |
| Code 节点 `$binary` undefined | `runOnceForAllItems` 下用 `items[0].binary` |

## 安全红线

- 本包 JSON 里的凭证 id 全是占位/原作者残留，不含明文密钥；导入后务必换成你自己的凭证。
- **原版别上生产**：10 坑未修；要改成生产版请至少完成 docs/05 的 🔴 三坑（#4 裸key / #7 零校验 / #8 静默死链）。
- Blotato 是社区节点；本厂没有生产账号，教学环节一律 mock（浏览器发布通道是我们的 L4 分发备选）。