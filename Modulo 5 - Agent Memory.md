# Módulo 5 — Agent Memory (Memoria del agente)

## What is Agent Memory? (¿Qué es la memoria del agente?)

**Qué es.** Toda la información que el agente conserva entre pasos y entre sesiones. Es necesaria porque el LLM es *stateless*: cada llamada a la API arranca sin recordar nada. Lo que llamamos "memoria" es infraestructura que vos construís y que termina, siempre, en tokens dentro del prompt.

**Para qué sirve.** Que el agente no repregunte lo mismo, que mantenga coherencia en conversaciones largas y que personalice según el historial del usuario.

**Dónde se aplica.** Soporte (no pedir el número de cliente tres veces), asistentes personales (preferencias), agentes de código (decisiones de arquitectura ya tomadas).

## Short Term Memory (Memoria de corto plazo)

**Within Prompt (dentro del prompt).** Es el historial de la conversación o del loop actual: mensajes del usuario, decisiones del modelo, resultados de tools. Vive en el array de mensajes que mandás en cada llamada.

Ventaja: inmediata y sin infraestructura. Límite: la ventana de contexto y el costo. Cada paso del agente reenvía todo el historial, así que una conversación de 40 turnos paga los 40 turnos en cada llamada.

```python
historial.append({"role": "tool", "tool_call_id": call.id, "content": str(salida)})
if contar_tokens(historial) > UMBRAL:
    historial = comprimir(historial)   # ver Summarization más abajo
```

## Long Term Memory (Memoria de largo plazo)

**Qué es.** Información que persiste entre sesiones, fuera del prompt, en almacenamiento propio.

**Vector DB / SQL / Custom (opciones de almacenamiento).** No es una decisión ideológica, es de forma del dato:

- **SQL / NoSQL** para datos estructurados y hechos exactos: preferencias, IDs, configuraciones, historial de operaciones. Consulta determinista. Es la opción por defecto y la más subestimada.
- **Vector DB** para recuperación semántica de texto libre: fragmentos de conversaciones pasadas, notas, documentos. Consulta por similitud, con resultados aproximados.
- **Custom / híbrido** es lo habitual en producción: un perfil en SQL (rápido, exacto) + un índice vectorial para lo narrativo.

**Cómo se aplica.** Al armar el prompt: recuperás el perfil por clave y los recuerdos relevantes por similitud, y los inyectás en el system prompt.

```python
perfil = db.get_perfil(user_id)                       # SQL: exacto
recuerdos = vector.search(embed(mensaje), top_k=3)    # vectorial: semántico
system = f"{BASE}\nPerfil: {perfil}\nContexto previo: {recuerdos}"
```

## Episodic vs Semantic Memory (Memoria episódica vs semántica)

Distinción tomada de la psicología cognitiva, útil para decidir dónde guardar cada cosa:

- **Episódica:** eventos con tiempo y lugar. "El 12/11 el usuario reportó un error en el checkout y se escaló al nivel 2." Se guarda como registro fechado, se recupera por recencia o similitud.
- **Semántica:** hechos generales desprendidos del evento. "El usuario trabaja con Next.js y prefiere respuestas sin preámbulo." Se guarda como afirmación atemporal, se recupera casi siempre.

En la práctica: los episodios son el log; los hechos semánticos son el perfil. Un proceso (a veces el propio LLM) destila episodios en hechos.

## RAG and Vector Databases (RAG y bases vectoriales)

Mismo mecanismo visto en el Módulo 1, aplicado a memoria en vez de documentos: se embeddean fragmentos de interacciones pasadas, se indexan y se recuperan los top-k más cercanos a la consulta actual.

Diferencia clave con RAG documental: la memoria **se escribe constantemente** y envejece. Un documento es estable; un recuerdo puede quedar contradicho por el siguiente mensaje. Por eso hacen falta estrategias de actualización y olvido.

Prácticas que importan: deduplicar antes de escribir (si no, el mismo hecho queda 15 veces y domina la recuperación), guardar metadata (`user_id`, fecha, fuente) para poder filtrar, y poner umbral de similitud para no inyectar ruido.

## User Profile Storage (Almacenamiento de perfil de usuario)

**Qué es.** Un registro estructurado por usuario con lo estable: nombre, idioma, zona horaria, preferencias de formato, restricciones.

**Para qué sirve.** Es la memoria con mejor relación costo/beneficio: pocas líneas, siempre relevantes, consulta exacta y barata, y es lo que el usuario percibe como "me conoce".

**Cómo se aplica.** Tabla simple con campos acotados, actualizada cuando el usuario declara algo explícitamente. Precaución necesaria: no guardar datos sensibles (salud, documentos, datos financieros) sin consentimiento, y permitir ver y borrar lo guardado. Esto es obligación legal además de buena práctica (Módulo 10).

## Summarization / Compression (Resumen / compresión)

**Qué es.** Reemplazar partes del historial por un resumen más corto para seguir operando dentro de la ventana.

**Estrategias habituales:**

- **Resumen por bloques:** cuando el historial supera un umbral, el LLM resume los turnos más viejos y se conservan los últimos N completos.
- **Resumen progresivo:** se mantiene un único resumen que se actualiza con cada bloque nuevo.
- **Poda selectiva:** se tiran los *payloads* largos de tools (ya usados) y se deja la conclusión. En agentes con muchas tool calls, esto suele recuperar más contexto que cualquier resumen.

```python
def comprimir(historial, conservar=6):
    viejos, recientes = historial[1:-conservar], historial[-conservar:]
    resumen = llm_resumir(viejos)   # "Hechos y decisiones relevantes: ..."
    return [historial[0], {"role": "system", "content": f"Resumen previo: {resumen}"}, *recientes]
```

Costo real a asumir: resumir **pierde información** y el modelo no sabe qué se perdió. Hay que conservar literal lo que es crítico (IDs, montos, decisiones) y resumir lo narrativo.

## Forgetting / Aging Strategies (Estrategias de olvido y envejecimiento)

**Qué es.** Reglas para que la memoria no crezca infinitamente ni quede desactualizada.

- **TTL / expiración:** los recuerdos operativos vencen; los hechos de perfil no.
- **Decaimiento por recencia:** puntuar al recuperar con similitud × recencia, para que lo nuevo gane.
- **Supersesión:** cuando un hecho nuevo contradice uno viejo, se reemplaza, no se acumulan los dos. Es la regla más importante y la que más se olvida.
- **Olvido a pedido:** el usuario tiene que poder decir "olvidate de esto" y que efectivamente se borre.

**Ejemplo concreto.** Asistente de soporte. Perfil en SQL: cliente, plan, idioma. Episodios en vector DB: cada ticket resuelto con su solución. Al abrir un chat nuevo se inyecta el perfil completo y los 3 tickets más parecidos al problema actual. Los episodios de más de 12 meses se archivan; si el cliente cambia de plan, el campo se sobrescribe. Resultado: el agente arranca sabiendo quién es y qué le pasó antes, con un prompt de tamaño constante.

## Conclusión del módulo

Lo que hay que llevarse: memoria = corto plazo (historial en el prompt, limitado por la ventana) + largo plazo (SQL para lo exacto, vectorial para lo semántico). Distinguir episodios de hechos; comprimir conservando literal lo crítico; y tener política explícita de supersesión y olvido. Todo termina en tokens, así que cada byte de memoria compite con el resto del prompt.

Cómo conecta con el siguiente: con tools y memoria resueltas, la pregunta pasa a ser cómo se **organiza** el razonamiento y cómo se conectan agentes y herramientas entre sistemas. Eso es el Módulo 6 (Agent Architectures), incluido MCP.
