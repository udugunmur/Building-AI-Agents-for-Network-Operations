# Parte 3: Operaciones del Mundo Real y Producción

## Capítulo 9: Hacia Agentes de Red Listos para Producción

Un agente de laboratorio puede resultar impactante: puede invocar una herramienta, inspeccionar un vecino BGP, resumir una anomalía y agilizar sustancialmente el diagnóstico frente a una investigación manual. Eso es valioso, pero dista mucho de estar listo para producción.

En producción las preguntas cambian radicalmente: en lugar de evaluar si el agente responde a un prompt, evaluamos si el sistema es confiable cuando un equipo no responde, una credencial expira, una herramienta devuelve datos parciales o un usuario solicita un comando no seguro. El agente de IA es solo una pieza dentro de ese ecosistema global.

En el capítulo anterior expusimos herramientas de red reutilizables mediante Model Context Protocol (MCP). Eso aportó una interfaz desacoplada, pero una interfaz reutilizable no es intrínsecamente segura a nivel operativo. Cuando múltiples clientes pueden invocar una herramienta, los límites y salvaguardas alrededor de ella se vuelven críticos.

Este capítulo constituye la verificación de realidad en producción de todo el libro. No presentaremos el repositorio de laboratorio como una plataforma enterprise terminada; en su lugar, utilizaremos los archivos del Laboratorio 6 como patrones de referencia para construir una lista de verificación práctica que permita evolucionar desde una demo local hacia un prototipo operativo controlado, observable y basado en el principio de **solo lectura primero**.

En este capítulo cubriremos los siguientes temas:

- Revisar los requisitos técnicos para la preparación para producción.
- Comprender por qué los entornos de producción difieren de los laboratorios.
- Definir el perímetro de producción (*production boundary*) para un agente de red.
- Mantener la primera fase operativa estrictamente en modo de solo lectura.
- Determinar qué procesos no deben automatizarse.
- Implementar autenticación, autorización y gestión segura de secretos.
- Validar entradas y salidas de herramientas antes de confiar en ellas.
- Incorporar puertas de aprobación (*approval gates*) para acciones de riesgo.
- Registrar, auditar y monitorizar el comportamiento del agente.
- Utilizar el Laboratorio 6 como patrón de referencia para producción.
- Planificar un despliegue por fases desde el laboratorio hasta la evaluación controlada.

Al finalizar este capítulo, dispondrás de una lista de verificación clara para determinar cuándo un agente está listo para salir del laboratorio y, fundamentalmente, qué preguntas formular antes de conectar cualquier flujo agéntico a la infraestructura real.

---

### Sección 1: Requisitos técnicos

Este capítulo utiliza los ejemplos de preparación para producción del repositorio. Las prácticas se mantienen en un entorno local, simulado y de solo lectura.

Requisitos y archivos necesarios:

- **Python 3.10** o superior.
- Dependencias del proyecto instaladas desde `requirements.txt`.
- Archivos del Laboratorio 6 en `labs/lab6-production-readiness/`.
- Backend simulado en `examples/mock_network_devices.py`.

Asegúrate de contar con el entorno listo:

```bash
pip install -r requirements.txt
```

Ejecuta los dos ejemplos de referencia del Laboratorio 6:

```bash
python3 labs/lab6-production-readiness/safe_tools.py
python3 labs/lab6-production-readiness/production_agent_skeleton.py
```

El Laboratorio 6 es un patrón de referencia que ilustra la capa de salvaguardas (*guardrails*) indispensable antes de interactuar con redes reales.

---

### Sección 2: Por qué producción es diferente a un laboratorio

Un laboratorio está diseñado para ser permisivo: las entradas son conocidas, los equipos están simulados, las credenciales no son críticas y el coste del fallo es nulo.

En producción, un comando erróneo, un destino equivocado o un resumen inexacto pueden retrasar la mitigación de una avería o provocar una interrupción del servicio (*outage*).

Por ello, el primer hito productivo debe ser un **prototipo de solo lectura con alta observabilidad**, orientado a evaluar el comportamiento del agente bajo condiciones reales sin concederle permisos para alterar la red.

La **Tabla 9.1** compara las asunciones del laboratorio frente a las exigencias de producción:

| Área técnica | Asunción en laboratorio | Exigencia en entorno de producción |
| :--- | :--- | :--- |
| **Backend de red** | Equipos simulados devuelven datos predecibles | Entornos reales exigen timeouts, reintentos y gestión de fallos |
| **Acceso a herramientas** | Cualquier usuario local ejecuta la demo | Usuarios y clientes deben estar autenticados y autorizados |
| **Control de comandos** | Se muestran ejemplos seguros controlados | Comandos restringidos a listas blancas estrictas y auditados |
| **Calidad de datos** | La salida de los datos es conocida de antemano | Salidas de herramientas validadas, tipadas y etiquetadas |
| **Gestión de fallos** | Una ejecución fallida se reinicia manualmente | Los fallos deben ser visibles, acotados, seguros y auditables |
| **Modificación de estado** | Sin cambios sobre configuraciones | Toda escritura exige aprobación, evidencias y plan de rollback |

> **Tabla 9.1:** Comparativa entre asunciones de laboratorio y requisitos para producción.

---

### Sección 3: Definición del perímetro de producción

Un agente de red no es únicamente el modelo de lenguaje: comprende al usuario, la aplicación del agente, el servidor de herramientas, las funciones autorizadas, el backend de red y las capas perimetrales de auditoría y aprobación.

> **Figura 9.1:** Perímetro de producción y separación de responsabilidades para un agente de red.

El principio rector es la **separación de responsabilidades**:
- El modelo razona sobre las evidencias.
- La aplicación gobierna el flujo de ejecución.
- La capa de herramientas aplica las validaciones.
- La capa de autorización gestiona los accesos.
- La capa de logging garantiza la trazabilidad.

---

### Sección 4: Mantener la primera fase operativa en solo lectura

La fase inicial fuera del laboratorio debe ser estrictamente de solo lectura. Un agente de solo lectura aporta un alto valor operativo: enriquece alertas, resume estados BGP, inspecciona interfaces, verifica tablas de rutas y prepara traspasos estructurados para los ingenieros de guardia.

Aunque una herramienta de solo lectura puede filtrar información o sobrecargar un equipo si no se controla, su radio de impacto (*blast radius*) es infinitamente menor que el de una herramienta de modificación.

Herramientas típicas de solo lectura:
- `device_status`: Inventario y salud general.
- `interface_status`: Estado y descripciones de puertos.
- `bgp_summary`: Sesiones de vecinos y recuento de prefijos.
- `topology`: Relaciones de adyacencia e interconexión.
- `show_command`: Comandos `show` autorizados en lista blanca.
- `ping`: Pruebas de alcanzabilidad con recuento de paquetes acotado.
- Herramientas de consulta de logs filtrados por eventos y marcas de tiempo.

---

### Sección 5: Delimitación de lo que no debe automatizarse

Un plan de producción riguroso define con precisión qué acciones quedan excluidas de la automatización. Las herramientas genéricas y abiertas (como un hipotético `run_command()`) introducen riesgos inasumibles; el diseño seguro exige herramientas acotadas con propósitos unívocos y validaciones estrictas.

La **Tabla 9.2** clasifica las herramientas según su nivel de riesgo:

| Herramienta o capacidad | Nivel de riesgo | Postura recomendada en producción |
| :--- | :--- | :--- |
| `device_status(device)` | Bajo | Permitir tras autenticación y validación de lista blanca de equipos |
| `bgp_summary(device)` | Bajo | Permitir en equipos autorizados registrando cada invocación en logs |
| `show_command(device, command)` | Medio | Permitir exclusivamente comandos de inspección (`show`) en lista blanca |
| `run_command(device, command)` | Alto | Evitar ejecución abierta; reemplazar por herramientas específicas |
| **Herramientas de cambio de configuración** | Alto | Exigir ticket de cambio, diff de configuración, aprobación y plan de rollback |
| **Herramientas de reinicio / borrado de sesiones** | Alto | Excluir de la fase inicial salvo control y supervisión extrema |

> **Tabla 9.2:** Clasificación de riesgo y posturas operativas recomendadas.

---

### Sección 6: Autenticación, autorización y control de acceso (RBAC)

La **autenticación** valida la identidad del solicitante; la **autorización** determina a qué herramientas y equipos tiene acceso.

La **Tabla 9.3** ilustra un modelo de control de acceso basado en roles (**RBAC**):

| Rol operativo | Herramientas permitidas | Acciones bloqueadas |
| :--- | :--- | :--- |
| **Visualizador (*Viewer*)** | Estado de solo lectura, topología y resúmenes | Ejecución de comandos, aprobaciones o alteraciones |
| **Operador de red (*Operator*)** | Herramientas de solo lectura y comandos `show` autorizados | Cambios de configuración y comandos abiertos |
| **Ingeniero sénior** | Diagnósticos avanzados y revisión de aprobaciones | Acciones de escritura no auditadas |
| **Mantenedor de automatización** | Pruebas de herramientas y validación en laboratorio | Ejecución en producción fuera de control de cambios |
| **Administrador** | Gestión de políticas y control de accesos | Eludir requisitos de auditoría, logging o aprobación |

> **Tabla 9.3:** Modelo de ejemplo de control de acceso a herramientas basado en roles (RBAC).

---

### Sección 7: Gestión de secretos y credenciales

Las credenciales nunca deben integrarse en los prompts, incluirse en el código fuente, subirse a repositorios Git ni imprimirse en registros o terminales.

Reglas obligatorias para la gestión de secretos:
- Inyectar credenciales mediante variables de entorno aisladas o gestores de secretos (*vaults*).
- Asignar cuentas de servicio bajo el principio de mínimo privilegio (*least privilege*).
- Emplear credenciales de solo lectura para las etapas iniciales.
- Rotar claves periódicamente y registrar formalmente su custodia.
- Aislar por completo las credenciales de laboratorio de las de producción.

---

### Sección 8: Validación rigurosa de entradas y salidas

La validación de entrada protege a la red frente a solicitudes inválidas; la validación de salida protege al operador frente a interpretaciones erróneas del modelo.

La **Tabla 9.4** detalla los puntos de control requeridos:

| Punto de validación | Aspecto a verificar | Ejemplo de comprobación |
| :--- | :--- | :--- |
| **Entrada de dispositivo** | El equipo existe y está dentro del alcance autorizado | Rechazar `core99` si no figura en el inventario permitido |
| **Entrada de comando** | El comando es de solo lectura y está en lista blanca | Permitir `show ip bgp summary`; bloquear `configure terminal` |
| **Límites de parámetros** | Parámetros numéricos y timeouts acotados | Limitar `count` en pings y aplicar timeouts por petición |
| **Estructura de salida** | Claves obligatorias y tipos de datos presentes | Verificar `total_peers`, `established_peers` y `neighbors` |
| **Cargas de error** | Retorno de errores estructurados ante fallos | Devolver `allowed: false` con motivo en lugar de una excepción |
| **Conclusión del agente** | El resumen final coincide con los datos objetivos | No afirmar salud global si un vecino figura en `Idle` |

> **Tabla 9.4:** Comprobaciones de validación para herramientas de red.

---

### Sección 9: Demostración de wrappers de seguridad del Lab 6

Ejecuta el script `safe_tools.py` desde la raíz del repositorio:

```bash
python3 labs/lab6-production-readiness/safe_tools.py
```

Salida esperada (fragmento):

```text
✅ Allowed BGP summary { "device": "leaf2", "total_peers": 2, "established_peers": 1, "neighbors": [ {"ip": "10.1.1.2", "state": "Established", "prefixes": 50}, {"ip": "10.1.2.2", "state": "Idle", "prefixes": 0} ] } 🚫 Blocked unsafe command { "error": "configuration and exec commands are blocked; only read-only show commands are allowed", "allowed": false, "risk": "high", "requires_approval": true }
```

El wrapper autoriza la consulta de solo lectura, bloquea comandos de configuración y retorna una estructura de denegación detallando el nivel de riesgo.

---

### Sección 10: Flujos de aprobación humana (Human-in-the-Loop)

Las acciones que impliquen modificaciones deben exigir validación humana explícita antes de su ejecución.

> **Figura 9.2:** Flujo de aprobación humana para acciones que alteran la red.

La solicitud de aprobación debe presentar al operador el destino exacto, la acción propuesta, las evidencias recopiladas, el impacto estimado y el plan de reversión (*rollback*).

---

### Sección 11: Registro y auditoría estructurada

Cada interacción con las herramientas debe generar una traza inmutable. La **Tabla 9.5** detalla los campos esenciales de un evento de auditoría:

| Campo del log | Justificación técnica | Valor de ejemplo |
| :--- | :--- | :--- |
| `timestamp` | Marca temporal de la invocación | `2026-06-28T18:36:38Z` |
| `user` / `client_id` | Identidad del usuario o servicio solicitante | `noc-operator-1` |
| `tool` | Nombre de la herramienta ejecutada | `bgp_summary` |
| `device` | Dispositivo de red objetivo | `leaf2` |
| `command` | Comando exacto cuando aplica | `show ip bgp summary` |
| `decision.allowed` | Dictamen de la política de seguridad | `true` o `false` |
| `decision.reason` | Justificación del dictamen de la política | `read-only command approved` |
| `result_summary` | Estado final del resultado | `success`, `blocked` o `timeout` |
| `duration_ms` | Tiempo total de ejecución para monitorización | `184` |
| `approval_id` | Identificador del registro de aprobación vinculado | `CHG-20491` |

> **Tabla 9.5:** Campos recomendados para el registro de auditoría de herramientas.

Ejemplo de evento estructurado generado por el Laboratorio 6:

```json
{ "timestamp": "2026-06-28T18:36:38.320934+00:00", "tool": "show_command", "device": "spine1", "command": "show ip bgp summary", "decision": { "allowed": true, "reason": "read-only command approved", "risk": "low", "requires_approval": false }, "result_summary": "success", "metadata": {} }
```

---

### Sección 12: Observabilidad y métricas del sistema

La **Tabla 9.6** enumera los indicadores clave de observabilidad para evaluar el comportamiento del agente:

| Métrica / Señal | Qué indica | Acción operacional derivada |
| :--- | :--- | :--- |
| **Recuento de llamadas por herramienta** | Frecuencia de uso de cada capacidad | Identificar herramientas críticas o de mayor riesgo |
| **Tasa de éxito (*Success rate*)** | Porcentaje de ejecuciones satisfactorias | Investigar degradaciones tras cambios de versión |
| **Comandos bloqueados** | Intentos de ejecución no permitidos | Revisar formación de usuarios, prompts o posibles anomalías |
| **Equipos desconocidos solicitados** | Intentos de acceso fuera de inventario | Actualizar inventario o clarificar el alcance del agente |
| **Latencia por herramienta** | Tiempos de respuesta de cada función | Optimizar consultas al backend o ajustar timeouts |
| **Tasa de timeouts** | Estabilidad de las conexiones al backend | Implementar reintentos, backoff exponencial o alertas |
| **Tasa de aprobación** | Decisiones sobre acciones de riesgo | Evaluar políticas de seguridad y carga de los operadores |
| **Discrepancias modelo/herramienta** | Solicitudes con parámetros o nombres inválidos | Perfeccionar la descripción de herramientas en el prompt |

> **Tabla 9.6:** Señales de observabilidad para agentes de red en producción.

---

### Sección 13: Gestión de fallos y principio de fallo seguro (Fail-Closed)

Ante cualquier inconsistencia o fallo en la validación, el sistema debe **fallar de forma segura** (*fail closed*). La **Tabla 9.7** ilustra las respuestas requeridas:

| Modo de fallo | Riesgo operacional | Respuesta segura del sistema |
| :--- | :--- | :--- |
| **Dispositivo desconocido** | Acciones sobre equipos equivocados | Rechazar la petición y devolver la lista de equipos permitidos |
| **Comando no soportado** | Exposición indebida o cambio de estado | Bloquear el comando y detallar la lista blanca permitida |
| **Timeout en el backend** | Decisiones basadas en datos incompletos | Devolver error de timeout e indicar explícitamente la incertidumbre |
| **Salida malformada** | Parseo de datos corruptos | Rechazar o reintentar con número acotado de intentos |
| **Datos parciales** | Conclusiones sobredimensionadas | Informar sobre lo confirmado y detallar lo no verificado |
| **Aprobación denegada** | Ejecución indebida | Detener el flujo de forma segura y auditar la decisión |
| **Petición de escritura inesperada** | Excesos en el alcance del agente | Bloquear por defecto y exigir revisión formal |

> **Tabla 9.7:** Modos de fallo y respuestas seguras del sistema.

---

### Sección 14: Patrón de esqueleto de producción

Ejecuta `production_agent_skeleton.py` para observar la estabilidad del contrato frente a cambios de backend:

```bash
python3 labs/lab6-production-readiness/production_agent_skeleton.py
```

Salida esperada:

```text
=== Mock backend demo === { "device": "leaf2", "summary": "BGP issue detected: 1/2 peers are established." } === Real backend placeholder demo === { "device": "leaf2", "summary": "Device status check failed: RealNetworkBackend.device_status is not implemented yet" }
```

El contrato expuesto al agente se mantiene invariante independientemente de si los datos provienen de mocks, APIs de controladores, NetBox o conexiones SSH mediante Netmiko/NAPALM.

---

### Sección 15: Planificación del despliegue escalonado

La **Tabla 9.8** establece la progresión recomendada para el despliegue de agentes:

| Fase | Alcance y cambios introducidos | Criterio de superación de fase |
| :--- | :--- | :--- |
| **1. Laboratorio local** | Datos simulados y herramientas locales | Laboratorios estables y salidas comprendidas |
| **2. Demo interna** | Equipo reducido prueba flujos de solo lectura | Incidencias documentadas y trazas auditables |
| **3. Piloto de solo lectura** | Usuarios autorizados consultan equipos reales | Control de acceso, logs y gestión de fallos operativos |
| **4. Modo recomendación** | El agente sugiere acciones sin ejecutarlas | Alta confianza del equipo en la calidad de las evidencias |
| **5. Acciones con aprobación** | Cambios acotados con aprobación humana | Flujos de aprobación, rollback y auditoría consolidados |
| **6. Producción ampliada** | Incorporación gradual de nuevos equipos y herramientas | Monitorización madura y gobernanza formalizada |

> **Tabla 9.8:** Fases de despliegue progresivo para agentes de red.

---

### Sección 16: Lista de verificación de preparación para producción

Preguntas clave que el equipo debe responder antes de iniciar un piloto:
1. ¿Cuál es el caso de uso exacto y el alcance de dispositivos?
2. ¿Qué herramientas puede invocar el agente automáticamente?
3. ¿Qué herramientas exigen aprobación previa?
4. ¿Dónde y cómo se almacenan las credenciales?
5. ¿Qué registros demuestran fehacientemente las acciones del agente?
6. ¿Cómo se notifican y gestionan las peticiones bloqueadas?
7. ¿Cómo reacciona el sistema ante timeouts del backend?
8. ¿Quién asume la propiedad operativa del agente tras el despliegue?
9. ¿Cuál es el procedimiento de desconexión de emergencia (*kill switch*)?
10. ¿Qué evidencias estructuradas se presentarán tras una revisión de incidentes?

La **Tabla 9.9** detalla los artefactos del paquete de revisión documental:

| Artefacto de revisión | Contenido requerido | Beneficio para la gobernanza |
| :--- | :--- | :--- |
| **Resumen del caso de uso** | Problema, alcance, usuarios y valor esperado | Evita que el agente se convierta en una herramienta difusa |
| **Inventario de herramientas** | Nombres, argumentos, salidas y responsables | Visibilidad integral de las capacidades expuestas |
| **Alcance de dispositivos** | Equipos, grupos, entornos y exclusiones | Delimita y acota el radio de impacto (*blast radius*) |
| **Política de comandos** | Comandos `show` permitidos y categorías bloqueadas | Facilita la auditoría de seguridad de comandos |
| **Modelo de acceso (RBAC)** | Asignación de roles y permisos por herramienta | Vincula las herramientas con el esquema de autorización |
| **Plan de auditoría y logs** | Campos capturados, retención y destino de logs | Soporte para revisiones forenses y cumplimiento normativo |
| **Plan de gestión de fallos** | Timeouts, reintentos, bloqueos y escalado | Define el comportamiento ante anomalías del sistema |
| **Procedimiento de parada** | Mecanismos de desactivación rápida del agente | Proporciona un mecanismo de control de emergencia |

> **Tabla 9.9:** Componentes del paquete de revisión para despliegue en producción.

---

### Sección 17: Matriz de pruebas previas al piloto

La **Tabla 9.10** define las pruebas indispensables antes del paso a producción:

| Área de prueba | Prueba ejecutada | Resultado esperado |
| :--- | :--- | :--- |
| **Validación de equipos** | Solicitar `device_status("core99")` | Rechazo con error estructurado de equipo desconocido |
| **Política de comandos** | Solicitar `configure terminal` | Bloqueo inmediato del comando marcando riesgo alto |
| **Canal de solo lectura** | Solicitar `show ip bgp summary` | Autorización del comando y registro en logs |
| **Evidencias BGP** | Consultar `leaf2` | Reporte de 1/2 vecinos establecidos e identificación del caído |
| **Placeholders de backend** | Invocar `RealNetworkBackend` | Retorno de error estructurado de funcionalidad no implementada |
| **Captura de auditoría** | Invocar cualquier wrapper seguro | Registro de evento con herramienta, equipo, dictamen y estado |
| **Mensajes ante fallos** | Forzar datos parciales o incompletos | El agente reporta incertidumbre sin inventar conclusiones |

> **Tabla 9.10:** Matriz de pruebas de preparación para producción.

---

### Sección 18: Elaboración del runbook operacional

La **Tabla 9.11** describe las secciones fundamentales que debe contener el runbook:

| Sección del Runbook | Contenido a incluir | Ejemplo práctico |
| :--- | :--- | :--- |
| **Propósito** | Objetivo y función operativa del agente | Diagnóstico y triaje de solo lectura de interfaces y BGP |
| **Alcance** | Entornos, equipos y usuarios autorizados | Equipos de laboratorio, staging o producción delimitada |
| **Arranque** | Comandos y servicios requeridos para iniciar | Inicio del servidor de herramientas, puente e interfaz |
| **Comprobaciones de salud** | Verificación de operatividad del servicio | Ejecución de tests de wrappers o consultas de prueba |
| **Ubicación de logs** | Destino de trazas de aplicación y auditoría | Pipeline de logs, archivos locales o índice SIEM |
| **Fallos comunes** | Errores típicos y comprobaciones iniciales | Timeouts de backend, equipos desconocidos o comandos bloqueados |
| **Escalado** | Propietario del sistema y criterios de guardia | Ingeniero de automatización o responsable de NetOps |
| **Desactivación** | Procedimiento de parada de emergencia | Parada del servicio, revocación de tokens o aislamiento de red |

> **Tabla 9.11:** Secciones del runbook operacional para agentes de red.

---

### Sección 19: Feature Flags y mecanismos de parada de emergencia (Kill Switch)

Modelo de configuración para control dinámico de capacidades:

```json
{ "tools": { "device_status": {"enabled": true}, "bgp_summary": {"enabled": true}, "show_command": {"enabled": true, "read_only_only": true}, "configure_interface": {"enabled": false} }, "global_kill_switch": false, "allowed_environment": "read_only_pilot" }
```

Permite habilitar herramientas progresivamente o suspender el agente de forma instantánea sin requerir modificaciones en el código.

---

### Sección 20: Modelo de gobernanza y propiedad

La **Tabla 9.12** define la matriz de responsabilidades organizativas:

| Responsabilidad técnica | Propietario principal | Aspectos bajo su supervisión |
| :--- | :--- | :--- |
| **Implementación de herramientas** | Ingeniero de automatización | Código, esquemas, tests unitarios y adaptadores |
| **Alcance de red** | Responsable de ingeniería de red | Equipos aprobados, comandos y límites operativos |
| **Políticas de acceso** | Responsable de seguridad / plataforma | Autenticación, roles RBAC, cuentas de servicio y tokens |
| **Soporte operacional** | Equipo NOC / NetOps | Runbook, monitorización, alertas y resolución de incidencias |
| **Auditoría y cumplimiento** | Revisor de gobierno / seguridad | Logs, políticas de retención, autorizaciones y evidencias |
| **Comportamiento del modelo** | Equipo de plataforma de IA / Agentes | Prompts, ajuste de modelos y calidad de respuestas |

> **Tabla 9.12:** Modelo de propiedad y gobierno para agentes de red en producción.

---

### Sección 21: Criterios de aceptación para pilotos

La **Tabla 9.13** resume los criterios cuantitativos para evaluar el piloto:

| Criterio de evaluación | Objetivo en piloto de solo lectura | Método de medición |
| :--- | :--- | :--- |
| **Calidad de evidencias** | Las respuestas citan datos de herramientas | Muestreo de transcripciones confrontadas con logs |
| **Seguridad operativa** | Cero comandos no autorizados ejecutados | Auditoría de logs de comandos permitidos y bloqueados |
| **Gestión de errores** | Los fallos devuelven estructuras claras | Inyección de pruebas con equipos y timeouts simulados |
| **Latencia** | Respuestas dentro de umbrales pactados | Monitorización de duración por herramienta |
| **Completitud de logs** | 100% de llamadas registradas en auditoría | Conciliación de peticiones de usuario frente a logs |
| **Utilidad percibida** | Reducción del tiempo de triaje inicial | Encuestas y retroalimentación de los operadores |

> **Tabla 9.13:** Criterios de aceptación para pilotos de solo lectura.

---

### Sección 22: Hoja de trabajo de preparación operativa

La **Tabla 9.14** proporciona una plantilla de trabajo para validar los requisitos:

| Pregunta de revisión | Responsable asignado | Evidencia documental adjunta |
| :--- | :--- | :--- |
| **¿Qué caso de uso está aprobado para el piloto?** | Propietario del agente | Declaración de alcance y casos de uso excluidos |
| **¿Qué equipos están dentro del alcance?** | Responsable de ingeniería de red | Lista blanca de equipos o grupo de inventario |
| **¿Qué herramientas están habilitadas?** | Ingeniero de automatización | Catálogo de herramientas y archivo de configuración |
| **¿Qué comandos están autorizados?** | Responsable de ingeniería de red | Lista blanca de comandos de inspección permitidos |
| **¿Quién tiene acceso a las herramientas?** | Responsable de seguridad / plataforma | Mapeo de roles RBAC y cuentas de servicio |
| **¿Dónde se almacenan los logs de auditoría?** | Responsable de operaciones | Destino, política de retención y muestra de log |
| **¿Cómo se desactiva el agente en emergencia?** | Responsable de operaciones | Sección del runbook o procedimiento probado |
| **¿Cómo se evaluará el éxito del piloto?** | Propietario del proyecto | Criterios de aceptación y fecha de revisión |

> **Tabla 9.14:** Hoja de trabajo de preparación para el piloto del agente de red.

---

### Sección 23: Registro formal de aprobaciones de cambio

Estructura de datos para registrar solicitudes de cambio que requieran supervisión:

```json
{ "approval_id": "CHG-20491", "requested_by": "network-agent", "reviewer": "senior-netops-engineer", "target_device": "leaf2", "proposed_action": "clear bgp neighbor 10.1.2.2", "risk": "high", "evidence": { "bgp_state": "Idle", "prefixes": 0, "last_checked": "2026-06-28T18:40:00Z" }, "rollback_plan": "No configuration change. Verify session state after action.", "decision": "pending" }
```

---

### Sección 24: Conexión con el itinerario formativo del libro

Este capítulo culmina el recorrido iniciado en el [Capítulo 1](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/1):
- **Capítulo 1:** Definición de agentes de IA y su encaje en la operativa de red.
- **Capítulo 2:** Fundamentos de LLMs y configuración de entornos locales.
- **Capítulo 3:** Ingeniería de prompts estructurados mediante el framework RACE.
- **Capítulo 4:** Parseo de salidas no estructuradas hacia JSON validado.
- **Capítulo 5:** Incorporación de memoria conversacional en la aplicación.
- **Capítulo 6:** Diseño de invocación controlada de herramientas.
- **Capítulo 7:** Construcción de un flujo completo de resolución de problemas.
- **Capítulo 8:** Exposición de herramientas reutilizables mediante MCP.
- **Capítulo 9:** Aplicación de controles integrales para producción.

---

### Sección 25: Revisión final Go/No-Go

Preguntas de control antes de autorizar el paso a producción:
1. ¿Cuál es el peor escenario posible con los permisos actuales del agente?
2. ¿Podemos detectar ese comportamiento de forma inmediata?
3. ¿Podemos detener el flujo de forma segura e instantánea?
4. ¿Podemos justificar cada respuesta del agente mediante logs de herramientas?
5. ¿Puede un ingeniero refutar, rechazar o anular la conclusión del agente?
6. ¿Están los secretos estrictamente aislados de prompts, logs y repositorios?
7. ¿Exige toda acción de riesgo una aprobación humana explícita?
8. ¿Estamos plenamente capacitados para presentar la traza de auditoría tras un incidente?

Si alguna respuesta es dudosa, el flujo debe permanecer en modo de solo lectura.

---

### Sección 26: Resumen

En este capítulo analizamos los requisitos y controles necesarios para avanzar desde herramientas de laboratorio hacia despliegues operativos listos para producción.

Examinamos la autenticación, autorización RBAC, gestión segura de secretos, validación estricta de entradas y salidas, puertas de aprobación humana, observabilidad, runbooks y estrategias de despliegue escalonado.

El principio cardinal se mantiene: **un agente es confiable solo si el sistema que lo gobierna es confiable**.

Con estos fundamentos completamos el cuerpo principal de la obra. El Apéndice A complementa este contenido con listas de verificación, matrices de decisión y plantillas de diseño reutilizables para tus proyectos de agentes de red.
