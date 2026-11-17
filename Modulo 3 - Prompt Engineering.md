# Módulo 3 — Prompt Engineering (Ingeniería de prompts)

## What is Prompt Engineering (Qué es la ingeniería de prompts)

**Qué es.** La práctica de escribir y refinar las instrucciones que recibe el modelo para obtener salidas correctas, consistentes y parseables. En un agente hay dos prompts críticos: el **system prompt** (define rol, reglas, cuándo usar cada tool, formato de salida) y las **descripciones de las tools** (que el modelo lee para decidir).

**Para qué sirve.** Es la forma más barata y rápida de cambiar comportamiento: no requiere dataset, ni entrenamiento, ni deploy de modelo. Antes de pensar en fine-tuning o en cambiar de framework, el prompt es lo primero que se ajusta.

**Dónde se aplica.** En el system prompt del agente, en las descripciones de tools, en los prompts de sub-tareas (resumir, clasificar, extraer) y en los *judges* de evaluación (Módulo 8).

## Writing Good Prompts (Escribir buenos prompts)

Los seis nodos del roadmap, con lo que significan en la práctica de un agente:

**Be specific in what you want (Sé específico).** Decí el objetivo, el formato y los límites. Un prompt vago produce salidas distintas en cada corrida, y en un agente eso rompe el parseo del paso siguiente.

- Vago: `Analizá estas ventas.`
- Específico: `Calculá el total por mes y devolvé solo un JSON {"mes": "YYYY-MM", "total": number}. Sin texto adicional.`

**Provide additional context (Dale contexto).** El modelo no conoce tu dominio ni tu negocio. Hay que inyectar las reglas que un empleado nuevo necesitaría: qué significa cada campo, qué unidades, qué casos son excepción.

**Use relevant technical terms (Usá términos técnicos precisos).** "Hacé un join entre `clientes` y `facturas` por `cliente_id` y agrupá por mes" dirige mucho mejor que "cruzá los datos". El vocabulario técnico activa el patrón correcto y reduce ambigüedad.

**Use Examples in your Prompt (Usá ejemplos / few-shot).** Mostrar 2-5 pares entrada→salida es la forma más efectiva de fijar un formato. Para un agente, el ejemplo más valioso es uno que muestre **el caso borde**: qué hacer cuando no hay datos.

```python
SYSTEM = """Clasificá el pedido del productor en: carga_datos | consulta | fuera_de_alcance.
Devolvé solo la etiqueta.

Ejemplos:
"anoté 40 bolsas de soja en el lote 3" -> carga_datos
"cuánto rinde el lote 3" -> consulta
"qué tiempo va a hacer el finde" -> fuera_de_alcance
"""
```

**Specify Length, format etc (Especificá largo y formato).** Pedí explícitamente JSON, markdown, cantidad de ítems o de palabras. Si el paso siguiente del agente parsea la salida, el formato no es cosmético: es contrato. Cuando el proveedor lo soporta, conviene reforzarlo con *structured output* / JSON schema en vez de confiar solo en el texto del prompt.

**Iterate and Test your Prompts (Iterá y testeá).** Un prompt no se escribe, se calibra: armá un set chico de casos reales (incluyendo los que fallaron), cambiá una cosa a la vez y medí. Esto es el puente directo al Módulo 8: sin un set de casos, "mejoré el prompt" es una opinión.

## Técnicas que conviene conocer

Complementan los nodos del cuadro y son consenso en la práctica:

- **Rol y alcance explícito.** "Sos el asistente de carga de datos de X. Solo podés usar las tools dadas. Si el pedido está fuera de alcance, decilo." Reduce invención de capacidades.
- **Chain of Thought (cadena de pensamiento).** Pedir que razone paso a paso antes de responder mejora tareas de varios pasos. Se amplía en el Módulo 6. Con modelos de reasoning es menos necesario: ya lo hacen internamente.
- **Decir qué hacer ante la duda.** La instrucción más rentable en un agente: "si no tenés el dato, no lo inventes: usá la tool o pedí la información faltante".
- **Delimitadores.** Separar instrucciones de datos del usuario con etiquetas (`<datos>...</datos>`) ordena el prompt y ayuda contra prompt injection (Módulo 10).

## Ejemplo concreto: antes y después

Un agente de consultas a base de datos con este system prompt:

> `Respondé preguntas sobre la base de datos.`

falla así: inventa nombres de tablas, devuelve prosa cuando el backend espera JSON, y ante una pregunta ambigua elige una interpretación al azar.

Reescrito:

> `Sos un asistente de consultas SQL de solo lectura sobre el esquema <esquema>...</esquema>. Reglas: (1) usá únicamente tablas y columnas de ese esquema; (2) si la pregunta es ambigua, pedí la aclaración en vez de suponer; (3) devolvé {"sql": "...", "explicacion": "..."}; (4) nunca generes UPDATE, DELETE ni DROP. Ejemplos: ...`

Mismo modelo, mismas tools: el comportamiento pasa de impredecible a auditable. Es la intervención con mejor relación costo/beneficio de todo el roadmap.

## Conclusión del módulo

Lo que hay que llevarse: el prompt es código y merece el mismo rigor (versionado, casos de prueba, un cambio a la vez). Las seis reglas del cuadro se resumen en tres: especificidad, contexto de dominio y formato explícito con ejemplos. Y siempre hay que definir el comportamiento ante la duda.

Cómo conecta con el siguiente: el prompt le dice al modelo *cómo* decidir; las **tools** son *entre qué* puede decidir. Y como el modelo elige la tool leyendo su descripción, escribir tools es, en buena medida, seguir haciendo prompt engineering. Eso es el Módulo 4.
