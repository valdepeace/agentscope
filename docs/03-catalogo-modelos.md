---
id: investigacion-agentes-modelos
title: Catálogo independiente de modelos
---

# 03. Catálogo independiente de modelos

Comparar por ID API + snapshot + fecha, no solo por familia. Modelo, agente y chat template son entidades distintas. Ventanas, tarifas y disponibilidad cambian; las páginas vivas se deben archivar al publicar.

| Familia | Proveedor | Fuente primaria | Disponibilidad / notas verificadas |
|---|---|---|---|
| Claude Opus/Sonnet/Haiku | Anthropic | [Model overview](https://platform.claude.com/docs/en/models/overview), [pricing](https://platform.claude.com/docs/en/about-claude/pricing) | API propietaria; IDs, contexto, tool use y tarifas por modelo/version |
| GPT y Codex | OpenAI | [Modelos](https://developers.openai.com/api/docs/models), [pricing](https://developers.openai.com/api/docs/pricing) | API; Codex puede nombrar un producto CLI o una familia de modelos |
| Gemini | Google | [Modelos](https://ai.google.dev/gemini-api/docs/models), [tokens](https://ai.google.dev/gemini-api/docs/tokens) | API y variantes; recuperar metadata exacta por ID |
| Qwen / Qwen Coder | Alibaba | [Qwen org](https://github.com/QwenLM), [Qwen3-Coder-Next card](https://huggingface.co/Qwen/Qwen3-Coder-Next) | La ficha consultada declara Apache-2.0, pesos abiertos, 80B total/3B activos y contexto nativo 262144; confirmar versión/SHA al publicar |
| DeepSeek | DeepSeek | [Model list](https://api-docs.deepseek.com/api/list-models/), [pricing](https://api-docs.deepseek.com/quick_start/pricing/) | IDs/endpoints y capacidades cambian; no transferir claims del proveedor a agente |
| Kimi | Moonshot AI | [API docs](https://platform.moonshot.ai/docs), [Kimi Code](https://github.com/MoonshotAI/kimi-code) | API; modelo exacto y pesos/licencia por release |
| GLM | Zhipu/Z.ai | [API](https://open.bigmodel.cn/dev/api), [ZCode](https://github.com/zai-org/ZCode) | API de modelos separada de harness ZCode |
| MiniMax | MiniMax | [Portal](https://www.minimax.io/), [MCode](https://github.com/MiniMax-AI/minimax-code) | registrar ID/región en captura |
| Doubao Seed | ByteDance/Volcengine | [Ark](https://www.volcengine.com/product/ark), [cookbook](https://github.com/ByteDance-Seed/Seed2.0) | API/modelos; cookbook Apache-2.0 declarado |
| Hunyuan | Tencent | [Docs](https://cloud.tencent.com/document/product/1729) | versión por API |
| ERNIE | Baidu | [Qianfan](https://cloud.baidu.com/product/wenxinworkshop) | versión por API |
| MiMo | Xiaomi | [MiMo Code repo](https://github.com/XiaomiMiMo/MiMo-Code) | distinguir agente de modelo |

Por modelo: proveedor, release date, ID/snapshot, capacidades de programación y tool calling demostradas, template/documentación, contexto de input/output, API/pesos, licencia, precio/región, benchmarks + condiciones y limitaciones. No se rellenan campos ausentes por inferencia. Métricas publicadas por proveedor se etiquetan como claims del proveedor; no son ranking independiente.
