# Módulo 7 — Building Agents (Construcción de agentes)

Tres vías: a mano, con function calling nativo del proveedor, o con framework. No son excluyentes; lo habitual es empezar a mano para entender y después decidir.

## Manual (from scratch) — A mano, desde cero

### Direct LLM API calls (Llamadas directas a la API)

**Qué es.** Hablar con el endpoint del proveedor sin capas intermedias: armás el array de mensajes, mandás el request, parseás la respuesta.

**Para qué sirve.** Control total y cero dependencias. Es también la mejor forma de aprender: si entendés el loop a mano, cualquier framework después es azúcar sintáctico.

**Dónde se aplica.** Agentes simples en producción (un loop de 100 líneas es perfectamente válido), y cuando querés evitar que una dependencia te rompa el build en cada release.

### Implementing the agent loop (Implementar el bucle)

El esqueleto completo con los controles que hacen falta en producción:

```python
def agente(objetivo, max_pasos=8):
    hist = [{"role": "system", "content": SYSTEM}, {"role": "user", "content": objetivo}]
    for _ in range(max_pasos):
        r = llm(hist, tools=SCHEMAS, temperature=0)
        hist.append(r.message)
        if not r.tool_calls:
            return r.content                      # terminó
        for c in r.tool_calls:
            out = ejecutar(c)                     # try/except adentro
            hist.append({"role": "tool", "tool_call_id": c.id, "content": json.dumps(out)})
        if contar_tokens(hist) > UMBRAL:
            hist = comprimir(hist)                # Módulo 5
    return "Límite de pasos alcanzado."
```

Los cuatro controles no negociables: **límite de pasos**, **errores como observación**, **compresión de contexto** y **trazas** (cada paso logueado, Módulo 9).

### Parsing model output (Parsear la salida)

Si usás tool calling nativo, el proveedor ya te devuelve la llamada estructurada y esto casi desaparece. Si trabajás con modelos sin tool calling, tenés que parsear texto, y ahí aparecen los problemas clásicos: JSON con comas finales, bloques ```json``` envolviendo el objeto, texto explicativo antes del JSON.

Estrategia: pedir el formato en el prompt, usar *structured output* / JSON schema cuando el proveedor lo soporte, y en el parser ser tolerante (extraer el primer objeto JSON válido) pero fallar explícito y devolverle el error al modelo para que reintente.

### Error & Rate-limit handling (Errores y límites de tasa)

Dos tipos de error distintos, con tratamiento distinto:

- **Errores de tool** (el lote no existe, timeout de la API): se devuelven al modelo como observación. El agente se corrige.
- **Errores de infraestructura** (HTTP 429 *rate limit*, 500, 529 *overloaded*): el modelo no puede hacer nada con eso. Se resuelven en el cliente con **reintentos con backoff exponencial y jitter**, respetando el header `retry-after` cuando viene.

```python
for intento in range(5):
    try:
        return client.messages.create(**kwargs)
    except RateLimitError as e:
        time.sleep(min(2 ** intento + random.random(), 30))
raise RuntimeError("Rate limit persistente")
```

Además: timeouts explícitos en toda llamada de red, y un tope de gasto por sesión (contá tokens y cortá) para que un bucle no te vacíe la cuenta.

## LLM Native "Function Calling" (Function calling nativo)

**Qué es.** La capacidad, entrenada en el modelo, de devolver una llamada a función estructurada en lugar de texto libre. Vos declarás los schemas en el request; el modelo responde con nombre y argumentos ya parseados.

**Para qué sirve.** Elimina el parseo frágil y mejora muchísimo la precisión de los argumentos. Es el estándar actual: salvo que uses un modelo que no lo soporte, esta es la vía.

Estado por proveedor (verificado a noviembre de 2026):

- **OpenAI Function Calling.** Disponible en Chat Completions y en la **Responses API**, que es hoy la recomendada para agentes. OpenAI publica además un **Agents SDK** con el loop ya armado.
- **OpenAI Assistant API.** **Ya no existe**: fue deprecada el 26/08/2025 y se apagó el **26 de agosto de 2026**. El PDF del roadmap la lista como nodo, pero hoy el camino es Responses API (+ Conversations para el estado). Si encontrás tutoriales con `client.beta.assistants`, están obsoletos.
- **Gemini Function Calling.** Soportado en los modelos Gemini vía su SDK, con *automatic function calling* en Python y modos de ejecución configurables.
- **Anthropic Tool Use.** Tool use en la API de Claude (bloques `tool_use` / `tool_result`), más un **Agent SDK** y soporte nativo de MCP.

Patrón común a los tres: declarás tools → el modelo pide una → ejecutás → devolvés el resultado con el id de la llamada → el modelo sigue. Cambian los nombres de los campos, no el concepto. Eso hace que migrar de proveedor sea trabajo mecánico.

## Building Using Frameworks (Construir con frameworks)

Qué aporta un framework: loop ya hecho, integraciones (modelos, vector DBs, loaders), manejo de estado, memoria, streaming y tracing. Qué te cuesta: abstracciones que esconden el prompt real, breaking changes, y debugging más difícil cuando algo falla adentro.

Estado del panorama (verificado a noviembre de 2026):

- **LangChain / LangGraph.** LangChain 1.0 y LangGraph 1.0 salieron GA el 22/10/2025, ambas designadas LTS y estables hasta la 2.0 (`create_agent` es la API de alto nivel; `langgraph.prebuilt` quedó deprecado a favor de `langchain.agents`). División práctica: LangChain para armar rápido, **LangGraph** para orquestar con grafos, estado persistente y durabilidad. Es el ecosistema más grande en integraciones.
- **LlamaIndex.** Orientado a datos y RAG. Es la mejor opción si el núcleo del problema es ingesta, indexado y recuperación sobre documentos propios.
- **Haystack** (deepset). Pipelines explícitos, perfil enterprise, fuerte en búsqueda. Enfoque más declarativo y predecible.
- **AutoGen.** Multi-agente conversacional; fue muy influyente. **Importante:** está en *maintenance mode* desde octubre de 2025 (solo bugs y seguridad). Microsoft publicó **Microsoft Agent Framework 1.0** (GA abril de 2026) como sucesor oficial, unificando AutoGen y Semantic Kernel; existe además **AG2**, fork comunitario Apache 2.0. No conviene arrancar un proyecto nuevo sobre AutoGen legacy.
- **CrewAI.** Multi-agente con metáfora de equipo: cada agente tiene rol, objetivo y backstory, y se reparten tareas. Muy rápido de prototipar, licencia MIT, cadencia de releases alta (chequeá la versión actual en su repo antes de fijar dependencias).
- **smolagents** (Hugging Face). El nodo del PDF que figura como "Smol Depot" es en realidad **smolagents**. Framework minimalista (~1000 líneas de core, Apache 2.0): su idea distintiva es el **CodeAgent**, que en vez de emitir JSON escribe código Python que llama a las tools, lo que suele resolver tareas compuestas en menos pasos. Soporta tools de servidores MCP y es model-agnostic. Requiere sandbox para ejecutar ese código.

**Criterio de elección (opinión, no consenso):** agente simple y acotado → a mano con tool calling nativo. Flujo de producción con estado, retries y paralelismo → LangGraph. Centrado en documentos → LlamaIndex. Equipo de roles para prototipar → CrewAI. Agentes que razonan con código → smolagents. Stack Microsoft/Azure → Microsoft Agent Framework.

**Ejemplo concreto.** Un agente de WhatsApp que registra y consulta datos de lotes, con cuatro tools y una base Postgres, no necesita framework: 150 líneas de Python con tool calling nativo, un loop con `max_pasos=6` y logging estructurado. Cuando ese mismo agente tenga que correr jobs largos, reintentar pasos individuales y sobrevivir a un reinicio, ahí sí paga migrarlo a LangGraph.

## Conclusión del módulo

Lo que hay que llevarse: el loop a mano con function calling nativo es el punto de partida correcto y alcanza para más casos de los que parece; el framework se adopta cuando aparece una necesidad concreta (estado durable, paralelismo, muchas integraciones), no por default. Y el panorama se mueve rápido: verificá siempre estado y versión antes de elegir, porque hay nodos del roadmap que ya quedaron desactualizados.

Cómo conecta con el siguiente: un agente que funciona en tres pruebas manuales no es un agente confiable. El Módulo 8 (Evaluation and Testing) trata de cómo medirlo.
