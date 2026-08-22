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