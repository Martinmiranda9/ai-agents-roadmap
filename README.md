# AI Agents — Roadmap de estudio

Apuntes de estudio sobre **AI Agents**

Cada archivo cubre un módulo con la misma estructura: qué es, para qué sirve, dónde se aplica, cómo se aplica (con ejemplos cortos en Python) y un ejemplo concreto, más una conclusión que lo conecta con el módulo siguiente.

**Última actualización:** noviembre de 2026

## Índice

| # | Módulo | Contenido principal |
|---|--------|---------------------|
| 1 | [LLM Fundamentals](Modulo%201%20-%20LLM%20Fundamentals.md) | Transformers, tokenización, ventana de contexto, pricing, controles de generación, open vs closed weight, reasoning, fine-tuning vs prompting, embeddings, RAG |
| 2 | [AI Agents 101](Modulo%202%20-%20AI%20Agents%20101.md) | Qué es un agente, qué son las tools, agent loop (percepción, razonamiento, acción, observación), casos de uso |
| 3 | [Prompt Engineering](Modulo%203%20-%20Prompt%20Engineering.md) | System prompt, especificidad, contexto, ejemplos few-shot, formato, iteración y testeo |
| 4 | [Tools / Actions](Modulo%204%20-%20Tools%20y%20Actions.md) | Definición de tools, schemas, manejo de errores, tipos de tools (search, REPL, DB, API, mensajería, filesystem) |
| 5 | [Agent Memory](Modulo%205%20-%20Agent%20Memory.md) | Corto y largo plazo, episódica vs semántica, vector DBs, perfil de usuario, resumen y compresión, olvido |
| 6 | [Agent Architectures](Modulo%206%20-%20Agent%20Architectures.md) | ReAct, CoT, RAG Agent, Planner-Executor, DAG, Tree-of-Thought y **MCP** (hosts, clients, servers, deployment) |
| 7 | [Building Agents](Modulo%207%20-%20Building%20Agents.md) | Loop manual, function calling nativo (OpenAI, Gemini, Anthropic), frameworks (LangChain/LangGraph, LlamaIndex, Haystack, CrewAI, smolagents) |
| 8 | [Evaluation and Testing](Modulo%208%20-%20Evaluation%20and%20Testing.md) | Métricas, unit testing de tools, testing de flujos, golden dataset, human in the loop, LangSmith/Ragas/DeepEval |
| 9 | [Debugging and Monitoring](Modulo%209%20-%20Debugging%20and%20Monitoring.md) | Logging estructurado, tracing, LangSmith, Langfuse, Helicone, OpenLLMetry, alertas de producción |
| 10 | [Security & Ethics](Modulo%2010%20-%20Security%20and%20Ethics.md) | Prompt injection, sandboxing y permisos, privacidad y PII, guardrails de sesgo, red teaming |

## Notas sobre el material

Estos apuntes no son una transcripción del roadmap: siguen su orden y sus nodos, pero el contenido está reescrito y, donde el roadmap quedó desactualizado, corregido y verificado contra fuentes primarias.

### Correcciones respecto del cuadro original

- **"Model Mechanis"** → el nodo correcto es **Model Mechanics** (figura cortado en el PDF del roadmap).
- **"Smol Depot"** → es **smolagents**, el framework de Hugging Face.
- **OpenAI Assistant API** → figura como nodo del roadmap, pero fue deprecada el 26/08/2025 y **se apagó el 26 de agosto de 2026**. El camino actual es la **Responses API**. Los tutoriales con `client.beta.assistants` están obsoletos.
- **AutoGen** → en *maintenance mode* desde octubre de 2025. El sucesor oficial es **Microsoft Agent Framework 1.0** (GA abril de 2026); existe además **AG2**, fork comunitario Apache 2.0.
- **MCP** → desde diciembre de 2025 es proyecto de la **Agentic AI Foundation** de la Linux Foundation (donado por Anthropic), con licencia Apache 2.0 y gobernanza neutral.
- **LangChain / LangGraph** → ambos llegaron a **1.0 el 22/10/2025**, designados LTS.

### Nodos agregados (están en el roadmap online, no en el PDF)

- **Multi-Agents** y **Self-critique / Reflexion** → cubiertos como complemento en el Módulo 6.
- **LangGraph** → cubierto en los módulos 6 y 7.

### Criterios

- Los **precios de modelos** se explican como mecanismo (costo por token de input y output) sin cifras concretas: cambian demasiado seguido como para dejarlas escritas. Hay que mirarlas en la página de pricing del proveedor.
- Donde una afirmación es **opinión** y no consenso de la industria (por ejemplo, qué framework elegir), está marcada como tal en el texto.
- Las **versiones de frameworks** se citan con fecha de verificación. Conviene chequear el repo oficial antes de fijar dependencias.

## Fuentes

- [roadmap.sh/ai-agents](https://roadmap.sh/ai-agents) — estructura y nodos
- [Documentación de MCP](https://modelcontextprotocol.io/) y [Agentic AI Foundation](https://aaif.io/)
- Documentación oficial de OpenAI, Anthropic y Google para function calling
- [LangChain 1.0 / LangGraph 1.0](https://www.langchain.com/blog/langchain-langgraph-1dot0)
- [smolagents](https://github.com/huggingface/smolagents)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
