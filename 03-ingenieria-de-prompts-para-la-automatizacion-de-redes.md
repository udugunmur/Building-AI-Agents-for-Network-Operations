# Parte 1: Fundamentos y Conceptos Clave

## Capítulo 3: Ingeniería de Prompts para la Automatización de Redes

La mayoría de los fallos en la automatización de redes no comienzan por malas intenciones. Comienzan con una solicitud imprecisa. Alguien le pide a un modelo que analice una configuración, resuelva un problema de red o resuma una alerta, y el modelo responde con algo que parece útil a primera vista. Sin embargo, el flujo de trabajo se interrumpe después porque la salida es demasiado locuaz (*chatty*), los campos son inconsistentes o el modelo realizó una suposición apresurada que en realidad no estaba presente en los datos.

Ahí es donde la ingeniería de prompts se vuelve práctica. No estamos intentando redactar frases ingeniosas para un chatbot. Estamos intentando dotar a un modelo de lenguaje grande (**LLM**) de la estructura suficiente para comportarse de manera predecible dentro de un flujo de operaciones de red (**NetOps**). El prompt se convierte en una parte integral del diseño del sistema.

Para el trabajo en redes, esto significa que el prompt debe indicar al modelo qué rol desempeña, qué ejemplos debe seguir, qué contexto es relevante y qué formato de salida espera el resto del flujo de trabajo. Si la salida debe alimentar a un script, no puede ser un párrafo elegante: necesita estar estructurada, restringida y ser fácil de validar.

En este capítulo desarrollaremos esa disciplina utilizando el marco de prompts **RACE** (*Role, Anchors, Context, and Expected output* — Rol, Anclas, Contexto y Salida esperada). El objetivo es directo: transformar prompts vagos en plantillas reutilizables que generen salidas útiles y verificables para operaciones de red.

En este capítulo cubriremos los siguientes temas:

- Revisar los requisitos técnicos para los laboratorios de ingeniería de prompts.
- Comprender por qué la ingeniería de prompts es crucial para la automatización de redes.
- Aplicar el marco de prompts RACE a tareas de NetOps.
- Comparar prompts vagos con prompts estructurados.
- Construir un prompt para parsear configuraciones.
- Validar la salida del modelo antes de utilizarla en flujos de trabajo.
- Crear plantillas de prompts reutilizables para tareas operativas habituales.
- Resolver los fallos más comunes en prompts (*troubleshooting*).

Al finalizar este capítulo, dispondrás de una metodología práctica para diseñar prompts capaces de respaldar tareas de parseo, triaje, documentación y futuros flujos agénticos. Esto es fundamental, ya que cada agente que desarrollemos posteriormente dependerá de la calidad de las instrucciones, ejemplos, contexto y reglas de salida que le proporcionemos.

---

### Sección 1: Requisitos técnicos

Este capítulo se basa en la configuración del entorno local desarrollada en el [Capítulo 2](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/2). Debes contar con Ollama instalado, el modelo `llama3.2:3b` descargado localmente y el repositorio del libro clonado en tu equipo.

Utilizaremos los siguientes archivos del repositorio:

- `labs/lab2-prompts/prompt_engineering_race.py`
- `labs/lab2-prompts/PROMPT_TEMPLATES.md`
- `labs/lab2-prompts/netmiko_config_parser.py`
- `prompts/bad_prompt.txt`
- `prompts/race_network_analysis_prompt.txt`

Los ejemplos de ingeniería de prompts realizan llamadas locales al modelo mediante Ollama. Los datos de red están simulados (*mocked*) por defecto, por lo que no requieres acceso a routers o switches reales para completar este capítulo. Si más adelante deseas conectarte a dispositivos reales, el archivo opcional `netmiko_config_parser.py` muestra dicho patrón, pero mantendremos el laboratorio principal en modo de solo lectura y basado en datos simulados por ahora.

Desde la raíz del repositorio, ejecuta el laboratorio principal de ingeniería de prompts con este comando:

```bash
python3 labs/lab2-prompts/prompt_engineering_race.py
```

Si tu repositorio local no se ha actualizado con los nombres definitivos basados en RACE, utiliza el archivo equivalente del Laboratorio 2 presente en la misma carpeta. El libro utiliza la nomenclatura RACE de manera consistente para mantener alineados el marco y la terminología del capítulo.

---

### Sección 2: Comprender por qué la ingeniería de prompts importa

Un prompt no es simplemente una pregunta. En un flujo de automatización, un prompt actúa como la **interfaz** entre tu código, los datos de tu red y el modelo. Si esa interfaz es imprecisa, la salida será imprecisa. Si esa interfaz está estructurada, el modelo tendrá muchas más probabilidades de generar un resultado utilizable por tu flujo de trabajo.

Esto es especialmente crítico en operaciones de red (**NetOps**) porque los datos de entrada suelen ser irregulares:
- La salida de la interfaz de línea de comandos (**CLI**) puede ser específica de un fabricante determinado.
- Un mensaje de log puede estar incompleto.
- Un ticket puede contener lenguaje coloquial de usuario en lugar de detalles técnicos precisos.
- Un fragmento de configuración puede carecer del contexto circundante necesario.

El modelo puede ayudar a interpretar estas entradas, pero solo si le indicamos en qué enfocarse y qué no debe inventar.

He aquí un ejemplo de un prompt que parece inofensivo pero genera problemas rápidamente:

```text
Parse this config and tell me what is wrong.
```

El modelo puede responder con un párrafo narrativo, una lista de viñetas, una suposición sobre la causa raíz o incluso un script de ejemplo para parsear. Ninguna de esas respuestas es intrínsecamente incorrecta en una conversación humana, pero no resulta fiable para la automatización. Si el paso siguiente requiere **JSON**, este tipo de prompt es completamente insuficiente.

Antes de redactar un prompt para producción, necesitamos un marco que haga predecible el comportamiento del modelo: aquí es donde entra **RACE**.

---

### Sección 3: Introducción al marco de prompts RACE

El marco **RACE** proporciona una estructura sencilla para construir prompts eficaces en la automatización de redes. No es magia ni sustituye a las pruebas: es una lista de verificación que ayuda a evitar los errores de formulación más comunes.

Los cuatro componentes se detallan en la **Tabla 3.1**:

| Elemento RACE | Función | Ejemplo en redes |
| :--- | :--- | :--- |
| **Rol (*Role*)** | Define la perspectiva o experiencia que el modelo debe adoptar | *Actúa como un ingeniero de automatización de redes revisando la salida de una interfaz* |
| **Anclas (*Anchors*)** | Proporciona ejemplos o patrones concretos que el modelo debe seguir | *Suministrar una entrada CLI de muestra y la estructura JSON exacta esperada* |
| **Contexto (*Context*)** | Aporta los datos de red relevantes, restricciones y detalles del entorno | *Indicar al modelo que la salida proviene de Arista EOS y que la tarea es de solo lectura* |
| **Salida esperada (*Expected output*)** | Establece el formato de respuesta, campos obligatorios y reglas | *Devolver un único objeto JSON, usar null para valores ausentes y no explicar la respuesta* |

> **Tabla 3.1:** El marco de prompts RACE para automatización de redes.

Con la estructura definida, examinemos cómo influye cada parte en el comportamiento del modelo en la práctica.

#### Definición del rol (*Role*)
El rol establece la perspectiva operativa del modelo. En el ámbito de redes, esto implica otorgarle una identidad técnica específica: ingeniero de automatización de redes, ingeniero senior de red, analista del Centro de Operaciones de Red (**NOC**), analista de seguridad o asistente de documentación.

Un rol débil se formula así:

```text
You are helpful.
```

Un rol sólido se formula así:

```text
You are a network automation engineer extracting structured facts from device CLI output.
```

La segunda versión no es más larga por capricho; delimita de forma precisa el comportamiento del modelo. No solicitamos una asistencia genérica, sino una extracción estructurada a partir de datos técnicos de red.

#### Incorporación de anclas (*Anchors*)
Las anclas son ejemplos representativos de entradas y salidas que ilustran al modelo el patrón exacto que debe reproducir. Los ejemplos suelen ser más determinantes que las instrucciones aisladas, ya que proporcionan una referencia concreta.

Por ejemplo, si deseamos convertir la salida de una interfaz a JSON, el prompt debe incluir una entrada corta de muestra y su correspondiente salida JSON formateada. Esto especifica los nombres de los campos, los tipos de datos y la jerarquía esperada, reduciendo la variabilidad entre distintas ejecuciones.

Las anclas resultan indispensables con modelos locales compactos, que tienden a interpretar las directrices de forma muy literal. Si solo describes el formato, el modelo puede interpretarlo con ligeras variaciones; si le muestras el formato exacto, seguirá el patrón.

#### Provisión de contexto (*Context*)
El contexto comprende la información operativa indispensable para responder correctamente: fabricante, plataforma, comando ejecutado, topología, función del dispositivo, severidad o restricciones operativas.

El contexto debe ser conciso y focalizado: no vuelques la configuración completa de la red solo porque la ventana de contexto lo permita. Si la tarea consiste en parsear una interfaz, proporciona la salida de esa interfaz y el esquema. Si la tarea es triar una alerta, suministra la alerta, el rol del equipo y su estado pertinente. El objetivo es **contexto útil**, no todo el contexto.

#### Definición de la salida esperada (*Expected output*)
La sección de salida esperada determina en gran medida el éxito o fracaso de los prompts de automatización. Si el resultado alimentará a un script, define el formato rigurosamente:
- Si requieres JSON, especifica `JSON`.
- Si pueden faltar campos, define si deben completarse con `null`, un array vacío o un objeto de error específico.
- Si no deseas introducciones narrativas ni explicaciones, indícalo expresamente.

Una sección de salida esperada práctica se estructura de la siguiente manera:

```text
EXPECTED OUTPUT: Return one JSON object only. Do not use markdown fences. Do not explain your answer. Use null for missing values. Do not invent values that are not present in the input.
```

No se trata de ser tajante con el modelo, sino de redactar restricciones que garanticen una salida segura para la siguiente fase del flujo de trabajo.

El flujo completo de **RACE** se sintetiza en la **Figura 3.1**, mostrando cómo transforma una petición vaga en un prompt estructurado y validable:

> **Figura 3.1:** Marco de prompts RACE para automatización de redes.

Todos los prompts que desarrollaremos en este capítulo aplicarán este mismo principio: definir el rol, anclar el comportamiento, suministrar el contexto idóneo y especificar la salida esperada.

---

### Sección 4: Comparación de prompts vagos con prompts estructurados

La vía más directa para comprobar la importancia de la estructura es contrastar un prompt impreciso con un prompt diseñado bajo el marco RACE. El prompt impreciso puede generar un texto legible para humanos, pero el prompt estructurado está concebido para integrarse en una automatización.

El repositorio incluye un ejemplo de prompt deficiente en `prompts/bad_prompt.txt`:

```text
Parse this config and tell me what is wrong.
```

Este prompt carece de rol, no contiene ejemplos ni contexto operativo, y omite el formato de respuesta esperado. El modelo se ve obligado a deducir la intención del usuario.

La **Tabla 3.2** compara ambas aproximaciones:

| Calidad del prompt | Contenido del prompt | Resultado probable |
| :--- | :--- | :--- |
| **Prompt vago** | *Parse this config* | Explicación en texto libre, campos inconsistentes, posibles suposiciones infundadas |
| **Prompt estructurado (RACE)** | Rol, ejemplos, contexto, esquema JSON y reglas de salida | JSON consistente, fácilmente procesable por código y validable mediante esquemas |

> **Tabla 3.2:** Comparación entre prompts vagos y prompts estructurados bajo RACE.

---

### Sección 5: Construcción de un prompt de parseo de configuración

Construyamos el primer prompt práctico del capítulo: un parseador de configuraciones. El objetivo es procesar la salida cruda de una interfaz y generar un objeto JSON con campos normalizados: nombre de interfaz, estado administrativo, estado operativo, dirección IP, longitud de prefijo, dirección MAC y MTU.

El laboratorio define el siguiente esquema (**schema**) que la salida debe satisfacer estrictamente:

```python
INTERFACE_SCHEMA = { "type": "object", "properties": { "interface": {"type": "string"}, "admin_status": {"type": "string", "enum": ["up", "down"]}, "oper_status": {"type": "string", "enum": ["up", "down"]}, "ip_address": {"type": ["string", "null"]}, "prefix_length": {"type": ["integer", "null"]}, "mac_address": {"type": ["string", "null"]}, "mtu": {"type": ["integer", "null"]} }, "required": [ "interface", "admin_status", "oper_status", "ip_address", "prefix_length", "mac_address", "mtu" ], "additionalProperties": False }
```

El valor del esquema reside en proporcionar una base formal para la validación: si el modelo omite un campo, altera un tipo de dato o introduce un estado no contemplado, el código lo detectará antes de propagar el dato erróneo aguas abajo.

#### Redacción del prompt RACE
El prompt RACE para este parseador articula los cuatro pilares: define el rol, presenta un ejemplo, suministra el texto crudo de la interfaz y establece las reglas de salida en JSON:

```text
You are a JSON extraction engine for network automation data. ROLE: Extract facts from network interface CLI output so another automation workflow can consume the result. ANCHORS: Example input: GigabitEthernet0/1 is up, line protocol is up Hardware is iGbE, address is 0000.0c07.ac01 Internet address is 10.0.0.1/24 MTU 1500 bytes Example output: { "interface": "GigabitEthernet0/1", "admin_status": "up", "oper_status": "up", "ip_address": "10.0.0.1", "prefix_length": 24, "mac_address": "0000.0c07.ac01", "mtu": 1500 } CONTEXT: Parse the interface output provided below. Only use facts present in the input. EXPECTED OUTPUT: Return one JSON object only. Do not write Python code. Do not use markdown fences. Use null for missing values. Do not invent values. NOW PARSE THIS CONFIG: {config_text}
```

El prompt es explícito y directo: establece la tarea, el patrón, los datos de entrada y las restricciones de formato.

#### Ejecución del laboratorio de ingeniería de prompts
Ejecuta el archivo principal del laboratorio desde la raíz del repositorio:

```bash
python3 labs/lab2-prompts/prompt_engineering_race.py
```

El script ejecuta primero el prompt vago y posteriormente el prompt estructurado con RACE. La salida esperada se ilustra a continuación:

```text
RACE Prompt Engineering Workshop ====================================================================== Framework: R - Role A - Anchors C - Context E - Expected output Goal: Show why vague prompts fail and structured prompts work better for network automation use cases. Config Parser Test ====================================================================== BAD PROMPT ---------------------------------------------------------------------- Prompt: Parse this config LLM Result: This appears to be interface output... GOOD PROMPT USING RACE ---------------------------------------------------------------------- Prompt length: 2100 characters LLM Result: { "interface": "GigabitEthernet0/2", "admin_status": "down", "oper_status": "down", "ip_address": null, "prefix_length": null, "mac_address": "0000.0c07.ac02", "mtu": 1500 } Evaluation ---------------------------------------------------------------------- Valid JSON detected. JSON passed schema validation.
```

El prompt RACE genera un JSON estructurado que supera satisfactoriamente la validación contra el esquema.

#### Análisis del funcionamiento del laboratorio
El laboratorio opera en tres fases: construye el prompt, invoca la API local de Ollama y extrae/valida el JSON resultante.

La función de invocación al modelo se estructura así:

```python
def call_llm(prompt, model="llama3.2:3b", temperature=0.0, timeout=60): payload = { "model": model, "prompt": prompt, "stream": False, "options": { "temperature": temperature } } response = requests.post( "http://localhost:11434/api/generate", json=payload, timeout=timeout ) response.raise_for_status() return response.json().get("response", "").strip()
```

La temperatura fijada en `0.0` es deliberada: para tareas de extracción estructurada, la repetibilidad y la consistencia prevalecen sobre la creatividad.

---

### Sección 6: Validación de la salida del modelo

Un prompt riguroso mitiga riesgos, pero no los elimina por completo. La validación debe formar parte del código: si la salida alimenta a otro script, panel, ticket o paso de un agente, la aplicación debe certificar la validez de la estructura antes de procesarla.

El laboratorio implementa una función de extracción y validación de JSON:

```python
parsed = extract_json(good_result) if parsed is None: print("Could not parse the LLM response as JSON") return errors = validate_interface_json(parsed) if errors: print("JSON parsed, but validation found issues") else: print("JSON passed schema validation")
```

Este flujo define el patrón de trabajo continuo:
1. Enviar el prompt al modelo.
2. Parsear el resultado obtenido.
3. Validar la estructura contra el esquema.
4. Continuar la ejecución únicamente si la validación es exitosa.

El diseño de prompts es un proceso iterativo. La **Figura 3.2** ilustra el ciclo de refinamiento continuo:

> **Figura 3.2:** Ciclo de pruebas y refinamiento de prompts.

El método es sistemático: partir de datos reales, definir la salida esperada, probar el prompt, analizar las discrepancias y refinar las instrucciones.

---

### Sección 7: Creación de plantillas de prompts reutilizables

Cuando un prompt demuestra eficacia, debe tratarse como código: guardarse, versionarse, probarse y reutilizarse. El archivo `labs/lab2-prompts/PROMPT_TEMPLATES.md` actúa como biblioteca inicial de plantillas.

Incluye tres patrones principales:
1. **Parseador de configuraciones:** transforma salidas de dispositivos en JSON normalizado.
2. **Triaje de alertas:** clasifica la severidad y propone acciones operativas inmediatas.
3. **Evaluación de riesgo de cambios:** califica el nivel de riesgo de una modificación y fundamenta los motivos.

#### Creación de un prompt para triaje de alertas
El triaje de alertas suele partir de información parcial. El prompt debe clasificar la severidad, explicar la causa y sugerir comprobaciones técnicas sin realizar suposiciones injustificadas:

```text
ROLE: You are a Network Operations Center analyst triaging network alerts. ANCHORS: Severity must be one of: critical, high, medium, low, false_positive. Next actions must be specific and operational. CONTEXT: Use only the alert text and provided device context. Do not invent business impact. EXPECTED OUTPUT: Return JSON with severity, reason, next_actions, and escalate. Use two to five next actions. TRIAGE THIS ALERT: {alert_text}
```

Este prompt proporciona al ingeniero un punto de partida depurado y fundamentado para iniciar la investigación.

#### Creación de un prompt de documentación
Los prompts de documentación son especialmente eficaces cuando la entrada posee cierta estructura previa (datos de topología, inventario, salidas de comandos y notas de incidentes).

El prompt puede solicitar una síntesis global, el listado de dispositivos, el resumen de VLANs o relaciones de enrutamiento y notas de diagnóstico. Es fundamental restringir al modelo a los datos suministrados para evitar que deduzca componentes que no existen en la topología real.

---

### Sección 8: Manejo de fallos comunes en prompts

Los fallos en los prompts responden a patrones recurrentes. Identificar el síntoma permite aplicar la corrección adecuada. La **Tabla 3.3** recopila los problemas habituales y sus soluciones prácticas:

| Modo de fallo | Síntoma observado | Corrección práctica |
| :--- | :--- | :--- |
| **Salida excesivamente discursiva** | El modelo añade explicaciones previas a los datos | Añadir en la salida esperada: *Return JSON only* |
| **Inconsistencia en nombres de campos** | El modelo devuelve `interface_name` en una ejecución e `intf` en otra | Suministrar un esquema estricto y un ejemplo de salida |
| **Generación de datos inexistentes** | El modelo rellena datos ausentes con valores supuestos | Indicar: *Use null for missing values and do not invent facts* |
| **Contexto de fabricante erróneo** | El modelo aplica sintaxis de Cisco a una salida de Arista | Declarar explícitamente el fabricante y la plataforma en el contexto |
| **Comportamiento no determinista** | El mismo prompt produce estructuras dispares en ejecuciones sucesivas | Reducir la temperatura a `0.0` e incorporar anclas (*anchors*) |

> **Tabla 3.3:** Modos de fallo habituales en prompts y correcciones prácticas.

---

### Sección 9: Extensión de prompts a datos de dispositivos en vivo

El desarrollo con datos simulados garantiza seguridad y repetibilidad. El archivo opcional `labs/lab2-prompts/netmiko_config_parser.py` ilustra cómo aplicar las mismas plantillas de prompts a datos obtenidos mediante **SSH** utilizando la librería **Netmiko**.

El script implementa la bandera `USE_MOCK` para alternar entre el modo simulado y la conexión a equipos reales:

```python
USE_MOCK = True
```

- Con `USE_MOCK = True`, el script emplea fragmentos de configuración locales.
- Con `USE_MOCK = False`, se conecta a los dispositivos configurados en `DEVICE_CONFIG`.

Este desacoplamiento separa el diseño del prompt de la recolección de datos: el prompt procesa la entrada y aplica las reglas de formato con independencia del origen de los datos.

---

### Sección 10: Comprobación de la realidad en producción

La ingeniería de prompts para entornos de producción exige tratar las instrucciones como activos de software sujetos a control de versiones, pruebas de regresión y validaciones automáticas.

Un prompt preparado para producción debe responder con claridad:
1. ¿Qué rol específico debe asumir el modelo?
2. ¿Qué ejemplos fijan el patrón esperado?
3. ¿Qué contexto es indispensable y cuál debe excluirse?
4. ¿Qué formato exacto debe tener la salida?
5. ¿Cómo se validará el resultado obtenido?
6. ¿Qué acción ejecutará el sistema si la validación falla?

Los prompts deben residir en archivos versionados, someterse a revisión de cambios y evaluarse ante casos límite para garantizar estabilidad operativa.

---

### Sección 11: Resumen

En este capítulo evolucionamos desde consultas abiertas hacia el diseño formal de prompts estructurados, donde la predictibilidad de las salidas es un requisito indispensable para la automatización.

Aplicamos el marco **RACE** (*Role, Anchors, Context, Expected output*) para construir prompts de parseo de configuración, triaje de alertas y documentación, complementándolos con mecanismos de validación contra esquemas JSON.

El principio central permanece: los prompts forman parte del código del sistema. Deben diseñarse con precisión, probarse con datos heterogéneos, validarse estrictamente y versionarse con rigor.

En el próximo capítulo aplicaremos estos fundamentos directamente sobre salidas crudas de red, parseando estados de interfaces, tablas BGP y comandos multifabricante en estructuras listas para alimentar a futuros agentes autónomos.
