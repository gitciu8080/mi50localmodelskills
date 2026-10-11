---
name: mi50-models
description: MI50 服务器上全部模型的使用方法手册。要调用对话 LLM（qwen36 :8200 / Strata 125B :8201）、看图/视觉理解、Embedding 向量、Rerank 重排、ASR 语音转文字、TTS 文字转语音、生图（Z-Image 轻量 / QwenImage 高质量）、RAG 检索链路、语音对话链路，或写对接这些模型的代码时使用。用户提到 8200/8201/8210/8211/8212/8213/8300/8301 端口、写调用示例、对接 Open WebUI 等客户端时触发。
---

# MI50 模型使用手册

所有端点统一 `Authorization: Bearer <YOUR_API_KEY>`。服务状态不确定时先跑 `/root/GenPic/ai-switch.sh`。

## 1. 对话 LLM — http://<IP>:8200（Qwen3.6-35B-A3B，OpenAI 兼容）

- 上下文：总量 512K，`--parallel 2` 每槽 256K；支持视觉；速度 ~49 t/s
- 端点：`/v1/chat/completions`、`/v1/models`；模型名 `Qwen3.6-35B-A3B`

```bash
curl http://<IP>:8200/v1/chat/completions -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" -d '{
    "messages":[{"role":"user","content":"你好"}], "max_tokens":200}'
```

视觉（图片走 base64 data URL）：

```bash
B64=$(base64 -w0 图.png)
curl http://<IP>:8200/v1/chat/completions -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" -d '{
    "messages":[{"role":"user","content":[
      {"type":"text","text":"图里是什么"},
      {"type":"image_url","image_url":{"url":"data:image/png;base64,'"$B64"'"}}]}]}'
```

注意：思考型模型，短回复时内容可能先出现在 `reasoning_content` 字段。

## 2. 对话 LLM（超长上下文）— http://<IP>:8201（Strata · Qwen3.8-Flash-Next 125B-A3B）

- **256K 上下文**（262144 token）+ 全量 KV 常驻显存；同会话追问走前缀缓存（24 万 token 前缀复用 <1s）
- 端点：`/v1/chat/completions`（OpenAI）、`/v1/messages`（Anthropic 兼容）、`/v1/models`；支持视觉
  （OpenAI 同款 image_url 格式，图片 ~300 token，走 CPU 编码 ~3s）；网页聊天直接开 `:8201/`
- 速度：短 prompt 解码 ~30-40 t/s；24 万 token 文档 prefill ~279 t/s（14 分钟读完不衰减）
- MTP 投机解码默认开：散文类无收益（接受率 ~55%），强规律文本（代码/数字/长文档问答）优

```bash
curl http://<IP>:8201/v1/chat/completions -H "Authorization: Bearer llamacppIsYes" \
  -H "Content-Type: application/json" -d '{
    "messages":[{"role":"user","content":"你好"}], "max_tokens":500}'
```

Anthropic 客户端（Claude Code 等）直接把 `ANTHROPIC_BASE_URL` 指到 `http://<IP>:8201`。

注意：**思考型模型，`max_tokens` 给足**（建议 ≥500）——太小会被思考 token 占满，
`message.content` 为 `null`、正文在 `reasoning_content` 里。与 :8200 的 35B 互为替代：
:8201 长文/文档问答更强，:8200 短对话并发（parallel 2）更方便；两档由 `ai-switch.sh` 切换。

## 3. Embedding — http://<IP>:8210（Qwen3-Embedding-0.6B，1024 维）

```bash
curl http://<IP>:8210/v1/embeddings -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3-embedding-0.6b","input":["文本1","文本2"]}'
```

坑：**必须用 `/v1/embeddings`**。旧版 `/embedding`（单数）返回裸数组，OpenAI 客户端会报
"Embedding response is not a JSON object: type=list"。Open WebUI 里引擎选 OpenAI、Base URL 填
`http://<IP>:8210/v1`。

## 4. Rerank 重排 — http://<IP>:8211（Qwen3-Reranker-0.6B，Jina/Cohere 风格）

```bash
curl http://<IP>:8211/v1/rerank -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" -d '{
    "model":"qwen3-reranker-0.6b", "query":"什么是显存",
    "documents":["显存是显卡上的内存","今天天气不错"], "top_n":3}'
# → {"results":[{"index":0,"relevance_score":0.99},...]}
```

## 5. ASR 语音转文字 — http://<IP>:8212（Qwen3-ASR-0.6B，OpenAI 兼容）

```bash
curl http://<IP>:8212/v1/audio/transcriptions -H "Authorization: Bearer <YOUR_API_KEY>" \
  -F file=@语音.wav -F model=qwen3-asr-0.6b
# → {"text":"language Chinese<asr_text>今天天气不错"}（前缀是模型格式，展示前剥掉）
```

multipart 上传，与 OpenAI Whisper 客户端直接兼容；3 秒音频转写约 0.75s。

## 6. TTS 文字转语音 — http://<IP>:8213（Qwen3-TTS-1.7B，OpenAI 风格壳）

```bash
curl http://<IP>:8213/v1/audio/speech -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"input":"要合成的文本","lang":"zh"}' -o out.wav
```

- `lang`: zh/en/de/it/pt/es/ja/ko/fr/ru（默认 zh）；返回 audio/wav 24kHz
- 可选 `speaker_file`（wav/mp3 路径）做零样本音色克隆
- 性能：合成 0.73×实时 + 每次调用 ~8s 模型加载；同一时刻只跑一个请求（单飞锁）

## 7. 轻量生图 — Z-Image-Turbo（ComfyUI REST API，LLM 在线时可用）

Skill / Agent 优先通过 HTTP API 远程调用，无需登录服务器运行 Python 脚本。
服务地址为 `http://<IP>:8301`，所有请求携带 Bearer key；从运行环境读取配置，不把实际密钥写入 skill 或提交到仓库。

```bash
export MI50_API_BASE="http://<IP>:8301"
export MI50_API_KEY="<YOUR_API_KEY>"
```

### 准备可复用的 API 工作流

首次配置时，在 **:8301** 打开 Workflows → Z-Image-Turbo，导出 **Save (API Format)**，
将节点图保存为 `zimage_workflow_api.json`。后续调用复用该文件，不必每次打开网页。
已有服务器脚本 `/root/GenPic/generate_zimage.py` 中的工作流构造也可作为参考。

根据导出文件中的节点类型和 `inputs` 修改提示词、宽高、步数及 seed；默认参考为
1024×1024、8 步。节点 ID 和模型文件名以实际工作流为准，不要猜测或写死其他实例的 ID。
必须使用 API 格式节点图，不能直接提交包含 `nodes` / `links` 的网页编辑器格式。

```bash
# 将 API 节点图包装为 /prompt 的请求体
jq '{prompt: .}' zimage_workflow_api.json > zimage_request.json
```

### 提交任务

```bash
curl -fsS "$MI50_API_BASE/prompt" \
  -H "Authorization: Bearer $MI50_API_KEY" \
  -H "Content-Type: application/json" \
  --data-binary @zimage_request.json -o zimage_submit.json

PROMPT_ID=$(jq -er '.prompt_id' zimage_submit.json)
```

请求外层结构为 `{"prompt": { ...API 格式 workflow 节点... }}`。
保存返回的 `prompt_id`，用于查询同一任务。若提交失败或缺少 `prompt_id`，
检查 HTTP 错误、`error` / `node_errors`，修正工作流后再提交。

### 查询进度并下载结果

```bash
curl -fsS "$MI50_API_BASE/history/$PROMPT_ID" \
  -H "Authorization: Bearer $MI50_API_KEY" -o zimage_history.json
```

每隔 2–5 秒查询一次，设置可配置的整体超时（例如 10 分钟，排队或冷启动时可延长）。
空历史记录表示任务尚未写入历史，不能据此判定失败；检查该 `prompt_id` 的
`status` 和 `outputs`，执行失败时报告错误，成功后读取各输出节点的 `images`。
超时后保留 `prompt_id` 供继续查询，避免重复提交同一请求。

每张图片提供 `filename`、`subfolder` 和 `type`，使用这些字段下载：

```bash
curl -fsS -G "$MI50_API_BASE/view" \
  -H "Authorization: Bearer $MI50_API_KEY" \
  --data-urlencode "filename=<images中的filename>" \
  --data-urlencode "subfolder=<images中的subfolder>" \
  --data-urlencode "type=<images中的type>" \
  -o zimage.png
```

遍历所有输出节点中的图片，保存到调用端的输出目录，再向用户展示图片或文件链接。
下载参数必须来自任务结果；不要把服务器输出路径当作本地路径。

1024² 图片约 80 秒（参考生成耗时，不含排队或冷启动）。Z-Image 可与 LLM 共存；
若服务未启动，在服务器运行 `/root/GenPic/ai-switch.sh zimage`。
网页仍可在 **:8301** → Workflows → Z-Image-Turbo → Queue Prompt 使用。

## 8. 高质量生图 — Qwen-Image-2.1（http://<IP>:8300，需切档独占 GPU）

```bash
/root/GenPic/ai-switch.sh image        # 自动停 LLM（CPU 服务不受影响）
/root/GenPic/venv/bin/python /root/GenPic/generate.py \
  --prompt "提示词" --width 1024 --height 1024 --out /root/GenPic/samples/y.png
# 1024²/25步 约 8.8 分钟；编辑图用 edit.py
```

用完 `ai-switch.sh llm36` 切回。两档生图模型都在 /nvme/GenPicModels/，workflow 在 ComfyUI 侧边栏。

## 9. 组合链路

**RAG 建库**：文档分块 → :8210 /v1/embeddings 存向量库 → 查询时同端点编码 query →
向量检索取 top20 → :8211 /v1/rerank 精排取 top3 → 塞进 :8200 或 :8201（长文档用 256K 的 :8201）的对话 prompt。

**语音对话**：录音 → :8212 转文字 → :8200/:8201 对话 → 回复文本 → :8213 合成语音播放。
（当前 llama.cpp 无 qwen3omni 架构，一体式 OMNI 不可用，此串联链路即等价实现）

## 性能/容量速查

| 能力 | 延迟参考 |
|---|---|
| qwen36 生成 | ~49 t/s |
| Strata 解码（短 prompt / 24 万 token 上下文） | ~30-40 t/s / ~20 t/s |
| Strata prefill | ~279 t/s（1.5K-24 万 token 同速） |
| Embedding 512-token 块 | ~0.5-1s |
| Rerank 20 条候选 | 数秒 |
| ASR | 0.25×实时 |
| TTS | 0.73×实时 + 8s 加载 |
| ZImage 1024² | ~80s |
| QwenImage 1024² 25 步 | ~8.8 分钟（独占 GPU） |
