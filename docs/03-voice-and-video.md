# 03 · 声音与数字人：全模板最贵的两跳

本讲覆盖生产线中段的"重工车间"：**ElevenLabs TTS → mpga→mp3 二进制手术 → tmpfiles 双上传 → FAL.ai VEED Fabric 数字人渲染 → Wait 死等 → 下载成片**。全模板的 10 个坑有 6 个（#1 #4 #5 #7 #8 及关联 #9）集中在这 7 个节点。

## 1. ElevenLabs Voice Synthesis（TTS，httpRequest）
- `POST https://api.elevenlabs.io/v1/text-to-speech/{voiceId}`，body 是 GPT 写好的脚本，响应格式选二进制（mpga）。
- 替身：**edge-tts 晓晓 +8%**（本厂标配，0 成本，本地真出 267KB/44.5s mp3 ✅）。
- 付费性：ElevenLabs 按字符计费——短剧/带货号量产时这是第一大开销。

## 2. Convert .mpga to .mp3（Code 节点 · 全模板唯一）
这个 10 行 Code 节点干的是**二进制属性改名手术**：
```js
const binary = items[0].binary['data_httpRequest'];            // 上游 httpRequest 的二进制属性名
return [{ json: item.json,
          binary: { audio: { ...binaryProps,
                              fileName: binary.fileName?.replace('.mpga','.mp3') || 'output.mp3',
                              mimeType: 'audio/mpeg' } } }];
```
L2-A 的 U2 单元 7 断言逐条固化了改名/改 mime/保留原属性/无二进制透传/默认文件名五种行为。
- **坑的娘家**：二进制属性名依赖`节点类型+上游结构`，节点改名或上游换节点立刻找不到 key——L2-B C 踩的工具坑：`runOnceForAllItems` 模式下 `$binary` 不可用，必须用 `items[0].binary`。这类"二进制链条手术"在 n8n 里最高频踩雷。

## 3. tmpfiles.org 双上传（免费的服务，最贵的不稳）
VEED Fabric 只接收**公网可下载**的 URL，所以照片和音频都要先传 tmpfiles.org 中转。模板在请求体内嵌正则把返回链接改写为直链：
```
http://tmpfiles.org/<数字路径>/<文件名>  →  https://tmpfiles.org/dl/<数字路径>/<文件名>
```
**坑#1（🟡 脆弱）**：正则只认 `http://` 开头且路径数字打头——上游一旦改返回 `https://` 或已含 `/dl/`，改写**静默不生效**，FAL 拿到的是"看得了网页下不了文件"的 URL。L2-A 的 U3 单元 5 断言固化四种输入形态的行为差异。
- **军规**：对正则改写一律"显式验证"——改写后立即 HEAD 请求验证 URL 可下载（200 + Content-Length > 0）。
- **替身**：C 版直接本地路径给 ffmpeg（跳过公网化）；真要用公网可换自建对象存储/CDN。

## 4. FAL.ai Video Generation（数字人渲染提交，httpRequest）
```
POST https://queue.fal.run/veed/fabric-1.0
headers: Authorization = {{ falApiKey }}          // ⚠️ 坑#4：裸 key，官方文档要求 "Key <apikey>"
body: { image_url: <公网图片>, audio_url: <公网音频>, resolution: "480p" }   // ⚠️ 坑#5：480p 硬编码
```
- **坑#4（🔴 疑似 bug）**：没带 `Key ` 前缀，真跑鉴权可能被拒——本课程无法免费验证 FAL，按官方文档加固。
- **坑#5（🟡）**：分辨率写死 480p；想高清只能改节点表达式。
- **坑#7（🟥 必炸）**：**零输入校验**——S3 场景复现：没照片时 `image_url=''` 照样提交，流程伪装成功。入口 IF 是必修。
- **这是全模板唯一"无法免费替代"的环节**（对口型数字人渲染）。课程的处理 = **C 版降级出片**（AI 图 + 音频 ffmpeg 合成），真数字人标注为"付费升级项"。

## 5. Wait for VEED + Download（异步渲染的"诅咒"）
```
[Wait] wait 10 分钟（写死） → [Download] GET https://queue.fal.run/.../requests/{request_id}
```
**坑#8（🟥 必炸 · 本课程最重量级的坑）**：
- 渲染快：白等 10 分钟（生产事故级延迟）；渲染慢/失败：**无任何轮询、超时、错误网、告警**——S4 场景复现：Download 返回 FAILED 无 video 字段 → 下游 Sheets 映射取 `.video.url` 直接 TypeError，**台账/发布/回写全部静默断链**，用户连"失败了"都收不到。
- **药方（生产必改）**：
  1. Wait 1 分钟 → Download → IF `status=='COMPLETED' && video` 放行；`FAILED` 走 Error 分支（微信/飞书告警 + 台账标失败）；否则**循环重试，最多 N 次**。
  2. 整体加 **Error Trigger 工作流**做兜底告警。
  3. 台账状态机：`PENDING → RENDERING → DONE/FAILED`，避免"假 DONE"。

## 真实工件（L2-C 替身链产出，零付费）
- `script.txt` 667B：DeepSeek 写的 30s 口播稿
- `voice.mp3` 267KB/44.5s：edge-tts 晓晓 +8%
- `lesson-10000-c.mp4` **1.82MB/46.4s/1920×1080 h264+aac**：ffmpeg 图音合成（zoompan 尾差 1.9s 属正常现象）
- `ledger.json`：IDEA/URL/STATUS 五列齐，STATUS=DONE

---
**本讲一句总结**：声音和画面是花钱的地方，更是**断链的地方**——二进制手术、免费中转、异步死等，每一跳都要上防御。