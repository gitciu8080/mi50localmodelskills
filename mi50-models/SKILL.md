---
name: mi50-models
description: MI50 服务器上全部模型的使用方法手册。要调用对话 LLM、看图/视觉理解、Embedding 向量、Rerank 重排、ASR 语音转文字、TTS 文字转语音、生图（Z-Image 轻量 / QwenImage 高质量）、RAG 检索链路、语音对话链路，或写对接这些模型的代码时使用。用户提到 8200/8210/8211/8212/8213/8300/8301 端口、写调用示例、对接 Open WebUI 等客户端时触发。
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

## 2. Embedding — http://<IP>:8210（Qwen3-Embedding-0.6B，1024 维）

```bash
curl http://<IP>:8210/v1/embeddings -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3-embedding-0.6b","input":["文本1","文本2"]}'
```

坑：**必须用 `/v1/embeddings`**。旧版 `/embedding`（单数）返回裸数组，OpenAI 客户端会报
"Embedding response is not a JSON object: type=list"。Open WebUI 里引擎选 OpenAI、Base URL 填
`http://<IP>:8210/v1`。

## 3. Rerank 重排 — http://<IP>:8211（Qwen3-Reranker-0.6B，Jina/Cohere 风格）

```bash
curl http://<IP>:8211/v1/rerank -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" -d '{
    "model":"qwen3-reranker-0.6b", "query":"什么是显存",
    "documents":["显存是显卡上的内存","今天天气不错"], "top_n":3}'
# → {"results":[{"index":0,"relevance_score":0.99},...]}
```

## 4. ASR 语音转文字 — http://<IP>:8212（Qwen3-ASR-0.6B，OpenAI 兼容）

```bash
curl http://<IP>:8212/v1/audio/transcriptions -H "Authorization: Bearer <YOUR_API_KEY>" \
  -F file=@语音.wav -F model=qwen3-asr-0.6b
# → {"text":"language Chinese<asr_text>今天天气不错"}（前缀是模型格式，展示前剥掉）
```

multipart 上传，与 OpenAI Whisper 客户端直接兼容；3 秒音频转写约 0.75s。

## 5. TTS 文字转语音 — http://<IP>:8213（Qwen3-TTS-1.7B，OpenAI 风格壳）

```bash
curl http://<IP>:8213/v1/audio/speech -H "Authorization: Bearer <YOUR_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"input":"要合成的文本","lang":"zh"}' -o out.wav
```

- `lang`: zh/en/de/it/pt/es/ja/ko/fr/ru（默认 zh）；返回 audio/wav 24kHz
- 可选 `speaker_file`（wav/mp3 路径）做零样本音色克隆
- 性能：合成 0.73×实时 + 每次调用 ~8s 模型加载；同一时刻只跑一个请求（单飞锁）

## 6. 轻量生图 — Z-Image-Turbo（http://<IP>:8301 ComfyUI，LLM 在线时可用）

命令行（推荐，约 80s/张 1024²）：

```bash
/root/GenPic/venv/bin/python /root/GenPic/generate_zimage.py \
  --prompt "一只戴宇航员头盔的橘猫，电影感" --width 1024 --height 1024 \
  --out /root/GenPic/samples/x.png        # --seed --steps(默认8) 可选
```

网页：打开 **:8301**（不是 8300）→ 侧边栏 Workflows → Z-Image-Turbo → Queue Prompt。
REST：ComfyUI 原生 API（/prompt 提交 + /history/{id} 轮询），generate_zimage.py 里有完整可复用构造。

## 7. 高质量生图 — Qwen-Image-2.1（http://<IP>:8300，需切档独占 GPU）

```bash
/root/GenPic/ai-switch.sh image        # 自动停 LLM（CPU 服务不受影响）
/root/GenPic/venv/bin/python /root/GenPic/generate.py \
  --prompt "提示词" --width 1024 --height 1024 --out /root/GenPic/samples/y.png
# 1024²/25步 约 8.8 分钟；编辑图用 edit.py
```

用完 `ai-switch.sh llm36` 切回。两档生图模型都在 /nvme/GenPicModels/，workflow 在 ComfyUI 侧边栏。

## 8. 组合链路

**RAG 建库**：文档分块 → :8210 /v1/embeddings 存向量库 → 查询时同端点编码 query →
向量检索取 top20 → :8211 /v1/rerank 精排取 top3 → 塞进 :8200 的对话 prompt。

**语音对话**：录音 → :8212 转文字 → :8200 对话 → 回复文本 → :8213 合成语音播放。
（当前 llama.cpp 无 qwen3omni 架构，一体式 OMNI 不可用，此串联链路即等价实现）

## 性能/容量速查

| 能力 | 延迟参考 |
|---|---|
| LLM 生成 | ~49 t/s |
| Embedding 512-token 块 | ~0.5-1s |
| Rerank 20 条候选 | 数秒 |
| ASR | 0.25×实时 |
| TTS | 0.73×实时 + 8s 加载 |
| ZImage 1024² | ~80s |
| QwenImage 1024² 25 步 | ~8.8 分钟（独占 GPU） |
