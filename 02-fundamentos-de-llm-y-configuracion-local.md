# Parte 1: Fundamentos y Conceptos Clave

## Capítulo 2: Fundamentos de LLM y Configuración Local

La mayoría de los ingenieros de red no necesitan convertirse en investigadores de inteligencia artificial (**IA**). Ese no es su trabajo. Su trabajo consiste en comprender lo suficiente sobre los modelos de lenguaje grandes (**LLMs**) para utilizarlos de manera segura, predecible y práctica en las operaciones de red (**NetOps**).

Si ya has utilizado un chatbot, has visto el lado útil de los LLMs. Pueden resumir registros (*logs*), explicar protocolos, redactar scripts y ayudar a razonar sobre textos complejos o desordenados. Pero si también has observado que el mismo prompt produce respuestas ligeramente diferentes, o has visto a un modelo generar con total seguridad una salida en el formato incorrecto, también has experimentado la parte que genera incertidumbre en los ingenieros.

Esto es crucial para NetOps porque nuestros casos de uso no son simples preguntas casuales. Queremos parsear salidas de comandos, resumir incidentes, construir asistentes de diagnóstico y, eventualmente, permitir que un agente invoque herramientas. Antes de hacer nada de eso, necesitamos un modelo mental operativo sobre cómo se comportan estos sistemas.

En este capítulo, mantendremos la teoría en un plano práctico. Analizaremos tokens, ventanas de contexto, temperatura, llamadas sin estado (*stateless*) y memoria. A continuación, configuraremos **Ollama** para poder ejecutar un modelo local e invocarlo desde Python. Al finalizar, dispondrás de un laboratorio local preparado para el resto del libro.

En este capítulo cubriremos los siguientes temas:

- Revisar los requisitos técnicos y los supuestos del laboratorio local.
- Explicar cómo se comportan los LLMs en las tareas de automatización de red.
- Comprender los tokens, las ventanas de contexto y las llamadas a interfaces de programación de aplicaciones (**APIs**) sin estado.
- Controlar el comportamiento del modelo mediante la temperatura y los límites de respuesta.
- Configurar Ollama y ejecutar un modelo local.
- Invocar un modelo local desde Python.
- Probar variaciones de temperatura y el comportamiento básico de los tokens.
- Resolver problemas comunes de configuración local (*troubleshooting*).

Estos temas proporcionan los cimientos necesarios antes de abordar la ingeniería de prompts, el parseo, la memoria y la invocación de herramientas. Si el [Capítulo 1](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/1) explicó por qué son importantes los agentes, este capítulo explica el comportamiento del modelo que debemos respetar al construirlos.

---

### Sección 1: Requisitos técnicos

Antes de ejecutar los ejemplos, asegúrate de tener instaladas las herramientas básicas. La vía rápida se detalla en `QUICKSTART.md`. Este capítulo sigue la misma ruta de instalación, pero explica por qué es importante cada componente.

Necesitas las siguientes herramientas:

- **Python 3.10** o superior.
- **Git** para clonar el repositorio.
- **Ollama** para ejecutar el modelo local.
- **Visual Studio Code (VS Code)** u otro editor de código.
- Las dependencias del repositorio listadas en `requirements.txt`.

No necesitas una cuenta en la nube ni una clave de API (*API key*) para este capítulo. Todo lo que hacemos aquí se ejecuta localmente una vez descargado el modelo.

---

### Sección 2: Comprender por qué los fundamentos de LLM son importantes para NetOps

Muchas explicaciones sobre LLMs profundizan excesivamente en la teoría del aprendizaje automático (*machine learning*) o se quedan en un nivel demasiado abstracto para ser útiles. Ninguno de esos extremos ayuda al intentar construir un flujo de trabajo para resolución de problemas de red. Necesitamos un punto medio: comprensión suficiente para tomar decisiones de ingeniería acertadas.

Para el trabajo en redes, el aspecto clave es que los LLMs son excelentes reconociendo patrones en texto. Esto los hace valiosos para leer logs, resumir tickets, generar borradores iniciales de documentación y transformar salidas de comandos desestructuradas en datos organizados. Esa misma fortaleza también introduce riesgos: un modelo puede generar texto fluido y convincente que parezca correcto a simple vista, pero que sea erróneo, incompleto o con un formato defectuoso.

Por esa razón tratamos al modelo como un componente dentro de un sistema más amplio. Puede asistir con el lenguaje y el razonamiento, pero la aplicación sigue requiriendo validación, límites en las herramientas, gestión de errores y supervisión humana cuando corresponda. Esta es la misma mentalidad adoptada en el [Capítulo 1](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/1): útil, pero bajo control.

Antes de escribir prompts o construir agentes, debemos entender cómo el modelo recibe entradas, genera salidas y olvida el contexto entre llamadas. Esto comienza con una pregunta directa: ¿qué hace realmente un LLM?

#### Comprensión de lo que es un modelo de lenguaje grande
Un modelo de lenguaje grande es un modelo entrenado para predecir el texto más probable en función de los patrones observados durante su entrenamiento. Esa es la definición en términos sencillos. No contiene una tabla de enrutamiento en su interior ni conoce el estado de tu red en tiempo real. Genera los siguientes fragmentos de texto más probables a partir de la entrada que le proporcionas y los patrones que ha aprendido.

Por eso los LLMs resultan tan útiles en tareas de red. Si introduces la explicación de un protocolo de enrutamiento, un fragmento de log o la salida de un comando, es muy probable que el modelo haya visto patrones similares anteriormente. Puede reconocer la estructura, extraer valores, sintetizar significados y explicar qué representa probablemente dicha información.

La advertencia práctica es igualmente importante: el reconocimiento de patrones no equivale a la verdad determinista. Un LLM puede generar una respuesta de apariencia válida sin haberla contrastado contra tu entorno real. Por eso lo empleamos para tareas como parseo, síntesis y asistencia, manteniendo siempre mecanismos de verificación sobre el resultado.

Esta distinción ahorra horas de depuración. La **Tabla 2.1** resume dónde encajan los LLMs en NetOps y dónde el sistema circundante debe asumir el control:

| Tipo de tarea | Dónde ayuda el LLM | Dónde requiere el sistema barreras de seguridad (Guardrails) |
| :--- | :--- | :--- |
| **Resumen de logs** | Condensa logs ruidosos en un resumen legible | Validar datos críticos contra los datos de origen |
| **Parseo de salida de comandos** | Extrae valores de texto semiestructurado | Verificar la salida contra un esquema (*schema*) antes de usarla |
| **Borrador de documentación** | Convierte notas de topología o incidentes en texto legible | Revisar la precisión antes de publicar |
| **Generación de configuraciones** | Crea un punto de partida o plantilla | Nunca desplegar configuración generada sin revisión y pruebas |
| **Soporte en resolución de problemas** | Sugiere siguientes comprobaciones basadas en evidencias | Exigir evidencias y descartar conclusiones sin sustento |

> **Tabla 2.1:** Dónde ayudan los LLMs en NetOps y dónde siguen siendo indispensables las barreras de seguridad.

El patrón es consistente: utilizamos el modelo donde es fuerte y lo encapsulamos con validaciones donde el resultado es crítico. Este enfoque resulta más intuitivo una vez comprendido cómo procesa el texto el modelo.

#### Trabajo con tokens y ventanas de contexto
Los LLMs no procesan texto como los humanos leen palabras. Procesan texto en forma de **tokens**. Un token es un fragmento de texto: a veces es una palabra completa, a veces una subpalabra, y otras veces signos de puntuación o espacios. No es necesario dominar la matemática exacta de los tokens, pero sí saber que cada prompt y cada respuesta consumen tokens.

Esto es relevante porque los datos de red crecen rápidamente. Un comando breve como `show ip interface brief` ocupa poco espacio. Sin embargo, la configuración completa de un equipo, un archivo de log extenso o la salida de comandos en una topología *spine-leaf* grande aumentan de tamaño con gran rapidez. Si a esto se añade el historial de conversación, ejemplos, esquemas e instrucciones, el contexto puede desbordarse antes de lo previsto.

La **ventana de contexto** (*context window*) es la cantidad máxima de texto que el modelo puede procesar a la vez. Funciona como el área de trabajo disponible para la solicitud actual. Si la información útil está dentro de la ventana, el modelo puede utilizarla. Si falta, está truncada o queda sepultada bajo datos irrelevantes, la calidad de la respuesta se degrada.

> **Figura 2.1:** Cómo el texto del prompt se divide en tokens dentro de la ventana de contexto del modelo.

Para NetOps, la regla es clara: no envíes todo solo porque sea posible. Proporciona al modelo el contexto adecuado, no todo el contexto. Más adelante, al diseñar plantillas de prompts y salidas de herramientas, aplicaremos este principio enviando información focalizada en lugar de volcar líneas masivas de logs.

#### Gestión del tamaño de salida y costes
Al ejecutar un modelo local con Ollama, no se paga por token consumido. Esa es una de las grandes ventajas del desarrollo local: permite experimentar libremente, iterar, probar prompts y entender el comportamiento del modelo sin incurrir en costes por uso.

Esto no significa que los tokens dejen de importar. Los prompts grandes requieren más tiempo de procesamiento y las salidas extensas consumen más recursos de cómputo. Si posteriormente trasladas el flujo de trabajo a un modelo en la nube, los tokens afectarán directamente a la facturación. La disciplina desarrollada en local sigue siendo esencial a escala productiva.

Una buena práctica consiste en solicitar únicamente lo necesario:
- Si necesitas **JSON**, solicita estrictamente JSON.
- Si requieres cinco campos, no pidas explicaciones extensas.
- Si la salida alimentará a otra herramienta, mantenla estructurada y lo bastante compacta para que el siguiente paso la valide fácilmente.

Esto conduce al primer parámetro del modelo que la mayoría de los ingenieros ajustan al comenzar a experimentar: la **temperatura**.

#### Control de la temperatura y salidas predecibles
La **temperatura** controla cuán aleatoria o variada puede ser la salida del modelo.
- Una temperatura baja genera salidas más predecibles y deterministas.
- Una temperatura alta permite mayor variabilidad y creatividad.

Para tareas creativas como la redacción o la lluvia de ideas, la variedad es útil. Para la automatización de redes, la variabilidad es precisamente lo que buscamos evitar.

Si solicitas al modelo una breve explicación sobre BGP, una ligera variación no supone un problema. Si le solicitas un objeto JSON con campos concretos, la variabilidad puede romper el flujo: una sola frase añadida antes del bloque JSON basta para invalidar un parser. Por eso, las tareas de red estructuradas utilizan habitualmente temperaturas bajas.

Para tareas operativas de red, parte siempre de salidas predecibles e incrementa la variación solo cuando el caso de uso lo justifique. La **Tabla 2.2** muestra valores de partida recomendados:

| Caso de uso | Temperatura sugerida | Justificación de la configuración |
| :--- | :--- | :--- |
| **Extracción estructurada en JSON** | `0.0` a `0.2` | Mantiene la salida consistente y facilita su validación |
| **Parseo de salida de comandos** | `0.0` a `0.3` | Reduce desviaciones en los nombres de campos y la estructura |
| **Plantillas de configuración** | `0.0` a `0.3` | Aumenta la repetibilidad de la estructura generada |
| **Resúmenes de documentación** | `0.5` a `0.8` | Permite una redacción más natural manteniéndose fiel a los datos |
| **Lluvia de ideas (*brainstorming*)** | `0.8` a `1.2` | Fomenta la variabilidad cuando no se requiere una estructura exacta |

> **Tabla 2.2:** Configuraciones prácticas de temperatura para tareas de operaciones de red.

Lo relevante no es memorizar los valores, sino identificar el objetivo de optimización: si la salida alimenta una automatización, optimiza para consistencia; si la salida es una explicación para humanos, existe margen para lenguaje natural.

#### Comparación del comportamiento de la temperatura
El repositorio incluye un ejemplo de prueba de temperatura en `examples/temperature.py`. El script envía un prompt a la API local de Ollama modificando el parámetro de temperatura para ilustrar los cambios en la respuesta.

El fragmento central es el siguiente:

```python
import requests prompt = "Generate a BGP configuration for AS 65001 with neighbor 10.0.0.1" response = requests.post("http://localhost:11434/api/generate", json={ "model": "llama3.2:3b", "prompt": prompt, "stream": False, "options": { "temperature": 1.5, "num_predict": 200 } }).json() print(response["response"])
```

Ejecuta el ejemplo desde la raíz del repositorio:

```bash
python3 examples/temperature.py
```

La respuesta obtenida variará respecto a otras ejecuciones, especialmente con temperaturas elevadas. Ejecútalo varias veces, reduce la temperatura y compara las diferencias: el objetivo es evaluar el nivel de control que requiere tu flujo de trabajo.

Comprendida la temperatura, el siguiente concepto es aún más crítico para los agentes: el modelo no retiene memoria de llamadas anteriores a menos que la aplicación se la suministre.

#### Comprensión de las llamadas sin estado (*stateless*) y la memoria
Las llamadas a las APIs de LLMs son **sin estado** (*stateless*) por defecto. Cada petición es independiente y no recuerda la anterior. Si preguntas al modelo *¿Qué es BGP?* y en una llamada posterior consultas *¿Qué te acabo de preguntar?*, el modelo no lo sabrá a menos que tu aplicación vuelva a enviarle el mensaje previo.

Esto suele sorprender porque las aplicaciones de chat aparentan recordar. Es la aplicación la que realiza ese trabajo: almacena el historial de conversación y reenvía las partes relevantes al modelo en cada turno. El modelo no mantiene sesiones de diagnóstico abiertas por sí mismo.

> **Figura 2.2:** Comparación entre llamadas a modelos sin estado y aplicaciones con gestión de memoria.

Esta distinción es fundamental para los agentes de red. Un diagnóstico real rara vez consta de una sola interacción: preguntas por un vecino BGP, luego por la interfaz y después por si los logs recientes explican el cambio de estado. Para que el asistente siga ese hilo, la aplicación debe gestionar el historial. Construiremos este patrón en el [Capítulo 5](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/5).

---

### Sección 3: Uso de Ollama para el laboratorio local

Para los ejercicios prácticos de este libro utilizamos **Ollama** como entorno de ejecución local. Ollama permite descargar y ejecutar modelos en tu propio equipo, ofreciendo un entorno seguro, controlado y sin costes antes de pasar a flujos más complejos.

Esto es especialmente relevante para ingenieros de red debido a la sensibilidad de los datos: nombres de dispositivos, topologías, configuraciones y logs contienen información que no debe exponerse en servicios públicos. Los modelos locales no resuelven todos los escenarios de producción, pero constituyen un excelente entorno de desarrollo.

El modelo estándar utilizado en el libro es `llama3.2:3b`, por ser lo bastante ligero para la mayoría de portátiles y ser el empleado en el repositorio. Modelos mayores pueden ofrecer mejores resultados a costa de mayor consumo de memoria y computación. Lo recomendable es dominar el flujo con el modelo base antes de evaluar opciones de mayor tamaño.

Si tu hardware lo permite, puedes probar `llama3.1:8b` para comparar, manteniendo `llama3.2:3b` como referencia para que las salidas coincidan con los ejemplos del libro.

La configuración es independiente del sistema operativo (macOS, Linux y Windows son compatibles). Las instrucciones se muestran desde la terminal.

#### Instalación de Ollama y descarga del modelo
El primer paso es instalar Ollama, iniciar el servicio y descargar el modelo `llama3.2:3b`.

En **macOS** (vía Homebrew):

```bash
brew install ollama ollama pull llama3.2:3b
```

En **Linux**:

```bash
curl -fsSL https://ollama.com/install.sh | sh ollama pull llama3.2:3b
```

En **Windows**, instala Ollama desde su instalador oficial, abre PowerShell y descarga el modelo:

```powershell
ollama pull llama3.2:3b
```

Verifica la instalación del modelo listando los modelos disponibles:

```bash
ollama list
```

La salida debe mostrar `llama3.2:3b` en la lista:

```text
$ ollama list NAME ID SIZE MODIFIED llama3.2:3b a80c4f17acd5 2.0 GB 2 minutes ago
```

Con el modelo disponible, realizamos una prueba desde la línea de comandos antes de escribir código en Python.

#### Ejecución del modelo desde la línea de comandos
Esta comprobación valida si Ollama puede ejecutar el modelo y responder adecuadamente.

Ejecuta el comando:

```bash
ollama run llama3.2:3b
```

Prueba con un prompt sencillo de redes:

```text
Explain BGP route selection in three bullet points
```

El modelo responderá con una breve explicación. En versiones compatibles de Ollama también se puede pasar el prompt directamente como argumento:

```bash
ollama run llama3.2:3b "Explain BGP route selection in three bullet points"
```

Una vez validada la respuesta desde la terminal, procedemos a configurar el entorno en Python.

#### Configuración del repositorio y entorno virtual de Python
Clona el repositorio e instala las dependencias en un entorno virtual:

```bash
git clone https://github.com/PacktPublishing/Building-AI-Agents-for-Network-Operations.git cd Building-AI-Agents-for-Network-Operations python3 -m venv .venv source .venv/bin/activate pip install -r requirements.txt
```

En **Windows PowerShell**:

```powershell
.venv\Scripts\Activate.ps1 pip install -r requirements.txt
```

El entorno virtual aísla las dependencias del proyecto, evitando conflictos entre librerías.

Con las dependencias instaladas, ejecuta el script de comprobación desde la raíz del repositorio:

```bash
python3 examples/test_setup.py
```

El script verifica la versión de Python, comprueba que Ollama esté en ejecución con el modelo `llama3.2:3b` y valida la presencia de los archivos de laboratorio:

```text
$ python3 examples/test_setup.py 🤖 AI Networking Workshop - Setup Test 100% Free with Ollama - No API Keys! ====================================================================== Python Version Check ====================================================================== ✅ Python 3.12.3 ====================================================================== Ollama Check ====================================================================== ✅ Ollama installed ✅ Ollama service running ✅ llama3.2:3b model installed ====================================================================== Lab Files Check ====================================================================== ✅ labs/lab1-ollama/simple_ollama_test.py ✅ labs/lab3-chatbot/chatbot_v2_with_memory.py ✅ labs/lab4-agentic/agentic_network_bot_ollama.py ====================================================================== Summary ====================================================================== ✅ All checks passed! You're ready! Next: python3 labs/lab1-ollama/simple_ollama_test.py
```

Si el script concluye con éxito, el entorno está listo. Si reporta fallos, el mensaje indicará el paso a corregir (por ejemplo, iniciar el servicio de Ollama o descargar el modelo).

---

### Sección 4: Llamadas a Ollama desde Python

El siguiente paso es interactuar con el modelo desde Python. El código de la aplicación debe enviar prompts, recibir respuestas y procesar los resultados.

El archivo principal para esta sección es `labs/lab1-ollama/simple_ollama_test.py`, que utiliza la librería `requests` para invocar la API local de Ollama en `http://localhost:11434/api/generate`:

```python
import requests def chat_with_ollama( prompt, model="llama3.2:3b", temperature=0.7, max_tokens=500 ): url = "http://localhost:11434/api/generate" payload = { "model": model, "prompt": prompt, "stream": False, "options": { "temperature": temperature, "num_predict": max_tokens } } response = requests.post(url, json=payload, timeout=30) response.raise_for_status() data = response.json() return data["response"] answer = chat_with_ollama("Explain OSPF in two sentences") print(answer)
```

Este patrón representa la base de las interacciones posteriores: el código construye la petición, la envía al modelo, recibe el resultado y determina los siguientes pasos.

Ejecuta el script completo desde la raíz del repositorio:

```bash
python3 labs/lab1-ollama/simple_ollama_test.py
```

La salida esperada incluye la respuesta del modelo, el conteo de tokens y el modo interactivo:

```text
$ python3 labs/lab1-ollama/simple_ollama_test.py 🤖 Ollama API Test - AI Networking Workshop ====================================================================== 📝 Test 1: Simple Chat Response: OSPF is a link state routing protocol used inside an autonomous system. Tokens: 48 📝 Test 3: Temperature Effects ====================================================================== 💬 Interactive Mode - Type quit to exit ======================================================================
```

La función `chat_with_ollama()` establece la estructura que se reutilizará en los laboratorios de chatbots, llamadas a herramientas y agentes.

#### Análisis del código Python
El script realiza tres operaciones principales:
1. Construye el payload JSON especificando el modelo y el prompt.
2. Envía la petición POST a la API local de Ollama.
3. Decodifica la respuesta JSON y extrae el texto generado.

El parámetro `stream` se define en `False` para recibir la respuesta completa en un único bloque.

El bloque `options` define los parámetros de generación:
- `temperature`: regula la variabilidad.
- `num_predict`: limita el número máximo de tokens generados.

El código del repositorio incluye además captura de excepciones de conexión para reportar fallos comunes (servicio detenido, modelo ausente o puerto inaccesible) de manera clara.

#### Experimentación con la temperatura desde Python
El archivo `simple_ollama_test.py` implementa una función de comparación de temperatura que envía el mismo prompt con distintos valores:

```python
temperatures = [0.0, 0.7, 1.5] for temp in temperatures: result = chat_with_ollama( "Generate a creative name for a network monitoring tool", model="llama3.2:3b", temperature=temp ) print(f"Temperature {temp}: {result}")
```

Para tareas de extracción estructurada, prueba sustituyendo el prompt por uno más restrictivo:

```text
Return three BGP neighbor states as a JSON array
```

Al comparar ejecuciones con temperatura baja y alta se aprecia la necesidad de utilizar prompts estrictos, esquemas y validación en flujos destinados a automatización.

#### Revisión de tokens en el laboratorio
El archivo `examples/tokens_test.py` ilustra el conteo de tokens utilizando el paquete de Ollama en Python sobre un fragmento de configuración BGP:

```text
router bgp 65001 neighbor 10.0.0.1 remote-as 65002 neighbor 10.0.0.1 description CORE-RTR-01
```

El hábito esencial consiste en controlar el volumen de datos suministrado al modelo. Al transferir salidas de herramientas, logs o historiales de conversación, la disciplina en el uso de tokens es indispensable para el rendimiento y la estabilidad.

---

### Sección 5: Resolución de problemas y consideraciones de producción

El entorno local permite consolidar buenas prácticas de diagnóstico. Ante cualquier fallo, se debe verificar la infraestructura capa por capa antes de modificar el código del agente.

#### Resolución de problemas del entorno local
La **Tabla 2.3** detalla los problemas de configuración más frecuentes y sus comprobaciones iniciales:

| Problema | Causa probable | Primera comprobación |
| :--- | :--- | :--- |
| **Comando `ollama` no encontrado** | Ollama no está instalado o no está en el `PATH` | Reinstalar Ollama y abrir una nueva terminal |
| **No se puede conectar a Ollama** | El servicio de Ollama no está en ejecución | Ejecutar `ollama serve` e intentar de nuevo |
| **Modelo no encontrado** | `llama3.2:3b` no ha sido descargado | Ejecutar `ollama pull llama3.2:3b` |
| **Error de importación en Python** | Faltan dependencias en el entorno virtual | Ejecutar `pip install -r requirements.txt` |
| **Archivos de laboratorio ausentes** | El comando se ejecuta desde un directorio incorrecto | Ejecutar los comandos desde la raíz del repositorio |
| **Respuesta excesivamente lenta** | Limitación de hardware o modelo demasiado grande | Usar `llama3.2:3b` antes de evaluar modelos mayores |

> **Tabla 2.3:** Problemas comunes de configuración local y primeros pasos de diagnóstico.

Ante dudas sobre el estado del entorno, ejecuta siempre `python3 examples/test_setup.py`.

#### Comprobación de la realidad en producción
El uso de modelos locales resulta idóneo para aprendizaje y desarrollo rápido sin costes de API. Sin embargo, en producción los requisitos varían según rendimiento, gobernanza de datos, latencia y precisión.

Algunos entornos emplearán modelos en la nube, otros modelos privados en infraestructura interna y otros un esquema híbrido. Por ello, conviene desacoplar las llamadas al modelo mediante funciones modulares como `chat_with_ollama()`, facilitando la sustitución del backend sin reescribir la lógica del agente.

El principio de diseño es directo: mantener interfaces limpias, validar salidas y asegurar la modularidad de los componentes.

---

### Sección 6: Resumen

En este capítulo analizamos los fundamentos de LLM aplicados a operaciones de red: tokens, ventanas de contexto, temperatura, llamadas sin estado y la necesidad de gestionar la memoria en la capa de aplicación.

Asimismo, instalamos y validamos Ollama con el modelo `llama3.2:3b`, comprobamos su funcionamiento desde la terminal e implementamos llamadas controladas desde Python, dejando listo el laboratorio local.

Los LLMs alcanzan su mayor utilidad cuando reciben contexto preciso, operan bajo parámetros controlados y se integran con código que valida rigurosamente sus respuestas.

En el siguiente capítulo utilizaremos esta base para diseñar mejores prompts de automatización de red, evolucionando de consultas abiertas a estructuras rigurosas orientadas a tareas reales de NetOps.
