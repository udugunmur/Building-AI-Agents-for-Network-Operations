# Parte 1: Fundamentos y Conceptos Clave

## Capítulo 4: Parseo de Salidas de Red a Datos Estructurados

Los ingenieros de red pasan gran parte de su tiempo examinando texto. Salidas de dispositivos, registros (*logs*), alertas, fragmentos de configuración, tablas de enrutamiento y notas de tickets son, en última instancia, texto en diversas formas. Los humanos pueden leer ese texto y comprenderlo, pero la automatización requiere estructuras mucho más predecibles.

Ahí es donde entran los datos estructurados. Si convertimos la salida de la interfaz de línea de comandos (**CLI**) en **JSON** (*JavaScript Object Notation*), podemos validarla, filtrarla, transferirla a otro script, almacenarla en bases de datos o suministrarla a un agente como contexto depurado. El modelo de lenguaje grande (**LLM**) ayuda en esa transformación, siempre que hagamos que la salida sea comprobable por código.

Este capítulo aplica la disciplina de ingeniería de prompts aprendida en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/3) a salidas reales de red. Parsearemos estados de interfaces, resúmenes de BGP (*Border Gateway Protocol*) y salidas multifabricante. Asimismo, abordaremos la gestión de fallos, ya que la primera regla de la automatización operativa es clara: las entradas de datos erróneas ocurren siempre.

Al finalizar este capítulo, comprenderás cómo pasar de texto de red desordenado a datos estructurados utilizables de forma segura por chatbots, herramientas y agentes. El objetivo no es asumir que un LLM es un parser infalible, sino construir un flujo de parseo práctico con validación y recuperación de errores integradas.

En este capítulo cubriremos los siguientes temas:

- Revisar los requisitos técnicos y los archivos de laboratorio de parseo.
- Comprender por qué la salida estructurada es fundamental para la automatización de redes.
- Pasar de texto CLI sin formato a JSON validado.
- Parsear salidas de interfaces en campos estructurados.
- Parsear resúmenes BGP para evaluar el estado de los vecinos (*neighbors*).
- Normalizar salidas de interfaces de múltiples fabricantes.
- Gestionar salidas malformadas y fallos de parseo.
- Decidir cuándo emplear parseo asistido por LLM o parseo determinista.

Estos temas constituyen el puente entre los prompts estructurados y los agentes operativos. Antes de que un agente razone sobre el estado de la red, dicho estado debe representarse en un formato en el que el resto de la aplicación pueda confiar plenamente.

---

### Sección 1: Requisitos técnicos

Antes de ejecutar los ejemplos, verifica que el entorno local del [Capítulo 2](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/2) funcione correctamente: Ollama en ejecución, el modelo `llama3.2:3b` disponible y capacidad de ejecutar scripts de Python desde la raíz del repositorio.

Utilizaremos los siguientes archivos del repositorio:

- `labs/lab1-ollama/json_output_challenge.py`
- `labs/lab1-ollama/challenge_1_interface_parser.py`
- `labs/lab1-ollama/challenge_2_bgp_parser.py`
- `labs/lab1-ollama/challenge_3_multi_vendor.py`
- `labs/lab1-ollama/challenge_4_error_handling.py`
- `labs/lab2-prompts/netmiko_config_parser.py`
- `examples/interface_output.json`
- `examples/bgp_output.json`

Los ejemplos emplean datos simulados (*mock data*) por defecto, por lo que no es necesario el acceso a routers o switches reales. El ejemplo con Netmiko muestra cómo conectar este patrón a dispositivos reales, manteniéndolo de solo lectura y opcional.

---

### Sección 2: Comprender por qué la salida estructurada importa para NetOps

Un humano puede interpretar una salida extensa de CLI: localizar visualmente el nombre de una interfaz, detectar un vecino BGP en estado `Idle` o notar la ausencia de un campo. El código no es tan flexible: si una columna se desplaza un solo espacio, un parser rígido puede fallar; si el modelo devuelve un párrafo en lugar de JSON, el siguiente paso del flujo fallará.

Para las operaciones de red (**NetOps**), la salida estructurada formaliza un **contrato**. En lugar de pedirle al modelo una opinión libre, le exigimos campos específicos en un formato concreto. Esa es la diferencia entre un asistente experimental y un flujo automatizado robusto ante variaciones de entrada.

Antes de programar un parser, conviene definir qué campos se requieren. La **Tabla 4.1** muestra salidas de red habituales y ejemplos de campos estructurados útiles:

| Salida de red | Campos estructurados de interés | Justificación operativa |
| :--- | :--- | :--- |
| **Estado de interfaz** | `interface`, `admin_status`, `oper_status`, `ip_address`, `mtu` | Verificación de enlaces, enriquecimiento de alertas y actualización de inventario |
| **Resumen BGP** | `router_id`, `local_as`, `neighbor`, `state`, `prefixes_received` | Identificación de sesiones caídas y anuncios de rutas vacíos |
| **Errores de interfaz** | `input_errors`, `crc_errors`, `drops`, `last_change` | Diagnóstico de capa física y problemas de congestión |
| **Inventario de dispositivos** | `hostname`, `model`, `version`, `serial_number` | Control de activos y planificación de cambios |
| **Entradas de registro (Logs)** | `timestamp`, `device`, `severity`, `message`, `event_type` | Reconstrucción de cronologías y resumen de incidentes |

> **Tabla 4.1:** Salidas de red comunes y campos estructurados de utilidad.

Cada flujo de trabajo debe definir sus expectativas de antemano. Esa definición previa permite validar los resultados en lugar de asumir que son correctos por su apariencia.

---

### Sección 3: Pasar de texto CLI sin formato a JSON validado

El patrón de parseo aplicado comprende las siguientes etapas:
1. Tomar el texto de red sin procesar.
2. Solicitar al modelo una estructura JSON estricta.
3. Parsear la respuesta del modelo en la aplicación.
4. Validar los campos devueltos.
5. Transferir los datos validados al siguiente componente.

El modelo asume la transformación de texto; la aplicación asume la validación estricta.

> **Figura 4.1:** De salida CLI sin procesar a JSON validado.

Esta separación desacopla la generación del modelo de la confianza en los datos: el modelo sugiere una estructura, pero el código verifica que incluya los campos requeridos, respete los tipos de datos y gestione valores nulos de forma segura. La respuesta del LLM se trata como un resultado candidato, nunca como un dato verificado por defecto.

Los scripts del repositorio encapsulan el prompt en instrucciones específicas para JSON y consultan la API local de Ollama:

```python
def ask_ollama(prompt: str, model: str = "llama3.2:3b")-> dict | None: json_prompt = f"""You are a JSON-only API. Return ONLY valid JSON. No markdown, no explanation, no code fences - just the JSON object. {prompt} Output only valid JSON:""" response = requests.post( "http://localhost:11434/api/generate", json={ "model": model, "prompt": json_prompt, "stream": False, "options": {"temperature": 0.1}, }, timeout=30, ) raw = response.json().get("response", "").strip() return json.loads(raw)
```

La temperatura baja (`0.1`) garantiza una estructura predecible que la siguiente línea de código pueda procesar sin errores. Este ayudante se reutiliza en todos los desafíos del capítulo.

---

### Sección 4: Parseo de salida de interfaces

El estado de las interfaces es un excelente caso inicial: es familiar pero contiene suficientes variaciones para requerir normalización.

El archivo `labs/lab1-ollama/challenge_1_interface_parser.py` procesa una salida típica de Cisco y solicita un objeto JSON con campos concretos. Ejecútalo desde la raíz del repositorio:

```bash
python3 labs/lab1-ollama/challenge_1_interface_parser.py
```

La salida esperada se ilustra a continuación:

```text
============================================================ Challenge 1: Interface Parser ============================================================ Input (raw CLI output): GigabitEthernet0/1 is up, line protocol is up Hardware is iGbE, address is 0000.0c07.ac01 Internet address is 10.0.0.1/24 MTU 1500 bytes, BW 1000000 Kbit/sec Asking Ollama to parse it... ✅ Parsed JSON: { "interface": "GigabitEthernet0/1", "admin_status": "up", "oper_status": "up", "ip_address": "10.0.0.1", "subnet_mask": null, "mac_address": "0000.0c07.ac01", "mtu": 1500 } ✅ All expected fields present!
```

#### Análisis del prompt de interfaz
El prompt define el tipo de salida a procesar y enumera los campos obligatorios:

```text
Return a JSON object with exactly these fields: - interface: string - admin_status: "up" or "down" - oper_status: "up" or "down" - ip_address: string or null - subnet_mask: string or null - mac_address: string - mtu: integer
```

Esta lista delimita el formato objetivo para el modelo y establece los criterios de comprobación para el código.

---

### Sección 5: Parseo de la salida del resumen de BGP

A diferencia de una interfaz individual, el resumen de BGP contiene habitualmente una lista de pares (*neighbors*), cada uno con un estado operativo que puede requerir seguimiento.

El archivo `labs/lab1-ollama/challenge_2_bgp_parser.py` parsea la tabla BGP en un objeto que agrupa la lista de vecinos. Ejecútalo desde la raíz del repositorio:

```bash
python3 labs/lab1-ollama/challenge_2_bgp_parser.py
```

La salida muestra el objeto JSON estructurado y la detección de sesiones no establecidas:

```text
============================================================ Challenge 2: BGP Summary Parser ============================================================ ✅ Parsed JSON: { "router_id": "10.0.0.11", "local_as": 65001, "neighbors": [ { "ip": "10.1.1.1", "remote_as": 65011, "state": "Established", "uptime": "3d02h", "prefixes_received": 150 }, { "ip": "10.1.2.0", "remote_as": 65013, "state": "Idle", "uptime": "0:00:00", "prefixes_received": 0 } ] } 📊 Total neighbors: 2 ⚠️ Non-established sessions: 10.1.2.0 - Idle
```

Con los datos en un objeto Python, el código determinista puede iterar, filtrar y alertar sobre sesiones caídas:

```python
neighbors = result.get("neighbors", []) down = [n for n in neighbors if n.get("state") != "Established"]
```

El LLM asume la conversión del texto irregular; Python asume la lógica determinista de filtrado.

---

### Sección 6: Normalización de salidas multifabricante

Los entornos reales combinan múltiples fabricantes con diferentes formatos de salida:
- Cisco: `GigabitEthernet0/1 is up, line protocol is up`
- Arista: `Ethernet1 is up, line protocol is up (connected)`
- Juniper: `ge-0/0/1.0 up up`

El significado operativo es idéntico, pero la sintaxis varía. El archivo `labs/lab1-ollama/challenge_3_multi_vendor.py` demuestra cómo una misma plantilla de prompt normaliza las salidas de estos tres fabricantes hacia un único esquema JSON.

Ejecútalo desde la raíz del repositorio:

```bash
python3 labs/lab1-ollama/challenge_3_multi_vendor.py
```

Salida esperada:

```text
============================================================ Challenge 3: Multi-Vendor Parser ============================================================ Goal: same JSON shape from three different CLI formats -- Cisco IOS -- Input: GigabitEthernet0/1 is up, line protocol is up Output: {"vendor": "Cisco IOS", "interface": "GigabitEthernet0/1", "admin_status": "up", "oper_status": "up"} -- Arista EOS -- Input: Ethernet1 is up, line protocol is up (connected) Output: {"vendor": "Arista EOS", "interface": "Ethernet1", "admin_status": "up", "oper_status": "up"} -- Juniper JunOS -- Input: ge-0/0/1.0 up up Output: {"vendor": "Juniper JunOS", "interface": "ge-0/0/1.0", "admin_status": "up", "oper_status": "up"} ============================================================ Normalised results (same shape, any vendor): Vendor Interface Admin Oper ------------------------------------------------------------ Cisco IOS GigabitEthernet0/1 up up Arista EOS Ethernet1 up up Juniper JunOS ge-0/0/1.0 up up
```

En lugar de mantener tres parsers independientes basados en expresiones regulares, el modelo mapea las variantes textuales a una misma estructura normalizada.

---

### Sección 7: Validación de la salida estructurada antes de usarla

Obtener JSON no garantiza que los datos sean correctos: pueden faltar campos, contener tipos de datos erróneos o incluir valores inventados.

Como mínimo, el código debe comprobar la presencia de las claves obligatorias antes de procesar el resultado:

```python
expected = ["interface", "admin_status", "oper_status", "ip_address", "mtu"] missing = [field for field in expected if field not in result] if missing: print(f"Missing fields: {missing}") else: print("All expected fields present!")
```

En entornos de producción se recomienda el uso de librerías de validación estricta de esquemas como **Pydantic**, asegurando tipos y rangos de valores admitidos (por ejemplo, que `admin_status` sea estrictamente `up`, `down` o `unknown`, y no una frase explicativa).

---

### Sección 8: Manejo de salidas malformadas y estados inciertos

Las respuestas de los modelos y los datos de red pueden presentar irregularidades: bloques de código markdown no solicitados, texto introductorio o estados de error en los dispositivos.

El archivo `labs/lab1-ollama/challenge_4_error_handling.py` implementa técnicas de limpieza de formato y control de errores sin provocar caídas (*crashes*) en la aplicación.

> **Figura 4.2:** Manejo de parseos fallidos o estados inciertos.

Ejecuta el laboratorio de gestión de errores desde la raíz del repositorio:

```bash
python3 labs/lab1-ollama/challenge_4_error_handling.py
```

Salida esperada:

```text
============================================================ Challenge 4: Error Handling & Graceful Recovery ============================================================ -- Error-disabled port -- Input: GigabitEthernet0/1 is up, line protocol is down (err-disabled) ✅ {"interface": "GigabitEthernet0/1", "admin_status": "up", "oper_status": "err-disabled", "error_reason": "port-security violation", "warning": null} -- Flapping / unknown -- Input: GigabitEthernet0/2 is up, line protocol is unknown ✅ {"interface": "GigabitEthernet0/2", "admin_status": "up", "oper_status": "unknown", "error_reason": null, "warning": "Last flap 00:00:03 ago"} -- No data available -- Input: % No interface information available ✅ {"interface": null, "admin_status": "unknown", "oper_status": "unknown", "error_reason": null, "warning": "No interface information available"} ============================================================ Key patterns used in ask_ollama(): 1. Strip markdown fences (``` blocks) 2. Grab first {...} block if model adds preamble text 3. Return (result, error) tuple - never raise, never crash 4. Caller decides what to do with a failure
```

La **Tabla 4.2** resume los principales modos de fallo y sus mecanismos de mitigación:

| Modo de fallo | Síntoma observado | Mecanismo de mitigación |
| :--- | :--- | :--- |
| **JSON inválido** | Texto adicional, bloques markdown o llaves rotas | Limpiar delimitadores comunes y reintentar el parseo |
| **Campos ausentes** | Falta `ip_address`, `state` o `interface` | Comprobar claves obligatorias antes de usar la salida |
| **Tipo de dato incorrecto** | `mtu` devuelto como cadena en lugar de entero | Validar los tipos de datos de los campos |
| **Alucinación de valores** | El modelo inventa una interfaz o estado no presente | Comparar contra los datos de origen siempre que sea posible |
| **Salida ambigua del equipo** | Mensajes de error, salidas vacías o desconocidas | Representar la incertidumbre explícitamente (`null` / `unknown`) |
| **Error de red o timeout** | Fallo en la llamada a Ollama o en la conexión | Devolver un error controlado y continuar la ejecución |

> **Tabla 4.2:** Modos de fallo en parseo y salvaguardas operativas.

---

### Sección 9: Conectar el parseo a datos de dispositivos en vivo con precaución

El archivo `labs/lab2-prompts/netmiko_config_parser.py` demuestra la conexión con equipos reales vía **SSH** mediante Netmiko, manteniendo el control de seguridad a través de la bandera `USE_MOCK`:

```python
USE_MOCK = True # Flip to False with real devices after the workshop
```

- Con `USE_MOCK = True`, el script utiliza datos locales simulados.
- Con `USE_MOCK = False`, se conecta a los dispositivos declarados en `DEVICE_CONFIG`.

El principio de diseño separa la recolección de datos del parseo: la herramienta obtiene los datos, el modelo los estructura y la aplicación los valida, manteniendo siempre un enfoque inicial de solo lectura.

---

### Sección 10: Elección entre parseo asistido por LLM y determinista

El parseo asistido por LLMs no reemplaza al parseo determinista en todos los casos: si un dispositivo entrega JSON nativo o existe una plantilla de **TextFSM** probada y estable, la solución determinista es preferible.

El LLM destaca ante entradas heterogéneas, salidas semiestructuradas, diferencias entre fabricantes o texto mezclado con lenguaje natural. La **Tabla 4.3** orienta la elección técnica:

| Escenario técnico | Enfoque recomendado | Justificación |
| :--- | :--- | :--- |
| **El equipo entrega JSON nativo** | Parseo determinista | Los datos ya están estructurados en origen |
| **Salida conocida con formato invariable** | TextFSM, regex o parser dedicado | El texto predecible debe procesarse de forma determinista |
| **Múltiples fabricantes con sintaxis dispar** | Parseo asistido por LLM con validación | El modelo normaliza formatos heterogéneos |
| **Notas de incidentes, tickets o logs mixtos** | Extracción asistida por LLM | La entrada combina lenguaje natural y datos técnicos |
| **Decisiones críticas de seguridad o auditoría** | Validación determinista + revisión humana | El flujo exige alta certeza y trazabilidad formal |

> **Tabla 4.3:** Criterios de elección entre parseo determinista y asistido por LLM.

Los sistemas más eficaces combinan ambos mundos: el LLM procesa y normaliza el lenguaje no estructurado, mientras que el código determinista valida, filtra y ejecuta la lógica operativa.

---

### Sección 11: Comprobación de la realidad en producción

En producción, la salida del LLM debe tratarse como no confiable hasta ser validada. Se deben registrar: la entrada cruda, el prompt, la respuesta textual del modelo, el JSON parseado y el resultado de la validación.

Buenas prácticas indispensables:
- Preservar la entrada cruda para auditoría y depuración.
- Solicitar la estructura JSON más compacta y necesaria.
- Validar claves requeridas y tipos admitidos.
- Representar valores desconocidos de forma explícita (`null` o `unknown`) en lugar de deducirlos.
- Aplicar fallo seguro (*fail closed*) ante discrepancias en la validación.
- Registrar las respuestas del modelo antes y después de la limpieza de formato.
- Usar comprobaciones deterministas en puntos críticos de decisión.

---

### Sección 12: Resumen

En este capítulo convertimos salidas no estructuradas de red en datos organizados y analizables. Procesamos estados de interfaces, resúmenes BGP y salidas multifabricante mediante llamadas a LLMs locales con salidas restringidas a JSON.

Se reforzó la frontera esencial: el modelo asiste en la interpretación textual, pero la aplicación asume la validación formal de los datos.

El parseo de solo lectura constituye el punto de partida ideal para la integración de IA en redes, aportando valor operativo inmediato con mínimo riesgo.

En el siguiente capítulo construiremos sobre esta base para desarrollar un chatbot de red con memoria conversacional, capaz de retener el contexto a lo largo de un proceso completo de diagnóstico.
