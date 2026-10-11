# mi50localmodelskills

MI50 32G（ROCm）单卡服务器上的本地模型服务栈配套 ZCode Skills。

## Skills

| Skill | 用途 |
|---|---|
| `mi50-ops` | 运维手册：9 个 systemd 服务全景（qwen36 / strata / 4 个 CPU 服务 / 双档生图 / 旧 27B）、ai-switch 切换器、互斥关系、显存/内存水位、诊断命令、故障速查、回滚点、Strata gfx906 构建链与 MTP 开关 |
| `mi50-models` | 模型使用：双档 LLM（qwen36 :8200 512K 并发 / Strata :8201 256K 超长上下文+视觉）、Embedding、Rerank、ASR、TTS、双档生图（Z-Image-Turbo / Qwen-Image-2.1）、RAG 与语音链路，全部带 curl 示例 |

## 安装到 ZCode

```bash
git clone https://github.com/gitciu8080/mi50localmodelskills.git
mkdir -p ~/.agents/skills
cp -r mi50localmodelskills/mi50-ops mi50localmodelskills/mi50-models ~/.agents/skills/
```

## 对应服务布局

- LLM :8200（Qwen3.6-35B-A3B，512K 上下文，parallel 2，GPU 视觉）· Strata :8201（Qwen3.8-Flash-Next 125B-A3B MoE，256K 上下文，KV q4_0 全常驻，MTP 投机解码，CPU 视觉，专家分层 GPU+RAM）
- Embed :8210 · Rerank :8211 · ASR :8212 · TTS :8213（全 CPU，与任何 GPU 档共存）
- 轻量生图 :8301（Z-Image-Turbo，与 LLM 共存）· 整卡生图 :8300（Qwen-Image-2.1）
- 监控面板 :80（gpu-monitor：GPU 曲线 + 全部 9 个服务状态卡片 + 内存水位）

## 2026-10-11 更新（Strata 接入）

- 新增 `strata.service`（:8201）：Strata 引擎 gfx906/HIP 自建构建（ROCm 6.3.3 直编，无需双版本共存），
  跑 Qwen3.8-Flash-Next 125B MoE（IQ2_XS），24576 专家分层缓存，256K 上下文 + q4_0 KV 全量常驻 +
  MTP 投机解码 + CPU 视觉；整卡互斥（与 qwen36/qwen/qwen-image/zimage）。
- ai-switch.sh 新增 `strata` 档；监控面板新增 Strata 卡片与内存水位行。
- 实测：短 prompt 解码 ~30-40 t/s（MTP 开）、24 万 token prefill ~279 t/s、末尾口令精确召回、
  24 万 token 前缀复用 <1s；MTP 开/关 A/B 与开关方法见 mi50-ops 坑 13。
