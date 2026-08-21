# Parte 2: Construcción de Agentes y Herramientas

## Capítulo 6: Diseño de Herramientas y Flujos de Trabajo Agénticos

Un chatbot con memoria puede recordar lo que preguntaste previamente. Eso es valioso, pero sigue siendo incapaz de comprobar un dispositivo, inspeccionar un vecino BGP o consultar el estado actual de la red por sí mismo. En algún momento, un asistente de red necesita un mecanismo seguro para solicitar información externa al modelo.

Aquí es donde entra en juego la **invocación de herramientas** (*tool calling*). El modelo no recibe acceso directo a tus routers: no abre una terminal SSH ni escribe comandos libremente. En su lugar, tu aplicación expone un conjunto acotado de funciones aprobadas. El modelo puede solicitar la ejecución de una de esas funciones, tu código decide si la solicitud está permitida, se ejecuta la acción en un entorno controlado y el resultado se reincorpora a la conversación.

Esa diferencia es fundamental. Un agente no es útil porque pueda hacerlo todo; es útil cuando puede realizar unas pocas acciones acotadas de forma segura, observar los resultados y continuar la investigación respaldado por evidencias. Esa es la frontera práctica entre un chatbot que habla sobre redes y un agente capaz de asistir en las operaciones de red (**NetOps**).

En este capítulo construiremos este modelo mental utilizando el flujo local con Ollama y las herramientas de red simuladas (*mock tools*) del repositorio. Nos centraremos en acciones de solo lectura, solicitudes estructuradas de herramientas, ejecución segura en Python y límites de iteración en bucles. Al finalizar, comprenderás cómo opera el patrón agéntico antes de aplicarlo a diagnósticos más complejos en el siguiente capítulo.

En este capítulo cubriremos los siguientes temas:

- Revisar los requisitos técnicos para el laboratorio de flujos de trabajo agénticos.
- Comprender por qué los agentes necesitan herramientas y no solo memoria.
- Revisar las herramientas de red simuladas utilizadas por el bot.
- Describir herramientas al modelo con límites operativos claros.
- Parsear solicitudes de herramientas emitidas por el modelo de forma segura.
- Ejecutar funciones de Python aprobadas mediante un mapa de herramientas (*tool map*).
- Devolver los resultados de las herramientas al modelo para un nuevo paso de razonamiento.
- Limitar el bucle del agente para prevenir comportamientos descontrolados.
- Ejecutar el bot de red agéntico local.
- Aplicar patrones de seguridad para producción en agentes basados en herramientas.

Estos temas nos permiten evolucionar desde un chatbot con memoria hacia un agente controlado basado en herramientas. El objetivo sigue siendo eminentemente práctico: construir el patrón seguro más compacto que permita al asistente observar el estado de la red sin conceder al modelo acceso irrestricto.

---

### Sección 1: Requisitos técnicos

Este capítulo utiliza el flujo agéntico del Laboratorio 4 del repositorio. La ruta principal se ejecuta en local empleando datos simulados, por lo que no requieres dispositivos reales, credenciales ni una topología física para seguir los ejercicios.

Necesitas las siguientes herramientas y archivos:

- **Python 3.10** o superior.
- **Ollama** instalado y en ejecución.
- El modelo `llama3.2:3b` descargado localmente.
- Las dependencias del repositorio instaladas desde `requirements.txt`.
- El script principal en `labs/lab4-agentic/agentic_network_bot_ollama.py`.
- El backend simulado en `examples/mock_network_devices.py`.
- Las notas del Laboratorio 4 en `labs/lab4-agentic/README.md`.

Desde la raíz del repositorio, ejecuta el script principal del Laboratorio 4 con el siguiente comando:

```bash
python3 labs/lab4-agentic/agentic_network_bot_ollama.py
```

El repositorio incluye variantes para redes en vivo, pero las trataremos como rutas avanzadas opcionales. El flujo formativo central se mantiene con herramientas de solo lectura simuladas para asegurar un comportamiento reproducible para todos los lectores.

---

### Sección 2: Comprender por qué los agentes necesitan herramientas

El chatbot que construimos en el capítulo anterior disponía de memoria: podía mantener el contexto del diagnóstico a lo largo de varios turnos, lo cual supuso un avance importante. Sin embargo, la memoria no le proporciona el estado en tiempo real de la red. Si el usuario pregunta si una sesión BGP está levantada en este instante, el modelo no puede saberlo únicamente a partir de sus datos de entrenamiento o del historial conversacional.

El asistente necesita una vía controlada para solicitar información a la aplicación: esa es la función de una **herramienta** (*tool*). En este libro, una herramienta es una función de Python que realiza una tarea específica y acotada, como devolver el estado de un dispositivo, comprobar interfaces, obtener la tabla BGP o verificar alcanzabilidad en la topología simulada.

Esto difiere sustancialmente de permitir que el modelo ejecute comandos arbitrarios. La aplicación define qué funciones existen, qué argumentos aceptan y qué estructura devuelven. El modelo puede solicitar una herramienta, pero tu código mantiene la potestad exclusiva de la ejecución. Esa separación constituye la **barrera de seguridad**.

Un ejemplo práctico ilustra el concepto: si un usuario pregunta si todas las sesiones BGP están saludables, un chatbot genérico explicará cómo funciona BGP; un agente dotado de herramientas consultará los datos del dispositivo, detectará que un vecino está en estado inactivo (*Idle*) y reportará el hallazgo respaldado por datos objetivos.

El bucle agéntico sigue un ciclo continuo: el agente observa la solicitud, razona sobre la información requerida, solicita una herramienta, recibe el resultado y evalúa si dispone de suficientes evidencias para responder o si necesita invocar otra herramienta adicional.

> **Figura 6.1:** Bucle de invocación de herramientas en un flujo de trabajo de red agéntico.

El bucle no es magia: es código de aplicación que envuelve las respuestas del modelo. El modelo propone el siguiente paso; la aplicación lo valida y lo ejecuta.

---

### Sección 3: Pasar de respuestas de chatbot a razonamiento asistido por herramientas

El cambio cualitativo respecto al [Capítulo 5](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/5) reside en que el asistente ya no responde exclusivamente a partir del prompt: puede invocar herramientas cuando necesita datos actualizados o estructurados.

En el script del Laboratorio 4, la clase `AgenticNetworkBot` gestiona el modelo, el historial conversacional, las herramientas disponibles y el bucle agéntico. Invoca a Ollama mediante su **API** local, pero ahora el prompt incorpora el catálogo de herramientas disponibles.

Esta arquitectura garantiza una separación modular:
1. El modelo determina qué información necesita para responder al usuario.
2. La aplicación verifica si la herramienta solicitada existe y está autorizada.
3. La función de Python ejecuta la herramienta aprobada.
4. El resultado devuelto se añade al historial de conversación.
5. El modelo razona sobre el nuevo dato y decide si solicita otra herramienta o emite su conclusión final.

Esta modularidad simplifica la depuración: ante cualquier anomalía, es posible auditar de forma independiente la solicitud del modelo, los argumentos suministrados, la respuesta de la herramienta y la conclusión final.

---

### Sección 4: Revisión de las herramientas de red simuladas (mock)

El archivo `examples/mock_network_devices.py` proporciona una topología simulada de centro de datos compuesta por dos switches troncales (*spines*) y dos switches de acceso (*leafs*), junto con sus estados de interfaces y tablas BGP:

```text
spine1 (192.168.0.11) ─┬─ leaf1 (192.168.0.21) └─ leaf2 (192.168.0.22) spine2 (192.168.0.12) ─┘
```

Esta estructura permite ejercitar el patrón sin necesidad de Containerlab, imágenes cEOS de Arista, credenciales SSH ni infraestructura física.

Las herramientas principales expuestas al bot se detallan en la **Tabla 6.1**:

| Herramienta | Qué devuelve | Por qué es segura en el laboratorio |
| :--- | :--- | :--- |
| `get_device_status()` | Inventario y estado general del dispositivo | Datos simulados de solo lectura |
| `get_interface_status()` | Estado detallado de interfaces, IP y MAC | Datos simulados de solo lectura |
| `get_bgp_summary()` | Vecinos BGP y sus estados operativos | Datos simulados de solo lectura |
| `ping_device()` | Resultado de prueba de alcanzabilidad | Prueba simulada |
| `execute_command()` | Salida de comandos de visualización permitidos | Restringido exclusivamente a comandos `show` |
| `get_topology_info()` | Metadatos globales de la topología | Datos estáticos |

> **Tabla 6.1:** Herramientas de red simuladas utilizadas por el bot agéntico.

Incluso la herramienta genérica `execute_command()` restringe su ejecución a comandos de inspección (`show`), consolidando el principio de **solo lectura primero**.

---

### Sección 5: Revisión del script principal del Lab 4

El archivo `labs/lab4-agentic/agentic_network_bot_ollama.py` importa las herramientas simuladas, las registra en un mapa de ejecución, las documenta ante el modelo y gestiona el bucle de interacción:

```python
from mock_network_devices import ( get_device_status, get_interface_status, get_bgp_summary, ping_device, execute_command, get_topology_info )
```

El modelo nunca invoca estas funciones directamente: es la aplicación Python la que gestiona su llamada a través de un mapeo autorizado.

---

### Sección 6: Mapeo de nombres de herramientas a funciones de Python aprobadas

La clase del agente define un diccionario que asocia los nombres textuales de las herramientas con sus funciones ejecutables en Python. Este es uno de los controles de seguridad esenciales: el modelo solo puede solicitar nombres explícitamente registrados.

```python
self.tools_map = { "get_device_status": get_device_status, "get_interface_status": get_interface_status, "get_bgp_summary": get_bgp_summary, "ping_device": ping_device, "execute_command": execute_command, "get_topology_info": get_topology_info }
```

> **Figura 6.2:** Mapeo entre las herramientas solicitadas por el modelo y las funciones de Python aprobadas.

Si el modelo solicita una herramienta no registrada en `tools_map`, la aplicación bloquea la petición y no ejecuta ningún código.

---

### Sección 7: Descripción de herramientas para el modelo

Para que el modelo sepa cuándo y cómo utilizar cada herramienta, el prompt debe describir su catálogo en lenguaje claro, indicando propósito y ejemplos de uso:

```text
Available Tools: 1. get_device_status(device) - Get device info (hostname, version, uptime, role) - Example: get_device_status("spine1") 2. get_interface_status(device, interface) - Get interface state, IP, MAC - Example: get_interface_status("leaf1", "Ethernet1") 3. get_bgp_summary(device) - Get BGP neighbor status - Example: get_bgp_summary("spine1")
```

Una descripción adecuada debe responder a tres preguntas:
1. ¿Qué función realiza esta herramienta?
2. ¿En qué escenario debe utilizarse?
3. ¿Qué parámetros y tipos de datos espera recibir?

---

### Sección 8: Construcción del prompt de sistema para el uso de herramientas

Dado que esta implementación local no depende de esquemas propietarios de *function calling*, el prompt de sistema instruye al modelo para emitir las solicitudes bajo un formato textual estructurado y predecible:

```text
TOOL: tool_name ARGS: {"arg1": "value1", "arg2": "value2"}
```

El prompt de sistema se estructura de la siguiente manera:

```text
You are an expert network engineer troubleshooting a data center network. When you need information, output a tool call in this EXACT format: TOOL: tool_name ARGS: {"arg1": "value1", "arg2": "value2"} After getting tool results, analyze them and either: 1. Call another tool if you need more info 2. Provide your final answer Be concise and practical. Focus on solving problems.
```

El requerimiento de formato exacto es determinante: permite que el parser de la aplicación identifique inequívocamente si la respuesta es una llamada a herramienta o una conclusión final.

---

### Sección 9: Parseo de llamadas a herramientas solicitadas por el modelo

El método `_parse_tool_call()` examina la respuesta del modelo línea por línea, localiza los prefijos `TOOL:` y `ARGS:`, y deserializa los argumentos desde JSON:

```python
def _parse_tool_call(self, response: str): lines = response.split("\n") tool_name = None tool_args = {} for line in lines: line = line.strip() if line.startswith("TOOL:"): tool_name = line.replace("TOOL:", "").strip() elif line.startswith("ARGS:"): args_str = line.replace("ARGS:", "").strip() tool_args = json.loads(args_str) if tool_name and tool_name in self.tools_map: return {"name": tool_name, "args": tool_args} return None
```

La salida del modelo es simplemente texto hasta que la aplicación la valida, la reconoce y la convierte en una invocación controlada.

---

### Sección 10: Ejecución segura de herramientas

Tras validar la solicitud, la aplicación recupera la función del mapa `tools_map` y le pasa los argumentos:

```python
def _execute_tool(self, tool_name: str, args: dict)-> dict: tool_func = self.tools_map[tool_name] try: result = tool_func(**args) return result if isinstance(result, dict) else {"result": str(result)} except TypeError as e: return {"error": f"Invalid arguments for {tool_name}: {str(e)}"}
```

El control de excepciones evita caídas de la aplicación si el modelo suministra parámetros incompatibles, devolviendo un error estructurado para que el modelo pueda corregir su petición.

---

### Sección 11: Retorno de resultados de herramientas al modelo

Una vez ejecutada la herramienta, el resultado se serializa en JSON y se incorpora al historial conversacional:

```python
result = self._execute_tool(tool_name, tool_args) result_str = json.dumps(result, indent=2) self.conversation_history.append({ "role": "assistant", "content": f"TOOL: {tool_name}\nARGS: {json.dumps(tool_args)}" }) self.conversation_history.append({ "role": "user", "content": f"Tool result:\n{result_str}" })
```

Aquí convergen la memoria del [Capítulo 5](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/5) y las herramientas: la llamada siguiente del modelo incluye la solicitud original, la herramienta ejecutada y el dato devuelto, proporcionando la base factual para razonar el siguiente paso.

---

### Sección 12: Limitación del bucle del agente

Todo bucle agéntico requiere un límite superior de iteraciones para impedir bucles infinitos ante prompts ambiguos o respuestas circulares del modelo:

```python
def chat(self, user_message: str, max_iterations: int = 5)-> str:
```

Cada llamada a herramienta incrementa el contador. Si el modelo no solicita más herramientas, se retorna la respuesta final; si alcanza `max_iterations`, la aplicación fuerza el cierre del bucle y solicita una conclusión con los datos acumulados hasta ese momento.

---

### Sección 13: Manejo de modos de fallo en llamadas a herramientas

La **Tabla 6.2** resume los fallos habituales en la invocación de herramientas y las salvaguardas que deben implementarse:

| Modo de fallo | Ejemplo observado | Salvaguarda operativa |
| :--- | :--- | :--- |
| **Herramienta desconocida** | El modelo solicita `get_route_status()`, que no existe | Rechazar la petición y reportar error de herramienta no encontrada |
| **Argumentos inválidos** | El modelo pasa `device=core1` cuando solo existen `spine1`, `leaf1`, etc. | Validar nombres de equipos antes de la ejecución |
| **JSON de argumentos malformado** | Sintaxis JSON rota en la línea `ARGS:` | Devolver error de parseo y solicitar reintento con formato correcto |
| **Comando no autorizado** | El modelo intenta ejecutar `configure terminal` en `execute_command()` | Permitir únicamente comandos de inspección en lista blanca (`show`) |
| **Bucle descontrolado** | El modelo invoca herramientas indefinidamente sin emitir respuesta | Aplicar `max_iterations` y finalizar de forma segura |
| **Respuesta incompleta de herramienta** | La herramienta devuelve error o datos parciales | Reenviar el error al modelo para evitar conclusiones infundadas |

> **Tabla 6.2:** Modos de fallo en invocación de herramientas y salvaguardas aplicables.

---

### Sección 14: Ejecución del bot de red agéntico

Ejecuta el bot del Laboratorio 4 desde la raíz del repositorio:

```bash
python3 labs/lab4-agentic/agentic_network_bot_ollama.py
```

Salida esperada en la consulta inicial de estado:

```text
🤖 Agentic Network Bot - Ollama Edition ====================================================================== No API keys required! Using Ollama (llama3.2:3b) ====================================================================== 🎯 Running test scenarios... ====================================================================== 👤 User: What's the status of spine1? ====================================================================== 🔧 Agent is calling: get_device_status({"device": "spine1"}) 📊 Result: { "device": "spine1", "ip": "192.168.0.11", "status": "up", "model": "cEOS", "version": "4.28.0F", "uptime": "5d", "role": "spine" } 🤖 Agent: spine1 is up and operating as a spine switch. It is running cEOS 4.28.0F and has been up for 5 days.
```

El bot no dedujo el estado de `spine1`: solicitó `get_device_status()`, la aplicación ejecutó la función y el modelo elaboró su respuesta basándose en los datos devueltos.

#### Consulta de múltiples pasos
En el modo interactivo, prueba formulando una consulta de ámbito global:

```text
Are all BGP sessions up?
```

El agente consultará `get_bgp_summary()` en los distintos dispositivos, identificará que `leaf2` cuenta con un vecino en estado `Idle` y reportará el diagnóstico basado en los datos obtenidos.

---

### Sección 15: Mantener opcional el acceso a la red en vivo

El repositorio incluye implementaciones para redes en vivo mediante Netmiko (`lab4b_agentic_network_bot_netmiko.py` y `live_network_devices.py`).

El laboratorio principal prescinde de ellas deliberadamente para evitar que incidencias de credenciales, latencia o conectividad interfieran en la asimilación del diseño agéntico y sus mecanismos de control.

---

### Sección 16: Aplicación de límites de producción a herramientas que usan agentes

La transición desde el entorno de laboratorio hacia producción exige reforzar los controles en cada capa. La **Tabla 6.3** sintetiza estas diferencias:

| Área | Comportamiento en laboratorio | Requisito en entorno de producción |
| :--- | :--- | :--- |
| **Acceso a herramientas** | Funciones Python sobre datos simulados | APIs de solo lectura autenticadas o comandos SSH controlados |
| **Validación de argumentos** | Parseo básico y captura de `TypeError` | Validación estricta de equipos, comandos, tipos y permisos RBAC |
| **Límites de comandos** | `execute_command()` filtra prefijos `show` | Listas blancas estrictas de comandos y control de acceso granular |
| **Gestión de secretos** | Sin credenciales | Variables de entorno aisladas o gestores de secretos (*vaults*) |
| **Auditoría y logging** | Salida por consola en la terminal | Logs estructurados con usuario, herramienta, argumentos, estado y duración |
| **Límites de bucle** | `max_iterations` fija el tope de iteraciones | Límites por usuario, control de tasa (*rate limits*) y políticas de timeout |
| **Acciones de modificación** | Bloqueadas por diseño | Aprobación humana explícita, registro de auditoría y planes de rollback |

> **Tabla 6.3:** Comparativa entre herramientas de laboratorio y límites requeridos en producción.

---

### Sección 17: Registro (logging) de las acciones del agente

Un agente que invoca herramientas debe generar una traza auditable. En producción, cada invocación debe registrarse en formato estructurado (usuario solicitante, marca de tiempo, herramienta, argumentos, resultado y latencia), permitiendo auditar el razonamiento del agente y validar sus conclusiones.

---

### Sección 18: Evitar errores comunes de diseño de agentes

1. **Exceso de herramientas en etapas tempranas:** Demasiadas opciones aumentan el espacio de decisión del modelo. Comienza con pocas herramientas alineadas con preguntas operativas concretas.
2. **Ocultar la ejecución de herramientas:** El operador debe poder inspeccionar qué herramienta se ejecutó y qué datos devolvió.
3. **Reemplazar automatizaciones deterministas existentes:** Si dispones de un script fiable, exponlo como herramienta; no intentes que el LLM lo reinvente.
4. **Omitir la validación:** Toda entrada, salida y conclusión intermedia debe ser verificada antes de continuar el flujo.

---

### Sección 19: Preparación para el agente principal de resolución de problemas

Disponemos ya de todos los componentes esenciales: descripción de herramientas, parseo de peticiones, ejecución controlada en Python, retroalimentación de resultados al modelo y control de bucles.

En el [Capítulo 7](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/7) utilizaremos estos cimientos para abordar un escenario completo de resolución de problemas de red (*troubleshooting*), analizando el proceso de investigación de extremo a extremo.

---

### Sección 20: Resumen

En este capítulo transformamos un chatbot con memoria en un agente autónomo controlado capaz de interactuar con herramientas de red.

Definimos la topología simulada, el catálogo de herramientas de inspección, el mapa de ejecución en Python, el protocolo estructurado de invocación (`TOOL` / `ARGS`) y el bucle agéntico con límites de iteración.

El principio rector se consolida: **el modelo propone; la aplicación gobierna y ejecuta**.

En el siguiente capítulo aplicaremos esta arquitectura a escenarios reales de diagnóstico, evaluando la capacidad del agente para recopilar evidencias fiables y asistir eficazmente a los ingenieros de operaciones.
