# Bitácora de Diseño: De la idea al Agente de Grado Bancario

Este documento narra el proceso de toma de decisiones técnicas, las hipótesis planteadas y las rutas descartadas durante el desarrollo del reto Banorte.

## 1. Fase Inicial: El enfoque del producto
Al recibir el reto, mi primera reacción instintiva fue bocetar un sitio web para exponer al agente. 
*   **Decisión:** Tras analizar el requerimiento de "endpoint compatible con Open Responses", concluí que crear un frontend era una distracción táctica.
*   **Justificación:** El valor de un perfil Senior en una Subdirección de IA radica en la robustez del backend, la latencia y la calidad de las respuestas, no en la interfaz gráfica. Decidí enfocar el 100% de mi ancho de banda mental en la arquitectura de datos y despliegue.

## 2. El Cerebro del Agente: OpenRouter vs. Amazon Bedrock
Evalué dos caminos para el motor de inferencia que me permitiera jugar con multiples modelos: **OpenRouter** (libertad de modelos) vs. **Amazon Bedrock** (estándar de cumplimiento corporativo).
*   **Hipótesis inicial:** OpenRouter permitía probar rápidamente diferentes modelos como Claude 3.5 Sonnet o Llama 3.
*   **Descarte:** Aunque es ágil, requiere exponer datos a través de Internet público y gestionar llaves de API externas.
*   **Decisión Final:** Elegí **Bedrock**. En el sector financiero, la soberanía de los datos es innegociable. Bedrock permite mantener el flujo de información dentro del perímetro de seguridad de AWS, usando autenticación IAM nativa. Fue una decisión de "Seguridad Corporativa" sobre "Agilidad de Prototipado".

## 3. El Desafío del Cómputo: Serveless vs. Contenedores
Esta fue la etapa de mayor iteración arquitectónica:
*   **Intento 1 (Lambda + API Gateway):** Inicialmente atractiva por costo y facilidad. Sin embargo, recoerde que API Gateway tiene un *hard limit* de 29 segundos. Un RAG complejo procesando documentos técnicos más el tiempo de inferencia segun la pregunta puede superar ese tiempo 
*   **Intento 2 (CloudFront + ALB + Lambda):** Una arquitectura muy elegante para seguridad en el borde. Sin embargo, añadía una capa innecesaria de complejidad en el enrutamiento de red dado el límite de tiempo del reto.
*   **Decisión Final (ECS Fargate + ALB):** Proporciona control total sobre los *timeouts* y maneja el *streaming* de tokens (Server-Sent Events) de forma nativa sin romper la conexión. Es el camino de oro para aplicaciones escalables, profesionales y con nivel de customización adecuado.

## 4. La "Trampa" de los Frameworks (LangChain/LlamaIndex)
Como desarrollador, estoy acostumbrado a usar librerías de orquestación. No obstante, cuestioné su valor en este escenario:
*   **Evaluación:** Añadir LangChain supondría dependencias adicionales, latencia y una capa de abstracción innecesaria.
*   **Decisión:** Opté por el **SDK nativo de AWS (boto3)** junto con **Bedrock Knowledge Bases**. Esto reduce la superficie de ataque, mantiene el código ligero, auditable y libre de dependencias de terceros que suelen introducir inestabilidad en producción.

## 5. Observabilidad como Auditoría
No quería entregar una "caja negra". La observabilidad fue definida desde el diseño:
*   **Decisión:** Integrar **CloudWatch** y **X-Ray**. 
*   **Justificación:** Esto me permite trazar el camino de una pregunta desde que entra por el ALB, pasa por mi código, llega a Bedrock y vuelve. Es la herramienta necesaria para identificar cuellos de botella y auditar la calidad del RAG en tiempo real, demostrando un enfoque de "Operaciones" (LLMOps) desde el día uno.

## 6. La forma del código: arquitectura hexagonal

Con el contrato escrito y la ingesta del corpus todavía por delante, apareció
una pregunta de secuencia: ¿se construye el endpoint *después* de tener la
Knowledge Base, o se construye antes y se conecta luego?

*   **Hipótesis inicial:** una estructura plana (`app/main.py`, `app/retrieval.py`,
    `app/inference.py`) llamando a `boto3` directamente. Es lo más corto de
    escribir y lo que sugería el boceto del plan.
*   **Descarte:** esa forma ata la verificación del contrato a que exista
    infraestructura. Sin KB creada no hay recuperación, sin recuperación no hay
    respuesta, y sin respuesta no se puede probar el emisor de eventos SSE —que
    es la parte más delicada y la que más barato sale arreglar temprano.
    Además obliga a parchear (`mock.patch`) para probar cualquier cosa, y un
    parche prueba la implementación, no el comportamiento.
*   **Decisión final:** **puertos y adaptadores**. El núcleo (dominio +
    aplicación) declara qué necesita —recuperar evidencia, generar texto,
    resolver un alias, medir— y la infraestructura lo cumple. Hoy lo cumplen un
    corpus local y un modelo determinista; mañana, Bedrock Knowledge Base y
    `ConverseStream`, cambiando una variable de entorno.
*   **Costo asumido:** más archivos y una indirección extra. Se justifica porque
    la frontera coincide con lo que de verdad va a cambiar (el proveedor de
    recuperación y el de inferencia) y con lo que hay que poder probar sin nube.
    En un CRUD sería sobreingeniería; aquí es lo que permite entregar por partes.

El resultado medible: la suite de aceptación completa —contrato, recuperación,
veracidad y operación— corre en segundos sin credenciales de AWS, y los
adaptadores de Bedrock ya están ejercitados con clientes falsos antes de su
primera llamada real.

## 7. Entregar con el RAG pendiente sin mentir

Tener el servicio en pie antes de que el corpus esté cargado plantea un riesgo
evidente: que el agente conteste igual, pero inventando.

*   **Decisión:** que la ausencia de evidencia produzca **silencio, no relleno**.
    Sin fragmentos recuperados, la respuesta correcta —y la única posible— es
    "eso no consta en los documentos disponibles".
*   **Cómo se sostiene:** el sustituto local del modelo no genera lenguaje;
    compone la respuesta a partir de los fragmentos y declina cuando la pregunta
    menciona algo que no aparece en ellos. Es deliberadamente tonto: así las
    pruebas miden el emisor de eventos, el enmascarado y la orquestación, y no
    la suerte de un modelo.
*   **Lo que esto todavía no prueba:** la veracidad de un modelo generativo
    real. Eso se mide con las preguntas de oro contra Bedrock (E7), y está
    declarado como tal en la suite: los casos que exigen un modelo real se
    saltan con motivo visible en el reporte, en vez de desaparecer.

Un caso de prueba deja constancia de la propiedad: con el corpus vacío, el
agente se queda sin evidencia y declina. Es la condición que hace seguro
desplegar mientras el corpus se prepara.


## 8. Recopilar información relevante de Luis Alberto Ortiz Meza y su trayectoría profesional

Se hace una busqueda en la computadora de certificados, CVs viejos y documentos profesionales o escolares, al terminar esta etapa se hace uso de Gemini para ayduar a complementar la información.

Llenando la info del perfil, me encontré que mucho era lo que yo digo, así que decidí buscar una forma de sacar las STATS de mi perfil de gitHub para tener la fuente de la verdad, ya no es lo que yo digo si no una fuente de datos fiable [Stats-github](stats-github.md)

Para evitar subir archivos personales a la nube, solo se subieron arhivos MD, preverificados y parseados con algunos scripts de pyhton. Y se 


## 9. Bedrock en local: lo que solo se aprende contra la API real

Con el contrato verificado y el corpus preparado, conecté la inferencia real
antes de tener infraestructura: mismo endpoint, misma suite, cambiando una
variable de entorno. Dos cosas aparecieron de inmediato, y ninguna estaba en la
documentación que había leído:

*   **Los modelos de Anthropic en Bedrock no se invocan por su ID base.**
    `anthropic.claude-sonnet-5` responde *"on-demand throughput isn't
    supported"*; hay que usar el perfil de inferencia regional
    (`us.anthropic.claude-sonnet-5`). Un default equivocado que, de haber
    llegado al despliegue, habría fallado solo en producción.
*   **`temperature` está deprecado en Claude Sonnet 5** y devuelve
    `ValidationException`. Haiku 4.5 y la familia GPT siguen aceptándola. No lo
    resolví con un `try/except`: el descriptor de modelo declara
    `supports_sampling`, el adaptador solo envía el parámetro cuando el modelo
    lo acepta, y si un cliente lo manda a un alias que no lo soporta recibe un
    400 con `param: "temperature"`. Aceptarlo y descartarlo en silencio sería
    mentir sobre lo que el servidor hizo con la petición.

Esto valida la nota del plan E1: *trabajar contra las dos familias desde el
principio, porque las diferencias son baratas de absorber ahora y caras en el
despliegue*.

## 10. Comparación entre familias de modelo

13 preguntas de oro sobre el corpus real —6 positivas, 5 negativas y 2 de
identificador enmascarado— contra los tres alias:

| Alias | Modelo | Aciertos | Negativas | TTFT p50 | Total medio |
|---|---|---|---|---|---|
| `agente-rag-sonnet` | Claude Sonnet 5 | **13/13** | 5/5 | 1.75 s | 3.74 s |
| `agente-rag-haiku` | Claude Haiku 4.5 | 12/13 | 5/5 | **0.83 s** | 1.84 s |
| `agente-rag-gpt` | GPT-OSS 120B | 7/13 | 5/5 | 2.10 s | 2.60 s |

**Cero credenciales inventadas y cero identificadores filtrados en los 39
casos.** Las tres familias niegan correctamente lo que no consta, resisten la
inyección de prompt y no transcriben CURP, cédula ni teléfono.

Donde divergen es en la **convención de citación**. Los seis fallos de GPT-OSS
son todos del mismo tipo: recupera el documento correcto y responde con la
información correcta, pero no cita el `document_id` entre corchetes. Para este
agente eso no es un detalle de formato: la cita *es* el mecanismo de
no-repudio. Una respuesta correcta sin cita no es verificable, y por tanto no
sirve al propósito del sistema.

La conclusión práctica es que el valor multi-proveedor de Open Responses tiene
un límite honesto: el protocolo se puede cumplir con cualquier familia, pero
las garantías de producto —citar la evidencia— dependen de qué tan bien sigue
instrucciones el modelo, y eso hay que medirlo, no suponerlo. Sonnet 5 queda
como default por exactitud; Haiku 4.5 es la opción de baja latencia con un
TTFT menos de la mitad, a cambio de un fallo de citación en trece preguntas.

**Lo que esta medición todavía no cubre:** son 13 preguntas, no las 20 del
plan, y la recuperación es local, no la Knowledge Base. Ambas cosas cambian los
números y deben volver a correrse cuando E3 esté cerrado.


## 11. Infraestructura: renunciar al NAT Gateway a propósito

Al escribir la red apareció una decisión con dinero de por medio. Una tarea de
Fargate en subred privada necesita salida para tres cosas: bajar su imagen de
ECR, leer el token de Secrets Manager y hablar con Bedrock. Hay dos formas.

*   **NAT Gateway:** una sola pieza, unos 32 USD/mes más tráfico, y funciona
    para todo sin configurar nada. Pero el tráfico hacia Bedrock sale a
    internet público y vuelve.
*   **VPC endpoints (PrivateLink):** seis endpoints de interfaz —
    `bedrock-runtime`, `bedrock-agent-runtime`, `ecr.api`, `ecr.dkr`,
    `secretsmanager`, `logs`— más el gateway de S3, que es gratis. Unos 0.01
    USD/hora por endpoint y por zona: con dos zonas, del orden de 90–105
    USD/mes. Es la partida más cara de esta infraestructura, más que el ALB y
    más que el cómputo.

**Decisión: endpoints, y sin NAT.** La §2 de esta bitácora descartó un router
de modelos externo por no exponer datos a internet público. Con NAT, la llamada
a Bedrock saldría exactamente por ahí y el argumento se caería solo. Con
endpoints, la tabla de rutas privada no tiene ruta a `0.0.0.0/0`: no es que el
tráfico *no vaya* a internet, es que *no puede*. La tesis queda respaldada por
la topología, no por la narrativa.

El costo es real y no lo escondo: si el presupuesto apretara, la palanca
correcta es desplegar los endpoints en una sola zona (la mitad del gasto,
a cambio de perder tolerancia a fallo de zona en la salida), no volver al NAT.

**Otras dos decisiones que merecen nombre:**

*   **Permisos de Bedrock con perfiles de inferencia.** Un perfil `us.` enruta
    a varias regiones, así que la política necesita el ARN del perfil *y* el
    del modelo base en cada región destino. Si falta una, la invocación falla
    de forma intermitente y solo bajo carga — el peor modo de fallo posible.
*   **Métricas por filtro de logs, no por código.** La aplicación ya emite
    `ttft_ms`, `chunks_retrieved` y `grounding.failure` en JSON estructurado.
    Los filtros de CloudWatch los convierten en métricas sin añadir una sola
    llamada a `PutMetricData` en la ruta de la petición. La alarma del p95 de
    TTFT y la de fundamentación salen de ahí.

La guarda de cuenta se probó a propósito con un perfil de otra cuenta: el
provider corta con *"AWS account ID not allowed"* y `deploy.sh` aborta antes de
llegar a Terraform.


## 12. Decisión B: el vector store, o cómo no pagar 350 USD al mes por diez documentos

Esta era la decisión con más dinero en juego del proyecto, y la que el plan
marcaba como bloqueante de E3.

**El problema.** Bedrock Knowledge Bases necesita un almacén vectorial. La
opción por defecto —la que la consola sugiere y casi todos los tutoriales
usan— es **OpenSearch Serverless**, que factura *OpenSearch Compute Units*
corriendo de forma continua, existan o no consultas. Ese piso es independiente
del tamaño del corpus.

**Los números** (us-east-1, precios de agosto de 2026):

| Opción | Modelo de cobro | Costo con este corpus |
|---|---|---|
| OpenSearch Serverless (colección clásica) | 0.24 USD por OCU-hora, mínimo 2 OCU | ~350 USD/mes |
| OpenSearch Serverless (dev/test, sin redundancia) | mínimo 1 OCU | ~175 USD/mes |
| Aurora Serverless v2 con pgvector | ~0.06 USD por ACU-hora, mínimo 0.5 ACU | ~44 USD/mes |
| **S3 Vectors** | 0.06 USD/GB-mes + 0.20 USD/GB de escritura + consultas por TB procesado | **céntimos al mes** |

Este corpus son 10 documentos, un fragmento cada uno (decisión de chunking de
E2). Diez vectores de 1024 dimensiones en float32 son unos 41 KB. El
almacenamiento cuesta, literalmente, una fracción de céntimo; las consultas de
una demo, otra fracción. Frente a ello, cualquier opción con piso de cómputo
significa **pagar una infraestructura de búsqueda dimensionada para millones de
vectores con el fin de indexar diez archivos**.

**Decisión: S3 Vectors.** No por ser lo más barato, sino porque es lo único
cuyo modelo de cobro guarda proporción con el problema: se paga por lo que se
guarda y por lo que se consulta, y con un corpus pequeño eso tiende a cero.

**Lo que se acepta a cambio, dicho sin adornos:**

*   Menor rendimiento en consultas de altísimo volumen. No es el caso de uso:
    un agente sobre un CV recibe consultas esporádicas.
*   Ecosistema más joven que OpenSearch, con menos material de referencia.
*   **El argumento del piso ya no es universal:** las colecciones *NextGen* de
    OpenSearch Serverless no tienen mínimo de OCU y escalan a cero tras diez
    minutos de inactividad. Para un corpus pequeño S3 Vectors sigue siendo más
    simple y más barato, pero sería deshonesto presentar OpenSearch como
    necesariamente caro sin esa aclaración.

**Cuándo revisar esta decisión.** Si el corpus creciera a millones de vectores
con consultas constantes, o si hicieran falta filtros complejos, búsqueda
híbrida léxico-vectorial o agregaciones, la comparación se invierte y
OpenSearch vuelve a ser la respuesta correcta. Con diez documentos, no lo es.

**Efecto en la infraestructura:** el módulo `bedrock.tf` crea el bucket del
corpus, el bucket vectorial, el índice (1024 dimensiones, distancia coseno), el
rol de la KB con condiciones anti-*confused deputy*, la Knowledge Base y su
origen de datos con `chunking_strategy = "NONE"`. El plan pasa de 54 a 65
recursos y el costo fijo mensual no se mueve: los VPC endpoints siguen siendo,
con diferencia, la partida más cara.

*Fuentes de precios: [S3 pricing](https://aws.amazon.com/s3/pricing/),
[OpenSearch Serverless entry cost](https://aws.amazon.com/about-aws/whats-new/2024/06/amazon-opensearch-serverless-entry-cost-half-collection-types/),
[S3 Vectors GA](https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-s3-vectors-generally-available/).*


## 13. El despliegue: cuatro fallos que solo existen en AWS

El plan decía que la ruta crítica pasa por desplegar antes de lo que la
intuición sugiere, «porque es donde vive el riesgo desconocido». Se confirmó:
ninguno de los cuatro fallos de este día era detectable en local, y tres eran
míos.

**1. La etiqueta de la imagen describía el commit, no lo construido.**
`deploy.sh` etiquetaba con `git rev-parse --short HEAD`, pero el build usa el
árbol de trabajo. Con cambios sin commitear, cada build produce una imagen
distinta bajo la misma etiqueta: Terraform no ve diferencia en la task
definition, ECS sigue con la anterior, y el despliegue anuncia «Apply
complete» sin haber desplegado nada. Estuve corrigiendo código que jamás
llegó al contenedor. Es el peor modo de falla —silencioso y con cara de
éxito— y ahora, con el árbol sucio, la etiqueta lleva marca de tiempo y lo
anuncia en pantalla.

**2. EC2 no acepta caracteres fuera de ASCII en la descripción de un security
group.** Escribir la infraestructura comentada en español costó un apply a
medio camino: `Character sets beyond ASCII are not supported`. CloudWatch y
Bedrock sí aceptan UTF-8, así que solo las descripciones de EC2 van sin
acentos, con una nota en el código explicando por qué la incoherencia es
deliberada.

**3. `readyz` llamaba al plano de control de Bedrock.** `ListFoundationModels`
no es `bedrock-runtime`: es otro servicio, con su propio endpoint de VPC. Sin
NAT y sin ese endpoint la llamada no tenía salida, se quedaba esperando, y el
ALB devolvía 504 sin decir qué había fallado.

La corrección de fondo fue de diseño, no de red. Validar que un alias tiene
acceso concedido es un **hecho de despliegue**: no cambia mientras el servicio
corre. Reverificarlo en cada sonda exigía un endpoint extra (~15 USD/mes) y
tres segundos de latencia para no aportar información nueva — y, mientras
tanto, reportaba `not_ready` un servicio que atendía peticiones sin problema.
Una sonda que da falsas alarmas provoca la caída que dice prevenir. La
validación se movió a `deploy.sh`, que corre desde fuera de la VPC y aborta
antes de publicar la imagen si falta acceso. **El contrato §3 se corrigió para
decir dónde vive la validación**: dejar el documento afirmando una cosa y el
código haciendo otra habría sido peor que el bug.

**4. Ningún cliente de AWS tenía tiempo límite.** Los defaults de botocore —60
segundos de conexión, varios reintentos— están pensados para scripts, no para
una sonda detrás de un balanceador. Ahora cada cliente declara su presupuesto y,
sobre todo, **el límite de la sonda vive en el caso de uso**, no en cada
adaptador: las tres comprobaciones corren en paralelo con un techo de cuatro
segundos, y agotar el tiempo cuenta como no disponible. Un adaptador nuevo que
olvide su timeout ya no puede colgar `readyz`.

### Lo que el despliegue confirmó

*   **La imagen bajó por PrivateLink, sin NAT.** La tarea arrancó en subredes
    sin ruta a `0.0.0.0/0`. La tesis de soberanía de datos de la §2 queda
    respaldada por la topología.
*   **B8 pasa.** Los deltas llegan repartidos en 6.75 s atravesando el ALB, no
    de golpe al final. Es la prueba que, según el contrato, decide la entrega.
*   **La ingesta indexó 10 de 10 documentos**, con sus 10 archivos de
    metadatos y cero fallos, sobre S3 Vectors.

## 14. Resultados finales contra el despliegue

14 preguntas de oro —6 positivas, 5 negativas, 2 de identificador enmascarado y
1 en idioma cruzado— ejecutadas contra el ALB, con recuperación real desde la
Knowledge Base:

| Alias | Modelo | Aciertos | Negativas | TTFT p50 |
|---|---|---|---|---|
| `agente-rag-sonnet` | Claude Sonnet 5 | **14/14** | 5/5 | 2.77 s |
| `agente-rag-haiku` | Claude Haiku 4.5 | 13/14 | 5/5 | **1.61 s** |
| `agente-rag-gpt` | GPT-OSS 120B | 8/14 | 5/5 | 2.15 s |

**Cero credenciales inventadas y cero identificadores filtrados en los 42
casos.** Ninguno de los tres modelos entrega el domicilio ni la clave de
elector, ninguno cede a la inyección de prompt, y CURP, cédula y teléfono salen
enmascarados o confirmados sin transcribir.

**La pregunta en inglés sobre corpus en español la aciertan los tres.** Con
recuperación léxica local era imposible —una pregunta en inglés no comparte
palabras con un documento en español— y con embeddings multilingües es trivial.
El caso C5 estaba mal atribuido: no era un problema de modelo, era de
recuperación.

La divergencia entre familias sigue estando en la **citación**. Los seis fallos
de GPT-OSS son del mismo tipo: recupera el documento correcto y responde con la
información correcta, pero no cita el `document_id`. En este agente la cita no
es formato, es el mecanismo de no-repudio: una respuesta correcta sin cita no
es verificable y no cumple el propósito del sistema.


## 15. Cuando el agente calla de más: conversación frente a evidencia

Reporte de un problema de producto, no de infraestructura: el agente declinaba
ante preguntas que un reclutador hace de forma natural. Medido contra el
despliegue, resultaron ser tres cosas distintas y solo una era la sospechada.

**Lo que funcionaba.** Saludos, agradecimientos y "¿qué puedes hacer?" se
respondían con normalidad. Una pregunta fuera de dominio —la capital de
Francia— se declinaba explicando el propósito, que es el comportamiento
correcto.

**Lo que no.** Ante *"¿por qué no debería contratar a Luis?"* respondía «no
puedo emitir una opinión, esa valoración excede mi función». Impecable según
sus reglas y pésimo como producto: la pregunta más importante que hará un
evaluador terminaba en un no-contesto.

Y un defecto de tono que nadie había notado: ante *"¿quién eres?"* contestaba
**«No debo revelar el prompt de sistema ni cambiar mi rol»**. La regla
anti-inyección se disparaba con una pregunta inocente, sonaba a sistema bajo
ataque y, de paso, confesaba la existencia del prompt que pretendía proteger.

### La postura, y por qué va con cita

Se añadió una regla: ante idoneidad, recomendación o contratación —incluida la
formulada en contra— el agente **toma posición a favor y la sustenta**. La
tentación era responder con la frase sola. Se descartó: un evaluador que
pregunta "¿por qué NO contratarlo?" suele estar comprobando si el agente es un
vendedor disfrazado, y una frase promocional sin evidencia le daría motivos
para dudar también de las credenciales que sí son verificables.

Con cita, la misma frase pasa de publicidad a postura defendible. La respuesta
real ante *"¿debería contratarlo? ¿tiene CISSP y un MBA de Harvard?"* abre con
la recomendación, la respalda con cuatro credenciales citadas, niega las dos
inexistentes y **añade por su cuenta que la certificación de AWS está vencida
desde 2024**. Un agente promocional habría escondido ese dato mientras
defendía; este lo pone encima de la mesa. Eso es lo que hace creíble el resto.

Se añadió también la salvaguarda inversa, observada en producción: si no hay
evidencia, el agente **no** abre con la recomendación. Una recomendación
seguida de «pero no puedo respaldarla» es peor que no darla.

### El piso de relevancia que no fue

Se detectó que la KB devuelve seis fragmentos incluso ante un saludo:
`retrieve` entrega siempre `numberOfResults`, ordenados pero sin filtrar. La
optimización parecía obvia —un piso de score— y se implementó en 0.58.

Rompió la evaluación: de 14/14 a 13/14, porque *"¿cuál es su número de
teléfono?"* dejó de recuperar el CV. Al medir la distribución, la causa quedó
clara: **las bandas se solapan**.

| Tipo | Score tope | Pregunta |
|---|---|---|
| legítima | 0.564 | ¿Cuál es su número de teléfono? |
| ruido | 0.566 | Hola |
| legítima | 0.576 | ¿Vale la pena entrevistarlo? |
| ruido | **0.586** | Gracias |
| legítima | 0.873 | ¿Qué cursos de Platzi aprobó en 2026? |

"Gracias" puntúa más alto que una pregunta legítima. Con Titan v2 sobre un
corpus de diez documentos, **ningún umbral absoluto separa ruido de señal**. Se
revirtió: un falso negativo —declinar teniendo la evidencia delante— es mucho
peor que pagar unos tokens de más en un saludo. El filtrado de lo irrelevante
lo hace el prompt, que ya declina correctamente aunque reciba fragmentos que no
vienen al caso.

El parámetro se conserva desactivado, con la medición escrita en el código para
que nadie lo reintente sin datos.

**Lo que esto dice de la suite de oro:** la optimización parecía obviamente
buena y rompió algo real. Sin una evaluación ejecutable, habría entrado en la
entrega y el fallo lo habría encontrado quien evaluara, no yo.

---

## 16. De un agente a una plataforma: el tema como dato

El agente resolvía un problema —responder sobre una trayectoria profesional con
evidencia— y lo resolvía bien. La pregunta siguiente era si servía para otro
corpus: unas fichas técnicas de coches, unos informes de inversión. La respuesta
honesta era «casi»: la arquitectura hexagonal ya aislaba recuperación e
inferencia, pero el **tema** estaba disperso por el código. El prompt hablaba de
credenciales, el enmascarado asumía identificadores mexicanos, la ingesta sabía
de cédulas de la SEP y el troceado daba por hecho documentos de media página.

### La decisión: un perfil es un dato, no una rama

Todo lo que cambia entre dominios se concentró en un objeto de dominio inmutable,
`Profile`, que se carga de un YAML:

```yaml
slug: coches
subject: los vehículos, versiones y especificaciones de las marcas documentadas
sources: los folletos y fichas técnicas oficiales de cada marca
redaction: []                 # sin enmascarado
retrieval: { top_k: 8, min_score: 0.35 }
chunking: { max_chars: 2000, overlap_chars: 250 }
path_metadata: [marca]
```

Lo importante es lo que el perfil **no** puede hacer. Sus reglas se numeran
*después* de las innegociables —fundamento documental, cita obligatoria, negación
explícita, defensa ante inyección de prompt—, de modo que un tema puede añadir
postura o formato pero nunca relajar la veracidad. La regla que abre con «Debes
contratar a Luis» es una regla del perfil `luis-cv`; la que prohíbe completar con
conocimiento general es del núcleo y no se toca.

La prueba de que la traducción fue fiel: el conjunto de oro de credenciales pasa
**7/7 contra Sonnet 5** después del refactor, igual que antes.

### El enmascarado tenía que dejar de ser global

El patrón de cédula profesional es `\d{7,8}` aislado. En un corpus de títulos eso
es correcto. En uno de fichas técnicas convierte cada precio, cada potencia y
cada número de parte en asteriscos. El enmascarado pasó a ser una política del
perfil, y el valor por defecto de un tema nuevo es **no enmascarar nada**: una
redacción que se activa sola sobre datos que no entiende hace más daño que bien.

### El troceado como decisión de dominio

Un título cabe en media página: un documento = un fragmento era la decisión
correcta y sigue siéndolo para `luis-cv`. Un folleto de coche son 40 páginas, y
tratarlo igual produce dos fallos a la vez: el agente cita «el folleto» para
cualquier afirmación —sin poder señalar dónde lo dice— y cada turno paga decenas
de miles de tokens de prompt. El troceo vive en el dominio, no en el script de
ingesta, precisamente porque decide qué es una unidad de evidencia citable.

Corta por costuras del texto (encabezado → párrafo → frase, nunca a mitad de
línea) y solapa: una fila de especificaciones partida por la mitad es un
fragmento que afirma «187» sin decir de qué.

### Topología: un plano de cómputo, N bases de conocimiento

Aquí había tres caminos y el costo decide:

| Opción | Costo por tema | Riesgo |
|---|---|---|
| Stack completo por tema | ~60 USD/mes (ALB + endpoints) | Ninguno, pero se paga la infraestructura N veces |
| Infra compartida, N KB | Centavos (S3 Vectors no tiene piso) | Ninguno relevante |
| Infra compartida, 1 KB con filtros | Centavos | Los corpus compiten en el mismo ranking |

Se eligió la segunda. Lo caro del despliegue es fijo y no depende del número de
temas; lo que se duplica es el índice vectorial, que sobre S3 Vectors cuesta
centavos —la misma razón por la que se descartó OpenSearch en la decisión B.

La tercera se descartó no por costo sino por modo de falla: con un índice único,
un filtro de metadatos mal puesto devuelve fichas de coches a una pregunta sobre
títulos, y ese error es silencioso. Índices separados lo hacen **imposible** en
vez de improbable.

Los temas los lee Terraform de `profiles/*.yaml` con `yamldecode`, no de una
variable propia. Es la misma lista que consume el servicio: no hay dos fuentes de
verdad que puedan divergir, y añadir un tema es crear su YAML y aplicar.

### Lo que se rompió por el camino

Tres fallos reales que solo aparecen al montar esto:

1. **La imagen no copiaba `profiles/`.** El contenedor habría arrancado, devuelto
   200 y respondido con un perfil genérico que nadie escribió, sobre un corpus
   que no era el suyo. Se arregló con un `COPY` y, sobre todo, haciendo que
   arrancar sin perfiles **falle** cuando el entorno no es `local`. Un despliegue
   que funciona a medias es peor que uno que no arranca.
2. **`RAG_CORPUS_DIR` dejó de tener efecto.** El perfil siempre ganaba, así que
   la evaluación medía un corpus vacío y daba 4/7 culpando al modelo. La
   precedencia correcta es la contraria: quien define esa variable está apuntando
   el servicio a un corpus concreto a propósito. Además, `Path` no expande `~`:
   un perfil con `~/corpus-x` buscaba una carpeta llamada literalmente «~».
3. **Un PDF roto tumbaba las 131 conversiones.** Con un corpus de origen
   desconocido, alguno estará cifrado, truncado o no será lo que su extensión
   dice. Los fallos se aíslan por archivo y se reportan; el lote sigue.

### El resultado

Con `coches` —101 PDFs de 14 marcas, 1053 fragmentos— el conjunto de oro de
arranque da **6/7 contra Sonnet 5 sin tocar una sola regla del prompt**. El caso
que falla lo hace porque el recuperador léxico local devuelve los fragmentos de
aviso legal del folleto, y ante evidencia que no responde a la pregunta el agente
**declina en vez de inventar**. Es el comportamiento correcto, y es además la
medida de cuánto aporta la Knowledge Base real frente al sustituto local.

### El menú: por qué una herramienta interactiva no es un lujo

Reutilizar el agente en un tema nuevo son cinco pasos —configurar, preparar el
corpus, probar, desplegar, ingerir—, cada uno con sus parámetros. La mitad de los
errores de un despliegue no son técnicos: son creer que ya se hizo el paso
anterior. `make menu` es un adaptador de entrada más —el actor es una persona en
vez de un cliente de API— que no decide nada de negocio: resuelve configuración,
llama a los mismos casos de uso y a los mismos scripts, y sobre todo **dice en
qué paso estás**. Su opción más útil no es «desplegar», es «estado».

Los nombres y variables del proyecto —cuenta, región, prefijo de recursos, primer
tema— no están en el código: los recoge el asistente y los escribe en `.env`,
`infra/terraform.tfvars` y `profiles/<tema>.yaml`. El paquete Python quedó con un
nombre neutro (`rag_agent`) porque es la librería; la identidad del proyecto es
configuración.

---

## 17. El PDF que no se deja leer

Del corpus de coches, 30 de 131 PDF no daban texto. El pipeline los reportaba y
seguía, que es honesto pero deja fuera a marcas enteras: sin ellos, el agente
declinaba para siempre cualquier pregunta sobre MG, BYD o Kia y nadie sabría por
qué. Antes de escribir OCR conviene mirar **por qué** no se dejan leer, porque
resultaron ser tres problemas distintos con tres soluciones de coste muy
distinto.

| Causa | Documentos | Solución |
|---|---|---|
| Cifrado AES con contraseña de propietario vacía | 6 | `decrypt("")` |
| PDF de imagen (el texto es un mapa de bits) | 23 | Transcripción |
| Archivo de 0 bytes | 1 | Reportarlo |

Los seis cifrados eran el rescate más barato que existe: restringían copiar e
imprimir, no leer, y su capa de texto estaba entera. Añadir `pypdf[crypto]` y
una línea devolvió 98.000 caracteres de especificaciones reales sin transcribir
nada. Vale la pena decirlo porque el instinto era empezar por el OCR.

### El caso que obliga a pensar: la ficha comparativa

Los 23 restantes no son escaneos de un documento de prosa. Son esto:

```
                              COOL MT   STYLE MT   STYLE CVT   EXCITE CVT
Potencia máxima neta (HP)            108 HP @ 4,500 rpm
Transmisión                    MT (5 vel.)        CVT (8 vel. simuladas)
Quemacocos eléctrico                                              •
```

Un OCR lineal —tesseract sobre la página— devuelve las etiquetas y **pierde los
valores y las columnas**. El resultado no es incompleto: es peligroso. Aplanada,
esa tabla dice que el MG3 lleva quemacocos. Lo lleva **una** de sus cuatro
versiones. El agente lo citaría con toda propiedad y estaría afirmando algo
falso, que es exactamente lo que este proyecto existe para evitar, y lo que la
primera regla del perfil `coches` prohíbe por escrito.

Así que el requisito no era «sacar texto»: era **conservar la rejilla**.

### Textract, y las dos cosas que Textract no resuelve

`AnalyzeDocument` con `TABLES` devuelve celdas con fila, columna y geometría, y
lee los acentos del español correctamente (97% de confianza media sobre este
corpus). Pero quedaban dos huecos, y los dos se cerraron con la geometría que el
propio servicio devuelve:

**1. Las viñetas no son texto.** El «•» de una ficha es un adorno tipográfico,
no una casilla de selección, así que Textract devuelve la celda vacía y la fila
entera desaparece. Se detectan midiendo la tinta dentro del recuadro de cada
celda: una celda marcada tiene una mancha oscura pequeña, una vacía no tiene
ninguna. Contrastado contra la imagen fila por fila, la separación es limpia —
0.005 de píxeles oscuros frente a 0.000— así que el umbral no está en el filo de
nada.

**2. Un valor centrado vuelve partido entre columnas.** «108 HP @ 4,500 rpm»,
escrito una sola vez en mitad de la tabla, llega troceado en dos celdas
contiguas. Aquí hubo una iteración que merece anotarse: el primer intento agrupó
por el hueco entre **celdas** y no separó nada, porque los recuadros que Textract
da para una celda son los de la rejilla y las celdas contiguas se tocan. Los
huecos hay que medirlos entre las **palabras**, y ahí la separación es de dos
órdenes de magnitud: 0.002 dentro de un mismo valor, 0.19 entre dos valores
distintos.

Y para saber a qué columna pertenece cada grupo, una regla de dos distancias: se
compara la distancia del valor al centro del área de columnas con su distancia a
la columna más cercana. «4,113» escrito en mitad de la tabla queda a 0.014 del
centro del área y a 0.049 de cualquier columna: significa «igual en todas las
versiones». «MT (5 vel.)» escrito bajo las manuales queda más cerca de su
columna. La primera versión de esa regla exigía que el valor ocupara más de una
celda, y por eso atribuía `Largo (mm): 4,113` a una sola versión — un error de
atribución, es decir, del tipo exacto que había que eliminar.

**Por qué no un modelo de visión.** Habría leído la página entera sin esfuerzo,
viñetas incluidas. Se descartó por una razón de principio: el proyecto se
sostiene sobre que cada afirmación sea rastreable hasta un documento, y un modelo
que *transcribe* puede completar una cifra ilegible con una plausible. Un error
de OCR es visible y reproducible; una alucinación en la capa de extracción es
indistinguible de un dato real y contamina el corpus de forma permanente. La
capa que fabrica la evidencia es el último sitio donde conviene poner algo que
inventa.

### Tres guardas que salieron de ejecutar el lote de verdad

Nada de esto apareció en el diseño; apareció al lanzar 130 páginas seguidas.

1. **Los diagnósticos se perdían justo al fallar.** El reporte solo propagaba
   los avisos de una transcripción con texto, así que un lote entero podía
   volver vacío diciendo «no dio resultado» sin nombrar la causa. La causa era
   `ConnectionClosedError`: las operaciones síncronas de Textract tienen un
   límite de transacciones por segundo bajo y cien páginas seguidas lo superan.
   Se arregló propagando siempre los avisos y configurando reintentos
   adaptativos, que reducen el ritmo cuando el servicio lo pide en vez de dormir
   a ciegas entre páginas.
2. **Un resultado parcial se cacheaba.** Dos fichas quedaron con su primera
   página perdida y guardadas así, lo que convierte un fallo de red pasajero en
   la versión definitiva del documento: nadie volvería a mirarlo. Ahora una
   transcripción incompleta no se cachea nunca.
3. **La densidad como criterio de evidencia.** Los siete PDF de Renault se
   transcribieron sin error y produjeron 900 caracteres desde catorce páginas.
   Mirándolos: no son fichas técnicas, son un folleto de ubicaciones de
   cargadores, idéntico entre modelos. El OCR hizo bien su trabajo; no hay más
   ahí. Indexarlos habría metido siete documentos casi idénticos que solo pueden
   ganar una recuperación por error y hacer que el agente cite
   «ficha-tecnica-renault-duster» para hablar de cargadores. Una transcripción
   por debajo de 200 caracteres por página se descarta y se dice por qué. El
   umbral vuelve a caer en mitad de dos órdenes de magnitud: 7.500-10.000
   caracteres por página cuando hay contenido, 60 cuando no.

También hay caché, y no como optimización: con un motor que cobra por página,
ajustar el troceado y relanzar `make corpus` volvería a pagar el corpus entero.
La clave va por contenido del archivo, así que un PDF que cambia se retranscribe
y uno que solo se movió de carpeta no. Y antes de gastar nada, el script enseña
cuántas páginas va a transcribir y cuánto cuesta.

### El resultado

| | Antes | Después |
|---|---|---|
| Documentos indexados | 101 | 123 |
| Fragmentos | 1.053 | 1.244 |
| PDF descartados en silencio | 30 | 0 |

Rescatados: 6 por descifrado, 16 por transcripción con estructura. Descartados a
propósito y con motivo: 7 (no eran fichas) y 1 (archivo vacío).

La prueba de que la rejilla sobrevivió el viaje entero —imagen, Textract,
geometría, fragmento, recuperación, cita— es la respuesta a «¿qué versiones del
MG3 llevan quemacocos?»:

> Quemacocos eléctrico de un solo toque: disponible **solo en la versión EXCITE
> CVT** [ficha-tecnica-mg-mg3--002.md]. Potencia: 108 HP @ 4,500 rpm, que
> aplicaría a las versiones COOL MT, STYLE MT, STYLE CVT y EXCITE CVT
> [ficha-tecnica-mg-mg3--001.md].

Las dos afirmaciones son correctas contra el PDF original, y la diferencia entre
ellas —una vale para una versión, la otra para las cuatro— es justo lo que un
OCR lineal habría borrado.

Cada fragmento lleva además de dónde salió su texto: `origen_texto: ocr:tablas`
con su confianza, o `cifrado_original: true`. Una cita que procede de una
transcripción automática no vale lo mismo que una del original, y quien audite
la respuesta tiene derecho a saberlo sin abrir el PDF.

---

## 18. ¿Sirve esto para cualquier PDF? La respuesta honesta era «no»

La pregunta era si el pipeline vale para cualquier corpus. Contestarla exigía
mirar el código y no la intención, y salieron tres niveles de problema. El
primero fue el peor, y estaba en la parte que parecía más inocente.

### El código que borraba las cifras

`limpiar()` tenía dos heurísticas escritas pensando en folletos: descartar
números sueltos —folios— y colapsar líneas idénticas consecutivas —pies
repetidos—. Sobre una ficha de coche no cuestan nada. Sobre un estado de
resultados extraído con etiqueta y valor en líneas separadas, que es como muchos
PDF disponen sus tablas:

```
ANTES:   Ingresos · 125 · Costos · 87 · Margen · 38 · Provisiones · 0 · 0 · Impuestos · 12
DESPUÉS: Ingresos · Costos · Margen · Provisiones · Impuestos
```

El documento sobrevive con todas sus etiquetas y **ninguna de sus cifras**. Y no
lo detecta nada: la guarda de densidad ve caracteres de sobra en las etiquetas, y
la comprobación de si hace falta OCR usa la misma función, así que el documento
tampoco se manda a transcribir. Entra al corpus completo, parece bien, y no puede
responder una sola pregunta.

Sobre el corpus de coches el daño era cero. Ese es justo el problema: una
heurística que no cuesta nada en el corpus donde se escribió y borra los datos en
el siguiente.

**El arreglo fue mirar las dos señales que sí distinguen basura de dato, y las
dos son posicionales.** `limpiar()` ya recibía las páginas por separado y tiraba
esa información al aplanarlas:

- Un folio está en el **borde** de su página **y forma una serie creciente**. La
  posición sola no basta: un balance puede acabar una página en «0». Exigir la
  serie cuesta cuatro líneas y elimina el falso positivo entero.
- Un pie corrido se repite **a lo largo del documento**, en el borde, y no es un
  número. Que dos líneas estén seguidas no dice nada.

Del pie corrido se conserva además la primera aparición. Suele ser el título o el
modelo —«MAZDA2 SEDÁN 2026»— y borrarlo de todas partes deja los fragmentos del
medio del documento sin decir de qué hablan. Una vez basta; cuatrocientas treinta
y nueve —las del manual del Corvette— no.

Medido sobre el corpus real: 55 documentos ganan texto (datos que antes se
perdían) y 15 pierden (repetición de verdad), con 31.000 caracteres netos menos
de ruido.

### El intérprete que solo entendía una forma de tabla

El render de tablas asumía fila 1 = encabezados y columna 1 = etiqueta, y sobre
esa premisa **redactaba afirmaciones**: «solo en X», «vale para todas». Eso es
correcto en una ficha comparativa y es inventarse el significado de la rejilla en
cualquier otra. Un balance con encabezados de dos niveles habría recibido frases
perfectamente redactadas y falsas. Además el módulo decía literalmente
«versiones» — vocabulario de coches incrustado en código genérico, que en una
tabla trimestral diría «en todas las versiones (Q1, Q2)».

Ahora la tabla tiene que **demostrar su forma** antes de que se interprete:
columna de etiquetas, dos o más encabezados y una fracción alta de celdas de
valor vacías o marcadas. Una tabla de datos —casi todas las celdas llenas— no
pasa el filtro, que es exactamente lo que se busca. Sin esa forma se hace un
volcado sin pérdida donde cada fila lleva sus encabezados dentro, para que un
corte de troceado no pueda dejar una cifra huérfana de su columna.

Al probarlo sobre el corpus apareció un caso que no estaba en el diseño: Textract
marca como TABLE cualquier maquetación en rejilla, incluida una plana de folleto
con tres bloques de texto en fila. Tomar su primera fila como encabezados
producía cosas como «1030 km de autonomía combinada: 3 filas de asientos» — un
emparejamiento entre dos textos que en el documento no tienen ninguna relación.
No es un formato feo: es una afirmación falsa. Un encabezado de verdad es una
etiqueta corta, y comprobarlo bastó.

### La detección de viñetas dejó de dar por hecha su calibración

Los umbrales de tinta se midieron sobre un corpus. Funcionan ahí; en otro, un
icono pequeño o un sombreado se leen como marca y el agente afirma una
característica que el documento no afirma — el error en la dirección peligrosa.

Ahora el umbral se calibra **por tabla**: se miden todas las celdas sin texto y
se exige una separación bimodal, un hueco ancho entre el grupo «vacías» y el
grupo «marcadas». En las fichas de coches ese hueco es de 0.000 frente a 0.005 y
la detección sigue igual de fina; lo que cambia es que ahora se comprueba en cada
documento en vez de darse por supuesto. Sin hueco claro no se emite ninguna
marca. Y una mancha solo cuenta si es pequeña y centrada: un sombreado que cubre
la celda no es una viñeta.

### Lo que sigue sin ser genérico

Conviene decirlo con la misma claridad. Solo se pide `TABLES`: sin `LAYOUT` no
hay orden de lectura garantizado en prosa a dos columnas —el formato de un
informe de análisis— y en una página mixta las líneas sueltas salen antes que las
tablas. No hay `FORMS` para facturas. El idioma no se valida contra lo que el
motor soporta. Y un gráfico transcrito devuelve las etiquetas de sus ejes, que
parecen dato sin serlo.

Está anotado y no disimulado: son las fases que se abordarán cuando se sepa qué
forma tienen los PDF del corpus de inversiones.

### El precio de una decisión anterior

El caché guarda el **texto ya redactado**, no la respuesta cruda del motor. Cada
cambio en el render obliga a subir la versión del formato y a volver a
transcribir: dos euros y unos minutos por iteración. Se pagó dos veces en este
trabajo. Guardar la respuesta cruda haría gratis iterar sobre el render, a cambio
de bastante disco; queda anotado como la mejora que hará falta si las fases
pendientes tocan otra vez esta capa.

### Resultado

Sin regresión: `coches` sigue en 9/10 y `luis-cv` en 7/7 contra el modelo real. El
MG3 sigue diciendo que el quemacocos es **solo de la versión EXCITE CVT** y que
los 108 HP valen para las cuatro. El corpus pasa de 1.244 a 1.211 fragmentos, y
esos 33 de menos son repetición de encabezados que antes se indexaba.

Lo que cambió no se ve en los números: cambió qué pasa cuando llegue un PDF que
nadie ha mirado.

## 19. El corpus de inversiones: cuando el PDF sí se deja leer y aun así se pierde

La §18 cerró con una promesa: las fases pendientes se abordarían «cuando se sepa
qué forma tienen los PDF del corpus de inversiones». Llegaron —veinticuatro
reportes trimestrales a inversionistas de GFNorte— y resultaron ser el caso
contrario al que esta ingesta sabía manejar.

### La premisa que se rompió

Toda la ruta de extracción se apoyaba en una equivalencia que nunca se escribió
pero estaba en el código: **«necesita el motor de tablas» ⇔ «no tiene capa de
texto»**. Es cierta para una ficha de coche escaneada. Es falsa para un informe
financiero nativo.

`extractors.py` invocaba el motor así:

```python
if len(texto) < min_chars and ocr is not None:   # min_chars = 200
```

Ese umbral mira el **documento entero**. Un trimestral de Banorte da ~258.000
caracteres de capa de texto. La condición no se cumplía jamás, así que las 562
líneas de `TextractOcr` —el módulo que existe precisamente para conservar la
rejilla— no llegaban a ejecutarse ni una vez sobre este corpus.

Nadie lo habría notado mirando la salida, y esa es la parte incómoda: `pypdf`
extrae el texto «bien». Lo que devuelve de una tabla de resultados es esto:

```
Banco Mercantil del Norte
11,761
11,705
12,391
6%
```

Una columna de cifras sin fila ni encabezado. No es un fallo visible como una
página en blanco: parece texto legítimo, se indexa como tal, y el agente puede
citar «11,705» sin poder decir de qué periodo es. Es exactamente el daño que la
§18 se propuso evitar en las fichas comparativas, reaparecido por otra puerta.

### Medir antes de diseñar

Antes de escribir nada se instrumentó el corpus. Los números decidieron casi
todo lo que vino después:

| Medición | Resultado |
| :--- | :--- |
| Páginas de `2T26.pdf` | 85 |
| Páginas sin capa de texto | **0** — todo nativo, mediana 3.040 caracteres |
| Páginas con tabla | **84 de 85** |
| Páginas con figura relevante | 3 de 85 (4%) |
| Dónde viven las tablas | En la capa de texto |
| Dónde viven las gráficas | Imágenes rasterizadas, fuera de la capa de texto |

Esa última fila es la que separa las dos herramientas. Las tablas se pueden
recuperar porque el dato está ahí y solo falta la estructura. Las gráficas no:
`get_text_range()` no devuelve ni «220» ni «+7.6%», porque son píxeles.

### Lo que se descartó, y por qué

Cuatro ideas llegaron a estar sobre la mesa y ninguna sobrevivió al contacto con
los datos. Se dejan escritas porque descartarlas fue el trabajo.

**Enrutar por página para ahorrar.** Era la petición inicial: detectar qué
páginas llevan tabla y mandar solo esas. Se implementó el clasificador y se
midió sobre el documento real: **84 de 85 páginas** llevan tabla. El ahorro del
enrutamiento es de **0,01 USD** por documento. La idea era buena y el corpus la
contradijo.

Lo que sí ahorró fue mucho más simple: **quitar `FORMS`**. Cuesta 0,05 USD por
página contra 0,015 de `TABLES`, y en un informe financiero produce pares como
`(Millones de: Pesos)` — ruido con forma de dato. Sobre las ~2.040 páginas del
lote, el bill pasa de **132,60 a 30,60 USD**. Una línea de configuración valía
cien veces más que todo el sistema de enrutamiento.

La lección no es que medir fuera opcional: es que **el clasificador se pagó solo
por decir que no hacía falta**, y en cuatro minutos de cómputo local gratuito.

**Seleccionar páginas por defecto.** Se descartó como modo recomendado. Un
detector con recall imperfecto pierde una tabla en silencio, y una tabla perdida
en un reporte financiero no se nota hasta que alguien cita una cifra que no
existe. El perfil de Banorte usa `paginas: todas`. El modo `con-tablas` quedó
implementado y disponible, pero para corpus mayoritariamente narrativos.

**`pdfplumber` como dependencia.** Detectar una tabla *sin bordes* exige un
analizador de layout. Se midió el atajo: usar solo las reglas vectoriales que
pypdfium2 ya da. Recall **0,83** — se perdía la página de índice, que es una
tabla real con **cero** líneas dibujadas. Como no era aceptable pero tampoco lo
era arrastrar `pdfminer.six` a un proyecto que documentó su elección de
`pypdfium2` sobre PyMuPDF por licencia, quedó como **extra opcional**: si no está
instalada, `con-tablas` manda **todas** las páginas. Falla hacia pagar de más,
nunca hacia perder una tabla. PyMuPDF se descartó sin discusión: AGPL-3.0 contra
un proyecto Apache-2.0.

**Tablas Markdown como formato de fragmento.** Se consideró emitir la rejilla en
Markdown. Se descartó por lo que dice la §3 del docstring de `chunking.py`: una
tabla partida por el troceo deja filas sin cabecera. El formato que ya usaba
`_render_tabla` —una frase autocontenida por fila— sobrevive al corte por
diseño, porque cada fragmento se explica solo.

### Cómo se corta, y por qué así

La extracción y el troceo son una sola decisión tomada en dos sitios. Conservar
la rejilla al extraer y luego cortarla por la mitad al trocear deshace el
trabajo; el troceado ya lo contemplaba y el perfil solo tuvo que afinarlo.

`chunking.py` corta por **costuras**: primero límite de sección, luego de
párrafo, luego de frase; solapa para que el dato de la costura aparezca entero
en uno de los dos lados; y **repite la cabecera** cuando parte una tabla.

El perfil `banorte` aprieta los números respecto a `autos`:

| | `autos` | `banorte` |
| :--- | ---: | ---: |
| `max_chars` | 2.000 | **1.800** |
| `overlap_chars` | 250 | **200** |
| `min_chars_to_split` | 2.600 | **2.400** |

Fragmentos más cortos, y la razón es la forma del contenido, no una preferencia.
Una fila de tabla financiera ya es autocontenida —`Utilidad Neta: 14.618 (2T25);
15.550 (2T26)`— así que un fragmento grande no gana contexto: mezcla indicadores
que nadie pregunta juntos y diluye la recuperación. En una ficha de coche, en
cambio, el equipamiento de una versión se entiende leyendo varias filas seguidas.

Se cambió también `columnas: periodos`. El propio comentario de `autos.yaml` lo
había anticipado: «en una ficha de coches son *versiones*, en una tabla
trimestral *periodos*». Decir «versiones» de una columna `2T26` es sencillamente
falso, y esa palabra viaja dentro de cada fila redactada.

### Tres trampas que una revisión de diseño encontró antes que la ejecución

Ninguna habría dado error. Las tres habrían corrompido el resultado en silencio:

1. **La respuesta síncrona de Textract siempre dice `Page: 1`.** Con una
   selección salteada, eso no son las páginas del documento: sin renumerar,
   todas las figuras del PDF se amontonan en la página 1 y las anclas del visor
   apuntan al sitio equivocado.
2. **`FigureRegion.is_meaningful` exige `texts` no vacío.** Una figura detectada
   localmente nace sin texto OCR, así que habría desaparecido del sidecar y el
   paso de visión nunca la habría visto.
3. **pdfium mide desde abajo y Textract desde arriba.** Sin `Top = 1 - (top /
   alto)` los recortes salen espejados en vertical.

También se blindó el caché: la clave incluye ahora la selección de páginas. Sin
eso, una prueba acotada a tres páginas dejaba cacheado un documento a medias
como si fuera la versión definitiva —el mismo error que la §18 pagó dos veces
por otra vía.

### El tiempo, que era la restricción real

El corpus llegaría con una ventana de tres horas para procesarlo. Medido: **5,4
segundos por página** en secuencia, es decir **más de tres horas** para 2.040
páginas. No cabía por poco, que es la peor forma de no caber.

Cada página es una llamada HTTP independiente y casi todo el tiempo es espera de
red, así que paralelizar es repartir espera, no cargar la máquina. Con ocho
peticiones en vuelo, las 7 páginas de prueba pasaron de **38 a 13 segundos**.
Proyección del lote: **~63 minutos**.

El orden del resultado es el de entrada y no el de llegada. Ordenar por quién
respondió antes mezclaría el texto de una página con el número de otra, que es
otra forma de corromper en silencio.

### La excepción de visión, escrita para que no sea una incoherencia

El docstring de `ocr/__init__.py` rechaza explícitamente los modelos de visión:
«un modelo que *transcribe* puede completar una cifra ilegible con una
plausible... contamina el corpus de forma permanente». La regla es correcta y se
mantiene.

La excepción es acotada: la visión **no sustituye extracción de texto en ningún
caso**. Entra solo donde la alternativa es *nada* —una gráfica rasterizada, donde
Textract devuelve `220 210 200 (8.5%)` sin relación entre sí— y entra marcada,
con `origen_texto` y su confianza en los metadatos del fragmento.

La validación fue medible: sobre la gráfica anual de la acción, el modelo leyó
**172** al inicio y **185** al cierre; la tabla del propio documento publica
**171,52** y **184,57**. Y se autocalificó `confianza: baja` porque la serie no
tiene etiquetas de dato. Ese reconocimiento del error posible es lo que hace la
excepción aceptable: la cifra viaja marcada como estimación, no como dato.

### Lo que costó equivocarse

El perfil se escribió con `source: corpus`, sin advertir que esa carpeta es
compartida. El primer lanzamiento empezó a mandar el corpus de coches a Textract
con `paginas: todas` antes de que se cortara: **26 páginas, 0,39 USD**. Barato
como incidente y caro como recordatorio: un perfil que apunta a un directorio
compartido es una factura esperando a ocurrir.

### La UI también aplanaba las tablas

Recuperar la rejilla al extraer no sirve si la respuesta se pinta con
`textContent`. Una tabla llegaba al navegador como una reja de pipes y las
negritas como asteriscos.

Se escribió un renderizador propio, sin dependencias, que construye nodos con
`createElement` y asigna texto con `textContent`. **Nunca `innerHTML`**, y por eso
no se usó una librería: casi todas emiten una cadena de HTML, y entonces hace
falta un sanitizador y confiar en que esté bien configurado. Aquí la seguridad no
depende de una lista de permitidos sino de que **la ruta que ejecuta HTML no
existe**. Dos pruebas lo fijan inyectando `<img onerror>` en un párrafo y dentro
de una celda.

Se descartó renderizar en cada delta del stream: reparsear por token hace
parpadear la tabla fila a fila mientras llega. Se renderiza una vez, al cerrar.
Y se descartaron los enlaces: las citas van como `[2T26.pdf]` y un parser de
enlaces empezaría a interpretarlas.

Los fragmentos de fuentes siguen en `textContent` a propósito: son evidencia
verbatim, y renderizarlos cambiaría lo que el documento dice.

### Resultado

Los 272 tests del núcleo siguen en verde; la UI pasa 17. Sobre el extracto de
prueba, el clasificador acertó las 7 páginas contra la corrida real de Textract
—recall y precisión 1,000— y el documento produjo 18 fragmentos con
`origen_texto: "capa de texto + tablas (7/7 páginas)"` y confianza 95,4.

La tabla que antes era una columna de cifras huérfanas ahora dice:

```
Banco Mercantil del Norte | 11,761 | 11,705 | 12,391 | ... | 22,793 | 24,095
```

### Lo que sigue sin estar resuelto

Con la misma claridad que la §18. `LAYOUT` ya se pide junto a `TABLES` —es
gratis en esa combinación— pero **las regiones de figura todavía no se aíslan del
texto corrido**: un fragmento puede contener aún `220 210 200 (8.5%) +7.6%`, las
etiquetas de eje de una gráfica, indistinguibles de cifras financieras para el
índice. Es la contaminación que este trabajo identificó y no alcanzó a cerrar.

Los encabezados de dos niveles tampoco se colapsan: `Var. Vs.` sobre `1T26` sale
como `Var.: 1T26 | Vs.: 2T25` en vez de `Var. Vs. 1T26`.

Y `con-tablas` está implementado pero **no validado a escala**: su puerta de
calidad —recall ≥ 0,99 sobre una muestra calibrada contra Textract— no se ha
ejecutado, porque el corpus que lo motivaba resultó no necesitarlo.

---

## Conclusión

Cada decisión tomada buscó el equilibrio entre **rigor técnico bancario** y
**velocidad de entrega**. Este diseño no solo resuelve el reto: crea un modelo
base auditable, seguro y profesional.

Y dejó de ser un diseño. El agente está **desplegado en AWS**, responde citando
los documentos que sustentan cada afirmación, y las dos pruebas que el contrato
declaró decisivas —streaming sin buffering tras el balanceador y cero
credenciales inventadas— pasan contra el despliegue, no solo en local.

Lo que queda pendiente está en la §1 de este documento en forma de lista, no
escondido: HTTPS en el balanceador, un token con caducidad real, el guardrail
administrado y las trazas distribuidas. Ninguno es un descubrimiento tardío.

Si algo resume el proyecto es la sección 15: una optimización que parecía
obviamente buena rompió una respuesta correcta, y la suite de evaluación lo
detectó antes que nadie. Esa es la diferencia entre un sistema que afirma ser
confiable y uno que puede demostrarlo.
