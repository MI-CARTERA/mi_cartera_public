# Mi Cartera

**Mi Cartera** es una plataforma web de finanzas personales construida como un ecosistema de microservicios. Centraliza gastos, ahorro, inversiones, cuentas bancarias y metas en una sola aplicación, y automatiza la parte más tediosa: convertir un estado de cuenta bancario en PDF en movimientos estructurados y clasificados.

Este repositorio es el punto de entrada público del proyecto: explica cómo está construido y dónde vive cada pieza.

🇬🇧 [Read this in English](./README.en.md)

---

## Por qué existe

El proyecto nació para reemplazar un manejo fragmentado de las finanzas personales: primero una planilla mensual de gastos, después otra planilla de inversiones, y finalmente la necesidad de relacionar gastos, ahorro, inversiones, cuentas bancarias y metas en un mismo lugar.

De ahí salen las dos decisiones que ordenan todo el diseño:

1. **Entidades relacionadas en vez de hojas sueltas**: movimientos, cajas, gastos fijos y metas se consultan y se cruzan entre sí.
2. **Menos carga manual**: los movimientos entran por PDF del banco, no a mano, y la clasificación la propone el sistema con revisión del usuario.

## Qué permite hacer hoy

- Gestionar movimientos del mes por caja y moneda
- Registrar ahorro e inversiones como movimientos, no como cuentas aparte
- Mantener una cuenta bancaria con varias cajas operativas (`caja de ahorro pesos`, `caja de ahorro dólares`)
- Administrar gastos fijos recurrentes con estado mes a mes (`pending`, `paid`, `unpaid`, `not_applicable`)
- Definir metas financieras y generar un plan de acción
- Consultar histórico, saldos, totales del mes y agregados por categoría
- Subir estados de cuenta en PDF, revisar la clasificación propuesta y confirmarla
- Ver reportes de en qué falló la interpretación de cada PDF y qué lecciones quedaron

El frontend organiza eso en las secciones: Gastos fijos, Movimientos, Ahorros, Inversiones, Objetivos económicos, Consulta histórica, Archivos y clasificación, Reportes de interpretación y Mis cuentas.

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

Características de la arquitectura:

- **Un servicio por capacidad de negocio**, con su propia base de datos y su propio usuario de Postgres
- **Un único punto de entrada HTTP**: el gateway valida el JWT, aplica rate limit por IP, agrega headers de seguridad y propaga la identidad hacia abajo
- **Sin service registry**: los servicios se descubren por DNS de Docker Compose (`ms-users`, `ms-transactions`, `ms-ai-classifier`, ...)
- **Dos redes Docker**: una pública para frontend y gateway, y una interna para el tráfico entre servicios
- **HTTP para CRUD y consultas, RabbitMQ para reacciones entre servicios**, con retry, DLQ e idempotencia en cada consumidor
- **Contrato de eventos compartido y agnóstico de lenguaje**, para que Spring y Python hablen el mismo idioma

## Mapa de repositorios

| Repositorio | Rol | Stack |
| --- | --- | --- |
| [`mi_cartera_infra`](https://github.com/MI-CARTERA/mi_cartera_infra) | Orquestación local, contrato de eventos, init de Postgres, observabilidad, generadores | Docker Compose |
| [`frontend`](https://github.com/MI-CARTERA/frontend) | SPA web: dashboard, revisión de clasificaciones, reportes | React 18 + Vite |
| [`api_gateway`](https://github.com/MI-CARTERA/api_gateway) | Entrada HTTP única, auth, CSRF, rate limit, headers | Spring Boot 4 + Spring Cloud Gateway |
| [`ms_users`](https://github.com/MI-CARTERA/ms_users) | Usuarios, login, MFA TOTP, cookie de sesión | Spring Boot 3.5 |
| [`ms_bank_accounts`](https://github.com/MI-CARTERA/ms_bank_accounts) | Cuenta bancaria y cajas por moneda, saldos | Spring Boot 3.5 |
| [`ms_files`](https://github.com/MI-CARTERA/ms_files) | Carga de PDFs, metadata y estado del workflow | Spring Boot 3.5 |
| [`ms_transactions`](https://github.com/MI-CARTERA/ms_transactions) | Movimientos, balances mensuales, agregados por categoría, detección de duplicados | Spring Boot 3.5 |
| [`ms_fixed_expenses`](https://github.com/MI-CARTERA/ms_fixed_expenses) | Gastos fijos recurrentes con estado por mes | Spring Boot 4 |
| [`ms_goals`](https://github.com/MI-CARTERA/ms_goals) | Metas financieras y plan de acción | Spring Boot 3.5 |
| [`ms_notifications_cartera`](https://github.com/MI-CARTERA/ms_notifications_cartera) | Notificaciones persistidas y stream SSE | Spring Boot 3.5 |
| [`ms_ai_classifier`](https://github.com/MI-CARTERA/ms_ai_classifier) | Extracción de PDFs, clasificación con reglas + memoria + LLM | Python 3.12 + FastAPI |

---

## El flujo que distingue al proyecto

Subir un estado de cuenta y que termine siendo movimientos clasificados atraviesa seis servicios y tres saltos de RabbitMQ:

```text
Usuario sube el PDF eligiendo cuenta y caja
  -> API Gateway
  -> ms_files guarda el archivo y su metadata (bankAccountId, bankAccountBoxId, currency)
     y publica FILE_UPLOADED
  -> ms_ai_classifier lee el PDF del volumen compartido
       extrae filas (pdfplumber; OCR como último recurso)
       valida cada fila contra el saldo impreso
       clasifica tipo y categoría
       publica AI_CLASSIFICATION_STARTED / COMPLETED / FAILED
  -> El usuario revisa y corrige las clasificaciones pendientes y confirma
  -> ms_ai_classifier publica TRANSACTIONS_CONFIRMED
  -> ms_transactions crea los movimientos y publica TRANSACTIONS_RECORDED
  -> ms_bank_accounts actualiza el saldo de la caja
  -> ms_files cierra el estado del workflow
  -> ms_notifications_cartera persiste la notificación y la empuja por SSE
```

Como el impacto real es asincrónico, el frontend no asume un delay fijo tras confirmar: hace polling con tope de intentos sobre los movimientos importados y el cambio de saldo.

### Qué hace y qué no hace la IA

- La **extracción es local**: fechas, importes, signos y saldos los resuelve el código con `pdfplumber`, priorizando tablas con encabezado propio y leyendo cada columna por su celda. OpenAI no lee el PDF ni hace OCR.
- La **clasificación resuelve en tres capas, en orden**: reglas fijas, memoria de correcciones del usuario, y recién entonces el LLM con Structured Outputs para lo que quedó sin resolver.
- Cada resultado guarda **quién lo decidió** (`rule`, `user_correction`, `historical_classification`, `llm`, `fallback`) y cuál fue la propuesta original, incluso si el usuario la corrige después.

### El sistema registra en qué se equivocó

Cada PDF interpretado deja un rastro auditable, porque la primera lectura de un formato nuevo suele repetir el mismo error:

- **Errores de lectura**: se valida que `saldo anterior + importe = saldo` en cada fila. Las que no cuadran se guardan con un código (`inverted_sign`, `amount_is_balance`, `balance_mismatch`, `balance_chain_break`) y se marcan en el panel de revisión antes de confirmar.
- **Errores de clasificación**: solo cuenta como error un cambio de tipo o categoría hecho por el usuario.
- **Lecciones**: las correcciones se agrupan por descripción normalizada y se inyectan en el prompt, para que la IA generalice a descripciones parecidas; las idénticas ya las resuelve la memoria antes de llegar al LLM. Cuando una corrección se repite, el sistema sugiere promoverla a regla fija; cuando el error lo produce una regla, lo marca como regla a revisar.

Todo eso se consulta desde la sección `Reportes de interpretacion` y se expone como métricas Prometheus.

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

Garantías de los consumidores: retry con backoff, DLQ durable por cola e idempotencia respaldada por constraints (`source_event_id` en el clasificador y en notificaciones, `source_job_id` + `source_sequence` en movimientos). `ms_bank_accounts` aplica el saldo como valor absoluto, no como suma incremental, para que una redelivery no desplace el saldo.

---

## Decisiones de dominio que vale la pena contar

El valor del proyecto no está solo en el stack: está en las reglas que evitan que los números mientan.

**Una cuenta bancaria, varias cajas por moneda.** Gastos, ahorros e inversiones no son cuentas distintas: son movimientos dentro de la caja correspondiente. Todo el dashboard depende de la caja activa.

**Un egreso no es un gasto.** Pasar pesos a la caja en dólares genera dos movimientos independientes, uno por caja, importados de dos estados de cuenta distintos. La pata que sale baja el saldo pero no es gasto: el gasto ocurre cuando la caja destino recibe el débito. Contar las dos patas duplicaría el total del mes, así que los movimientos internos se acumulan aparte y no pueden marcar un gasto fijo como pagado.

**El discriminador es la categoría, no el tipo.** La pata entrante de un fondeo llega como `deposit` + `transfer_in`. `deposit` solo no alcanza: también puede ser plata que entra de afuera, como un depósito en ventanilla. El sueldo, en cambio, se clasifica como `income`.

**Un importe positivo nunca suma a gastos**, aunque el tipo parezca de gasto. El caso real es el reintegro de IVA, que llega con el mismo tipo que el gasto que lo originó. Un único helper decide si un movimiento es egreso, y su espejo en el backend calcula los agregados: si se separan, un recuadro marcaría filas que su propio total no incluye.

**Varios estados de cuenta del mismo mes no reimportan lo ya cargado.** La identidad de un movimiento es fecha + descripción + importe. El saldo acumulado no es condición, porque el banco puede entregar los movimientos de un mismo día en otro orden entre dos PDFs; se usa solo como preferencia para desempatar. El tipo tampoco entra, porque es salida del clasificador y el usuario pudo corregirlo. Una misma implementación alimenta la importación real y el panel de revisión, para que el panel no anuncie una cosa y la importación haga otra.

---

## Seguridad

El modelo se revisó contra OWASP Top 10 y las decisiones están implementadas, no solo documentadas:

- **La identidad sale del token, nunca del header.** El gateway valida el JWT y propaga `X-User-Id`, pero cada servicio vuelve a validar la firma y reescribe ese header con el `sub` del token. Confiar solo en el gateway dejaría que cualquiera con acceso a la red interna leyera la cartera de otro usuario.
- **Scoping por usuario en cada endpoint**, y el recurso de otro usuario se responde 404 en vez de 403, para no confirmar que ese id existe.
- **La sesión vive en una cookie HttpOnly**, no en `localStorage` ni en el body de la respuesta, así un XSS no puede exfiltrarla. Como la cookie la manda el navegador sola, hay protección CSRF double-submit, y el chequeo corre solo cuando el token vino de la cookie.
- **Dos capas contra fuerza bruta**: rate limit por IP en el gateway con Redis, y conteo de intentos fallidos por identidad en `ms_users`. Un código MFA inválido también cuenta.
- **Sin credenciales en archivos versionados**: cada servicio entra a Postgres con su propio usuario, los secretos salen del entorno y hay un script de rotación que opera sobre el entorno vivo sin tocar los datos.
- **Superficie mínima**: solo el gateway y el frontend se publican al host; Postgres, Redis, RabbitMQ y los microservicios no. Todas las imágenes corren con usuario no-root y `no-new-privileges`.
- **Errores genéricos hacia afuera, trazas completas en el log del servidor.**

Lo que falta está anotado abiertamente en la documentación interna (migración del resto de los servicios a Spring Boot 4, registro abierto, MFA opcional, revocación de tokens y migraciones de esquema versionadas).

## Observabilidad

El entorno local incluye una pila completa para seguir el flujo de punta a punta:

- **Prometheus** recolecta Actuator de los servicios Spring, métricas del clasificador y de RabbitMQ, PostgreSQL, Redis y contenedores
- **Loki + Grafana Alloy** para logs de contenedores
- **Tempo + OpenTelemetry Collector** para trazas HTTP, JDBC y RabbitMQ, con contexto W3C propagado también al publicar y consumir eventos
- **Grafana** como panel único
- Un `X-Request-Id` nace en el frontend, viaja como `correlation_id` del evento y se conserva hasta la confirmación de los movimientos

No se exportan JWT, PDFs, texto extraído, prompts, importes ni datos personales a la pila de observabilidad.

## Stack técnico

| Capa | Tecnología |
| --- | --- |
| Frontend | React 18, React Router, Vite, `fetch` centralizado, `EventSource` para SSE |
| Gateway | Java 21, Spring Boot 4, Spring Cloud Gateway, Redis para rate limit |
| Servicios de dominio | Java 21, Spring Boot 3.5 / 4, Spring Data JPA |
| Servicio de IA | Python 3.12, FastAPI, SQLAlchemy, Pydantic, pdfplumber, PyMuPDF, Tesseract, OpenAI Structured Outputs |
| Datos | PostgreSQL (una base y un usuario por servicio), Redis |
| Mensajería | RabbitMQ con exchange de eventos, DLX y DLQ por cola |
| Orquestación | Docker Compose con redes pública e interna |
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

Rutas públicas del gateway: `/api/auth`, `/api/users`, `/api/bank-accounts`, `/api/files`, `/api/transactions`, `/api/fixed-expenses`, `/api/goals`, `/api/notifications` y `/api/ai` (reescrita a `/api/v1` del clasificador).

---

## Hacia dónde va

Hoy la IA cumple una función operativa: extraer, normalizar y clasificar movimientos. El paso siguiente es que pase de interpretar el pasado a asistir el futuro:

- detectar patrones de gasto y analizar hábitos
- sugerir objetivos económicos realistas a partir del histórico
- proponer planes financieros en el tiempo para alcanzarlos

La base ya está: los agregados por mes y categoría existen justamente para responder preguntas históricas sin recorrer los movimientos uno por uno.

---

Proyecto personal de finanzas propias, desarrollado como ejercicio de arquitectura distribuida. Cada repositorio de la organización es la fuente de verdad de su propio componente.
