# Módulo 9 — Debugging and Monitoring (Debugging y monitoreo)

Un agente falla distinto que una app común: no tira stack trace, devuelve algo plausible pero equivocado. Sin visibilidad del razonamiento y de las llamadas, el debugging es adivinanza.

## Structured logging & tracing (Logging estructurado y trazas)

**Qué es.** **Logging estructurado** es emitir eventos como objetos (JSON) con campos consultables, en vez de strings libres. **Tracing** es encadenar esos eventos en una jerarquía que representa una ejecución completa: un *trace* por corrida del agente, con un *span* por cada paso (llamada al LLM, ejecución de tool, recuperación).

**Para qué sirve.** Responder preguntas que con `print()` no se pueden responder: ¿en qué paso se desvió? ¿qué prompt exacto recibió el modelo (con la memoria ya inyectada)? ¿qué devolvió la tool? ¿cuántos tokens y cuánta plata costó esa conversación? ¿por qué entró en bucle?

**Dónde se aplica.** En los cuatro puntos del loop: entrada, decisión del modelo, ejecución de la tool, observación. Más el cierre de la corrida con totales.

**Cómo se aplica.** Lo mínimo que hay que loguear por paso, con un `trace_id` que ate todo:

```python
import json, time, uuid

trace_id = str(uuid.uuid4())

def log(evento, **campos):
    print(json.dumps({"ts": time.time(), "trace_id": trace_id, "evento": evento, **campos}))

log("llm_request", paso=n, modelo=MODELO, tokens_in=t_in, prompt_hash=h)
log("llm_response", paso=n, tool=call.name, args=call.args, tokens_out=t_out, ms=dur)
log("tool_result", paso=n, tool=call.name, ok=out.get("ok", True), ms=dur_tool,
    salida=str(out)[:500])                      # truncar: las salidas grandes inundan el log
log("run_end", pasos=n, tokens_total=tt, costo_usd=round(costo, 4), exito=ok)
```

Reglas prácticas:
- **Un `trace_id` por corrida** y propagarlo a todo (incluidas las tools). Es lo que te deja reconstruir el caso cuando el usuario reporta "no me funcionó".
- **Guardar el prompt final**, no el template. El bug suele estar en lo que se inyectó (memoria, RAG), no en el template.
- **Nunca loguear secretos ni datos personales en claro.** Redactá antes de emitir (Módulo 10).
- **Versionar prompt y modelo en cada traza** (`prompt_version`, `modelo`): sin eso no podés atribuir una regresión.

## Observability Tools (Herramientas de observabilidad)

Las cuatro del cuadro, con lo que aporta cada una:

- **LangSmith** (LangChain). Tracing detallado del árbol de ejecución, con el prompt y la respuesta de cada paso, tokens y costo; integra datasets y evaluaciones (Módulo 8). Funciona con o sin LangChain. Servicio administrado, plan gratuito para desarrollo. Es la opción más cómoda si ya estás en ese ecosistema.
- **Langfuse.** Observabilidad open source (self-hosted o cloud) para aplicaciones LLM: traces, sesiones, usuarios, costos, gestión de prompts, evaluaciones y anotación humana. Es la alternativa habitual cuando necesitás que los datos queden en tu infraestructura.
- **Helicone.** Open source, enfocado en el **gateway/proxy**: se integra cambiando la `base_url` del cliente, sin tocar el código de la app, y desde ahí da logging, métricas de costo y latencia, caching y rate limiting por usuario. La vía más rápida para tener visibilidad de costos.
- **OpenLLMetry** (Traceloop). Conjunto de extensiones sobre **OpenTelemetry** que instrumentan llamadas a LLMs, vector DBs y frameworks. Ventaja grande: exporta a cualquier backend compatible con OTel (Datadog, Grafana, New Relic, Jaeger), así que las trazas del agente viven junto al resto de tu observabilidad en vez de en un silo aparte. Es la opción natural si la empresa ya tiene OTel.

**Criterio de elección (opinión):** si el equipo ya usa OpenTelemetry, OpenLLMetry evita un silo. Si querés self-hosted con UI completa, Langfuse. Si lo urgente es controlar costos sin tocar código, Helicone. Si vas a hacer evals en serio sobre las trazas, LangSmith.

## Qué monitorear en producción

Las trazas sirven para debuggear un caso; el monitoreo sirve para enterarte antes de que te avisen. Alertas que vale tener:

- **Costo por hora/día** y **tokens promedio por tarea** (una regresión de prompt se ve primero acá).
- **Tasa de error de tools** por tool (una API externa que empezó a fallar).
- **Porcentaje de corridas que agotan `max_pasos`** (indicador de bucles).
- **Latencia p95**.
- **Tasa de escalamiento a humano** y thumbs down.

Los dos síntomas más comunes y su lectura: **bucle** (misma tool con mismos argumentos repetida) casi siempre es una tool que devuelve un error poco claro o una descripción ambigua; **respuesta plausible pero falsa** casi siempre es contexto recuperado insuficiente y falta de la instrucción "si no lo tenés, no lo inventes".

**Ejemplo concreto.** El agente de lotes empieza a responder mal un lunes. Con trazas: filtrás por `exito=false` del día, ves que el 70% tiene `tool_result` con `ok=false` en `consultar_rinde`, abrís una traza y en `salida` aparece un timeout del ERP. El modelo no estaba alucinando: la tool venía fallando y el error que devolvía ("error inesperado") no le daba al agente nada para corregirse. Arreglo: mensaje de error explícito, retry con backoff y alerta sobre esa tool. Sin trazas, el mismo diagnóstico es medio día de probar prompts al azar.

## Conclusión del módulo

Lo que hay que llevarse: `trace_id` por corrida, logs estructurados en los cuatro puntos del loop, guardar el prompt final y versionar prompt y modelo. Elegí una herramienta según dónde quieras los datos (OTel, self-hosted, proxy o plataforma) y poné alertas sobre costo, error de tools y corridas que agotan pasos. La observabilidad no es un lujo de producción: es lo que hace posible evaluar y mejorar.

Cómo conecta con el siguiente: los logs y las trazas contienen datos sensibles, y un agente con tools tiene capacidad real de hacer daño. El Módulo 10 (Security & Ethics) cierra el roadmap.
