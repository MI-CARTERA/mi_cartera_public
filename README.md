# Mi Cartera

**Mi Cartera** es una plataforma web de finanzas personales construida como un ecosistema de microservicios. Centraliza gastos, ahorro, inversiones, cuentas bancarias y metas en una sola aplicacion, y automatiza la parte mas tediosa: convertir un estado de cuenta bancario en PDF en movimientos estructurados y clasificados.

Este repositorio es el punto de entrada publico del proyecto: explica como esta construido y donde vive cada pieza.

---

## Por que existe

El proyecto nacio para reemplazar un manejo fragmentado de las finanzas personales: primero una planilla mensual de gastos, despues otra planilla de inversiones, y finalmente la necesidad de relacionar gastos, ahorro, inversiones, cuentas bancarias y metas en un mismo lugar.

De ahi salen las dos decisiones que ordenan todo el diseño:

1. **Entidades relacionadas en vez de hojas sueltas**: movimientos, cajas, gastos fijos y metas se consultan y se cruzan entre si.
2. **Menos carga manual**: los movimientos entran por PDF del banco, no a mano, y la clasificacion la propone el sistema con revisión del usuario.

## Que permite hacer hoy

- Gestionar movimientos del mes por caja y moneda
- Registrar ahorro e inversiones como movimientos, no como cuentas aparte
- Mantener una cuenta bancaria con varias cajas operativas (`caja de ahorro pesos`, `caja de ahorro dolares`)
- Administrar gastos fijos recurrentes con estado mes a mes (`pending`, `paid`, `unpaid`, `not_applicable`)
- Definir metas financieras y generar un plan de accion
- Consultar historico, saldos, totales del mes y agregados por categoria
- Subir estados de cuenta en PDF, revisar la clasificacion propuesta y confirmarla
- Ver reportes de en que fallo la interpretacion de cada PDF y que lecciones quedaron

El frontend organiza eso en las secciones: Gastos fijos, Movimientos, Ahorros, Inversiones, Objetivos economicos, Consulta historica, Archivos y clasificacion, Reportes de interpretacion y Mis cuentas.

---

## Arquitectura

```text
          Frontend (React + Vite)
                   |
                   v
        API Gateway (Spring Cloud Gateway)
                   |
   +---------------+----------------+
   |               |                |
   v               v                v
ms_users    ms_bank_accounts    ms_files   ms_transactions
ms_goals    ms_fixed_expenses   ms_notifications_cartera
                ms_ai_classifier (Python + FastAPI)
                   |
      PostgreSQL / Redis / RabbitMQ
```

Caracteristicas de la arquitectura:

- **Un servicio por capacidad de negocio**, con su propia base de datos y su propio usuario de Postgres
- **Un unico punto de entrada HTTP**: el gateway valida el JWT, aplica rate limit por IP, agrega headers de seguridad y propaga la identidad hacia abajo
- **Sin service registry**: los servicios se descubren por DNS de Docker Compose (`ms-users`, `ms-transactions`, `ms-ai-classifier`, ...)
- **Dos redes Docker**: una publica para frontend y gateway, y una interna para el trafico entre servicios
- **HTTP para CRUD y consultas, RabbitMQ para reacciones entre servicios**, con retry, DLQ e idempotencia en cada consumidor
- **Contrato de eventos compartido y agnostico de lenguaje**, para que Spring y Python hablen el mismo idioma

## Mapa de repositorios

| Repositorio | Rol | Stack |
| --- | --- | --- |
| [`mi_cartera_infra`](https://github.com/MI-CARTERA/mi_cartera_infra) | Orquestacion local, contrato de eventos, init de Postgres, observabilidad, generadores | Docker Compose |
| [`frontend`](https://github.com/MI-CARTERA/frontend) | SPA web: dashboard, revision de clasificaciones, reportes | React 18 + Vite |
| [`api_gateway`](https://github.com/MI-CARTERA/api_gateway) | Entrada HTTP unica, auth, CSRF, rate limit, headers | Spring Boot 4 + Spring Cloud Gateway |
| [`ms_users`](https://github.com/MI-CARTERA/ms_users) | Usuarios, login, MFA TOTP, cookie de sesion | Spring Boot 3.5 |
| [`ms_bank_accounts`](https://github.com/MI-CARTERA/ms_bank_accounts) | Cuenta bancaria y cajas por moneda, saldos | Spring Boot 3.5 |
| [`ms_files`](https://github.com/MI-CARTERA/ms_files) | Carga de PDFs, metadata y estado del workflow | Spring Boot 3.5 |
| [`ms_transactions`](https://github.com/MI-CARTERA/ms_transactions) | Movimientos, balances mensuales, agregados por categoria, deteccion de duplicados | Spring Boot 3.5 |
| [`ms_fixed_expenses`](https://github.com/MI-CARTERA/ms_fixed_expenses) | Gastos fijos recurrentes con estado por mes | Spring Boot 4 |
| [`ms_goals`](https://github.com/MI-CARTERA/ms_goals) | Metas financieras y plan de accion | Spring Boot 3.5 |
| [`ms_notifications_cartera`](https://github.com/MI-CARTERA/ms_notifications_cartera) | Notificaciones persistidas y stream SSE | Spring Boot 3.5 |
| [`ms_ai_classifier`](https://github.com/MI-CARTERA/ms_ai_classifier) | Extraccion de PDFs, clasificacion con reglas + memoria + LLM | Python 3.12 + FastAPI |

---

## El flujo que distingue al proyecto

Subir un estado de cuenta y que termine siendo movimientos clasificados atraviesa seis servicios y tres saltos de RabbitMQ:

```text
Usuario sube el PDF eligiendo cuenta y caja
  -> API Gateway
  -> ms_files guarda el archivo y su metadata (bankAccountId, bankAccountBoxId, currency)
     y publica FILE_UPLOADED
  -> ms_ai_classifier lee el PDF del volumen compartido
       extrae filas (pdfplumber; OCR como ultimo recurso)
       valida cada fila contra el saldo impreso
       clasifica tipo y categoria
       publica AI_CLASSIFICATION_STARTED / COMPLETED / FAILED
  -> El usuario revisa y corrige las clasificaciones pendientes y confirma
  -> ms_ai_classifier publica TRANSACTIONS_CONFIRMED
  -> ms_transactions crea los movimientos y publica TRANSACTIONS_RECORDED
  -> ms_bank_accounts actualiza el saldo de la caja
  -> ms_files cierra el estado del workflow
  -> ms_notifications_cartera persiste la notificacion y la empuja por SSE
```

Como el impacto real es asincronico, el frontend no asume un delay fijo tras confirmar: hace polling con tope de intentos sobre los movimientos importados y el cambio de saldo.

### Que hace y que no hace la IA

- La **extraccion es local**: fechas, importes, signos y saldos los resuelve el codigo con `pdfplumber`, priorizando tablas con encabezado propio y leyendo cada columna por su celda. OpenAI no lee el PDF ni hace OCR.
- La **clasificacion resuelve en tres capas, en orden**: reglas fijas, memoria de correcciones del usuario, y recien entonces el LLM con Structured Outputs para lo que quedo sin resolver.
- Cada resultado guarda **quien lo decidio** (`rule`, `user_correction`, `historical_classification`, `llm`, `fallback`) y cual fue la propuesta original, incluso si el usuario la corrige despues.

### El sistema registra en que se equivoco

Cada PDF interpretado deja un rastro auditable, porque la primera lectura de un formato nuevo suele repetir el mismo error:

- **Errores de lectura**: se valida que `saldo anterior + importe = saldo` en cada fila. Las que no cuadran se guardan con un codigo (`inverted_sign`, `amount_is_balance`, `balance_mismatch`, `balance_chain_break`) y se marcan en el panel de revision antes de confirmar.
- **Errores de clasificacion**: solo cuenta como error un cambio de tipo o categoria hecho por el usuario.
- **Lecciones**: las correcciones se agrupan por descripcion normalizada y se inyectan en el prompt, para que la IA generalice a descripciones parecidas; las identicas ya las resuelve la memoria antes de llegar al LLM. Cuando una correccion se repite, el sistema sugiere promoverla a regla fija; cuando el error lo produce una regla, lo marca como regla a revisar.

Todo eso se consulta desde la seccion `Reportes de interpretacion` y se expone como metricas Prometheus.

## Contrato de eventos

La fuente de verdad vive en `contracts/events/mi_cartera_events.yaml`, dentro del repo de infraestructura.

| Evento | Routing key | Productor | Consumidores |
| --- | --- | --- | --- |
| `FILE_UPLOADED` | `files.file_uploaded` | `ms_files` | `ms_ai_classifier` |
| `AI_CLASSIFICATION_STARTED` | `ai.classification_started` | `ms_ai_classifier` | `ms_files` |
| `AI_CLASSIFICATION_COMPLETED` | `ai.classification_completed` | `ms_ai_classifier` | `ms_files` |
| `AI_CLASSIFICATION_FAILED` | `ai.classification_failed` | `ms_ai_classifier` | `ms_files`, `ms_notifications_cartera` |
| `TRANSACTIONS_CONFIRMED` | `ai.transactions_confirmed` | `ms_ai_classifier` | `ms_transactions` |
| `TRANSACTIONS_RECORDED` | `transactions.recorded` | `ms_transactions` | `ms_bank_accounts`, `ms_files`, `ms_notifications_cartera` |

Garantias de los consumidores: retry con backoff, DLQ durable por cola e idempotencia respaldada por constraints (`source_event_id` en el clasificador y en notificaciones, `source_job_id` + `source_sequence` en movimientos). `ms_bank_accounts` aplica el saldo como valor absoluto, no como suma incremental, para que una redelivery no desplace el saldo.

---

## Decisiones de dominio que vale la pena contar

El valor del proyecto no esta solo en el stack: esta en las reglas que evitan que los numeros mientan.

**Una cuenta bancaria, varias cajas por moneda.** Gastos, ahorros e inversiones no son cuentas distintas: son movimientos dentro de la caja correspondiente. Todo el dashboard depende de la caja activa.

**Un egreso no es un gasto.** Pasar pesos a la caja en dolares genera dos movimientos independientes, uno por caja, importados de dos estados de cuenta distintos. La pata que sale baja el saldo pero no es gasto: el gasto ocurre cuando la caja destino recibe el debito. Contar las dos patas duplicaria el total del mes, asi que los movimientos internos se acumulan aparte y no pueden marcar un gasto fijo como pagado.

**El discriminador es la categoria, no el tipo.** La pata entrante de un fondeo llega como `deposit` + `transfer_in`. `deposit` solo no alcanza: tambien puede ser plata que entra de afuera, como un deposito en ventanilla. El sueldo, en cambio, se clasifica como `income`.

**Un importe positivo nunca suma a gastos**, aunque el tipo parezca de gasto. El caso real es el reintegro de IVA, que llega con el mismo tipo que el gasto que lo origino. Un unico helper decide si un movimiento es egreso, y su espejo en el backend calcula los agregados: si se separan, un recuadro marcaria filas que su propio total no incluye.

**Varios estados de cuenta del mismo mes no reimportan lo ya cargado.** La identidad de un movimiento es fecha + descripcion + importe. El saldo acumulado no es condicion, porque el banco puede entregar los movimientos de un mismo dia en otro orden entre dos PDFs; se usa solo como preferencia para desempatar. El tipo tampoco entra, porque es salida del clasificador y el usuario pudo corregirlo. Una misma implementacion alimenta la importacion real y el panel de revision, para que el panel no anuncie una cosa y la importacion haga otra.

---

## Seguridad

El modelo se reviso contra OWASP Top 10 y las decisiones estan implementadas, no solo documentadas:

- **La identidad sale del token, nunca del header.** El gateway valida el JWT y propaga `X-User-Id`, pero cada servicio vuelve a validar la firma y reescribe ese header con el `sub` del token. Confiar solo en el gateway dejaria que cualquiera con acceso a la red interna leyera la cartera de otro usuario.
- **Scoping por usuario en cada endpoint**, y el recurso de otro usuario se responde 404 en vez de 403, para no confirmar que ese id existe.
- **La sesion vive en una cookie HttpOnly**, no en `localStorage` ni en el body de la respuesta, asi un XSS no puede exfiltrarla. Como la cookie la manda el navegador sola, hay proteccion CSRF double-submit, y el chequeo corre solo cuando el token vino de la cookie.
- **Dos capas contra fuerza bruta**: rate limit por IP en el gateway con Redis, y conteo de intentos fallidos por identidad en `ms_users`. Un codigo MFA invalido tambien cuenta.
- **Sin credenciales en archivos versionados**: cada servicio entra a Postgres con su propio usuario, los secretos salen del entorno y hay un script de rotacion que opera sobre el entorno vivo sin tocar los datos.
- **Superficie minima**: solo el gateway y el frontend se publican al host; Postgres, Redis, RabbitMQ y los microservicios no. Todas las imagenes corren con usuario no-root y `no-new-privileges`.
- **Errores genericos hacia afuera, trazas completas en el log del servidor.**

Lo que falta esta anotado abiertamente en la documentacion interna (migracion del resto de los servicios a Spring Boot 4, registro abierto, MFA opcional, revocacion de tokens y migraciones de esquema versionadas).

## Observabilidad

El entorno local incluye una pila completa para seguir el flujo de punta a punta:

- **Prometheus** recolecta Actuator de los servicios Spring, metricas del clasificador y de RabbitMQ, PostgreSQL, Redis y contenedores
- **Loki + Grafana Alloy** para logs de contenedores
- **Tempo + OpenTelemetry Collector** para trazas HTTP, JDBC y RabbitMQ, con contexto W3C propagado tambien al publicar y consumir eventos
- **Grafana** como panel unico
- Un `X-Request-Id` nace en el frontend, viaja como `correlation_id` del evento y se conserva hasta la confirmacion de los movimientos

No se exportan JWT, PDFs, texto extraido, prompts, importes ni datos personales a la pila de observabilidad.

## Stack tecnico

| Capa | Tecnologia |
| --- | --- |
| Frontend | React 18, React Router, Vite, `fetch` centralizado, `EventSource` para SSE |
| Gateway | Java 21, Spring Boot 4, Spring Cloud Gateway, Redis para rate limit |
| Servicios de dominio | Java 21, Spring Boot 3.5 / 4, Spring Data JPA |
| Servicio de IA | Python 3.12, FastAPI, SQLAlchemy, Pydantic, pdfplumber, PyMuPDF, Tesseract, OpenAI Structured Outputs |
| Datos | PostgreSQL (una base y un usuario por servicio), Redis |
| Mensajeria | RabbitMQ con exchange de eventos, DLX y DLQ por cola |
| Orquestacion | Docker Compose con redes publica e interna |
| Observabilidad | Prometheus, Grafana, Loki, Alloy, Tempo, OpenTelemetry |

## Entorno local

El stack completo se levanta desde el repositorio de infraestructura:

```bash
git clone git@github.com:MI-CARTERA/mi_cartera_infra.git
cd mi_cartera_infra
cp .env.example .env     # completar secretos y OPENAI_API_KEY
docker compose up -d --build
```

Accesos principales:

| Puerto | Servicio |
| --- | --- |
| `5173` | Frontend |
| `8080` | API Gateway |
| `3000` | Grafana |
| `15672` | RabbitMQ Management |

Los microservicios no se publican al host: se acceden por el gateway. Los datos de negocio viven en el volumen `pg_data`, y los PDFs en `files_data`, compartido entre `ms_files` (lectura/escritura) y `ms_ai_classifier` (solo lectura).

Rutas publicas del gateway: `/api/auth`, `/api/users`, `/api/bank-accounts`, `/api/files`, `/api/transactions`, `/api/fixed-expenses`, `/api/goals`, `/api/notifications` y `/api/ai` (reescrita a `/api/v1` del clasificador).

---

## Hacia donde va

Hoy la IA cumple una funcion operativa: extraer, normalizar y clasificar movimientos. El paso siguiente es que pase de interpretar el pasado a asistir el futuro:

- detectar patrones de gasto y analizar habitos
- sugerir objetivos economicos realistas a partir del historico
- proponer planes financieros en el tiempo para alcanzarlos

La base ya esta: los agregados por mes y categoria existen justamente para responder preguntas historicas sin recorrer los movimientos uno por uno.

---

Proyecto personal de finanzas propias, desarrollado como ejercicio de arquitectura distribuida. Cada repositorio de la organizacion es la fuente de verdad de su propio componente.
