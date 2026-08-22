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
