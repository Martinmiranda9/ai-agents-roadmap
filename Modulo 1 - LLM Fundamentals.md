# Módulo 1 — LLM Fundamentals (Fundamentos de LLM)

Prerrequisito del roadmap. Sin esto, todo lo que viene después (agent loop, tools, memoria) se usa a ciegas.

## Transformer Models and LLMs (Modelos transformer y LLMs)

**Qué es.** Un LLM es una red neuronal con arquitectura *transformer* entrenada para predecir el próximo token. El mecanismo central es *self-attention*: cada token pesa su relación con los demás tokens de la secuencia, lo que permite capturar dependencias largas sin recurrencia.

**Para qué sirve.** Es el motor de razonamiento del agente. El agente no "piensa": le pide al modelo que genere texto, y ese texto incluye las decisiones.

**Dónde se aplica.** Todo: chatbots, generación de código, clasificación, extracción de datos.

**Cómo se aplica.** Como dev consumís el modelo por API. Lo importante no es implementar attention, sino entender tres consecuencias prácticas: el modelo es *stateless* (no recuerda nada entre llamadas, vos le mandás el historial), es *probabilístico* (misma entrada puede dar salidas distintas) y tiene *límite de contexto*.

## Model Mechanics (Mecánica del modelo)

> Nota: en el PDF este nodo figura cortado como "Model Mechanis". El nodo correcto es **Model Mechanics**.

Agrupa tres cosas que impactan directo en costo y diseño:

**Tokenization (Tokenización).** El texto se parte en tokens (subpalabras). En inglés ≈ 4 caracteres por token; en español rinde algo peor por acentos y palabras más largas. Se factura por token, de entrada y de salida.

```python
import tiktoken
enc = tiktoken.get_encoding("cl100k_base")
print(len(enc.encode("El agente consultó la base de datos")))  # cuenta tokens
```

**Context Windows (Ventana de contexto).** Es el máximo de tokens que entran en una llamada (prompt + respuesta). Cuando un agente corre muchos pasos, el historial crece y choca con ese techo: de ahí nace la necesidad de memoria y resumen (Módulo 5).

**Token Based Pricing (Precio por token).** Se cobra por millón de tokens, con tarifa distinta para input y output (output suele ser bastante más caro). Muchos proveedores ofrecen *prompt caching* y *batch* más baratos. Los precios cambian seguido: el número hay que mirarlo en la página de pricing del proveedor, no memorizarlo.

## Generation Controls (Controles de generación)

Parámetros que cambian el comportamiento del modelo sin tocar el prompt:

- **Temperature.** Cuánta aleatoriedad. `0` ≈ determinista y repetible (lo que querés en un agente que decide tools); valores altos, más creatividad y más desvío.
- **Top-p (nucleus sampling).** Recorta el muestreo al conjunto de tokens que acumulan probabilidad `p`. Se ajusta `temperature` **o** `top_p`, no los dos a la vez.
- **Frequency penalty.** Penaliza tokens según cuántas veces ya aparecieron: baja la repetición literal.
- **Presence penalty.** Penaliza tokens que ya aparecieron al menos una vez: empuja a introducir temas nuevos.
- **Stopping criteria / Max length.** `stop` corta la generación al encontrar una secuencia; `max_tokens` pone techo duro a la salida. En agentes, el techo es red de seguridad contra bucles y facturas infladas.

```python
resp = client.chat.completions.create(
    model="<modelo>",
    messages=[{"role": "user", "content": "Decidí la próxima acción"}],
    temperature=0,          # decisiones estables
    max_tokens=300,         # techo de salida
    stop=["\nObservation:"] # cortar antes de alucinar la observación
)
```

**Ejemplo concreto.** Un agente que elige entre `buscar_precio` y `crear_orden` con `temperature=0.9` elige distinto en cada corrida con el mismo input: imposible de testear. Con `temperature=0` el comportamiento se vuelve reproducible.

## Open vs Closed Weight Models (Modelos de pesos abiertos y cerrados)

**Open weight:** los pesos se descargan y se corren propios (familias Llama, Mistral, Qwen, DeepSeek, Gemma). Ventaja: datos que no salen de tu infra, costo fijo de hardware, fine-tuning libre. Costo: ops, GPU, y generalmente menor capacidad de razonamiento que el tope de gama cerrado.

**Closed weight:** se accede solo por API (Claude, GPT, Gemini). Ventaja: lo mejor disponible sin infra. Costo: dependencia del proveedor, datos que salen, precio por uso.

**Model Families and Licences (Familias y licencias).** "Open weight" no es lo mismo que "open source": varias licencias (por ejemplo las de la familia Llama) imponen restricciones de uso comercial o de escala. Antes de meter un modelo abierto en producción, leé la licencia.

## Streamed vs Unstreamed Responses (Respuestas con y sin streaming)

Sin streaming esperás la respuesta completa; con streaming recibís tokens a medida que se generan (SSE). Streaming mejora la percepción de velocidad en UI de chat, pero complica el parseo cuando necesitás JSON completo para decidir una tool. Regla práctica: streaming en la respuesta final al usuario, sin streaming en los pasos internos del agente.

## Reasoning vs Standard Models (Modelos de razonamiento vs estándar)

Los modelos de *reasoning* generan pasos intermedios antes de responder (razonamiento extendido), lo que mejora planificación, matemática y código difícil, a cambio de más latencia y más tokens. Los estándar son más rápidos y baratos. En un agente conviene mezclar: reasoning para el nodo de planificación, estándar para extraer un campo de un texto.

## Fine-tuning vs Prompt Engineering

**Prompt engineering** cambia el comportamiento con instrucciones y ejemplos: costo casi nulo, iteración inmediata. **Fine-tuning** reentrena el modelo con tus pares entrada/salida: sirve para fijar formato o estilo muy específico y para bajar costo por token al usar un modelo chico, pero necesita dataset, dinero y se desactualiza con cada modelo nuevo. Orden recomendado: prompt → ejemplos (few-shot) → RAG → fine-tuning como último recurso.

## Embeddings and Vector Search (Embeddings y búsqueda vectorial)

**Qué es.** Un embedding convierte texto en un vector de N dimensiones donde la cercanía geométrica (coseno) aproxima la cercanía semántica.

**Para qué sirve.** Buscar por significado y no por palabra exacta: "no me anda el pago" matchea con "error en checkout".

```python
from openai import OpenAI
client = OpenAI()
v = client.embeddings.create(model="<modelo-embeddings>", input="no me anda el pago")
# v.data[0].embedding -> lista de floats; se guarda en una vector DB
```

Se almacenan en bases vectoriales (pgvector, Qdrant, Chroma, Pinecone, Weaviate) que indexan para búsqueda aproximada (ANN).

## Understand the Basics of RAG (Fundamentos de RAG)

**Qué es.** *Retrieval-Augmented Generation*: antes de responder, se recupera contexto relevante de una fuente externa y se inyecta en el prompt.

**Pipeline.** Ingesta → chunking (fragmentos con solapamiento) → embeddings → índice vectorial. En consulta: embeddear la pregunta → traer top-k → armar prompt con esos fragmentos → generar respuesta citando la fuente.

**Para qué sirve.** Dar al modelo datos que no están en su entrenamiento (manuales internos, precios, documentación) sin reentrenarlo, y reducir alucinación al anclar la respuesta en texto recuperado.

**Ejemplo concreto.** Soporte técnico sobre 400 PDFs de manuales: el agente recupera los 5 fragmentos más cercanos a la consulta y responde citando página. Si no recupera nada por encima de un umbral, responde "no lo encuentro" en vez de inventar.

## Conclusión del módulo

Lo que hay que llevarse: el LLM es un predictor de tokens *stateless*, probabilístico y con ventana limitada; `temperature=0` y `max_tokens` son las dos perillas que vuelven un agente testeable; el costo se mide en tokens y crece con el historial; y embeddings + RAG son la forma estándar de darle conocimiento propio sin reentrenar.

Cómo conecta con el siguiente: todas esas limitaciones son exactamente las que el patrón de **agente** viene a resolver. El Módulo 2 (AI Agents 101) arma el *agent loop* que convierte este predictor de texto en algo que percibe, planifica, actúa y observa.
