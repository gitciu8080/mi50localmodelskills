---
name: mi50-ops
description: MI50 服务器的模型服务运维手册。涉及这台机器上任何服务的启停、切换、重启、状态查看、显存/内存问题、开机自启、systemd 单元、ai-switch 切换器、80 端口监控面板、报错排查、回滚时使用。用户提到 qwen36/qwen-embed/qwen-asr/qwen-tts/zimage/qwen-image 服务、端口 8200-8301、显存不够、服务冲突等一律触发本 skill。
---

# MI50 服务器运维手册

单卡 MI50 32G（gfx9006，ROCm，llama.cpp build 10802 + ComfyUI），59G 内存 / 30 vCPU（2 NUMA）。
所有服务的 API key 统一为 `<YOUR_API_KEY>`（TTS 壳同）。

## 服务全景（2026-10 布局）

| systemd 单元 | 端口 | 位置 | 内容 |
|---|---|---|---|
| `qwen36.service` | :8200 | GPU 26.2G | Qwen3.6-35B-A3B Q4_K_XL，512K ctx（每槽 256K×parallel 2），视觉 mmproj，脚本 /root/Qwen36.sh |
| `qwen-embed.service` | :8210 | CPU | Qwen3-Embedding-0.6B Q8（/v1/embeddings） |
| `qwen-rerank.service` | :8211 | CPU | Qwen3-Reranker-0.6B Q8（/v1/rerank） |
| `qwen-asr.service` | :8212 | CPU | Qwen3-ASR-0.6B Q8 + 音频 mmproj（/v1/audio/transcriptions） |
| `qwen-tts.service` | :8213 | CPU | Qwen3-TTS-1.7B（llama-tts CLI + /root/tts_server.py 的 OpenAI 壳） |
| `zimage.service` | :8301 | GPU ~5G | Z-Image-Turbo 轻量生图（ComfyUI `--cpu-vae`），与 LLM 共存 |
| `qwen-image.service` | :8300 | GPU 整卡 | Qwen-Image-2.1 ComfyUI（峰值 24-28G） |
| `qwen.service` | :8200 | GPU | 旧 27B（保留可切回），默认不自启 |
| `gpu-monitor.service` | :80 | CPU | 监控面板（GPU 曲线 + qwen* 服务状态卡片） |

## 统一切换入口

```bash
/root/GenPic/ai-switch.sh            # 状态总览（推荐第一步）
/root/GenPic/ai-switch.sh llm36      # 35B 模型组（5 个 CPU 服务一起拉起）
/root/GenPic/ai-switch.sh llm        # 切回 27B
/root/GenPic/ai-switch.sh image      # 整卡生图（自动停所有 LLM 档）
/root/GenPic/ai-switch.sh zimage     # 轻量生图（与 LLM 共存，仅停整卡生图）
```

**互斥关系**（systemd `Conflicts=` 单侧声明即对称生效）：
- GPU 整卡三选一：`qwen36` ↔ `qwen` ↔ `qwen-image`
- `zimage` 只与 `qwen-image` 互斥，可与任一 LLM 共存（DiT 4.7G + TE/VAE 走 CPU）
- 4 个 CPU 服务（embed/rerank/asr/tts）与一切共存，生图时 RAG/语音仍可用

**开机自启默认**：qwen36 + 4 个 CPU 服务 + gpu-monitor。改默认：
`systemctl disable qwen36 && systemctl enable qwen`（zimage/qwen-image 按需启动，不自启）。

## 常用诊断

```bash
systemctl status qwen36                       # 单服务状态
journalctl -u qwen36 -f                       # 看日志（加载/报错）
journalctl -u zimage --since "-10 min"        # ComfyUI 日志
rocm-smi --showmeminfo vram --json            # 显存（约 26.2G 为 llm36 正常水位，>31G 危险）
free -g                                       # 内存（CPU 服务合计 ~7G 为正常）
curl -s -H "Authorization: Bearer <YOUR_API_KEY>" http://127.0.0.1:8200/health   # LLM 健康
curl -s http://127.0.0.1:80/api/services      # 面板的服务状态 API
```

80 端口面板（http://<IP>/）顶部"模型服务"卡片实时显示全部 8 个单元，3 秒刷新。

## 已知规则与坑

1. **CPU 服务必须 `HIP_VISIBLE_DEVICES=-1`**（已在 4 个单元里）：否则 ROCm build 的 llama-server
   即使 `-ngl 0` 也各白占 ~1.5G 显存的 HIP 上下文。新增 CPU 服务时照抄这个 Environment。
2. **两个 ComfyUI 实例**（8300/8301）共用 workflows 目录和 extra_model_paths.yaml（模型根
   `/nvme/GenPicModels`）。浏览器打开已停实例会显示缓存页，先确认端口对应服务在跑。
3. **显存水位**：llm36 26.2G + zimage DiT ~2.5G（Q4_K_M 4.7G 文件，常驻部分）≈ 28.7G，余量 ~3G。
   别再往 GPU 塞东西；新增 GPU 服务前先算账。
4. **8200 端口被 27B/35B 共用**（互斥保证只有一个跑）——客户端不用改配置，但别同时手动 start 两个。
5. qwen36 的 `-c 524288` 是总量（parallel 2 每槽 262144，正好模型原生上限），改上下文别超 524288。
6. TTS 每次请求冷加载 ~8s 是 llama-tts CLI 的固有行为（无 server 端点），不是故障。
7. ASR 返回带 `language Chinese<asr_text>` 前缀是模型输出格式，客户端展示前剥掉。

## 故障处理速查

| 症状 | 处置 |
|---|---|
| 服务反复重启 | `journalctl -u <单元> -n 50` 看报错；模型文件在不在（`.aria2` 是残留可删） |
| 8200 报连接拒绝 | 服务在加载（35B 约 15s-3min）；`systemctl is-active qwen36` 确认 |
| ComfyUI 出图转 lowvram 变慢 | 显存不够，zimage 换 `z-image-turbo-Q3_K_S.gguf`（设 ZIMG_UNET 环境变量） |
| Embedding 报 "not a JSON object: type=list" | 客户端调到了旧版 `/embedding`，改用 `/v1/embeddings` |
| 面板看不到服务卡片 | 卡片在页面顶部；浏览器缓存旧页面则 Ctrl+F5 |
| 显存涨到 31G+ 降不下来 | `rocm-smi` 确认哪个进程；通常是 CPU 服务丢了 HIP_VISIBLE_DEVICES 或起了重复实例 |

## 回滚点

- 27B 全套保留：/root/Qwen.sh + qwen.service，`ai-switch.sh llm` 即回
- qwen36 部署明细与实测数据：/root/GenPic/README.md（llm36 模型组一节）
- 监控面板：/root/gpu-monitor/（server.py + index.html，纯标准库）
