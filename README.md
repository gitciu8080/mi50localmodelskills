# mi50localmodelskills

MI50 32G（ROCm）单卡服务器上的本地模型服务栈配套 ZCode Skills。

## Skills

| Skill | 用途 |
|---|---|
| `mi50-ops` | 运维手册：8 个 systemd 服务全景、ai-switch 切换器、互斥关系、显存水位、诊断命令、故障速查、回滚点 |
| `mi50-models` | 模型使用：LLM 对话/视觉、Embedding、Rerank、ASR、TTS、双档生图（Z-Image-Turbo / Qwen-Image-2.1）、RAG 与语音链路，全部带 curl 示例 |

## 安装到 ZCode

```bash
git clone https://github.com/gitciu8080/mi50localmodelskills.git
mkdir -p ~/.agents/skills
cp -r mi50localmodelskills/mi50-ops mi50localmodelskills/mi50-models ~/.agents/skills/
```

## 对应服务布局

- LLM :8200（Qwen3.6-35B-A3B，512K 上下文）· Embed :8210 · Rerank :8211 · ASR :8212 · TTS :8213
- 轻量生图 :8301（Z-Image-Turbo，与 LLM 共存）· 整卡生图 :8300（Qwen-Image-2.1）
- 监控面板 :80（gpu-monitor，含服务状态卡片）
