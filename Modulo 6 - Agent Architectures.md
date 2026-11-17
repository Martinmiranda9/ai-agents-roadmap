# Módulo 6 — Agent Architectures (Arquitecturas de agentes)

Patrones probados para organizar el razonamiento, más el protocolo que estandarizó la conexión con herramientas (MCP).

## Common Architectures (Arquitecturas comunes)

### ReAct (Reason + Act)

**Qué es.** El patrón base: el agente alterna explícitamente *Thought* (razonamiento) → *Action* (tool) → *Observation* (resultado), y repite. Viene del paper "ReAct: Synergizing Reasoning and Acting in Language Models" (2022).

**Para qué sirve.** Obliga al modelo a justificar cada acción antes de ejecutarla, lo que mejora la selección de tools y deja una traza legible para debuggear.

**Dónde se aplica.** Es el default de la mayoría de los frameworks y la implementación típica del agent loop del Módulo 2.

```
Thought: necesito el id del cliente antes de facturar
Action: buscar_cliente({"razon_social": "Agro SRL"})
Observation: {"id": 412, "estado": "al dia"}
Thought: ya tengo el id, puedo consultar la facturación
Action: facturacion({"cliente_id": 412, "mes": "2026-10"})
```

Hoy con modelos que soportan tool calling nativo no hace falta parsear ese texto a mano, pero el esquema conceptual es el mismo.

### Chain of Thought (CoT, Cadena de pensamiento)

**Qué es.** Pedirle al modelo que desarrolle el razonamiento paso a paso antes de dar la respuesta final.

**Para qué sirve.** Mejora tareas de varios pasos (cálculo, lógica, planificación). Diferencia con ReAct: CoT es razonamiento puro, sin acciones externas; ReAct intercala acciones.

**Nota de actualidad.** Con los modelos de *reasoning* el CoT explícito en el prompt rinde menos, porque el modelo ya razona internamente. Sigue siendo útil en modelos estándar y cuando querés ver el razonamiento.

### RAG Agent (Agente con RAG)

**Qué es.** Un agente que tiene la recuperación como una tool y decide **si** buscar, **qué** buscar y **si lo recuperado alcanza**. Si no alcanza, reformula la consulta y vuelve a buscar.

**Para qué sirve.** Supera al RAG clásico de una sola pasada en preguntas que requieren varias búsquedas o descomposición ("comparame el rinde de soja de los últimos tres años": tres recuperaciones, no una).

**Ejemplo.** Pregunta sobre dos manuales distintos: el agente busca en el primero, nota que falta la mitad de la respuesta, busca en el segundo y recién entonces responde.

## Other Architecture Patterns (Otros patrones)

**Planner-Executor (Planificador-Ejecutor).** Un modelo fuerte genera el plan completo por adelantado; uno más barato ejecuta cada paso. Sirve para tareas largas con pasos conocidos: baja costo y hace el proceso auditable (podés revisar el plan antes de ejecutarlo). Contra: si el plan arranca mal, todo sale mal; conviene permitir replanificación.

**DAG Agents (Agentes como grafo dirigido acíclico).** El flujo se modela como grafo de nodos con dependencias; lo que no depende entre sí corre en paralelo. Da control, retries por nodo y estado persistente. Es el modelo de LangGraph. Para flujos semi-estructurados suele ser la opción más confiable en producción.

**Tree-of-Thought (ToT, Árbol de pensamiento).** El modelo genera varias ramas de razonamiento, las evalúa y se queda con la mejor. Mejora problemas de búsqueda y planificación, a costa de multiplicar llamadas y costo. Uso justificado en problemas difíciles y acotados, no en un chat de soporte.

**Complemento (no está en el PDF, sí en el roadmap online):** **Multi-Agent** (varios agentes especializados que se coordinan, cada uno con su propio set chico de tools; es la salida al problema de "demasiadas tools") y **Self-critique / Reflexion** (el agente critica su propia salida y la corrige antes de entregarla; funciona mejor cuando hay un verificador objetivo, como un test que corre).

## Model Context Protocol (MCP)

**Qué es.** Un protocolo abierto que estandariza cómo una aplicación de IA se conecta a herramientas y datos externos. Creado por Anthropic en noviembre de 2024 y **donado a la Linux Foundation en diciembre de 2025**, donde es proyecto fundacional de la Agentic AI Foundation (AAIF) junto a goose de Block y AGENTS.md de OpenAI, bajo licencia Apache 2.0.

**Para qué sirve.** Resolver el problema N×M: antes, cada aplicación implementaba su propia integración para cada herramienta. Con MCP, el que publica la herramienta escribe **un** servidor y cualquier cliente compatible lo usa. La analogía que se repite es "USB-C para IA".

**Dónde se aplica.** Está soportado por Claude (apps y Claude Code), y por clientes y SDKs de otros proveedores (la Responses API de OpenAI es MCP-native, Codex CLI, Gemini CLI, entre otros). Hay miles de servidores comunitarios publicados.

### Arquitectura

- **MCP Host (anfitrión).** La aplicación donde vive el modelo y que coordina todo: Claude Desktop, un IDE, tu propia app. Decide qué servidores habilitar y pide permiso al usuario.
- **MCP Client (cliente).** El componente dentro del host que mantiene la conexión 1-a-1 con un servidor y traduce entre el protocolo y el modelo.
- **MCP Server (servidor).** El proceso que expone las capacidades. Puede ser propio o de terceros (GitHub, Slack, Postgres, filesystem).

### Core Components (Componentes que expone un servidor)

- **Tools (herramientas).** Funciones ejecutables que el modelo puede invocar. Es lo más usado.
- **Resources (recursos).** Datos de solo lectura identificados por URI que el host puede leer y pasar como contexto (archivos, registros).
- **Prompts.** Plantillas reutilizables que el servidor ofrece al usuario (por ejemplo "revisá este PR").

La separación importa: *tools* las decide el modelo, *resources* y *prompts* los elige la aplicación o el usuario. Es un límite de control, no solo de nomenclatura.

### Creating MCP Servers (Crear servidores MCP)

Hay SDKs oficiales (Python, TypeScript, y otros). Con el SDK de Python, un servidor mínimo:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("agro")

@mcp.tool()
def consultar_rinde(lote_id: int, campania: str) -> dict:
    """Devuelve el rinde en qq/ha de un lote para una campaña."""
    return {"lote": lote_id, "campania": campania, "rinde": 38.5}

if __name__ == "__main__":
    mcp.run()
```

El docstring y los type hints se convierten en el schema que ve el modelo: vale todo lo del Módulo 4 sobre descripciones.

### Deployment Modes (Modos de despliegue)

- **Local Desktop.** El servidor corre como proceso local y se comunica por **stdio**. Ideal para acceso a filesystem, bases locales o credenciales que no deben salir de la máquina. Es el modo más simple y el más seguro por defecto.
- **Remote / Cloud.** El servidor corre como servicio HTTP (transporte *Streamable HTTP*; SSE quedó como legado) y se comparte entre usuarios. Requiere autenticación y autorización reales. Las revisiones recientes de la especificación agregaron, entre otras cosas, operaciones asíncronas, mejoras de escalabilidad e identidad de servidor; la spec se versiona con fecha y las deprecaciones tienen ventana de gracia, así que conviene fijar la versión contra la que desarrollás.

**Ejemplo concreto.** En vez de programar tools de Postgres dentro de tu agente, corrés un servidor MCP de Postgres en modo local; el mismo servidor te sirve en Claude Desktop, en tu IDE y en tu agente propio. Si mañana cambiás de framework, las tools siguen funcionando: el acoplamiento quedó en el protocolo, no en tu código.

## Conclusión del módulo

Lo que hay que llevarse: ReAct es el default razonable; DAG/Planner-Executor son la respuesta cuando necesitás control y confiabilidad; ToT y multi-agente son para problemas que lo justifiquen por costo. MCP es la pieza de infraestructura que conviene aprender bien: desacopla las herramientas del agente y hoy es un estándar con gobernanza neutral, no una apuesta de un solo proveedor.

Cómo conecta con el siguiente: elegida la arquitectura, hay que construir. El Módulo 7 (Building Agents) recorre las tres vías: manual, function calling nativo y frameworks.
