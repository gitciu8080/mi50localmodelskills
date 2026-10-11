---
name: mi50-ops
description: MI50 服务器的模型服务运维手册。涉及这台机器上任何服务的启停、切换、重启、状态查看、显存/内存问题、开机自启、systemd 单元、ai-switch 切换器、80 端口监控面板、报错排查、回滚时使用。用户提到 qwen36/qwen-embed/qwen-asr/qwen-tts/zimage/qwen-image/strata 服务、端口 8200-8301、显存不够、服务冲突等一律触发本 skill。
---

# MI50 服务器运维手册

单卡 MI50 32G（gfx906），三套推理栈并存：llama.cpp（build 10802，qwen36/27B/CPU 服务）、
Strata 0.1.42（gfx906/HIP 自建，125B MoE）、ComfyUI（生图）。59G 内存 / 30 vCPU（2 NUMA）。
所有服务的 API key 统一为 `llamacppIsYes`（TTS 壳同）。

## 服务全景（2026-10 布局）

| systemd 单元 | 端口 | 位置 | 内容 |
|---|---|---|---|
| `qwen36.service` | :8200 | GPU 26.2G | Qwen3.6-35B-A3B Q4_K_XL，512K ctx（每槽 256K×parallel 2），视觉 mmproj，脚本 /root/Qwen36.sh |
| `strata.service` | :8201 | GPU 整卡 31.6G + 内存 ~40G | Strata 0.1.42（gfx906/HIP 自建）跑 Qwen3.8-Flash-Next 125B MoE IQ2_XS，**256K ctx，KV q4_0 全量常驻显存（--kv-resident 0 --kv-grow）**，MTP spec4，**多模态（CPU 视觉编码器，发图走 OpenAI image_url）**；配置 /nvme/Strata/strata-iq2_xs.json，引擎日志 /nvme/Strata/strata-iq2_xs.log |
| `qwen-embed.service` | :8210 | CPU | Qwen3-Embedding-0.6B Q8（/v1/embeddings） |
| `qwen-rerank.service` | :8211 | CPU | Qwen3-Reranker-0.6B Q8（/v1/rerank） |
| `qwen-asr.service` | :8212 | CPU | Qwen3-ASR-0.6B Q8 + 音频 mmproj（/v1/audio/transcriptions） |
| `qwen-tts.service` | :8213 | CPU | Qwen3-TTS-1.7B（llama-tts CLI + /root/tts_server.py 的 OpenAI 壳） |
| `zimage.service` | :8301 | GPU ~5G | Z-Image-Turbo 轻量生图（ComfyUI `--cpu-vae`），与 LLM 共存 |
| `qwen-image.service` | :8300 | GPU 整卡 | Qwen-Image-2.1 ComfyUI（峰值 24-28G） |
| `qwen.service` | :8200 | GPU | 旧 27B（保留可切回），默认不自启 |
| `gpu-monitor.service` | :80 | CPU | 监控面板（GPU 曲线 + qwen*/strata 服务状态卡片 + 内存行） |

## 统一切换入口

```bash
/root/GenPic/ai-switch.sh            # 状态总览（推荐第一步）
/root/GenPic/ai-switch.sh llm36      # 35B 模型组（5 个 CPU 服务一起拉起）
/root/GenPic/ai-switch.sh strata     # Strata 125B（:8201，自动停一切 GPU 服务）
/root/GenPic/ai-switch.sh llm        # 切回 27B
/root/GenPic/ai-switch.sh image      # 整卡生图（自动停所有 LLM 档）
/root/GenPic/ai-switch.sh zimage     # 轻量生图（与 LLM 共存，仅停整卡生图）
```

**互斥关系**（systemd `Conflicts=` 单侧声明即对称生效）：
- GPU 整卡四选一：`qwen36` ↔ `qwen` ↔ `qwen-image` ↔ `strata`（strata 在自己单元里声明 Conflicts 其余三个）
- `zimage` 只与 `qwen-image`、`strata` 互斥，可与任一 llama.cpp LLM 共存（DiT 4.7G + TE/VAE 走 CPU）
- 4 个 CPU 服务（embed/rerank/asr/tts）与一切共存；但 strata 内存吃 ~40G，共存时余量 ~17G，紧但够用

**开机自启默认**：qwen36 + 4 个 CPU 服务 + gpu-monitor。改默认：
`systemctl disable qwen36 && systemctl enable qwen`（zimage/qwen-image/strata 按需启动，不自启；
不要把 strata 和 qwen36 同时 enable——开机后谁后到谁活）。

## 常用诊断

```bash
systemctl status qwen36                       # 单服务状态
journalctl -u qwen36 -f                       # 看日志（加载/报错）
journalctl -u zimage --since "-10 min"        # ComfyUI 日志
journalctl -u strata -f                       # Strata 加载进度；详细速度行在 /nvme/Strata/strata-iq2_xs.log
rocm-smi --showmeminfo vram --json            # 显存（llm36 约 26.2G、strata 约 31.6G 为正常水位，>31G 时 llm36 危险）
free -g                                       # 内存（CPU 服务 ~7G；strata 在跑时 used ~41G 为正常）
curl -s -H "Authorization: Bearer llamacppIsYes" http://127.0.0.1:8200/health   # LLM 健康
curl -s -H "Authorization: Bearer llamacppIsYes" http://127.0.0.1:8201/v1/models # Strata 健康
curl -s http://127.0.0.1:80/api/services      # 面板的服务状态 API
```

80 端口面板（http://<IP>/）顶部"模型服务"卡片实时显示全部 9 个单元 + 一行内存水位，3 秒刷新。

## 已知规则与坑

1. **CPU 服务必须 `HIP_VISIBLE_DEVICES=-1`**（已在 4 个单元里）：否则 ROCm build 的 llama-server
   即使 `-ngl 0` 也各白占 ~1.5G 显存的 HIP 上下文。新增 CPU 服务时照抄这个 Environment。
2. **两个 ComfyUI 实例**（8300/8301）共用 workflows 目录和 extra_model_paths.yaml（模型根
   `/nvme/GenPicModels`）。浏览器打开已停实例会显示缓存页，先确认端口对应服务在跑。
3. **显存水位**：llm36 26.2G + zimage DiT ~2.5G（Q4_K_M 4.7G 文件，常驻部分）≈ 28.7G，余量 ~3G。
   strata 独占整卡（~31.6G）不能与任何 GPU 服务共存。新增 GPU 服务前先算账。
4. **8200 端口被 27B/35B 共用**（互斥保证只有一个跑）——客户端不用改配置，但别同时手动 start 两个。
5. qwen36 的 `-c 524288` 是总量（parallel 2 每槽 262144，正好模型原生上限），改上下文别超 524288。
6. TTS 每次请求冷加载 ~8s 是 llama-tts CLI 的固有行为（无 server 端点），不是故障。
7. ASR 返回带 `language Chinese<asr_text>` 前缀是模型输出格式，客户端展示前剥掉。
8. **Strata 冷启动**：服务刚起的前 1-2 个请求解码只有 4-8 tok/s（HIP 内核首次加载），第 3 个起恢复
   ~30-40 tok/s，不是故障。专家装入 RAM ~70-80s，整个启动约 1.5-2 分钟。
9. **Strata 思考模型**：`max_tokens` 太小会让思考 token 占满输出，`message.content` 为 null（在
   `reasoning_content` 里）。客户端要么给足 max_tokens，要么处理 reasoning 字段。
10. **Strata 重跑 setup 的坑**：`./setup.sh --setup ...` 会把 `--kv-resident` 写回 32768、丢 `--kv-grow`、
    清空 host/api_key——重跑后必须检查 strata-iq2_xs.json 并修回（`--context/--kv` 显式传参可保住）。
11. **Strata 长文工况**：24 万 token prefill 全程 ~279 tok/s，结温峰值 ~101°C 属 AMD 卡正常工况
   （临界 105°C，用户确认不设功耗墙）；同会话追问走前缀缓存（24 万 token 复用 <1s）。
12. **Strata 的 gfx906 构建链**：源码/构建在 /nvme/Strata（build-906 目录，ROCm 6.3.3 直编，
   `-DSTRATA_HIP_GFX906=ON`，CMAKE_PREFIX_PATH=/opt/rocm）；setup.py 有本地补丁——`STRATA_GFX906=1`
   时 AMD_ARCHS 才认 gfx906；钉住版 llama.cpp 在 third_party/llama.cpp 免下载（github.com 主站被墙，
   codeload.github.com 与 hf-mirror.com 可直连；模型下载一律 `HF_ENDPOINT=https://hf-mirror.com`）。
   升级 = git pull 后手动重编（CMake 参数照 build-906 的 cache），setup 不会替你编 gfx906。
13. **Strata 的 MTP 开关**：native IQ 量化包**强制要求 `--spec >= 2`**（引擎拒绝无投机启动），但 MTP
   草稿模型本身可选——删 `--spec-min-p 0.5` 与 `--mtp <path>` 两对参数即退回后缀自投机。
   实测（2026-10-11）：散文类 MTP 接受率仅 52-60%，开着反而慢 ~15%；强规律文本（数字/代码）
   接受率 88%，MTP 略优（27.7 vs 23.7 tok/s 峰值）。当前保持 MTP 开（长上下文 242K 时接受率 79%）。

## 故障处理速查

| 症状 | 处置 |
|---|---|
| 服务反复重启 | `journalctl -u <单元> -n 50` 看报错；模型文件在不在（`.aria2` 是残留可删） |
| 8200 报连接拒绝 | 服务在加载（35B 约 15s-3min）；`systemctl is-active qwen36` 确认 |
| 8201 报连接拒绝 | strata 在加载（~2 分钟，日志看 "experts loaded / ready"）；`systemctl is-active strata` |
| Strata 解码只有个位数 tok/s | 刚启动的前 1-2 个请求是 HIP 内核冷加载，跑热即恢复 ~40 tok/s；若持续低，看 strata-iq2_xs.log 的 hit rate（应 >95%） |
| ComfyUI 出图转 lowvram 变慢 | 显存不够，zimage 换 `z-image-turbo-Q3_K_S.gguf`（设 ZIMG_UNET 环境变量） |
| Embedding 报 "not a JSON object: type=list" | 客户端调到了旧版 `/embedding`，改用 `/v1/embeddings` |
| 面板看不到服务卡片 | 卡片在页面顶部；浏览器缓存旧页面则 Ctrl+F5 |
| 显存涨到 31G+ 降不下来 | llm36 档才异常：`rocm-smi` 确认哪个进程；strata 档 31.6G 是正常水位 |
| strata 启动后 swap/卡死 | 内存不够：expert 装入要 ~39G，先停几个 CPU 服务再启 strata |

## 回滚点

- 27B 全套保留：/root/Qwen.sh + qwen.service，`ai-switch.sh llm` 即回
- qwen36 部署明细与实测数据：/root/GenPic/README.md（llm36 模型组一节）
- strata 全套：/nvme/Strata（源码+引擎）+ /nvme/Strata-data（模型 72G）；删档 = `systemctl disable --now strata`
  后删以上两目录 + /etc/systemd/system/strata.service；明细与实测：/root/GenPic/README.md（strata 模型组一节）
- 监控面板：/root/gpu-monitor/（server.py + index.html，纯标准库）
