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

## Conclusión
Cada decisión tomada buscó el equilibrio entre **rigor técnico bancario** y **velocidad de entrega**. Este diseño no solo resuelve el reto, sino que crea un modelo base auditable, seguro y profesional, listo para ser escalado a producción.

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


## 8. Bedrock en local: lo que solo se aprende contra la API real

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

## 9. Comparación entre familias de modelo

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


## 10. Infraestructura: renunciar al NAT Gateway a propósito

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


## 11. Decisión B: el vector store, o cómo no pagar 350 USD al mes por diez documentos

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


## 12. El despliegue: cuatro fallos que solo existen en AWS

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

## 13. Resultados finales contra el despliegue

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
