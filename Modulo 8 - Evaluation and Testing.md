# Módulo 8 — Evaluation and Testing (Evaluación y testing)

El problema de fondo: un agente es no determinista y la salida es texto. No hay un `assert resultado == esperado` universal. Hay que construir el criterio de corrección.

## Metrics to Track (Métricas a seguir)

**Qué es.** El conjunto de indicadores que te dicen si el agente anda bien, separados en cuatro familias:

**1. Calidad de la tarea (lo que más importa).**
- **Task success rate:** porcentaje de casos resueltos correctamente. Es la métrica madre; requiere definir "correcto" por caso.
- **Exactitud factual / groundedness:** si la respuesta está respaldada por lo que las tools devolvieron, o el agente inventó.
- **Tool selection accuracy:** si eligió la tool correcta y con los argumentos correctos.

**2. Eficiencia.**
- **Pasos por tarea** (y cuántas veces pega el techo de `max_pasos`).
- **Tokens por tarea** y **costo por tarea**: la métrica que mira el negocio.
- **Latencia** p50 y p95.

**3. Robustez.**
- **Tool error rate** y **tasa de recuperación** (de los errores, cuántos resolvió solo).
- **Tasa de bucles** (repite la misma tool con los mismos argumentos).

**4. Experiencia.**
- Escalamiento a humano, abandono, thumbs up/down.

**Cómo se aplica.** Instrumentá el loop para emitir estos números por corrida y guardalos; sin eso, "mejoró" es impresión personal. Lo mínimo viable: success rate, costo por tarea y tool error rate.

## Unit Testing for Individual Tools (Testing unitario de tools)

**Qué es.** Tests clásicos y deterministas sobre las funciones de las tools, **sin LLM en el medio**.

**Para qué sirve.** Es la parte del sistema que sí es determinista: aprovechala. Buena parte de los fallos que parecen "el modelo se confundió" son en realidad una tool que devuelve mal un caso borde.

```python
def test_registrar_carga_lote_inexistente():
    r = registrar_carga(lote_id=9999, cultivo="soja", bolsas=10)
    assert r["ok"] is False
    assert "no existe" in r["error"]          # el mensaje guía al modelo

def test_registrar_carga_es_idempotente():
    a = registrar_carga(lote_id=3, cultivo="soja", bolsas=10, idem_key="k1")
    b = registrar_carga(lote_id=3, cultivo="soja", bolsas=10, idem_key="k1")
    assert a["id"] == b["id"]                 # el reintento no duplica
```

Qué testear: validación de argumentos, casos borde, y sobre todo que **los mensajes de error sean accionables** (son parte del contrato con el modelo).

## Integration Testing for Flows (Testing de integración de flujos)

**Qué es.** Ejecutar el agente completo sobre casos reales y verificar el resultado y la traza.

**Cómo se aplica.** Tres niveles de verificación, de más barato a más caro:

1. **Assertions estructurales (deterministas).** No mires el texto: mirá los efectos. ¿Llamó a `registrar_carga` exactamente una vez? ¿Quedó la fila en la base? ¿Terminó en menos de 6 pasos? Esto es rápido, estable y atrapa la mayoría de las regresiones.
2. **LLM-as-judge.** Otro modelo evalúa la respuesta contra una rúbrica ("¿responde la pregunta? ¿está respaldada por las observaciones? ¿inventó datos?"). Útil y escalable, pero es un evaluador falible: conviene rúbrica cerrada, salida estructurada y validar el judge contra juicios humanos en una muestra.
3. **Golden dataset.** 30-100 casos con entrada y criterio de éxito, que corren en CI. Se arma incrementalmente: **cada bug reportado entra como caso nuevo**. Es el activo más valioso que vas a construir.

```python
CASOS = [
  {"input": "metí 40 bolsas de soja en el 3",
   "espera_tool": "registrar_carga",
   "espera_args": {"lote_id": 3, "cultivo": "soja", "bolsas": 40}},
]

def test_flujo():
    for c in CASOS:
        traza = agente(c["input"], trazar=True)
        calls = [t.name for t in traza.tool_calls]
        assert c["espera_tool"] in calls
```

Detalle práctico: fijá `temperature=0` y mockeá las tools externas para que los tests sean repetibles y no gasten plata.

## Human in the Loop Evaluation (Evaluación con humano en el circuito)

**Qué es.** Que una persona revise salidas del agente, en dos momentos distintos:

- **En producción, antes de actuar:** el agente propone y el humano aprueba acciones irreversibles (mandar el mail, emitir la factura, borrar). No es evaluación, es control de riesgo — y es obligatorio en acciones con efecto externo.
- **Como evaluación periódica:** muestreo de N conversaciones por semana, etiquetadas por una persona que conoce el dominio. Es la única fuente de verdad real y la que calibra al LLM-as-judge.

**Cómo se aplica.** Interfaz mínima de revisión (conversación + traza de tools + etiqueta correcto/incorrecto + motivo), y los casos marcados incorrectos pasan directo al golden dataset. Sumale thumbs up/down en la app para recolectar señal gratis del usuario final.

## Frameworks (Herramientas de evaluación)

- **LangSmith** (LangChain). Plataforma de tracing + evals + datasets. Permite guardar datasets, correr evaluaciones automáticas y manuales, comparar versiones de prompt y pasar de una traza real a un caso de test. Funciona con o sin LangChain. Es servicio administrado (hay plan gratuito).
- **Ragas.** Open source, especializado en **pipelines de RAG**: mide *faithfulness* (si la respuesta se sostiene en el contexto), *answer relevancy*, *context precision* y *context recall*. Lo que te dice si el problema está en la recuperación o en la generación. Si tu agente usa RAG, es la herramienta indicada.
- **DeepEval** (Confident AI). Open source, framework de evals con sabor pytest: métricas listas (alucinación, relevancia, toxicidad, corrección por G-Eval), datasets y corridas en CI. Buena opción si querés tus evals dentro del repo y no en un servicio.

Elección razonable: Ragas si el núcleo es RAG, DeepEval si querés evals como tests versionados, LangSmith si querés tracing y evaluación integrados en una plataforma. No son excluyentes.

**Ejemplo concreto.** Agente de consultas sobre lotes: 40 casos en el golden set (20 consultas, 10 cargas, 10 fuera de alcance). En CI corre con `temperature=0` y tools mockeadas, y verifica tool + argumentos + que los fuera de alcance se rechacen. Semanalmente, 20 conversaciones reales revisadas a mano; las falladas entran al set. Al cambiar el system prompt, se corren los 40 casos y se compara: así "mejoré el prompt" pasa a ser un número.

## Conclusión del módulo

Lo que hay que llevarse: testeá determinista lo que sea determinista (tools, efectos, cantidad de pasos) y reservá el LLM-as-judge para lo que es irreductiblemente textual, siempre calibrado contra humanos. El golden dataset construido a partir de fallas reales es lo que separa un demo de un producto. Y medí costo por tarea desde el día uno.

Cómo conecta con el siguiente: para evaluar y para arreglar necesitás ver qué pasó dentro de cada corrida. Eso requiere trazas y observabilidad: Módulo 9.
