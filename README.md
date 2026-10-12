# Aurora: agente de WhatsApp con RAG para una academia de inglés

Agente conversacional en producción desde el 15 de enero de 2026 para una academia de inglés
con dos sedes en Antioquia, Colombia. Atiende el WhatsApp de la academia, responde
con la información oficial, califica a cada persona y le pasa al equipo solo a quien está
listo para inscribirse o necesita a un humano.

## Resultados en producción

15 de enero a 10 de octubre de 2026 (269 días), sin tráfico de prueba, cada persona contada
una vez:

- **2.066 mensajes** de **339 personas**
- **98,9% de autonomía**: 23 mensajes (de 14 personas) pasaron al equipo. Ningún mes bajó de 97,5%.
- **81 personas interesadas** entregadas al equipo: 52 pidieron clase demo, 20 familias dejaron
  sus datos para el programa infantil y 13 pidieron inscripción directa
- **$21.757.000 COP de ingreso potencial**: personas con datos completos para inscribirse,
  a precio de matrícula. No son pagos confirmados.
- **Hora pico: 11 am.** El grueso entra en plena jornada de clases: el valor del agente es la
  capacidad, no el horario.
- **18 personas con username** de WhatsApp atendidas desde que se soportó el cambio de Meta
  (sección "Agosto 2026")

Las cifras salen de un workflow de informe que se corrigió en octubre de 2026: contaba el
tráfico de prueba, contaba leads por mensaje en lugar de por persona y agrupaba las horas en
UTC. La consulta nueva se validó contra un informe que el cliente ya había recibido: la
diferencia fue exactamente el tráfico de prueba.

## Arquitectura

El sistema es un agente conversacional 24/7 construido sobre n8n y conectado a la API oficial
de WhatsApp (Meta). Responde usando una base de conocimiento vectorizada en MongoDB Atlas Vector
Search mediante RAG (retrieval augmented generation).

### Flujo

1. El mensaje entra por WhatsApp. Si es audio, se transcribe.
2. La identidad del usuario se resuelve una sola vez: teléfono si existe, identificador de
   usuario de Meta (BSUID) si la persona usa nombre de usuario. Todos los nodos leen de ahí.
3. Un extractor (LLM) saca del mensaje nombre, nivel, intención y sede, y el perfil se crea
   o se actualiza en MongoDB.
4. Búsqueda vectorial sobre la KB (top 20) y reordenamiento con Cohere Rerank (top 4).
5. El agente (gpt-4o-mini) responde con el contexto recuperado, la memoria de la conversación
   y sus instrucciones. Cuando la persona completa los datos de un flujo, emite una etiqueta.
6. Enrutamiento por etiqueta (demo, programa infantil, inscripción directa o humano): alerta
   a Slack con el resumen del caso, registro de la interacción y respuesta. Las preguntas
   generales solo se registran y se responden.
7. La respuesta se parte en mensajes cortos antes de enviarse, para que lea como una
   conversación y no como un bloque.

### Componentes

1. Orquestación: n8n self-hosted, workflows visuales y conexiones por API y webhooks.
2. LLM: gpt-4o-mini para el agente y el extractor, elegido por costo-beneficio con un
   razonamiento suficiente para el caso.
3. Persistencia y RAG: MongoDB Atlas, con colecciones separadas para historial de
   conversaciones, logs, perfiles y base de conocimiento.
4. Envío: HTTP Request directo a la API de WhatsApp. El nodo nativo de n8n no puede escribirle
   a un usuario que solo tiene nombre de usuario.

## Decisión de diseño

**Contexto:** el agente necesitaba mantener el hilo conversacional entre sesiones con un volumen
bajo, del orden de cientos de mensajes al mes.

**Opciones consideradas:** Supabase, que no se exploró a fondo, y MongoDB Atlas con Vector
Search para el RAG y colecciones para historial, logs y perfiles.

**Decisión:** MongoDB Atlas. Cubría el RAG y el almacenamiento en un solo servicio.

**Consecuencia asumida:** se pierden las herramientas de Supabase para desarrollo agéntico y
las ventajas de una base relacional sobre PostgreSQL.

## Agosto 2026: tres problemas en producción

### 1. Personas con nombre de usuario

- **Qué pasó:** WhatsApp empezó a dejar que la gente escriba con un nombre de usuario, sin
  mostrar su número. Aurora reconocía a cada persona por su número, así que a esas personas
  no les respondía nada.
- **El riesgo mayor:** varias personas sin número podían quedar con la misma identificación
  vacía y ver conversaciones que no eran suyas.
- **Arreglo:** la identidad se resuelve en un solo punto: el número si existe y, si no, el
  código que WhatsApp le asigna a cada usuario. Las respuestas se mandan por otra vía, porque
  la herramienta original de n8n solo sabía escribirle a números.
- **Lo que no funcionó:** la primera regla fue "si llega el código, no hay número". Probando
  con mensajes reales se vio que el código llega siempre, así que se descartó.
- **Resultado:** 18 personas atendidas que antes no habrían recibido respuesta. En
  septiembre fueron 13 de 27.

### 2. La base de conocimiento le ganaba a las instrucciones

- **Qué pasó:** algunas personas daban todos sus datos y Aurora no le avisaba al equipo. Se
  perdían interesados.
- **Primer intento:** corregir las instrucciones del agente. Con 66 conversaciones de prueba
  subió de 30 a 36 aciertos, pero el agente cambia casi la mitad de sus respuestas aunque
  reciba lo mismo, así que esa mejora no probaba nada.
- **Lo que sí probó algo:** dos casos fallaron 12 de 12 veces, aunque la instrucción decía
  exactamente qué hacer. Si falla siempre no es azar: el texto de la base de conocimiento que
  leía en ese momento pesaba más que la instrucción.
- **Arreglo:** reorganizar la base. Antes se cortaba en pedazos de 1000 caracteres sin mirar
  el contenido y separaba preguntas de sus respuestas. Ahora cada pedazo es una pregunta con
  su respuesta completa.
- **Hallazgo extra:** 4 pedazos nunca llegaban al agente (los contenidos de los niveles A1 a B2).

### 3. Incidente: bucle de avisos de entrega y lectura

- **De dónde salió:** no existía antes. Lo introdujeron los cambios de la misma intervención
  del punto 1: al rediseñar el workflow se agregó una regla para responder "todo lo que no sea
  texto ni audio", pensada para fotos y documentos.
- **Qué pasó:** WhatsApp le avisa al sistema cada vez que un mensaje se entrega o se lee (los
  chulitos). Esos avisos cayeron en la regla nueva y Aurora les respondía como si fueran
  mensajes. Cada respuesta generaba avisos nuevos, que volvían a entrar, y se multiplicó en
  cadena hasta que WhatsApp y Slack lo bloquearon por exceso.
- **Arreglo:** el mismo día se apagó la regla. Después se cambió para que solo responda a
  fotos y documentos, que traen un tipo de mensaje; los avisos no lo traen.
- **Lección:** la prueba revisaba que WhatsApp contestara "recibido" (código 200), no que la
  persona viera la respuesta en su chat. Ahora se verifica mirando el chat.

## Qué mediría distinto con evals

Lo que ya existe (agosto 2026):

- **Pruebas offline sobre el JSON real del workflow:** estructura, código real de los nodos Code
  y expresiones de n8n evaluadas como JavaScript. Detectan el error antes de importar.
- **Banco de evals del agente:** 66 casos con la etiqueta esperada, un marcador de corridas que
  señala filas inestables y un gate de PII sobre los datos. Primera lección: con 42% de varianza,
  un cambio solo se acepta cuando mueve casos de 0 de N, no cuando sube un promedio.

Lo que falta:

1. **Tiempo de respuesta.** Hoy no se mide antes y después de cada cambio (modelo, nodos, top N
   del rerank). Se mediría con las mismas consultas del banco y una línea base por cambio.
2. **Regresión automática de edge cases.** Cada edge case documentado tiene su resultado
   esperado y una prioridad. Umbral: prioridad alta y media en 100% verde, baja en 75%. Si uno
   de prioridad alta cae, el cambio se revierte.
3. **Backups de MongoDB medibles.** Hoy son manuales. Propuesta: backup programado con una
   verificación de restauración, no solo de que el archivo exista.

## Limitaciones conocidas

1. **Condición de carrera.** Si una persona manda dos mensajes en menos de 3 segundos, se
   procesan por separado y el segundo puede no ver el primero. No corrompe perfiles: desordena
   el contexto. Hay un diseño de debounce (buffer en MongoDB) que no está activo, porque va en
   la entrada del workflow, el punto donde un error alcanza a todos los usuarios.
   Impacto: medio.
2. **Campo de intención contaminado.** El campo que guarda la intención del usuario recibe
   etiquetas largas o de versiones anteriores del extractor. No afecta el enrutamiento, que va
   por la etiqueta del agente, pero rompe la analítica por intención: todas las cifras se cuentan
   por etiqueta. Abierto. Impacto: bajo.
3. **Agente no determinista.** 42% de las etiquetas cambian entre corridas idénticas. Se mitiga
   decidiendo con ceros y con el banco de evals, no con una corrida. Impacto: medio.
4. **Sin integración de CRM.** Los leads quedan en MongoDB y llegan al equipo por Slack; no hay
   exportación automática. Impacto: bajo.
5. **Un solo documento en la KB.** Suficiente para el alcance actual; múltiples documentos y
   formatos se evaluarían si el alcance cambia. Impacto: bajo.
