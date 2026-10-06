# Aurora: agente de WhatsApp con RAG para una academia de inglés

## Arquitectura

El sistema es un agente conversacional de customer support 24/7 construido sobre la plataforma de automatización n8n y conectado mediante la API oficial de WhatsApp (Meta), responde usando la base de conocimiento ingestada a la base de datos de MongoDB Atlas Vector Store mediante la técnica de RAG (retrieval augmented generation).

### Flujo

1. Mensaje entra por el canal de WhatsApp
2. Un conjunto de nodos se encargan de extraer y pasar por el workflow la información relevante extraída de la conversación tales como número de teléfono, nombre, nivel, intención, ubicación, etc.
3. Se hace una query a la base de datos, si el usuario no existe: se crea, si existe: se actualiza la información.
4. Después de eso el sistema hace una búsqueda vectorial a la KB, la reordena con Cohere Rerank a un top 4 de resultados más relevantes.
5. El agente haciendo uso del modelo gpt-4o-mini de OpenAI procesa el texto en el turno correspondiente y en base a sus instrucciones, y los resultados obtenidos en la búsqueda vectorial, decide qué acciones tomar y genera la respuesta correspondiente.
6. Enrutamiento por intención: una vez el agente detecta la intención (demo, clase para niños inscripción directa, o escalamiento a humano), el sistema notifica al equipo por Slack, registra la interacción, y responde cada rama con su propia acción, salvo FAQ que solo registra y responde.

### Componentes

1. Orquestación: n8n, workflows visuales y conexiones mediante APIs/Webhooks.
2. LLM: gpt-4o-mini como modelo principal de razonamiento del agente, elegido por costo-beneficio y un razonamiento medio aceptable para el propósito principal.
3. Persistencia y RAG: MongoDB, con colecciones separadas para: historial de conversaciones, logs, información de prospecto, base de conocimiento.

## Decisión de diseño

Contexto: El agente necesitaba la capacidad de mantener el contexto/hilo conversacional entre sesiones, el volumen esperado era bajo (<300 mensajes al mes).

Opciones consideradas: Se consideró Supabase, pero no se exploró a profundidad esta opción. La otra opción era MongoDB (NoSQL) con Atlas vector store para el RAG y colecciones para historial de conversaciones, logs, información del usuario, etc.

Decisión: Se optó por MongoDB (NoSQL) con Atlas vector store para el RAG ya que ofrecía las capacidades requeridas para solucionar el problema y colecciones para almacenar los datos.

Consecuencia asumida: Se pierden funciones y herramientas ofrecidas en Supabase para desarrollo agéntico, además de las ventajas que pueda ofrecer usar una base de datos que corra sobre PostgreSQL como es el caso de Supabase.

## Qué mediría distinto con evals

1. Tiempos de respuesta del agente: Actualmente no se mide el tiempo de respuesta al darse un cambio aplicado en el workflow ya sea: un cambio de modelo del agente conversacional principal, actualización de nodos code, cambiar o quitar más pasos en el flujo, top N rank de chunks en Cohere Rerank, etc.

- Qué se mide: Tiempo de respuesta.
- Umbral/Verificación: Baseline de tiempo de respuesta actual vs tiempo después del cambio. Aquí se concluye si el cambio mejoró el tiempo de respuesta o no.
- Cómo se mide: 10 queries predefinidas en una hoja de Excel en línea, contiene campos de respuesta del agente, tiempo de respuesta del agente, semáforo rojo/amarillo/verde para indicar si fue buena, neutra o mala. Se ejecuta el mismo conjunto de queries antes y después de cada cambio al workflow para tener una baseline y poder comparar.

2. Testing de edge cases documentados y sin documentar automático: Actualmente la única forma de evaluar un edge case es realizarlo de manera manual, se evalúa el edge case actual, pero no se tiene un método para evaluar o identificar si el cambio aplicado en el workflow no rompió algún otro edge case, a menos que se hagan manualmente uno por uno.

- Qué se mide: edge cases documentados y sin documentar (el "¿qué pasa si...?"). Por cada uno defines cómo se vería el resultado/decisión óptima del agente, y lo calificas Verde (pasa) o Rojo (no pasa). Los edge cases se priorizan en alta/media/baja.
- Umbral/Verificación: Alta y media prioridad, 100% en verde. Baja prioridad, 75% en verde. Si un edge case de alta prioridad cae en Rojo, el cambio se deshace y se revisa.
- Cómo se mide: Crear una base de datos separada de pruebas, y un workflow que ejecute los edge cases en paralelo.

3. MongoDB backups manuales → medible

- El problema: hoy el backup es manual, así que ante corrupción de datos dependes del último backup que alguien haya hecho a mano.
- Solución propuesta: un trigger en n8n que dispara un backup automático cada vez que un workflow escribe o afecta MongoDB.
- Cómo se verifica que funcionó: cada acción que toca MongoDB activa la secuencia de backup, con una nota al pie con la causa. Alternativa: backup programado por schedule (cada semana, cada 2 días, etc.) en vez de por evento.

## Limitaciones conocidas

1. Race conditions

- Qué es: Si un usuario escribe en una ventana de 3 segundos aproximadamente 2 o más mensajes, ej: "hola", "buenos días". Aurora procesará cada mensaje por separado, no como conjunto, esto resulta en una respuesta para "hola" y otra respuesta para "buenos días" siendo redundante y dando la sensación de que no leyó lo primero para el usuario.
- Estado actual: Ya se posee una propuesta que contempla el uso de MongoDB con lógica de Debounce como opción primaria. Otra opción viable considerada es usar Redis.
- Impacto: Medio

2. Sin integración de CRM

- Qué es: Actualmente no se tiene una integración con ningún CRM, por lo que los leads obtenidos por Aurora se almacenan en MongoDB en colecciones siendo esta la única forma de acceder a ellos actualmente.
- Estado actual: Los leads se pueden exportar desde MongoDB, pero no hay una integración directa con ningún CRM ni un proceso claro para manejar el formateo, limpieza y exportación de los datos.
- Impacto: Bajo

3. Un único PDF para la KB (knowledge base)

- Qué es: El sistema actualmente tiene la capacidad de recibir, analizar y vectorizar documentos PDF con la información necesaria que usará posteriormente para responder preguntas de manera adecuada mediante el uso de RAG. No está hecho o pensado para vectorizar múltiples documentos y/o múltiples formatos ya que en este caso no era necesario.
- Estado actual: Es suficiente dado el scope actual, si el scope cambia, se considerarían otras opciones más adelante.
- Impacto: Bajo
