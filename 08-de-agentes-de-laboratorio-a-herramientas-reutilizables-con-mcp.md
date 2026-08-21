# Parte 2: Construcción de Agentes y Herramientas

## Capítulo 8: De Agentes de Laboratorio a Herramientas Reutilizables con MCP

En el capítulo anterior, el agente de resolución de problemas invocaba directamente funciones de Python. Ese era el punto de partida adecuado: la aplicación mapeaba el nombre de una herramienta solicitada por el modelo a una función aprobada, la ejecutaba y devolvía el resultado al modelo. Sin embargo, ese patrón presenta una limitación: se encuentra fuertemente acoplado a un único script.

En un entorno operativo real, las herramientas útiles deben transcender una aplicación aislada. Un equipo de red puede requerir que las comprobaciones de estado de dispositivos, los resúmenes BGP, la inspección de interfaces o los comandos `show` seguros estén disponibles simultáneamente para una interfaz web, un asistente interno, un plugin de editor o un servicio de automatización. Reescribir el código de integración para cada cliente genera redundancia y dificulta el mantenimiento.

Aquí es donde el **Protocolo de Contexto de Modelo** (**MCP**, *Model Context Protocol*) aporta una solución estructural. MCP ofrece un estándar desacoplado para exponer herramientas a diversos clientes: la lógica de negocio permanece en Python, pero el cliente ya no necesita conocer los detalles de implementación interna. Puede descubrir herramientas disponibles, invocarlas a través del protocolo estandarizado y recibir datos estructurados.

El objetivo de este capítulo es aplicar MCP de forma práctica y rigurosa. Tomaremos las herramientas de red simuladas (*mock*) de los laboratorios anteriores, las encapsularemos tras un servidor MCP, conectaremos un puente HTTP (*bridge*) y utilizaremos una interfaz de navegador para interactuar con ellas, manteniendo siempre el principio de **solo lectura primero**.

En este capítulo cubriremos los siguientes temas:

- Revisar los requisitos técnicos para el laboratorio de MCP.
- Comprender por qué la invocación directa de herramientas no escala de forma limpia.
- Explicar qué aporta MCP al flujo de trabajo de los agentes.
- Revisar la estructura del Laboratorio 5 de MCP.
- Probar los wrappers seguros de herramientas de red.
- Exponer herramientas de red a través de un servidor MCP.
- Conectar la interfaz web del navegador mediante el puente HTTP.
- Ejecutar el servidor MCP y el puente en el orden operacional correcto.
- Probar herramientas MCP desde la interfaz del navegador.
- Preservar la seguridad mediante herramientas de solo lectura preparadas para producción.

Al finalizar este capítulo, comprenderás cómo empaquetar las herramientas de red utilizadas por un agente local tras una interfaz MCP estandarizada y reutilizable, preparando el terreno para los controles de producción del capítulo final.

---

### Sección 1: Requisitos técnicos

Este capítulo utiliza los archivos del Laboratorio 5 del repositorio. El laboratorio se ejecuta de forma local, simulada y en modo de solo lectura, sin requerir infraestructura física ni credenciales de nube.

Requisitos y archivos necesarios:

- **Python 3.10** o superior.
- Dependencias de `requirements.txt` instaladas.
- La carpeta del Laboratorio 5 en `labs/lab5-mcp/`.
- El servidor MCP en `labs/lab5-mcp/mcp_server.py`.
- El puente HTTP en `labs/lab5-mcp/http_bridge.py`.
- La interfaz de usuario web en `labs/lab5-mcp/ui.html`.
- Los wrappers de herramientas seguros en `labs/lab5-mcp/network_tools.py`.
- La prueba local de cordura en `labs/lab5-mcp/client_test.py`.
- El backend simulado en `examples/mock_network_devices.py`.

Asegúrate de tener las dependencias instaladas desde la raíz del repositorio:

```bash
pip install -r requirements.txt
```

El Laboratorio 5 utiliza `mcp`, `Starlette` y `Uvicorn` para la capa de servidor y puente HTTP.

---

### Sección 2: Más allá de la invocación directa de herramientas

La invocación directa es idónea para aprender y crear prototipos: la aplicación mantiene un mapa de herramientas y ejecuta la función local correspondiente.

La limitación radica en que el contrato de la herramienta queda confinado a esa única aplicación. Si una interfaz gráfica o un bot de chat requieren la misma funcionalidad, se ven forzados a duplicar la integración.

MCP desacopla la interfaz de las herramientas del cliente: el código de ejecución y las validaciones de seguridad permanecen centralizados en el servidor, mientras que los clientes interactúan mediante un protocolo estándar.

La **Tabla 8.1** compara ambos paradigmas:

| Patrón de integración | Cómo accede el cliente a las herramientas | Escenario idóneo |
| :--- | :--- | :--- |
| **Invocación directa (*Direct tool calling*)** | Una aplicación mapea peticiones a funciones locales de Python | Primeros laboratorios, prototipos y agentes monoproceso |
| **Servidor de herramientas MCP (*MCP tool server*)** | Un servidor expone herramientas descubribles por múltiples clientes | Herramientas compartidas entre varios asistentes y servicios |
| **Puente HTTP + Interfaz web** | Un puente traduce llamadas HTTP hacia el servidor MCP | Pruebas visuales desde navegadores web estándar |

> **Tabla 8.1:** Comparativa entre invocación directa y exposición reutilizable basada en MCP.

---

### Sección 3: Qué aporta el protocolo MCP

**Model Context Protocol (MCP)** es un estándar abierto que permite a clientes y servidores intercambiar capacidades de herramientas y resultados estructurados. El servidor MCP define qué herramientas proporciona, sus parámetros requeridos y el formato de los datos devueltos.

MCP establece un **contrato formal**: el proveedor gobierna la implementación y las salvaguardas operativas; el cliente descubre e invoca la herramienta; el resultado fluye como JSON estructurado.

> **Figura 8.1:** Comparativa entre invocación directa de herramientas y exposición desacoplada mediante MCP.

---

### Sección 4: Arquitectura del Laboratorio 5

El Laboratorio 5 desacopla la lógica de negocio, la exposición MCP, el puente HTTP y la interfaz web en módulos independientes:

```text
ui.html -> http_bridge.py -> mcp_server.py -> network_tools.py -> mock_network_devices.py
```

> **Figura 8.2:** Arquitectura y flujo de solicitudes del Laboratorio 5 con MCP.

La **Tabla 8.2** detalla los componentes del laboratorio:

| Archivo | Función en la arquitectura | Enfoque de revisión |
| :--- | :--- | :--- |
| `network_tools.py` | Envuelve las funciones simuladas con validaciones de seguridad | Ubicación de las reglas de validación y límites de solo lectura |
| `mcp_server.py` | Expone los wrappers seguros como herramientas MCP | Definición de la interfaz pública de herramientas |
| `http_bridge.py` | Conecta con el servidor MCP y expone endpoints HTTP | Mecanismo de enlace entre el navegador y el servidor MCP |
| `ui.html` | Proporciona una interfaz web interactiva | Validación visual del flujo de ejecución |
| `client_test.py` | Ejecuta comprobaciones locales directas sobre los wrappers | Depuración de la lógica de negocio antes de iniciar MCP |

> **Tabla 8.2:** Componentes del Laboratorio 5 de MCP y focos de atención.

---

### Sección 5: Wrappers seguros de herramientas de red

El archivo `network_tools.py` concentra las salvaguardas operativas previas a la ejecución, interactuando con el backend simulado y devolviendo diccionarios estructurados.

La **Tabla 8.3** resume los wrappers seguros implementados:

| Función wrapper | Propósito operativo | Comprobación de seguridad aplicada |
| :--- | :--- | :--- |
| `list_devices()` | Lista los equipos en la topología simulada | Sin argumentos; solo lectura |
| `safe_device_status(device)` | Devuelve el estado de un equipo conocido | Rechaza nombres de dispositivos no registrados |
| `safe_interface_status(device, interface=None)` | Consulta una interfaz específica o todas las del equipo | Valida la existencia del equipo antes de consultar |
| `safe_bgp_summary(device)` | Devuelve el estado de vecinos BGP | Valida la existencia del equipo antes de consultar |
| `safe_ping(target, count=4)` | Ejecuta una prueba de alcanzabilidad simulada | Limita `count` estrictamente entre 1 y 10 paquetes |
| `safe_show_command(device, command)` | Ejecuta comandos de inspección permitidos | Bloquea todo comando que no comience por `show` |
| `safe_topology_info()` | Devuelve el mapa de la topología | Datos estáticos de solo lectura |

> **Tabla 8.3:** Wrappers de red seguros utilizados por el servidor MCP.

---

### Sección 6: Pruebas unitarias de la capa de herramientas antes de MCP

Antes de iniciar el servidor MCP, se debe validar la lógica de negocio de forma aislada mediante `client_test.py`:

```bash
python3 labs/lab5-mcp/client_test.py
```

Salida esperada de las pruebas:

```text
====================================================================== Available devices ====================================================================== { "source": "mock_network_devices", "devices": ["spine1", "spine2", "leaf1", "leaf2"] } ====================================================================== BGP summary: leaf2 ====================================================================== { "device": "leaf2", "total_peers": 2, "established_peers": 1, "neighbors": [ {"ip": "10.1.1.2", "state": "Established", "prefixes": 50}, {"ip": "10.1.2.2", "state": "Idle", "prefixes": 0} ] } ====================================================================== Interface status: leaf2 Ethernet3 ====================================================================== { "device": "leaf2", "interface": "Ethernet3", "description": "server_rack_2", "status": "down", "speed": "1G" } ====================================================================== Blocked unsafe command ====================================================================== { "error": "Only read-only show commands are allowed in this lab", "blocked_command": "configure terminal", "example": "show ip bgp summary" }
```

La prueba confirma que los datos simulados se leen correctamente y que los comandos no autorizados (como `configure terminal`) quedan bloqueados antes de su procesamiento.

---

### Sección 7: Exposición de herramientas a través del servidor MCP

El archivo `mcp_server.py` utiliza `FastMCP` y registra las funciones mediante el decorador `@mcp.tool()`:

```python
from mcp.server.fastmcp import FastMCP mcp = FastMCP("network-agent-tools") @mcp.tool() def bgp_summary(device: str)-> dict: """Get BGP neighbor summary for a lab network device.""" return safe_bgp_summary(device)
```

La **Tabla 8.4** relaciona los nombres de herramientas MCP expuestos con sus wrappers internos:

| Herramienta MCP | Wrapper interno invocado | Capacidad expuesta al cliente |
| :--- | :--- | :--- |
| `devices` | `list_devices()` | Listar los equipos disponibles en la topología |
| `device_status` | `safe_device_status()` | Consultar el estado operativo de un equipo |
| `interface_status` | `safe_interface_status()` | Consultar una interfaz concreta o todas las del equipo |
| `bgp_summary` | `safe_bgp_summary()` | Obtener el estado de sesiones BGP |
| `ping` | `safe_ping()` | Ejecutar prueba de alcanzabilidad simulada |
| `show_command` | `safe_show_command()` | Ejecutar comando `show` de solo lectura permitido |
| `topology` | `safe_topology_info()` | Obtener el esquema de la topología spine-leaf |

> **Tabla 8.4:** Herramientas MCP expuestas por el servidor del Laboratorio 5.

---

### Sección 8: Ejecución del servidor MCP

El servidor puede operar en modo estándar (`stdio`) o en modo **Server-Sent Events** (`SSE`). Para la integración con la interfaz web utilizaremos el modo SSE:

```bash
python3 labs/lab5-mcp/mcp_server.py --sse
```

Salida esperada:

```text
Starting MCP server in sse mode... Listening on http://localhost:8000 Start the bridge next: python3 labs/lab5-mcp/http_bridge.py
```

Mantén este proceso en ejecución en su terminal.

---

### Sección 9: Conexión del puente HTTP

El archivo `http_bridge.py` actúa como cliente MCP, se conecta al endpoint SSE del servidor y publica rutas HTTP estándar en el puerto `8765`:

```python
MCP_SERVER_URL = "http://localhost:8000/sse"
```

Inicia el puente en una segunda terminal:

```bash
python3 labs/lab5-mcp/http_bridge.py
```

Salida esperada del puente:

```text
============================================================ Lab 5 — HTTP → MCP Bridge ============================================================ MCP server: http://localhost:8000/sse Bridge HTTP: http://localhost:8765 ============================================================ 🔌 Connecting to MCP server at http://localhost:8000/sse ... Tools registered: ['devices', 'device_status', 'interface_status', 'bgp_summary', 'ping', 'show_command', 'topology'] ✅ MCP session established 🌐 Bridge HTTP on http://localhost:8765 Open labs/lab5-mcp/ui.html in your browser
```

---

### Sección 10: Apertura y pruebas desde la interfaz web

Abre el archivo `ui.html` en tu navegador (en macOS mediante `open labs/lab5-mcp/ui.html`).

La **Tabla 8.5** describe el mapeo entre las acciones de la interfaz, las rutas HTTP del puente y las herramientas MCP:

| Acción en la interfaz | Endpoint invocado en el puente | Herramienta MCP ejecutada |
| :--- | :--- | :--- |
| **Listar dispositivos** | `/devices` | `devices` |
| **Estado del dispositivo** | `/status?device=spine1` | `device_status` |
| **Resumen BGP** | `/bgp?device=leaf2` | `bgp_summary` |
| **Consultar interfaz** | `/interface?device=leaf2&interface=Ethernet3` | `interface_status` |
| **Ejecutar Ping** | `/ping?target=10.99.99.99&count=4` | `ping` |
| **Ejecutar comando** | `/command?device=spine1&command=show%20ip%20bgp%20summary` | `show_command` |
| **Ver topología** | `/topology` | `topology` |

> **Tabla 8.5:** Mapeo entre acciones de interfaz gráfica, endpoints HTTP y herramientas MCP.

#### Pruebas funcionales y de seguridad en la UI
1. **Comando permitido:** Ejecuta `show ip bgp summary` y verifica la respuesta estructurada.
2. **Comando no autorizado:** Ejecuta `configure terminal`. El sistema retornará un error estructurado denegando la ejecución.

---

### Sección 11: Implementación interna del puente HTTP

El puente gestiona la sesión MCP de forma transparente para el navegador:

```python
async def handle_bgp(request: Request)-> JSONResponse: device = request.query_params.get("device", "") return JSONResponse( await call_tool("bgp_summary", {"device": device}) )
```

---

### Sección 12: Secuencia operacional de arranque

Para evitar errores de conexión, inicia los componentes en el siguiente orden estricto:

1. **Validación unitaria de herramientas:**
```bash
python3 labs/lab5-mcp/client_test.py
```
2. **Inicio del servidor MCP (Terminal 1):**
```bash
python3 labs/lab5-mcp/mcp_server.py --sse
```
3. **Inicio del puente HTTP (Terminal 2):**
```bash
python3 labs/lab5-mcp/http_bridge.py
```
4. **Apertura de la interfaz web:**
```bash
open labs/lab5-mcp/ui.html
```

---

### Sección 13: Diagnóstico y resolución de incidencias en MCP

La **Tabla 8.6** resume los fallos habituales y sus soluciones:

| Síntoma observado | Causa probable | Verificación recomendada |
| :--- | :--- | :--- |
| **El puente no conecta al servidor MCP** | Servidor MCP no iniciado en modo SSE | Iniciar `mcp_server.py --sse` antes de arrancar el puente |
| **Error de conexión en el navegador** | Puente HTTP inactivo | Verificar que `http://localhost:8765` responde |
| **Error de dispositivo desconocido** | Nombre no presente en la topología | Utilizar `spine1`, `spine2`, `leaf1` o `leaf2` |
| **Error de comando no autorizado** | El comando no inicia con `show` | Usar exclusivamente comandos de inspección (`show`) |
| **Error de importación en módulos** | Dependencias faltantes | Ejecutar `pip install -r requirements.txt` |
| **La interfaz abre pero no muestra datos** | Servidor o puente caídos | Reiniciar la secuencia desde el paso 2 |

> **Tabla 8.6:** Incidencias comunes en laboratorios MCP y pasos de resolución.

---

### Sección 14: Salvaguardas operativas en herramientas MCP

La **Tabla 8.7** detalla las salvaguardas implementadas en la capa de herramientas:

| Salvaguarda de seguridad | Archivo de implementación | Riesgo operacional mitigado |
| :--- | :--- | :--- |
| **Validación de dispositivos conocidos** | `validate_device()` | Impide acciones sobre objetivos fuera de inventario |
| **Lista blanca de comandos de solo lectura** | `safe_show_command()` | Bloquea comandos de configuración y alteración de estado |
| **Límite en recuento de pings** | `safe_ping()` | Previene sobrecarga por ráfagas excesivas de paquetes |
| **Cargas de error estructuradas** | Todos los wrappers | Permite a los clientes gestionar fallos sin excepciones fatales |
| **Backend simulado** | `mock_network_devices.py` | Asegura reproducibilidad sin riesgo sobre redes vivas |
| **Separación de capas (servidor/puente)** | `http_bridge.py` / `mcp_server.py` | Facilita la depuración modular de transporte y lógica |

> **Tabla 8.7:** Controles de seguridad integrados en la capa de herramientas MCP.

---

### Sección 15: Criterios para adoptar MCP

- **Invocación directa:** Adecuada para prototipos rápidos, scripts monoproceso o laboratorios iniciales.
- **Servidor MCP:** Recomendado cuando múltiples clientes (asistentes, GUIs, orquestadores) deben consumir el mismo catálogo de herramientas de red de forma desacoplada y estandarizada.

---

### Sección 16: Consideraciones para entornos de producción

La transición de herramientas MCP a producción exige controles de nivel empresarial. La **Tabla 8.8** sintetiza estos requerimientos:

| Requisito de producción | Justificación técnica | Dirección de diseño para producción |
| :--- | :--- | :--- |
| **Autenticación** | Evitar invocaciones anónimas | Identificación obligatoria de clientes y usuarios |
| **Autorización (RBAC)** | Restringir el alcance por perfil | Políticas granulares de acceso a herramientas y equipos |
| **Registro y auditoría** | Trazabilidad completa | Registro de usuario, herramienta, parámetros, estado y duración |
| **Gestión de secretos** | Eliminar credenciales en texto plano | Uso de variables de entorno seguras o gestores de secretos (*vaults*) |
| **Observabilidad** | Detección temprana de anomalías | Monitorización de tasas de error, latencias y volumetría |
| **Aprobación de cambios** | Control estricto de escrituras | Flujos con intervención humana (*Human-in-the-Loop*) y rollback |
| **Validación estricta** | Integridad de entradas y salidas | Validación tipada mediante esquemas estrictos (ej. Pydantic) |

> **Tabla 8.8:** Consideraciones técnicas de producción para despliegues MCP.

---

### Sección 17: Resumen

En este capítulo desacoplamos las herramientas de red del agente local, exponiéndolas como servicios reutilizables mediante Model Context Protocol (MCP).

Validamos los wrappers seguros, implementamos el servidor `FastMCP`, articulamos el puente HTTP y ejercitamos las consultas desde una interfaz de navegador web, garantizando en todo momento la restricción a operaciones de solo lectura.

En el próximo capítulo analizaremos las directrices integrales de seguridad, gobernanza, observabilidad y límites operativos necesarias antes de evaluar agentes de IA en infraestructuras de red de producción.
