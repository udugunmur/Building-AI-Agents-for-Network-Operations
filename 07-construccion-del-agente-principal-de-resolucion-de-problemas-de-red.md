# Parte 2: Construcción de Agentes y Herramientas

## Capítulo 7: Construcción del Agente Principal de Resolución de Problemas de Red

En el capítulo anterior construimos los mecanismos de un agente basado en herramientas: el modelo podía solicitar una función, la aplicación validaba la petición, la función aprobada en Python se ejecutaba y el resultado se reincorporaba al contexto del modelo. Ese patrón es fundamental, pero sigue representando únicamente la maquinaria.

Este capítulo transforma esa maquinaria en un **flujo de trabajo de resolución de problemas** (*troubleshooting*). En lugar de limitarnos a comprobar si el agente puede invocar una herramienta, analizaremos si es capaz de seguir el rastro de las evidencias ante un fallo de red concreto. El escenario es eminentemente práctico: una incidencia de BGP vinculada a síntomas de rutas no alcanzables o pérdida de conectividad con un host de destino.

El objetivo no es que el agente resuelva la red de forma mágica. El verdadero valor reside en su capacidad para recopilar el estado actual mediante herramientas simuladas, contrastar las evidencias y ofrecer un diagnóstico inicial estructurado al operador humano. El ingeniero mantiene la potestad sobre las decisiones, la aplicación gobierna la ejecución y el modelo asiste razonando sobre los datos.

Mantendremos este capítulo en un entorno local, simulado (*mocked*) y de solo lectura. Esto garantiza que la investigación sea reproducible para cualquier lector y enfoca el aprendizaje en la creación de un patrón de diagnóstico guiado por evidencias antes de abordar el empaquetado reutilizable con MCP en el siguiente capítulo.

En este capítulo cubriremos los siguientes temas:

- Revisar los requisitos técnicos para el agente de resolución de problemas.
- Comprender cómo evolucionar del [Capítulo 6](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/6) hacia un flujo de diagnóstico integral.
- Revisar la topología simulada y las fuentes de evidencia.
- Ejecutar el script principal del agente.
- Investigar el estado general de dispositivos como ejercicio inicial.
- Investigar la salud operativa de vecinos BGP.
- Emplear pruebas de alcanzabilidad ante síntomas de pérdida de rutas (*missing route to host*).
- Corregir conclusiones erróneas del modelo confrontándolas con los datos de las herramientas.
- Diseñar respuestas de diagnóstico con estructura profesional y fundamentada.
- Preservar la seguridad mediante flujos simulados y de solo lectura.
- Preparar al agente para la integración de herramientas reutilizables en el próximo capítulo.

Al finalizar este capítulo, dispondrás de una visión clara sobre el rol de un agente de diagnóstico práctico: recopilar datos objetivos, exponer sus evidencias, evitar conclusiones infundadas y proporcionar al ingeniero el siguiente paso operativo más conveniente.

---

### Sección 1: Requisitos técnicos

Este capítulo reutiliza la infraestructura local de los laboratorios anteriores. Debes disponer de Ollama en ejecución, las dependencias del proyecto instaladas y el modelo `deepseek-r1:8b` disponible localmente para el agente del Laboratorio 4.

El flujo principal reutiliza el bot agéntico del Laboratorio 4 para concentrarse en la metodología de investigación sin incorporar dependencias adicionales en esta etapa.

Requisitos y archivos necesarios:

- **Python 3.10** o superior.
- **Ollama** instalado y en ejecución.
- El modelo `deepseek-r1:8b` disponible localmente.
- Dependencias de `requirements.txt` instaladas.
- El script principal en `labs/lab4-agentic/agentic_network_bot_ollama.py`.
- El backend simulado en `examples/mock_network_devices.py`.
- Las notas del Laboratorio 4 en `labs/lab4-agentic/README.md`.

Antes de ejecutar el agente, descarga el modelo con el siguiente comando:

```bash
ollama pull deepseek-r1:8b
```

A continuación, ejecuta el script principal desde la raíz del repositorio:

```bash
python3 labs/lab4-agentic/agentic_network_bot_ollama.py
```

El capítulo utiliza datos simulados, prescindiendo de credenciales, accesos reales a equipos o rutas de modificación de configuraciones para garantizar un entorno seguro y reproducible.

---

### Sección 2: De la mecánica de herramientas al diagnóstico de redes

El [Capítulo 6](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/6) detalló el bucle de invocación de herramientas: el usuario envía una solicitud, el modelo evalúa qué datos requiere, solicita una herramienta, la aplicación la valida y se ejecuta una función en Python.

La resolución de incidencias exige un rigor metodológico superior: el agente debe determinar qué evidencias son prioritarias, invocar las herramientas en un orden lógico, contrastar las salidas y abstenerse de formular afirmaciones no respaldadas por los datos.

Un agente de diagnóstico eficaz debe operar con la prudencia de un ingeniero júnior meticuloso: solicitar los datos pertinentes, examinar las salidas, explicitar la incertidumbre y justificar el razonamiento de sus conclusiones.

> **Figura 7.1:** Bucle de diagnóstico guiado por evidencias para un agente de red.

El agente recibe un síntoma, recopila datos mediante herramientas autorizadas, verifica si los hechos respaldan una hipótesis y solicita más información o emite su informe fundamentado.

---

### Sección 3: Revisión de los escenarios de diagnóstico

El escenario central combina dos síntomas frecuentes en redes: una degradación en sesiones BGP y la falta de alcanzabilidad hacia un host o prefijo. En un entorno real, ambos eventos pueden estar o no correlacionados, por lo que el agente no debe asumir causalidad de manera apresurada.

Ante el reporte de un host inaccesible, una tabla de rutas incompleta o una alarma de sesión BGP caída, el agente debe recabar información para determinar lo confirmado, lo sospechoso y las comprobaciones subsiguientes requeridas.

La investigación se apoya en las siguientes fuentes de evidencia:
- Estado del dispositivo mediante `get_device_status()`.
- Estado de interfaces mediante `get_interface_status()`.
- Estado de vecinos BGP mediante `get_bgp_summary()`.
- Pruebas de alcanzabilidad mediante `ping_device()`.
- Contexto de la topología mediante `get_topology_info()`.

La **Tabla 7.1** relaciona las preguntas operativas habituales con las herramientas correspondientes y los datos esperados:

| Pregunta de diagnóstico | Herramienta de soporte | Qué debe verificar el agente |
| :--- | :--- | :--- |
| **¿El equipo responde en el inventario simulado?** | `get_device_status()` | Estado general, rol, modelo e IP de gestión |
| **¿La interfaz correspondiente está levantada (*up*)?** | `get_interface_status()` | `status`, `description` y `speed` |
| **¿Las sesiones BGP están establecidas?** | `get_bgp_summary()` | `total_peers`, `established_peers`, estados de vecinos y prefijos |
| **¿El host de destino es alcanzable?** | `ping_device()` | `status`, `packet_loss` y `error` |
| **¿La topología explica la ruta esperada?** | `get_topology_info()` | Roles de dispositivos, enlaces y adyacencias conocidas |

> **Tabla 7.1:** Preguntas habituales de resolución de problemas y herramientas de soporte.

---

### Sección 4: Revisión de la topología simulada

El laboratorio utiliza una topología simulada de centro de datos compuesta por dos switches troncales (*spines*) y dos switches de acceso (*leafs*):

```text
spine1 (192.168.0.11) ─┬─ leaf1 (192.168.0.21) └─ leaf2 (192.168.0.22) spine2 (192.168.0.12) ─┘
```

El caso degradado de interés reside en `leaf2`: su estado general figura como `up`, pero su resumen BGP contiene un vecino en estado `Idle` y su interfaz local `Ethernet3` se encuentra en estado `down`, configurando un escenario realista para la investigación.

---

### Sección 5: Ejecución del agente de resolución de problemas

Inicia el script del agente desde la raíz del repositorio:

```bash
python3 labs/lab4-agentic/agentic_network_bot_ollama.py
```

Salida inicial esperada:

```text
🤖 Agentic Network Bot - Ollama Edition ====================================================================== No API keys required! Using Ollama (deepseek-r1:8b) ====================================================================== 🎯 Running test scenarios...
```

---

### Sección 6: Escenario 1: Comprobación del estado del dispositivo

Como prueba inicial, el usuario consulta el estado de `spine1`:

```text
What's the status of spine1?
```

El agente invoca la herramienta y recibe los datos del inventario:

```text
🔧 Agent is calling: get_device_status({"device": "spine1"}) 📊 Result: { "device": "spine1", "ip": "192.168.0.11", "status": "up", "model": "cEOS", "version": "4.28.0F", "serial": "SPX2134567890", "uptime": "5d", "role": "spine" }
```

La respuesta confirma que `spine1` está operativo, indicando su rol, versión de software y tiempo de actividad a partir de los datos devueltos por `get_device_status()`.

---

### Sección 7: Escenario 2: Comprobación de la salud de BGP

En este escenario, el usuario pregunta por el estado global del enrutamiento:

```text
Are all BGP sessions up?
```

El agente debe consultar `get_bgp_summary()` e inspeccionar los campos detallados en la **Tabla 7.2**:

| Campo BGP | Significado operativo | Interpretación en el diagnóstico |
| :--- | :--- | :--- |
| `total_peers` | Número de vecinos configurados en los datos | Contrastar frente a `established_peers` |
| `established_peers` | Número de vecinos en estado establecido | Si es inferior a `total_peers`, existe una degradación |
| `state` | Estado operativo de cada vecino individual | Todo valor distinto de `Established` debe reportarse |
| `prefixes` | Cantidad de prefijos recibidos del vecino | Un valor de `0` puede indicar ausencia de rutas anunciadas |
| `uptime` | Tiempo transcurrido en el estado actual | Permite identificar caídas recientes (*flaps*) o fallos persistentes |

> **Tabla 7.2:** Campos de BGP que el agente debe auditar antes de concluir.

Regla imperativa: la respuesta final nunca debe contradecir estos campos. Si un vecino está en `Idle`, debe reportarse; si `established_peers` es menor que `total_peers`, no debe afirmarse que la red está sana.

---

### Sección 8: Escenario 3: Diagnóstico en leaf2

La evaluación de `leaf2` expone la presencia simultánea de un vecino BGP caído y una interfaz inactiva:

```text
Check if leaf2 has any issues
```

El agente recopila evidencias ejecutando las herramientas correspondientes:

```text
🔧 Agent is calling: get_device_status({"device": "leaf2"}) 🔧 Agent is calling: get_bgp_summary({"device": "leaf2"}) 🔧 Agent is calling: get_interface_status({"device": "leaf2"})
```

Salida del resumen BGP:

```text
📊 Result: { "device": "leaf2", "local_as": 65012, "router_id": "10.0.1.22", "total_peers": 2, "established_peers": 1, "neighbors": [ { "ip": "10.1.1.2", "remote_as": 65001, "state": "Established", "uptime": "2d3h", "prefixes": 50 }, { "ip": "10.1.2.2", "remote_as": 65001, "state": "Idle", "uptime": "0h", "prefixes": 0 } ] }
```

Salida del estado de interfaces:

```text
📊 Result: { "device": "leaf2", "interfaces": [ { "name": "Ethernet1", "description": "to_spine1", "status": "up", "speed": "10G" }, { "name": "Ethernet2", "description": "to_spine2", "status": "up", "speed": "10G" }, { "name": "Ethernet3", "description": "server_rack_2", "status": "down", "speed": "1G" }, { "name": "Management1", "description": "oob_management", "status": "up", "speed": "1G" } ] }
```

Una respuesta rigurosa debe distinguir los elementos operativos de los degradados:

```text
leaf2 is up, but there are two issues to investigate. BGP has 1/2 peers Established. Neighbor 10.1.2.2 is in Idle state and is receiving 0 prefixes. Ethernet3 (server_rack_2) is down. This may explain missing routes that depend on that peer, while Ethernet3 may affect the server-facing segment. Next checks: verify the peer path, interface toward the peer, recent logs, BGP configuration, and the local server-facing interface state.
```

---

### Sección 9: Escenario 4: Síntoma de ruta a host no encontrada (missing route)

Cuando un host resulta inalcanzable, la causa raíz puede deberse a enrutamiento, enlaces caídos, filtrado o fallos en el propio extremo. El agente no debe atribuir el problema a BGP sin evidencias directas.

Consulta de ejemplo:

```text
Host 10.99.99.99 is unreachable. Check whether this could be related to BGP on leaf2.
```

> **Figura 7.2:** Ruta de investigación para diagnóstico de BGP y falta de alcanzabilidad a host.

La prueba de alcanzabilidad reporta la pérdida total de paquetes:

```text
🔧 Agent is calling: ping_device({"target": "10.99.99.99", "count": 4}) 📊 Result: { "target": "10.99.99.99", "packets_sent": 4, "packets_received": 0, "packet_loss": 100, "status": "unreachable", "error": "Destination host unreachable" }
```

Este resultado confirma el síntoma pero no la causa raíz. Al cruzar este dato con la sesión BGP en estado `Idle` en `leaf2` y la interfaz `Ethernet3` caída, el agente puede plantear hipótesis fundadas, manteniendo la prudencia terminológica (*"podría estar relacionado"* en lugar de *"está definitivamente causado por"*).

---

### Sección 10: Estructuración de respuestas basadas en evidencias

Una respuesta técnica rigurosa debe organizarse en cuatro componentes:
1. **Hechos confirmados:** Estados validados por las herramientas.
2. **Evidencias degradadas:** Anomalías o errores detectados.
3. **Causa probable:** Hipótesis formulada con cautela.
4. **Comprobaciones siguientes:** Acciones sugeridas para el operador.

La **Tabla 7.3** ilustra la función de cada elemento:

| Elemento de la respuesta | Ejemplo operativo | Propósito técnico |
| :--- | :--- | :--- |
| **Hechos confirmados** | `leaf2` está operativo; `Ethernet3` está caída | Delimita los estados conocidos y verificados |
| **Evidencias degradadas** | Vecino `10.1.2.2` en `Idle` con 0 prefijos; `Ethernet3` en `down` | Identifica con precisión las anomalías detectadas |
| **Causa probable** | Las rutas ausentes pueden depender del vecino caído; el segmento de servidores puede estar afectado | Vincula el síntoma a las evidencias sin sobredimensionar |
| **Comprobaciones siguientes** | Verificar enlace hacia `spine2`, logs de BGP y estado físico de `Ethernet3` | Orienta al ingeniero hacia los siguientes pasos prácticos |

> **Tabla 7.3:** Estructura recomendada para las respuestas de diagnóstico del agente.

---

### Sección 11: Detección y corrección de conclusiones erróneas del modelo

Si la salida de una herramienta contradice la conclusión generada por el LLM, **la evidencia de la herramienta prevalece siempre**.

Ejemplo de conclusión riesgosa:

```text
leaf2 appears to be functioning normally. It has two established peers and no major connectivity issues.
```

Corrección obligatoria alineada con los datos:

```text
leaf2 is up, but it has issues to investigate. Only 1 of 2 BGP peers is established. Neighbor 10.1.2.2 is Idle and receiving 0 prefixes. Ethernet3 (server_rack_2) is down. This may explain missing routes learned through that neighbor, while Ethernet3 may affect the server-facing segment.
```

---

### Sección 12: Optimización de prompts para diagnóstico

Para minimizar discrepancias, el prompt debe incorporar directivas operativas estrictas:

```text
When summarizing troubleshooting results: - Report any BGP neighbor not in Established state. - Compare established_peers with total_peers. - Treat prefixes: 0 as possible routing impact. - Do not claim no issues if any tool returned an error or degraded state. - Separate confirmed facts, likely causes, and next checks.
```

---

### Sección 13: Pruebas en modo interactivo

En el modo interactivo, audita tanto el texto final como las herramientas invocadas por el modelo. La **Tabla 7.4** sintetiza las pruebas de validación:

| Prueba interactiva | Comportamiento esperado del agente | Señal de advertencia (*Warning*) |
| :--- | :--- | :--- |
| **Verificar problemas BGP en `leaf2`** | Invoca `get_bgp_summary()` y `get_interface_status()`, reportando el vecino caído y `Ethernet3` | Afirma que `leaf2` está sano sin consultar BGP |
| **Host `10.99.99.99` inalcanzable** | Invoca `ping_device()` y detalla el fallo de alcanzabilidad | Asume la causa raíz sin verificar con herramientas |
| **¿Todas las sesiones BGP están activas?** | Compara `total_peers` y `established_peers` | Se limita a explicar BGP en términos genéricos |
| **Mostrar evidencias** | Incluye datos textuales y nombres de campos devueltos | Devuelve una recomendación vaga sin datos objetivos |

> **Tabla 7.4:** Pruebas interactivas para evaluar el comportamiento del agente de diagnóstico.

---

### Sección 14: Alcance y límites operativos del agente

La **Tabla 7.5** delimita las capacidades soportadas en el laboratorio frente a los requisitos para entornos de producción:

| Soportado en este capítulo | Fuera de alcance en este capítulo |
| :--- | :--- |
| Invocar herramientas simuladas aprobadas | Conectarse directamente a dispositivos físicos de producción |
| Inspeccionar datos de dispositivos, interfaces, BGP y ping | Garantizar el estado en tiempo real de redes en vivo |
| Sintetizar causas probables a partir de evidencias | Asumir causas no respaldadas por las salidas de las herramientas |
| Proponer siguientes comprobaciones al operador | Realizar modificaciones directas de configuración |
| Exponer evidencias de herramientas para revisión humana | Sustituir el juicio y la responsabilidad del ingeniero |

> **Tabla 7.5:** Capacidades y límites operativos del agente de diagnóstico.

---

### Sección 15: Análisis paso a paso del caso leaf2

1. **Estado del dispositivo:**
```text
🔧 Agent is calling: get_device_status({"device": "leaf2"}) 📊 Result: { "device": "leaf2", "ip": "192.168.0.22", "status": "up", "model": "cEOS", "version": "4.27.3F", "serial": "LFX2134567891", "uptime": "2d", "role": "leaf" }
```
El equipo responde, pero esto no garantiza que el plano de control esté sano.

2. **Estado de BGP:**
```text
🔧 Agent is calling: get_bgp_summary({"device": "leaf2"}) 📊 Result: { "device": "leaf2", "local_as": 65012, "router_id": "10.0.1.22", "total_peers": 2, "established_peers": 1, "neighbors": [ { "ip": "10.1.1.2", "remote_as": 65001, "state": "Established", "uptime": "2d3h", "prefixes": 50 }, { "ip": "10.1.2.2", "remote_as": 65001, "state": "Idle", "uptime": "0h", "prefixes": 0 } ] }
```
Se detecta una sesión caída (`10.1.2.2` en `Idle` con 0 prefijos).

3. **Estado de interfaces:**
```text
🔧 Agent is calling: get_interface_status({"device": "leaf2"}) 📊 Result: { "device": "leaf2", "interfaces": [ { "name": "Ethernet1", "description": "to_spine1", "status": "up", "speed": "10G" }, { "name": "Ethernet2", "description": "to_spine2", "status": "up", "speed": "10G" }, { "name": "Ethernet3", "description": "server_rack_2", "status": "down", "speed": "1G" }, { "name": "Management1", "description": "oob_management", "status": "up", "speed": "1G" } ] }
```
Se identifica la interfaz `Ethernet3` en estado `down`.

---

### Sección 16: Creación de un registro de evidencias estructurado

La aplicación puede compilar un registro estructurado de evidencias independiente del texto del prompt para auditoría y validación determinista:

```json
{ "device": "leaf2", "evidence": { "device_status": { "tool": "get_device_status", "status": "up" }, "interface_status": { "tool": "get_interface_status", "down_interfaces": [ { "name": "Ethernet3", "description": "server_rack_2", "status": "down" } ] }, "bgp_status": { "tool": "get_bgp_summary", "total_peers": 2, "established_peers": 1, "non_established_neighbors": [ { "ip": "10.1.2.2", "state": "Idle", "prefixes": 0 } ] } } }
```

Función auxiliar de validación determinista:

```python
def bgp_has_issue(bgp_result: dict)-> bool: if "error" in bgp_result: return True if bgp_result.get("established_peers") != bgp_result.get("total_peers"): return True for neighbor in bgp_result.get("neighbors", []): if neighbor.get("state") != "Established": return True return False
```

---

### Sección 17: Plantilla de respuesta estructurada en cinco partes

Para estandarizar las respuestas, se establece el siguiente formato de prompt:

```text
Return the final troubleshooting answer using this format: Finding: Evidence: Likely cause: Next checks: Unknowns: Do not omit unhealthy evidence. Do not say the network is healthy if a tool result contains an error, Idle peer, down interface, or unreachable target.
```

Ejemplo de respuesta validada:

```text
Finding: leaf2 has two issues to investigate. The device is up, but one BGP neighbor is not established and Ethernet3 is down. Evidence: - leaf2 status is up - Ethernet3 on leaf2 is down - BGP total_peers is 2 - BGP established_peers is 1 - neighbor 10.1.2.2 is Idle - neighbor 10.1.2.2 has 0 prefixes Likely cause: Missing routes may be related to the idle BGP neighbor if the route should be learned through that peer. Ethernet3 may also affect the local server-facing segment. Next checks: Check the peer at 10.1.2.2, inspect BGP logs, verify link state toward spine2, check Ethernet3 locally, and confirm whether the missing prefix should be advertised by that peer.
```

La **Tabla 7.6** ilustra el contraste entre conclusiones imprecisas y conclusiones respaldadas por datos:

| Conclusión deficiente | Conclusión basada en evidencias |
| :--- | :--- |
| *El host es inalcanzable porque BGP está caído.* | *El host no responde y `leaf2` presenta un vecino BGP en `Idle` con 0 prefijos. Si la ruta dependía de ese enlace, BGP es una causa probable. Asimismo, `Ethernet3` está en `down` y debe verificarse.* |
| *`leaf2` está sano porque el dispositivo está `up`.* | *`leaf2` está activo, pero presenta un vecino BGP en `Idle` e `Ethernet3` en `down`, requiriendo inspección en el plano de control y en el enlace local.* |
| *Todo BGP se ve correcto.* | *`leaf2` tiene `established_peers: 1` de `total_peers: 2`; el vecino `10.1.2.2` no está establecido.* |

> **Tabla 7.6:** Comparación entre conclusiones débiles y conclusiones fundamentadas en evidencias.

---

### Sección 18: Preparación para herramientas reutilizables con MCP

En este laboratorio las herramientas residen dentro de la propia aplicación de Python. En el [Capítulo 8](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/8) utilizaremos el protocolo **MCP** (*Model Context Protocol*) para desacoplar las herramientas de red y exponerlas mediante un servidor estandarizado reutilizable por múltiples asistentes e interfaces.

---

### Sección 19: Resumen

En este capítulo transformamos los mecanismos básicos de invocación de herramientas en un flujo completo de resolución de incidencias de red.

El agente recopiló datos de inventario, interfaces, tablas BGP y pruebas de alcanzabilidad ante síntomas de pérdida de rutas, evidenciando la necesidad de confrontar y validar las deducciones del modelo frente a los datos objetivos de las herramientas.

En el próximo capítulo avanzaremos hacia la estandarización y desacoplamiento de estas capacidades mediante la implementación de servidores de herramientas basados en Model Context Protocol (MCP).
