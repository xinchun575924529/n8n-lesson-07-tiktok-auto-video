# 05 · 排障图谱：10 坑照妖镜与药方

本讲是全课的"体检报告"：**10 个坑全部有本地断言固化**（L2-A 6 个 + L2-B 4 个），每一个都能在 `workflows/` 里找到"把坑按在地上验证"的断言现场。按危害从高到低排。

## 🔴 上线必修（真跑必炸 / 静默事故级）

### 坑#8 · 渲染失败 = 全链静默死亡（本课最重）
- **现场**：Wait 死等 10 分钟 → Download 返回 `FAILED` 无 video 字段 → Sheets 映射取 `.video.url` TypeError → 台账/发布/回写**全部静默断链**，无 Error Trigger、无重试、无告警（L2-B S4 断言固化 ✅）。
- **药方**：① Wait 改短 + 轮询循环（IF status==COMPLETED 放行 / FAILED 走告警分支 / 否则重试 N 次）；② Error Trigger 工作流兜底；③ 台账状态机 `PENDING→RENDERING→DONE/FAILED`。

### 坑#7 · 零输入校验：空照片照样提交渲染
- **现场**：无照片时 `photoUrl=''` → FAL `image_url=''` **无任何 IF 拦截直接提交**（S3 断言固化 ✅）。真跑必失败，流程却假装走完。
- **药方**：Extract 之后加 IF `photoUrl != ''` 才放行；否则回信"请带照片一起发"。

### 坑#4 · FAL Authorization 发裸 key
- **现场**：`Authorization: <apikey>` 无前缀，官方文档要求 `Key <apikey>`（A-U6 断言按模板原样固化了这个形态，真跑可能被 401）。
- **药方**：头值改成 `Key {{ config.falApiKey }}`。

## 🟡 健壮与数据质量（必修但不致命）

### 坑#2 · Perplexity 提示词拼 `message.caption`
- prompt 拼出字面 `undefined`（S2 断言固化 ✅）。**改 `$('Extract...').item.json.theme`（走兜底）**。

### 坑#6 · Sheets IDEA 列同病
- text-only 时台账 IDEA=`undefined`（S2 断言固化 ✅）。同上：统一取 `theme`。

### 坑#9 · LLM 输出零 schema 校验
- GPT 缺 `message.content` 时 TTS 拿 `undefined` 合成（E1 断言固化 ✅）。**每个 LLM 节点后加空值断言/IF，报错优于静默**。

### 坑#3 · `scriptMaxDuration=30` 是死配置
- prompt 里 "30-second" 硬编码，改了配置节点毫无作用（A-U5/U9 断言固化 ✅）。**改参数=把 prompt 文本也表达式化**。

### 坑#5 · 分辨率 480p 硬编码
- fabric-1.0 请求体 `resolution:"480p"` 写死，无配置入口（A-U6 断言固化 ✅）。纳入 Workflow Configuration 即可。

### 坑#1 · tmpfiles.org URL 改写正则太脆
- 只认 `http://` 且路径数字打头；上游返回格式一变改写静默失效（A-U3 五断言固化四形态 ✅）。药方：改写后立即 HEAD 验证 URL 可下载；生产换自建中转。

### 坑#10 · Merge1(chooseBranch, 9输入)：假成功合流
- 取第一个非空分支数据；任一平台失败无人察觉（S1 断言固化行为事实 ✅）。药方：平台各自回写状态列，或合流前汇总失败计数。

## 验证现场索引（想看断言怎么写的直接查）
| 坑 | 固化位置 | 工作流 |
|---|---|---|
| #1 #3 #4 #5 | L2-A U3/U5/U6/U9 | `workflows/L2A-unit-test.json` |
| #2 #6 | L2-B S2 场景 | `workflows/wf-l2-mock-10000-s2.json`（生成器 `scripts/build-l2-mock-10000.py`） |
| #7 | L2-B S3 | 同上 |
| #8 | L2-B S4 | 同上 |
| #9 | L2-B E1 | 同上 |
| #10 | L2-B S1 | `workflows/L2B-integration-mock-s1.json` |

## 环境/工具层怪癖（⚪ 不属模板缺陷，但踩过一次省一晚）
- Code 节点 `runOnceForAllItems` 模式下 `$binary` 不可用，必须 `items[0].binary`。
- 断言里动态遍历节点名用 `$(name)` 变量形式，不存在 `$('...')` 占位写法——否则会报 `Referenced node doesn't exist`。
- mock 生成器里 Python 拼 JS 注意花括号三层闭合（`return [{json:{...}}]`），少一层报 `Unexpected token ']'`。
- CLI 跑 child_process（ffmpeg/edge-tts）必须 `NODE_FUNCTION_ALLOW_BUILTIN` 加白名单。
- 绝对路径含空格的（如 edge-tts 安装路径）必须加双引号。

---
**全课一句总结**：这个模板的"教科书价值"不在功能多全，而在 **10 个坑几乎覆盖了 n8n 内容流水线的全部地雷位**——配置死信、表达式拼接、二进制属性链、免费第三方依赖、异步渲染、日志台账、扇出分发、LLM 零校验——修好这 10 坑，你就拥有了所有同类模板的排障模板。