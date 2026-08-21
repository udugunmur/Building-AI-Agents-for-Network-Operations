# Parte 1: Fundamentos y Conceptos Clave

## Capítulo 5: Creación de un Chatbot de Red con Memoria

Un chatbot que olvida todo tras una sola pregunta no resulta de gran utilidad durante un diagnóstico. Los problemas de red rara vez se presentan como incidencias aisladas de una sola pregunta y una sola respuesta. Llegan como un rastro sucesivo de pistas: una alerta, el nombre de un dispositivo, la dirección de un vecino, el estado de una interfaz, una entrada de registro (*log*) y, a continuación, una nueva pregunta derivada de lo anterior.

Ahí es donde la memoria cobra verdadera importancia. No nos referimos a memoria en el sentido de la ciencia ficción ni a una funcionalidad oculta dentro del modelo. Para los sistemas que estamos desarrollando, la memoria es un **patrón de diseño de la aplicación**: tu código determina qué conservar, qué reenviar al modelo y qué descartar.

En el [Capítulo 2](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/2) vimos que las llamadas a modelos de lenguaje grandes (**LLMs**) son sin estado (*stateless*) por defecto. En este capítulo traduciremos ese concepto a código Python funcional: ejecutaremos primero un chatbot sin memoria y, posteriormente, construiremos un chatbot con memoria que retenga el historial de la conversación para responder a preguntas de seguimiento.

Este capítulo se mantiene en un entorno estrictamente local y seguro, sin otorgar todavía acceso a la red real. El objetivo es comprender cómo una conversación adquiere estado antes de conectar herramientas, dispositivos simulados y flujos de trabajo agénticos.

En este capítulo cubriremos los siguientes temas:

- Revisar los requisitos técnicos para los laboratorios de chatbots.
- Comprender por qué los chatbots sin estado fallan durante la resolución de problemas.
- Ejecutar un chatbot sin estado con Ollama.
- Añadir memoria gestionada por la aplicación mediante el historial de conversación.
- Construir la clase `NetworkChatbot`.
- Gestionar prompts, historial, reinicio (*reset*) y modo interactivo.
- Controlar el crecimiento del contexto antes de que sature la ventana de trabajo.
- Preparar la memoria del chatbot para la invocación de herramientas y agentes.

Al finalizar este capítulo, dispondrás de un asistente de red local capaz de recordar el contexto conversacional. Este paso es uno de los puentes más determinantes entre un simple prompt y un agente operativo completo.

---

### Sección 1: Requisitos técnicos

Este capítulo utiliza el entorno local configurado en el [Capítulo 2](https://subscription.packtpub.com/book/cloud-and-networking/9781808346835/2). Debes contar con Ollama en ejecución y el modelo `llama3.2:3b` instalado antes de iniciar las prácticas.

Utilizaremos los siguientes archivos del repositorio:

- `labs/lab3-chatbot/chatbot_v1_stateless.py`
- `labs/lab3-chatbot/chatbot_v2_with_memory.py`
- `labs/lab3-chatbot/stateless.MD`
- `labs/lab3-chatbot/memory.MD`

También emplearemos la librería `requests` para invocar la **API** local de Ollama (incluida en `requirements.txt`).

El repositorio contiene además `chatbot_v3_live_ssh.py` y `live_ssh.MD`. Mantendremos el acceso vía **SSH** fuera del flujo principal de este capítulo, tratándolo como una extensión avanzada opcional una vez consolidados los conceptos de memoria y control.

---

### Sección 2: Comprender por qué la memoria del chatbot es importante

Un chatbot básico puede responder a una pregunta aislada. Esto es útil, pero no refleja el proceso real de resolución de problemas. Al investigar una incidencia de red, cada pregunta se apoya en la anterior: consultas por un protocolo, luego por un vecino, después por un dispositivo y finalmente por un mensaje de error. El significado de cada consulta depende directamente del contexto previo.

Un chatbot sin estado no arrastra ese contexto: cada llamada al modelo es independiente. Si la primera petición pregunta sobre **OSPF** (*Open Shortest Path First*) y la segunda consulta *¿Qué te acabo de preguntar?*, el modelo solo ve la segunda pregunta a menos que la aplicación vuelva a enviarle la primera.

Por ello, la memoria reside en la **aplicación**: esta mantiene el historial conversacional, construye un prompt que integra los mensajes anteriores relevantes, envía dicho prompt al modelo y almacena la respuesta para el siguiente turno.

> **Figura 5.1:** Comparación entre llamadas a chatbots sin estado y aplicaciones con gestión de memoria.

La diferencia radica en la ubicación del contexto: en el flujo sin estado, cada turno está aislado; en el flujo con memoria, la aplicación almacena y reenvía el historial, permitiendo al asistente responder con visión global del diagnóstico en curso.

---

### Sección 3: Ejecución de un chatbot sin estado (stateless)

El script `labs/lab3-chatbot/chatbot_v1_stateless.py` evidencia esta limitación. Define la función `simple_chat()` que envía un único mensaje de usuario a Ollama sin incluir el historial:

```python
def simple_chat(user_message: str, model: str = "llama3.2:3b")-> str: """Send single message with NO conversation history.""" url = "http://localhost:11434/api/generate" payload = { "model": model, "prompt": user_message, "stream": False, "options": { "temperature": 0.7 } } response = requests.post(url, json=payload, timeout=30) response.raise_for_status() data = response.json() return data.get("response", "")
```

El campo `prompt` contiene únicamente el mensaje actual del usuario, omitiendo preguntas y respuestas previas.

Ejecuta el script desde la raíz del repositorio:

```bash
python3 labs/lab3-chatbot/chatbot_v1_stateless.py
```

Salida esperada:

```text
🤖 Stateless Chatbot Demo (Ollama) ====================================================================== 👤 User: What is OSPF? 🤖 Bot: OSPF is a link-state routing protocol... 👤 User: What did I just ask you? 🤖 Bot: I do not have access to previous messages... ❌ FAILURE: The bot doesn't remember! Each API call is independent. 💡 Next: See chatbot_v2_with_memory.py for the solution!
```

El modelo se comporta de acuerdo con el código: al no recibir el primer mensaje en la segunda llamada, no tiene forma de conocerlo.

---

### Sección 4: Incorporación de memoria gestionada por la aplicación

El archivo `labs/lab3-chatbot/chatbot_v2_with_memory.py` resuelve este comportamiento almacenando los mensajes en una lista, la cual actúa como la memoria conversacional de la aplicación.

El mecanismo es directo:
1. Cuando el usuario envía un mensaje, la aplicación lo añade al historial.
2. Tras la respuesta del modelo, la aplicación añade la intervención del asistente al historial.
3. En el siguiente turno, la aplicación construye el prompt incorporando todo el historial acumulado.

La estructura de los mensajes utiliza el esquema estándar de rol (`role`) y contenido (`content`):

```python
[ {"role": "user", "content": "What is OSPF?"}, {"role": "assistant", "content": "OSPF is a link-state routing protocol..."} ]
```

> **Figura 5.2:** Construcción de un prompt a partir del historial de conversación.

Al integrar el historial dentro del prompt, el chatbot puede responder coherentemente a preguntas de seguimiento sin requerir bases de datos ni búsquedas vectoriales complejas en esta etapa.

---

### Sección 5: Construcción de la clase NetworkChatbot

El chatbot con estado encapsula su lógica dentro de la clase `NetworkChatbot`, gestionando el modelo, el historial conversacional y el prompt de sistema:

```python
class NetworkChatbot: """Chatbot with conversation memory for network engineering.""" def __init__(self, model: str = "llama3.2:3b"): self.model = model self.conversation_history = [] self.system_prompt = """You are a network engineer assistant. Available devices: - spine1, spine2 (core switches) - leaf1, leaf2 (access switches) Provide accurate, concise answers about networking."""
```

El inicializador define:
- El modelo base (`llama3.2:3b`).
- Una lista vacía para `conversation_history`.
- Un `system_prompt` que orienta al modelo hacia un rol de asistente de ingeniería de redes con un inventario básico de switches conocidos.

#### Implementación del método chat
El método `chat()` gestiona el ciclo completo de cada interacción:

```python
def chat(self, user_message: str)-> str: """Send message with full conversation history.""" self.conversation_history.append({ "role": "user", "content": user_message }) full_prompt = self._build_prompt() payload = { "model": self.model, "prompt": full_prompt, "stream": False, "options": { "temperature": 0.7, "num_predict": 1024 } } response = requests.post( "http://localhost:11434/api/generate", json=payload, timeout=60 ) response.raise_for_status() assistant_message = response.json().get("response", "") self.conversation_history.append({ "role": "assistant", "content": assistant_message }) return assistant_message
```

#### Construcción del prompt completo
El método auxiliar `_build_prompt()` concatena el prompt de sistema con los mensajes acumulados en el orden correspondiente:

```python
def _build_prompt(self)-> str: """Build prompt with system message and conversation history.""" prompt_parts = [self.system_prompt, "\n\n"] for msg in self.conversation_history: if msg["role"] == "user": prompt_parts.append(f"User: {msg['content']}\n") elif msg["role"] == "assistant": prompt_parts.append(f"Assistant: {msg['content']}\n") prompt_parts.append("Assistant: ") return "".join(prompt_parts)
```

---

### Sección 6: Ejecución del chatbot con memoria activada

Ejecuta el script de chatbot con memoria desde la raíz del repositorio:

```bash
python3 labs/lab3-chatbot/chatbot_v2_with_memory.py
```

Salida esperada:

```text
🤖 Stateful Chatbot Demo (Ollama) ====================================================================== 👤 User: What is OSPF? 🤖 Bot: OSPF is a link-state routing protocol... 👤 User: What did I just ask you? 🤖 Bot: You asked what OSPF is. ✅ SUCCESS: The bot remembers! Conversation length: 4 messages ====================================================================== 💬 Interactive Mode - Type 'quit' to exit, 'reset' to clear history ======================================================================
```

El historial registra 4 mensajes (2 del usuario y 2 del asistente), confirmando que el estado se acumula en la aplicación. A continuación, el script inicia el modo interactivo para formular preguntas sucesivas.

---

### Sección 7: Gestión del historial de conversación

El historial conversacional consume espacio dentro de la **ventana de contexto**. En sesiones prolongadas donde se adjuntan salidas de comandos o logs extensos, el contexto puede saturarse con rapidez.

La **Tabla 5.1** compara las estrategias de gestión de memoria más habituales:

| Estrategia de memoria | Mecanismo de funcionamiento | Cuándo utilizarla |
| :--- | :--- | :--- |
| **Historial completo (*Full history*)** | Envía todos los mensajes previos de la sesión | Laboratorios breves y consultas cortas |
| **Historial reciente (*Recent history*)** | Conserva únicamente los últimos $N$ turnos de conversación | Sesiones interactivas de diagnóstico operativo |
| **Memoria resumida (*Summary memory*)** | Sintetiza los mensajes antiguos y mantiene los recientes literales | Sesiones extensas con decisiones previas relevantes |
| **Almacenamiento externo** | Persiste el contexto en archivos, bases de datos o sistemas de recuperación | Asistentes en producción y soporte multisesión |

> **Tabla 5.1:** Estrategias habituales de memoria para chatbots.

#### Reinicio de la conversación (*Reset*)
La clase implementa el método `reset()` para vaciar el historial cuando se cambia de tema o se requiere limpiar contexto obsoleto:

```python
def reset(self): """Clear conversation history.""" self.conversation_history = []
```

En el modo interactivo, escribir `reset` invoca esta función, garantizando que datos previos no interfieran en nuevos diagnósticos.

---

### Sección 8: Uso de un prompt de sistema orientado a redes

El prompt de sistema establece el marco operativo del asistente:

```text
You are a network engineer assistant. Available devices: - spine1, spine2 (core switches) - leaf1, leaf2 (access switches) Provide accurate, concise answers about networking.
```

El prompt de sistema orienta el comportamiento y fomenta la concisión, pero no constituye un límite de seguridad: las restricciones de ejecución y la validación de herramientas deben implementarse siempre en el código de la aplicación.

---

### Sección 9: Preparación de la memoria del chatbot para agentes

Un chatbot con memoria mantiene la continuidad conversacional, pero sigue dependiendo exclusivamente del texto suministrado en el prompt. El siguiente salto cualitativo consiste en combinar **memoria** con **llamadas a herramientas** (*tool calling*).

En un flujo agéntico real:
1. El usuario reporta una caída de sesión BGP en `leaf1`.
2. El asistente recuerda el equipo y el vecino mencionados en turnos previos.
3. El asistente invoca una herramienta de solo lectura para consultar el estado de la interfaz.
4. El asistente incorpora el resultado de la herramienta al contexto conversacional.
5. El asistente explica la causa probable basándose en la evidencia reunida.

El archivo `chatbot_v3_live_ssh.py` ilustra la extensión hacia dispositivos reales vía SSH. No obstante, el laboratorio principal se centra en la memoria local para consolidar el control del flujo antes de abordar la gestión de credenciales y timeouts de red.

---

### Sección 10: Comprobación de la realidad en producción

En entornos de producción, la memoria conversacional requiere consideraciones adicionales:
- Los mensajes, nombres de equipos y logs constituyen información operativa sensible sujeta a retención y control de acceso.
- Las conversaciones extensas pueden arrastrar premisas obsoletas o errores no corregidos.
- Las arquitecturas de producción combinan ventanas de historial reciente con resúmenes periódicos y recuperación selectiva de información.

La directriz fundamental se mantiene: **útil, pero bajo control**. La memoria dota de continuidad al asistente, exigiendo a su vez una gestión rigurosa del contexto enviado al modelo.

---

### Sección 11: Resumen

En este capítulo avanzamos desde un chatbot sin estado hacia un asistente de red con memoria gestionada por la aplicación, demostrando que la retención de contexto es una responsabilidad del código y no una propiedad inherente al modelo.

Implementamos la clase `NetworkChatbot`, analizamos la concatenación dinámica de prompts, examinamos las estrategias de control del tamaño de contexto y establecimos la base necesaria para la invocación de herramientas.

En el próximo capítulo integraremos herramientas controladas para que el asistente pueda consultar el estado de la red en tiempo real en lugar de limitarse a razonar sobre datos estáticos.
