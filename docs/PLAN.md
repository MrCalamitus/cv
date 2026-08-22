# Plan de Construcción — `luis-cv`

Agente RAG compatible con Open Responses sobre AWS Bedrock + ECS Fargate.

> Documento operativo. Se ejecuta de arriba hacia abajo. Cada etapa tiene un
> criterio de salida binario: si no se cumple, no se avanza a la siguiente.
> Documento hermano: `contrato-open-responses.md` (contrato normativo y suite
> de aceptación). Este plan construye; ese documento define qué es correcto.

---

## Decisiones cerradas

Ya no se discuten. Cambiar alguna implica revisar este plan.

| # | Decisión | Fuente |
|---|---|---|
| 1 | Sin frontend. Solo endpoint | Bitácora §1 |
| 2 | Motor: Amazon Bedrock, no OpenRouter | Bitácora §2 |
| 3 | Cómputo: ECS Fargate + ALB | Bitácora §3 |
| 4 | Sin LangChain/LlamaIndex. `boto3` + Bedrock KB | Bitácora §4 |
| 5 | Observabilidad: CloudWatch + X-Ray desde el día uno | Bitácora §5 |
| 6 | Contrato: Open Responses 2026-04-24, superset conforme | Contrato §0 |
| 7 | RAG expuesto como herramienta hospedada `agente:knowledge_search` | Contrato §4 |
| 8 | `store: false`, sin `previous_response_id`, retención cero | Contrato §7 |
| 9 | Corpus: CV, título, cédulas, constancias de cursos, notas MD | — |
| 10 | Modelos con acceso concedido: familia Anthropic y familia GPT en Bedrock | — |
| 11 | Perfil de despliegue: `luis`, como variable con default | — |
| 12 | Prefijo y tag de toda la infra: `luis-cv` | — |

## Decisiones abiertas (bloquean etapas concretas)

| # | Decisión | Bloquea | Cuándo se resuelve |
|---|---|---|---|
| A | Terraform vs CDK | E0 | Antes de escribir IaC. **Default asumido: Terraform** |
| B | Vector store de la KB | E3 | Al crear la KB. Ver nota de costo en E3 |
| C | Las 20 preguntas de oro | E7 | Cuando el corpus esté cargado |
| D | Fecha límite de entrega | Alcance de E6–E8 | Ahora mismo, idealmente |

---

## Estructura del repositorio

```
luis-cv/
├── README.md                      # cómo desplegar y cómo probar
├── BITACORA.md                    # se actualiza al cerrar cada etapa
├── docs/
│   ├── contrato-open-responses.md # el contrato normativo
│   └── arquitectura.md            # capas, puertos y adaptadores
├── src/luis_cv/                   # arquitectura hexagonal
│   ├── domain/                    # reglas puras: prompts, redacción, eventos
│   ├── application/               # casos de uso y puertos
│   ├── infrastructure/
│   │   ├── inbound/http/          # FastAPI, esquema, SSE, auth, rate limit
│   │   └── outbound/              # bedrock/ · local/ · telemetry/
│   └── main.py                    # uvicorn luis_cv.main:app
├── tests/
│   ├── contract/                  # casos A y B del contrato
│   ├── rag/                       # casos C
│   ├── operation/                 # casos D
│   ├── unit/                      # dominio, adaptadores y guardas de capas
│   ├── deployed/                  # la misma aceptación contra el ALB
│   └── golden.yaml                # preguntas de oro + respuestas esperadas
├── infra/
│   ├── main.tf  variables.tf  outputs.tf  locals.tf
│   ├── network.tf                 # VPC, subredes, VPC endpoints
│   ├── ecs.tf  alb.tf  ecr.tf     # cómputo y borde
│   ├── bedrock.tf                 # KB, data source, guardrail
│   └── observability.tf           # log groups, métricas, dashboard, alarmas
├── corpus/                        # documentos fuente (NO se sube a git)
├── scripts/
│   ├── deploy.sh                  # guarda de cuenta + build + push + apply
│   ├── sync-kb.sh                 # subir corpus e iniciar ingesta
│   ├── eval.py                    # preguntas de oro y reporte comparativo
│   └── smoke.sh                   # verificación rápida contra el desplegado
├── Dockerfile
├── Makefile
└── .gitignore                     # corpus/, .env, *.tfstate, *.tfvars
```

La estructura plana `app/*.py` del boceto inicial se sustituyó por capas
(`domain` / `application` / `infrastructure`). El motivo está en
`arquitectura.md`: con la ingesta del RAG pendiente, poder sustituir la
recuperación y la inferencia por adaptadores locales es lo que permite tener el
contrato verificado antes de que exista la Knowledge Base.

`corpus/` fuera de git no es negociable: son documentos de identidad. Que estén
en un repo, aunque sea privado, es una superficie de exposición innecesaria.

---

## Etapa 0 — Fundaciones

**Objetivo:** que exista una cuenta segura, un repo con forma y una guarda que
impida desplegar en el lugar equivocado.

1. Repo con la estructura de arriba y `.gitignore` correcto **antes** del
   primer commit. Un documento de identidad commiteado no se borra del
   historial con un `rm`.
2. Perfil `luis` verificado: `aws sts get-caller-identity --profile luis`.
   Anotar el account ID; va a `terraform.tfvars`.
3. Presupuesto de AWS con alerta al 50 % y al 80 %.
4. `locals.tf` con el bloque de nombres y tags (`local.name`, `local.tags`).
5. Provider con `profile = var.aws_profile` y `allowed_account_ids`.
6. `scripts/deploy.sh` con la guarda de cuenta y de sesión SSO.

**Salida:** `terraform plan` corre limpio con cero recursos y falla en seco si
se apunta a otra cuenta. Probarlo a propósito con otro perfil.

---

## Etapa 1 — Endpoint en local, sin RAG

**Objetivo:** el contrato funciona antes de que exista infraestructura.

1. `schema.py` — todo el §2 del contrato en Pydantic, incluida la
   normalización de `input` como string y el rechazo de `store: true`.
2. `models.py` — mapa de alias, validado al arranque contra los modelos con
   acceso concedido.
3. `errors.py` — la tabla del §5, con supresión de detalles internos.
4. `events.py` — el corazón. Emisor SSE con `sequence_number` monotónico,
   `event:` espejo del `type`, sin campo `id:`, terminal `[DONE]` literal.
5. `inference.py` — `converse_stream()` de `bedrock-runtime`, traducido a la
   secuencia canónica del §4.
6. `main.py` — rutas y modo no-streaming (agrega el stream y responde JSON).

**Salida:** pasan los casos A1–A10 y B1–B7 en local. B6 (concatenar los deltas
reproduce exactamente el `output_text.done`) es el que prueba que el emisor
está bien; si falla, nada río abajo sirve.

**Nota:** trabajar contra los dos alias, Anthropic y GPT, desde el principio.
Las diferencias entre familias en `converse_stream` aparecen temprano y son
baratas de absorber ahora; encontrarlas en E6 duele.

---

## Etapa 2 — Preparación del corpus

**Objetivo:** que los documentos estén listos antes de que la KB exista, porque
la ingesta es un tiempo de espera que conviene solapar.

1. Inventario en `corpus/manifiesto.csv`: archivo, tipo (`cv`,
   `titulo`, `cedula`, `curso`, `nota`), año, emisor, si contiene PII.
2. Verificar que los PDF tienen **capa de texto**. Un título escaneado sin OCR
   es una imagen: la KB no extrae nada y el agente responderá que no consta.
   Comprobar con `pdftotext archivo.pdf -` sobre cada uno.
3. OCR de los que no la tengan (`ocrmypdf`), o transcripción manual a MD — con
   pocos documentos, transcribir es más rápido y más fiable que pelear con OCR.
4. Normalizar nombres: `tipo-descripcion-anio.pdf`. El nombre del archivo
   termina en el `document_id` que el agente cita; que sea legible importa.
5. Metadatos por documento en `<archivo>.metadata.json` (formato de Bedrock
   KB), con `tipo`, `anio`, `institucion`. Habilita filtros en la
   recuperación y hace las citas mucho más útiles.

**Salida:** todo documento del manifiesto rinde texto extraíble y tiene su
archivo de metadatos.

**Decisión de chunking:** un CV es corto y denso; partirlo en fragmentos de 300
tokens destroza la relación entre puesto, empresa y fechas. Usar chunking
jerárquico o directamente **un documento = un fragmento** para títulos, cédulas
y constancias, que son cortos por naturaleza. El default agresivo de la KB es
la causa número uno de RAG malo sobre corpus pequeños.

---

## Etapa 3 — Knowledge Base

**Objetivo:** recuperación funcionando y auditable.

1. Bucket `luis-cv-corpus-<account_id>`, cifrado, acceso público bloqueado,
   versionado activo.
2. **Decidir el vector store (decisión B) antes de crear nada.** OpenSearch
   Serverless cobra un mínimo de OCUs corriendo de forma continua, existan o no
   consultas. Para un corpus de decenas de archivos pequeños, ese piso puede
   ser el mayor gasto del proyecto. Alternativas: Aurora Serverless con
   pgvector, o Pinecone. Comparar el costo mensual estimado y **escribir la
   comparación en la bitácora** — es exactamente el tipo de análisis que el
   reto premia.
3. Crear la KB (`luis-cv-kb`), data source apuntando al bucket, estrategia de
   chunking de E2.
4. Ingesta. Verificar en la consola que cada documento produjo fragmentos, no
   cero.
5. `retrieval.py` — `retrieve()`, no `retrieve_and_generate()`. La segunda
   quita el control del prompt y del formato de los eventos que ya se construyó
   en E1.
6. Emitir el ítem `agente:knowledge_search` con `queries`, `results`
   (`document_id`, `chunk`, `score`, `metadata`) y `latency_ms`.

**Salida:** el ítem de recuperación aparece en la respuesta con el
`document_id` correcto para tres preguntas de prueba.

---

## Etapa 4 — Prompt, veracidad y redacción

**Objetivo:** que el agente no invente credenciales. Es el requisito de
producto más importante del proyecto.

1. `prompts.py` — sistema con reglas duras: responder solo desde los
   fragmentos; citar `document_id` en toda afirmación sobre una credencial; si
   la evidencia no alcanza, decir que no consta.
2. `redaction.py` — enmascarado de cédula, CURP y RFC en la salida (§6.2 del
   contrato) y en los logs.
3. Guardrail de Bedrock `luis-cv-guardrail`: filtro de PII, tema prohibido de
   suplantación de identidad, y verificación de fundamentación contextual
   contra alucinaciones.
4. Probar los adversariales: certificación inexistente, año equivocado,
   institución equivocada, inyección de prompt en el turno del usuario.

**Salida:** pasan C2, C3, C6 y C7. **C3 con cero tolerancia**: una sola
credencial inventada invalida la etapa.

---

## Etapa 5 — Contenedor y despliegue

**Objetivo:** que lo de E1–E4 corra en AWS con el streaming intacto.

1. Dockerfile multi-etapa, usuario no-root, `HEALTHCHECK` a `/healthz`.
2. `ecr.tf` — repo `luis-cv/api` con escaneo de imágenes al subir.
3. `network.tf` — VPC, subredes públicas y privadas, y **VPC endpoints
   (PrivateLink) para `bedrock-runtime`, `bedrock-agent-runtime`, S3, ECR,
   Secrets Manager y CloudWatch Logs**.

   Esto no es opcional: la bitácora descartó OpenRouter por no exponer datos a
   internet público. Sin los endpoints, la llamada de Fargate a Bedrock sale
   por NAT a internet y el argumento se cae solo. Con ellos, el tráfico nunca
   abandona la red de AWS y la §2 de la bitácora queda respaldada por la
   topología, no solo por la narrativa.
4. `alb.tf` — ALB, target group (`name_prefix`, no `name`), HTTPS,
   **`idle_timeout = 120`**.
5. `ecs.tf` — cluster, task definition, servicio en subredes privadas, rol de
   tarea con permisos mínimos: `bedrock:InvokeModelWithResponseStream`,
   `bedrock:Retrieve` sobre el ARN de la KB, `bedrock:ApplyGuardrail`. Nada de
   comodines.
6. Autenticación: token en Secrets Manager, inyectado como secreto de la task.
7. Límite de tasa en la aplicación.

**Salida:** el smoke test pasa contra la URL del ALB, y **B8 pasa**: los deltas
llegan espaciados en el tiempo, no todos juntos al final. Verificarlo en el
primer despliegue, no al final — el buffering silencioso entre el proxy y el
ALB es el modo de falla clásico de SSE y no se ve en local.

---

## Etapa 6 — Observabilidad

**Objetivo:** convertir la §5 de la bitácora en artefactos verificables.

1. Logs JSON estructurados con `request_id`, alias de modelo, conteo de
   fragmentos, tokens, latencias. **Sin texto del turno ni PII.**
2. X-Ray con subsegmentos explícitos y separados: `retrieval` e `inference`.
   Es la traza que demuestra el argumento de auditoría del RAG.
3. Métricas propias: `TimeToFirstToken`, `RetrievalLatency`, `ChunksRetrieved`,
   `GroundingFailures`, `ErrorRate`.
4. Dashboard `luis-cv-prod` y una alarma sobre p95 de TTFT.

**Salida:** pasan D1–D7. D5 (ningún log contiene PII) se verifica leyendo los
logs reales, no asumiendo.

---

## Etapa 7 — Evaluación

**Objetivo:** medir en vez de afirmar.

1. Escribir las 20 preguntas de oro en `tests/golden.yaml` (decisión C). Deben
   ser las preguntas de un reclutador o un evaluador, no las cómodas. Incluir
   entre 5 y 7 negativas: cosas que **no** están en el corpus.
2. Runner que ejecuta la suite contra el despliegue y emite un reporte con
   fundamentación, cita correcta, TTFT y latencia total.
3. Correr con los tres alias de modelo y tabular. La comparación entre familias
   es material de primer nivel para la bitácora, y es justo lo que el §2
   original quería lograr con OpenRouter — pero conseguido dentro del
   perímetro de Bedrock.

**Salida:** reporte con 100 % en las negativas y ninguna credencial inventada.

---

## Etapa 8 — Entrega

1. `README.md`: qué es, cómo desplegar con el perfil `luis`, cómo probar.
2. `docs/arquitectura.md` con el diagrama.
3. **Actualizar `BITACORA.md`** con lo que cambió durante la construcción: el
   ítem `knowledge_search` como mecanismo de auditoría, la decisión del vector
   store, los VPC endpoints cerrando el argumento de soberanía de datos, y la
   tensión honesta entre RAG y contexto completo sobre un corpus pequeño.
4. `terraform destroy` verificado en un entorno desechable, para poder afirmar
   que el modelo es reproducible y no un montaje manual.

---

## Riesgos, en orden de cuándo atacarlos

| Riesgo | Etapa | Mitigación |
|---|---|---|
| PDFs escaneados sin capa de texto | E2 | Verificar el día uno; OCR o transcribir |
| Costo piso del vector store | E3 | Comparar antes de crear la KB |
| Buffering de SSE tras el ALB | E5 | Spike temprano con tokens falsos, contra el ALB |
| Chunking que destroza el CV | E2 | Fragmentos a nivel documento en los cortos |
| Fuga de PII en logs | E6 | Redacción desde E4, verificada en E6 |
| Familias de modelo con comportamiento distinto | E1 | Probar ambos alias desde el inicio |

Los tres primeros son de espera o de infraestructura: si aparecen el último
día, no se recuperan escribiendo código más rápido.

---

## Orden de trabajo y paralelismo

E2 (corpus) no depende de E1 (endpoint) y su paso lento es la ingesta. Lanzar
la ingesta de E3 en cuanto el corpus esté listo y seguir con E1 mientras corre.

Ruta crítica real: **E1 → E5 → E3 → E4 → E7**. El despliegue va antes de lo
que la intuición sugiere, porque es donde vive el riesgo desconocido.

Si el tiempo aprieta, el orden de recorte es: E6 al mínimo (logs y una
métrica), luego el alias GPT, luego el guardrail administrado — pero **nunca**
E4, porque sin veracidad el agente no es entregable, y **nunca** los VPC
endpoints, porque sin ellos la tesis de la bitácora es falsa.

---

## Siguiente acción

Resolver la decisión A (Terraform o CDK) y arrancar E0 y E2 en paralelo: el
scaffolding del repo mientras se verifican los PDFs.