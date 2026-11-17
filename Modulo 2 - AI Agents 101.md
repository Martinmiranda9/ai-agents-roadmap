# Módulo 2 — AI Agents 101 (Agentes de IA 101)

## What are AI Agents? (¿Qué son los agentes de IA?)

**Qué es.** Un agente es un sistema donde un LLM decide **qué hacer** y no solo **qué responder**: recibe un objetivo, elige acciones de un conjunto de herramientas disponibles, ejecuta, mira el resultado y vuelve a decidir, hasta cumplir el objetivo o agotar un límite.

La diferencia con un chatbot: el chatbot hace una pasada (prompt → respuesta). El agente itera y tiene efectos en el mundo (consulta una DB, manda un mail, escribe un archivo).

La diferencia con un workflow tradicional: en un workflow el orden de los pasos lo escribiste vos; en un agente el orden lo decide el modelo en runtime. Eso da flexibilidad y quita determinismo — de ahí la importancia de los módulos 8 y 9.

**Para qué sirve.** Automatizar tareas donde los pasos no se pueden enumerar de antemano porque dependen de lo que se vaya encontrando.

**Criterio honesto de cuándo NO usar un agente:** si el flujo es fijo y conocido, un script o un workflow (tipo n8n) es más barato, más rápido y más confiable. El agente se justifica cuando hay ramificación impredecible.

## What are Tools? (¿Qué son las tools?)

**Qué es.** Una tool es una función de tu código que el modelo puede pedir que se ejecute. Al modelo le pasás un *schema* (nombre, descripción, parámetros); el modelo responde con un pedido estructurado (`{"tool": "buscar_cliente", "args": {"dni": "30111222"}}`); **tu código** la ejecuta y le devuelve el resultado.

Punto clave que se malentiende: el modelo nunca ejecuta nada. Solo emite la intención. Toda la ejecución, validación y permisos están de tu lado. Se profundiza en el Módulo 4.

## Agent Loop (El bucle del agente)

El corazón del patrón. Cuatro fases que se repiten:

**1. Perception / User Input (Percepción / entrada).** Entra el objetivo: mensaje del usuario, un webhook, un evento de cron. Acá se arma el contexto inicial: system prompt, tools disponibles, memoria relevante.

**2. Reason and Plan (Razonar y planificar).** El modelo decide si ya puede responder o si necesita una herramienta, y con qué argumentos. Es una llamada al LLM con el historial completo.

**3. Acting / Tool Invocation (Actuar / invocar la tool).** Tu runtime toma el pedido, valida los argumentos, ejecuta la función y captura el resultado o el error.

**4. Observation & Reflection (Observación y reflexión).** El resultado se agrega al historial como una nueva observación y el bucle vuelve al paso 2. Si falló, el modelo ve el error y puede corregir (reintentar con otros argumentos, probar otra tool, avisar que no puede).

**Cómo se aplica.** Esqueleto mínimo, sin frameworks:

```python
historial = [{"role": "system", "content": SYSTEM}, {"role": "user", "content": objetivo}]

for paso in range(MAX_PASOS):                      # el techo evita bucles infinitos
    r = llm(historial, tools=SCHEMAS, temperature=0)
    if not r.tool_calls:                           # el modelo ya puede responder
        return r.content
    for call in r.tool_calls:
        try:
            salida = TOOLS[call.name](**call.args)  # ejecutás vos, no el modelo
        except Exception as e:
            salida = f"ERROR: {e}"                  # el error también es observación
        historial.append({"role": "tool", "tool_call_id": call.id, "content": str(salida)})
return "No pude completar la tarea dentro del límite de pasos."
```

Tres decisiones de diseño que aparecen acá y vuelven en todo el roadmap: el **límite de pasos**, el **manejo de errores como observación** (no como excepción que corta el flujo) y el **historial creciente** (que es el problema que resuelve el Módulo 5).

**Ejemplo concreto.** Objetivo: "¿cuánto facturó el cliente Pérez el mes pasado?".
Paso 1 — el modelo pide `buscar_cliente(nombre="Pérez")` → devuelve dos clientes homónimos. Paso 2 — el modelo pide desambiguar al usuario o elige por fecha de alta. Paso 3 — pide `facturacion(cliente_id=412, desde="2026-10-01", hasta="2026-10-31")` → `$1.284.300`. Paso 4 — responde en texto. Ninguno de esos pasos estaba programado: el modelo los encadenó.

## Example Usecases (Casos de uso de ejemplo)

- **Personal assistant (asistente personal).** Lee calendario y mail, agenda, resume, prepara respuestas. Tools: Calendar, Gmail, búsqueda. Es el caso donde más importa el *human in the loop*: que confirme antes de mandar.
- **Code generation (generación de código).** Agentes tipo Claude Code: leen el repo, editan archivos, corren tests y usan el resultado del test como observación para corregirse. El loop se cierra solo porque hay un verificador objetivo (el test).
- **Data analysis (análisis de datos).** El agente escribe y ejecuta código Python sobre un dataset, ve el error o el gráfico, y ajusta. Requiere sandbox (Módulo 10).
- **Web scraping / crawling.** El agente navega, decide qué links seguir según el objetivo y extrae datos estructurados. Aguanta cambios de layout que romperían un scraper con selectores fijos.
- **NPC / Game AI.** Personajes no jugables con memoria de interacciones previas y objetivos propios, que reaccionan a lo que hace el jugador en vez de seguir un árbol de diálogo.

Patrón común: todos tienen un objetivo difuso, herramientas acotadas y necesidad de verificar resultados intermedios.

## Conclusión del módulo

Lo que hay que llevarse: un agente es LLM + tools + loop + criterio de parada. Las cuatro fases (percibir, razonar, actuar, observar) son el esqueleto de absolutamente todo lo que sigue, incluso cuando lo esconde un framework. Y el buen juicio de ingeniería es saber cuándo el problema **no** necesita un agente.

Cómo conecta con el siguiente: la calidad de la fase "razonar y planificar" depende casi por completo de cómo esté escrito el prompt. El Módulo 3 (Prompt Engineering) es la palanca más barata para mejorar un agente.
