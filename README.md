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


## Fuentes

- [roadmap.sh/ai-agents](https://roadmap.sh/ai-agents) — estructura y nodos
- [Documentación de MCP](https://modelcontextprotocol.io/) y [Agentic AI Foundation](https://aaif.io/)
- Documentación oficial de OpenAI, Anthropic y Google para function calling
- [LangChain 1.0 / LangGraph 1.0](https://www.langchain.com/blog/langchain-langgraph-1dot0)
- [smolagents](https://github.com/huggingface/smolagents)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
