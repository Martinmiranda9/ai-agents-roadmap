# Módulo 4 — Tools / Actions (Herramientas / acciones)

Las tools son lo que convierte al modelo en algo que *hace*. Un agente sin tools es un chat; con tools, es software con efectos sobre sistemas reales.

## Tool Definition (Definición de una tool)

Una tool tiene dos caras: el **schema** que ve el modelo y la **función** que corre tu runtime. El modelo elige basándose exclusivamente en el schema, así que el schema es parte del prompt.

### Name and Description (Nombre y descripción)

**Qué es.** El identificador y la explicación en lenguaje natural de qué hace la tool y cuándo usarla.

**Por qué importa.** Es el único criterio de selección del modelo. Dos tools con descripciones parecidas garantizan que el agente elija mal.

Regla práctica: nombre en `snake_case` con verbo (`buscar_cliente`, `crear_orden`), y descripción que diga **qué hace, cuándo usarla y cuándo NO usarla**.

- Pobre: `get_data` — "obtiene datos".
- Buena: `buscar_cliente` — "Busca clientes por DNI o razón social y devuelve id, nombre y estado de cuenta. Usar antes de cualquier operación que requiera `cliente_id`. No sirve para buscar proveedores."

### Input / Output Schema (Esquema de entrada y salida)

**Qué es.** La declaración tipada de los parámetros (JSON Schema) y la forma del retorno.

**Para qué sirve.** El schema restringe lo que el modelo puede pedir: tipos, enums, campos obligatorios. Cuanto más estricto, menos alucinación de argumentos. Los `enum` son especialmente efectivos.

```python
SCHEMA = {
    "name": "registrar_carga",
    "description": "Registra bolsas cosechadas de un lote. Usar solo con lote existente.",
    "input_schema": {
        "type": "object",
        "properties": {
            "lote_id": {"type": "integer", "description": "ID del lote"},
            "cultivo": {"type": "string", "enum": ["soja", "maiz", "trigo"]},
            "bolsas": {"type": "integer", "minimum": 1},
            "fecha": {"type": "string", "description": "YYYY-MM-DD"}
        },
        "required": ["lote_id", "cultivo", "bolsas"]
    }
}
```

Del lado de la salida: devolvé algo compacto y legible para el modelo. Volcarle un JSON de 300 líneas quema contexto y lo confunde; devolvé los campos que necesita para decidir.

### Error Handling (Manejo de errores)

**Qué es.** Qué recibe el modelo cuando la tool falla.

**Principio central.** El error es una **observación**, no una excepción que corta el programa. Si la tool tira error y vos lo capturás y se lo devolvés como texto útil, el modelo puede corregirse solo.

```python
def registrar_carga(lote_id, cultivo, bolsas, fecha=None):
    if not existe_lote(lote_id):
        return {"ok": False, "error": f"El lote {lote_id} no existe. Usá listar_lotes para ver los válidos."}
    ...
```

Ese mensaje le dice al modelo **qué** falló y **cómo** seguir. Comparalo con un `500 Internal Server Error`: con eso, el agente reintenta igual y vuelve a fallar. Agregá además idempotencia en las tools que escriben (para que un reintento no duplique la carga) y timeouts.

### Usage Examples (Ejemplos de uso)

Incluir 1-2 ejemplos de invocación en la descripción cuando los argumentos son no obvios (formatos de fecha, IDs compuestos, unidades). Barato y muy efectivo.

## Examples of Tools (Ejemplos de tools)

- **Web Search (búsqueda web).** Da acceso a información posterior al entrenamiento. Devolvé resultados resumidos con URL, no el HTML crudo. Riesgo a tener presente: el contenido recuperado es dato no confiable y puede contener prompt injection (Módulo 10).
- **Code Execution / REPL (ejecución de código).** La tool más potente y la más peligrosa: el agente escribe Python y lo corre. Resuelve cálculo, gráficos y transformación de datos sin que programes cada caso. Exige sandbox (contenedor aislado, sin red, con límite de CPU y tiempo). Servicios como E2B o Daytona existen justamente para esto.
- **Database Queries (consultas a base de datos).** Dos enfoques: tools acotadas (`facturacion_por_mes(cliente_id)`), o SQL libre contra una réplica de solo lectura. El primero es seguro y predecible; el segundo es flexible y requiere usuario read-only, whitelist de tablas y límite de filas.
- **API Requests (llamadas a APIs).** Envolver endpoints de terceros (ERP, CRM, clima, pagos). Nunca expongas un `http_request(url, method, body)` genérico: es un agujero de seguridad y el modelo arma mal los requests. Una tool por operación.
- **Email / Slack / SMS.** Acciones con efecto hacia afuera e irreversibles. Patrón estándar: el agente **redacta** y un humano **confirma** antes del envío (human in the loop).
- **File System Access (acceso al filesystem).** Leer y escribir archivos, base de los agentes de código. Restringí a un directorio raíz, validá *path traversal* (`../`) y mantené las escrituras auditables.

## Cómo se arma el mapa de tools

```python
TOOLS = {"registrar_carga": registrar_carga, "listar_lotes": listar_lotes}

def ejecutar(call):
    fn = TOOLS.get(call.name)
    if not fn:
        return {"ok": False, "error": f"Tool inexistente. Disponibles: {list(TOOLS)}"}
    try:
        return fn(**call.args)          # validar args antes si son sensibles
    except Exception as e:
        return {"ok": False, "error": str(e)}
```

**Ejemplo concreto.** Un agente de WhatsApp para un productor con cuatro tools: `listar_lotes`, `registrar_carga`, `consultar_rinde`, `resumen_campania`. El productor escribe "metí 40 bolsas de soja en el 3". El modelo llama `registrar_carga(lote_id=3, cultivo="soja", bolsas=40)`; la tool valida que el lote 3 exista y que el cultivo corresponda a ese lote; si no, devuelve el error explicativo y el agente repregunta. Cuatro tools bien descritas rinden más que veinte genéricas.

## Cuántas tools darle

Consenso práctico: más tools = peor selección. Arriba de 15-20 el modelo empieza a confundirse y el schema consume mucho contexto. Estrategias: agrupar por dominio y exponer solo el subconjunto relevante al contexto, o usar sub-agentes especializados (Módulo 6).

## Conclusión del módulo

Lo que hay que llevarse: una tool es schema + función + manejo de error; el schema se escribe pensando en que lo lee un modelo, no un humano; los errores devueltos como texto accionable son lo que le permite al agente auto-corregirse; y todo lo que escribe o manda algo afuera necesita confirmación humana o idempotencia.

Cómo conecta con el siguiente: cada tool ejecutada agrega una observación al historial, y el historial crece hasta chocar con la ventana de contexto. Administrar eso — qué se guarda, qué se resume, qué se olvida — es el Módulo 5 (Agent Memory).
