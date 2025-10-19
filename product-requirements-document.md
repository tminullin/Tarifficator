# Product Requirements Document

***

### 1. Информация о продукте

| Item          | Content                                                                                        |
| ------------- | ---------------------------------------------------------------------------------------------- |
| Название      | Микросервис журналирования статусов движения товаров в маркетплейсе (Tracking Journal Service) |
| Версия        | v1.0 / Phase 1 Development                                                                     |
| Автор         | Product Manager                                                                                |
| Creation Date | 2025-05-15                                                                                     |
| Last Updated  | 2025-05-15                                                                                     |
| Команда       | Product Team, Engineering Lead, QA Lead, Operations Lead                                       |

***

### 2. Обоснование необходимости системы

#### 2.1 Основания проекта

Текущая система маркетплейса испытывает трудности с полным и своевременным отслеживанием статусов движения товаров по всей цепочке поставок. Отсутствие единого, надежного журнала статусов приводит к:

* Невозможности оперативно реагировать на задержки и нарушения сроков со стороны поставщиков, складов и курьеров.
* Потерей видимости в реальном времени для операционных диспетчеров, что снижает их эффективность.
* Недостаточной детализации данных для анализа узких мест в логистической цепочке.
* Потребности в надежной инфраструктуре, способной обрабатывать большие объемы событий в условиях высокой нагрузки (highload), характерной для крупного маркетплейса с географическим охватом.

#### 2.2 Цели проекта

1. **Создать и поддерживать единый, надежный журнал статусов движения товаров** от поставщика, через склады и курьеров, до конечного покупателя.
2. **Обеспечить автоматическое выявление и уведомление о нарушениях нормативов времени** (SLA) для операций склада и курьеров.
3. **Разработать рабочее место (Interfaz `UI`) для Операционного контролера**, позволяющее в реальном времени отслеживать статусы, выявлять нарушения и инициировать корректирующие действия.
4. **Спроектировать архитектуру, устойчивую к высоким нагрузкам (highload)**, способную масштабироваться горизонтально по количеству событий, городов, поставщиков и сотрудников.
5. Обеспечить интеграцию с существующими микросервисами (поставщики, склады, курьеры, пользователи) для получения событий статусов.

***

### 3. Целевые пользователи

* **Операционный контролер `(UR-001)`**:
  * Роль: Мониторинг и управление логистическими операциями в реальном времени.
  * Опыт: в логистике, управление цепочками поставок.
  * Навыки: Работа с системами мониторинга, аналитика, принятие быстрых оперативных решений.
  * Основные потребности:
    * Полная видимость всех этапов доставки.
    * Уведомления о нарушениях SLA в реальном времени.
    * Инструменты для быстрого реагирования на инциденты.
* **Системный администратор / DevOps `(UR-004)`**:
  * Роль: Обеспечение работоспособности, масштабируемости и наблюдаемости системы.
  * Опыт: системноt администрирование, DevOps.
  * Навыки: Управление инфраструктурой, мониторинг, CI/CD, работа с highload системами.
  * Основные потребности:
    * Надежная и масштабируемая архитектура.
    * Эффективный мониторинг и алертинг.
    * Простота деплоя и обслуживания.

***

### 4. Описание функций

#### 4.1 Feature Structure Diagram (High-Level Architecture)

```mermaid
graph TD
    subgraph External Services
        A[Warehouse System] --> B(Tracking Journal API Gateway);
        C[Courier App Backend] --> B;
        D[Supplier Integration] --> B;
    end
    subgraph Tracking Journal Service
        B --> E{Message Queue (e.g., Kafka)};
        E --> F[Event Processor Service];
        F --> G[Database (e.g., PostgreSQL/Cassandra)];
        F --> H[SLA Monitoring Service];
        G --> I[Read API for Journal Data];
        H --> J[SLA Violation Detector];
        J --> K[Notification Service];
        K --> L[Operator Workspace UI];
    end
    subgraph Operational Controller Workspace
        L --> I;
        L --> M[Corrective Action Module];
        M --> K;
    end
    subgraph Observability
        F --> N[Metrics/Tracing Collector];
        H --> N;
        J --> N;
        N --> O[Monitoring Dashboard (e.g., Grafana)];
    end
    B --> N;
```

\[ARCH-1.0]

#### 4.2 Core Feature List

| Core Feature                          | Sub Feature                                                                      |
| ------------------------------------- | -------------------------------------------------------------------------------- |
| Event Ingestion & Journaling          | Event reception, Validation, Storage, Data exposure                              |
| SLA Monitoring & Violation Detection  | SLA parameter configuration, Real-time monitoring, Violation detection, Alerting |
| Operational Controller Workspace      | Real-time dashboard, Violation list, Action creation, Reporting                  |
| Highload Architecture & Observability | Scalability, Reliability, Monitoring, Tracing, Logging                           |

#### 4.3 Detailed Requirement for Core Features

@@@LOOP\_START@@@

**Feature Summary: F1-001 - Event Ingestion and Journaling**

| **Category**                   | **Content**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Feature ID**                 | F1-001                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| **Feature Name**               | Event Ingestion and Journaling                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **User Story ID**              | US-001, US-002, US-003                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| **Short Description**          | Сбор, валидация и сохранение событий статусов движения товаров в централизованный журнал.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **Business Context**           | Является основой для всех последующих функций: мониторинга, аналитики и операционного контроля. Обеспечивает единый источник истины о перемещении товара.                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **Goals / Acceptance Metrics** | <p>- 99.99% точность записи событий.<br>- Задержка приема и записи события &#x3C; 500 мс.<br>- Обработка до 10,000 событий/сек (MVP), масштабирование до 100,000+ событий/сек (Final).<br>- Идемпотентность обработки событий.</p>                                                                                                                                                                                                                                                                                                                                                            |
| **Dependencies**               | <p>- Микросервис Склада (Warehouse Service)<br>- Микросервис Курьера (Courier Service)<br>- Возможно, Микросервис Поставщика (Supplier Service)<br>- Message Queue (e.g., Kafka)<br>- База данных (PostgreSQL/Cassandra)</p>                                                                                                                                                                                                                                                                                                                                                                  |
| **Business Rules**             | <p>- Каждый статус должен быть привязан к уникальному идентификатору товара (<code>deliveryUnitId</code>).<br>- События должны содержать временную метку (<code>timestamp</code>).<br>- Обработка событий должна быть идемпотентной (повторная отправка одного и того же события не должна приводить к дублированию записей).<br>- Допустимые статусы для склада: "Принял", "Отправил".<br>- Допустимые статусы для курьера: "взял", "в работе", "отдал".<br>- Если источник - поставщик, определяются статусы (например, "Отгружен", "Передан в доставку") - <em>требует уточнения.</em></p> |
| **State Transitions (Brief)**  | State tracking for each `deliveryUnitId`: `Supplier_Shipment_Pending` → `Warehouse_Received` → `Warehouse_Shipped` → `Courier_Accepted` → `Courier_In_Progress` → `Courier_Delivered` / `Delivery_Failed`. (This is illustrative; actual states will evolve based on events).                                                                                                                                                                                                                                                                                                                 |
| **Notes / Constraints**        | <p>- Географическая распределенность требует учета задержек сети.<br>- Высокая нагрузка требует использования асинхронных паттернов и горизонтально масштабируемых решений ( Kafka, Cassandra/Sharded PostgreSQL).<br>- Данные могут нуждаться в партиционировании по региону/городу для масштабирования.<br>- Обеспечить безопасность передачи данных.</p>                                                                                                                                                                                                                                   |

#### F1-001 Input / Output / Error (Framework I/O & Error)

| **Type**       | **Field Name**       | **Data Type (Example)** | **Required** | **Description / Purpose**                              | **Format / Constraints**  | **Example Value / Notes**              |
| -------------- | -------------------- | ----------------------- | ------------ | ------------------------------------------------------ | ------------------------- | -------------------------------------- |
| Input          | `deliveryUnitId`     | string                  | Yes          | Уникальный идентификатор единицы поставки/товара.      | UUID                      | "a1b2c3d4-e5f6-7890-1234-567890abcdef" |
| Input          | `actor`              | object                  | Yes          | Информация об источнике события.                       |                           |                                        |
| Input          | `actor.type`         | string                  | Yes          | Тип актора: `WAREHOUSE`, `COURIER`, `SUPPLIER`.        | Enum                      | "WAREHOUSE"                            |
| Input          | `actor.id`           | string                  | Yes          | ID конкретного склада или курьера.                     | UUID / String             | "wh-12345" / "cr-67890"                |
| Input          | `actor.name`         | string                  | No           | Имя склада или курьера.                                | String (max 100)          | "Склад №3", "Иван Петров"              |
| Input          | `status`             | string                  | Yes          | Новый статус для `deliveryUnitId`.                     | Enum (см. Business Rules) | "Принял"                               |
| Input          | `timestamp`          | string (ISO 8601)       | Yes          | Время события.                                         | UTC                       | "2025-05-15T10:30:00Z"                 |
| Input          | `event_id`           | string                  | Yes          | Уникальный идентификатор события для идемпотентности.  | UUID                      | "f9e8d7c6-b5a4-3210-fedc-ba9876543210" |
| Input          | `location`           | object                  | No           | Географические координаты события (опционально).       |                           |                                        |
| Input          | `location.latitude`  | number                  | No           | Широта.                                                | float (double)            | 55.7558                                |
| Input          | `location.longitude` | number                  | No           | Долгота.                                               | float (double)            | 37.6173                                |
| Input          | `metadata`           | object                  | No           | Дополнительные данные, специфичные для статуса/актора. | JSON                      | `{"reason_for_delay": "traffic_jam"}`  |
| Success Output | `message`            | string                  | Yes          | Подтверждение приема события.                          |                           | "Event received successfully."         |
| Success Output | `event_id`           | string                  | Yes          | Подтверждение получения ID обработанного события.      | UUID                      | "f9e8d7c6-b5a4-3210-fedc-ba9876543210" |
| Error Output   | `error_code`         | string                  | Yes          | Код ошибки.                                            | See Error Code Table      | E1001                                  |
| Error Output   | `message`            | string                  | Yes          | Понятное сообщение об ошибке.                          |                           | "Invalid status provided."             |
| Error Output   | `detail`             | object                  | No           | Детальная информация для логирования/отладки.          |                           | `{"validation_errors": [...]}`         |

**Error Code Examples (Framework)**

| **Error Code** | **Category**   | **User Message (Brief)**         | **Dev Notes / Handling**                                                                                                             |
| -------------- | -------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| E1001          | Validation     | Invalid status provided.         | Недопустимый статус. Требуется возврат 400 Bad Request.                                                                              |
| E1002          | Validation     | Missing required fields.         | Недостает обязательных полей (deliveryUnitId, actor, status, timestamp, event\_id). Требуется возврат 400 Bad Request.               |
| E1003          | Idempotency    | Event already processed.         | Событие с таким `event_id` уже было обработано. Возврат 200 OK (или 409 Conflict).                                                   |
| E1004          | Data Integrity | Delivery unit not found.         | `deliveryUnitId` не существует в системе. Обработка зависит от бизнес-логики (создать, проигнорировать, вернуть ошибку). Логировать. |
| E1005          | System         | Internal server error.           | Сбой при сохранении в БД или обработке. Возврат 500 Internal Server Error. Логировать.                                               |
| E1006          | Highload       | Service temporarily unavailable. | Сервис перегружен. Возврат 503 Service Unavailable, с указанием `retry_after` если применимо.                                        |

#### F1-001 API Contract (Integration Contract — Framework Constraints)

| **Category**               | **Framework Requirements**                                                                                                                                                                                                                                                                                                                                                                                                       |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| API Scope                  | RESTful API для приема событий статусов (`POST /api/v1/events`).                                                                                                                                                                                                                                                                                                                                                                 |
| Input Requirements         | <p>- Строгая валидация входных данных по схеме (JSON Schema).<br>- Поддержка аутентификации (e.g., API Keys, OAuth2 client credentials) для внешних сервисов.<br>- <code>event_id</code> проверяется на уникальность за последний период (e.g., 24 часа) для идемпотентности.</p>                                                                                                                                                |
| Output Requirements        | <p>- Успешный ответ (<code>200 OK</code> или <code>201 Created</code>): <code>{ "event_id": "...", "message": "Event received successfully." }</code>.<br>- Ошибки (<code>4xx</code>, <code>5xx</code>): стандартный формат <code>{ "error_code": "...", "message": "...", "detail": {...} }</code>.<br>- Асинхронная обработка: API может отвечать <code>202 Accepted</code> с указанием, что событие поставлено в очередь.</p> |
| Error Handling             | <p>- Использовать стандартные HTTP статусы.<br>- Централизованное управление кодами ошибок.<br>- Дифференцировать ошибки валидации (<code>4xx</code>) от серверных ошибок (<code>5xx</code>).</p>                                                                                                                                                                                                                                |
| Versioning                 | API версии `v1`. Обязательно обеспечить обратную совместимость для `v1` в будущих релизах.                                                                                                                                                                                                                                                                                                                                       |
| Security                   | <p>- HTTPS обязателен.<br>- Аутентификация для всех внешних вызовов.<br>- Шифрование чувствительных данных при передаче.</p>                                                                                                                                                                                                                                                                                                     |
| Rate Limiting & Throttling | <p>- Установка лимитов на количество запросов <code>event_id</code> от каждого источника.<br>- Информативное сообщение об ошибке при превышении лимита (e.g., 429 Too Many Requests).</p>                                                                                                                                                                                                                                        |
| Dependency Management      | <p>- Зависимость от Message Queue (Kafka/RabbitMQ) для асинхронной обработки.<br>- Зависимость от БД для хранения.<br>- SLA для внешних систем (Warehouse, Courier) должны быть определены; в случае их недоступности, система должна иметь механизм повторных попыток или буферизации.</p>                                                                                                                                      |
| Observability              | <p>- Все запросы должны логироваться с уникальным <code>trace_id</code>.<br>- Внедрение метрик: <code>events_received_total</code>, <code>events_processed_total</code>, <code>event_processing_latency_ms</code>, <code>error_rate</code>.<br>- Трассировка для отслеживания пути события через компоненты.</p>                                                                                                                 |
| Extensibility              | <p>- Возможность добавления новых типов акторов (e.g., "CUSTOM_LOGISTICS_PARTNER").<br>- Расширяемая схема <code>metadata</code> для новых статусов.</p>                                                                                                                                                                                                                                                                         |
| Documentation              | - API spec (Swagger/OpenAPI) с примерами запросов/ответов, детальными описаниями полей, кодами ошибок.                                                                                                                                                                                                                                                                                                                           |

#### F1-001 UI / Interaction Description

| Category                 | Content Description                                                                                                                                                                                                                                                                                                                                                                         |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Page Structure           | <p>- <strong>API Endpoint <code>POST /api/v1/events</code></strong>: Принимает JSON payload.<br>- <strong>Internal Processing</strong>: Событие попадает в Message Queue, затем обрабатывается сервисом.<br>- <strong>Data Storage</strong>: Записи сохраняются в реляционную БД (для структурированных данных и запросов по ID) или NoSQL БД (для масштабируемости и больших объемов).</p> |
| Input Behavior           | <p>- Внешние системы (Warehouse, Courier backend) отправляют HTTP POST запросы.<br>- Валидация происходит на стороне API Gateway и/или Event Processor Service.</p>                                                                                                                                                                                                                         |
| Button Behavior          | N/A (This feature is typically integrated via API calls, not direct user interaction through a button for event submission.)                                                                                                                                                                                                                                                                |
| Animations & Effects     | N/A (No direct UI for this aspect of event ingestion.)                                                                                                                                                                                                                                                                                                                                      |
| Response & Notifications | <p>- API возвращает synchronous HTTP response (200 OK, 202 Accepted, 4xx, 5xx).<br>- Асинхронные уведомления для операционной команды могут быть настроены в рамках других фич (F2-001).</p>                                                                                                                                                                                                |
| Page Navigation Logic    | N/A (This is a backend service, no client-side navigation.)                                                                                                                                                                                                                                                                                                                                 |
| Compatibility Notes      | N/A                                                                                                                                                                                                                                                                                                                                                                                         |
| Component References     | <p>- API Gateway<br>- Message Queue (Kafka)<br>- Event Processor Service<br>- Database Service</p>                                                                                                                                                                                                                                                                                          |

#### F1-001 Testing Requirements (Event Ingestion and Journaling)

| Test Type             | Description                                                                                                                                                                                                                                                                                                                                                            |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Functional Testing    | <p>- Отправка валидного события успешного статуса (e.g., "Принял" от склада).<br>- Отправка события нового статуса (e.g., "взял" от курьера).<br>- Отправка события с <code>metadata</code>.<br>- Проверка записи в БД: <code>deliveryUnitId</code>, <code>actor</code>, <code>status</code>, <code>timestamp</code>, <code>event_id</code> должны быть корректны.</p> |
| Boundary Testing      | <p>- Отправка событий с граничными значениями (<code>timestamp</code>, UUID).<br>- Максимальная длина строк полей.<br>- Отправка событий с пустыми, но допустимыми полями (e.g., <code>location</code>).</p>                                                                                                                                                           |
| Exception Testing     | <p>- Отправка события с невалидным <code>deliveryUnitId</code>.<br>- Отправка события с недопустимым <code>status</code>.<br>- Отправка события с отсутствующими обязательными полями.<br>- Отправка события с невалидным форматом <code>timestamp</code>.<br>- Отправка дублирующегося <code>event_id</code> (проверка идемпотентности).</p>                          |
| Security Testing      | <p>- Тест на инъекции (SQL injection, XSS) в поля, где это возможно (хотя большинство полей не будет отображаться напрямую в UI).<br>- Проверка аутентификации/авторизации API.<br>- Проверка отсутствия логирования чувствительных данных.</p>                                                                                                                        |
| Performance Testing   | <p>- Тест под высокой нагрузкой (e.g., 10,000+ req/sec) для API Gateway и Event Processor.<br>- Измерение конца-в-конец задержки (от отправки запроса до записи в БД).<br>- Тест масштабируемости: увеличение количества партиций в Kafka, реплик/шардов в БД, добавление экземпляров сервиса.</p>                                                                     |
| Compatibility Testing | <p>- Проверка успешности интеграции с разными версиями систем-источников.<br>- Проверка различных форматов <code>timestamp</code> (хотя ISO 8601 — стандарт).</p>                                                                                                                                                                                                      |
| Observability Testing | <p>- Проверка генерации <code>trace_id</code>.<br>- Проверка корректности логирования ошибок и успешных операций.<br>- Проверка генерации и поступления метрик в систему мониторинга (Prometheus/Grafana), кастомных метрик.</p>                                                                                                                                       |
| Integration Testing   | <p>- Тестирование полного потока: склад → API → Kafka → Processor → DB → Read API.<br>- Проверка взаимодействия с Message Queue и Базой данных.</p>                                                                                                                                                                                                                    |

#### F1-001 Visualization

*   **Data Flow Diagram (DFD) - Level 0**

    ```mermaid
    flowchart TB
        A[Warehouse/Courier B/E] --> B(Tracking Journal Service);
        B --> C[Message Queue];
        C --> D[Event Processor];
        D --> E[Database];
        E --> F[Read API];
        B --> G[Observability System];
        D --> G;
        F --> G;
    ```
*   **Sequence Diagram - Event Submission**

    ```mermaid
    sequenceDiagram
        participant Source as Warehouse/Courier Backend
        participant API as Tracking Journal API Gateway
        participant MQ as Message Queue (Kafka Topic)
        participant Processor as Event Processor Service
        participant DB as Database

        Source->>API: POST /api/v1/events (event_data)
        API->>API: Validate Request & Authenticate
        API->>MQ: Publish event_data
        API-->>Source: 202 Accepted (or 200 OK)

        MQ->>Processor: Consume event_data
        Processor->>Processor: Validate event_data schema
        Processor->>Processor: Check for duplicate event_id (using cache/DB)
        Processor->>DB: Store new event record
        Processor-->>MQ: Acknowledge message
        DB-->>Processor: Record saved confirmation
    ```
*   **State Machine Diagram (Illustrative for Delivery Unit)**

    ```mermaid
    stateDiagram-v2
        [*] --> Supplier_Pending: Initial State

        Supplier_Pending --> Warehouse_Received: Warehouse "Принял"
        Warehouse_Received --> Supplier_Pending: Event Override/Correction? (If allowed)
        Warehouse_Received --> Warehouse_Shipped: Warehouse "Отправил"

        Warehouse_Shipped --> Courier_Accepted: Courier "взял"
        Courier_Accepted --> Courier_In_Progress: Courier "в работе"
        Courier_In_Progress --> Courier_Delivered: Courier "отдал"
        Courier_In_Progress --> Delivery_Failed: Courier "отдал" (with failure intent)
        Courier_Accepted --> Warehouse_Shipped: Incorrect event? (if rollback is supported)
    ```

@@@LOOP\_END@@@

@@@LOOP\_START@@@

**Feature Summary: F1-002 - SLA Monitoring and Violation Detection**

| **Category**                   | **Content**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Feature ID**                 | F1-002                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| **Feature Name**               | SLA Monitoring and Violation Detection                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| **User Story ID**              | US-005, US-011, US-012, US-013                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| **Short Description**          | Система отслеживает время выполнения операций по каждому товару относительно установленных нормативов (SLA) и выявляет нарушения.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| **Business Context**           | Ключевая функция для обеспечения своевременности доставки и качества логистических операций. Позволяет прогнозировать и предотвращать задержки, улучшать клиентский опыт.                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **Goals / Acceptance Metrics** | <p>- 99.9% SLA правил корректно применяются.<br>- Обнаружение нарушения &#x3C; 1 минуты после фактического превышения SLA.<br>- Возможность настройки SLA для каждого типа операции (склад/курьер) и для каждого города/региона.<br>- Задержка уведомления о нарушении: &#x3C; 5 минут.</p>                                                                                                                                                                                                                                                                                                                              |
| **Dependencies**               | <p>- F1-001: Event Ingestion and Journaling (Source of event data).<br>- Configuration Service / Database: для хранения SLA нормативов.<br>- Notification Service.<br>- Highload infrastructure.</p>                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **Business Rules**             | <p>- Для каждой операции (e.g., Warehouse "Принял" from supplier event) задается норматив времени (e.g., 2 часа).<br>- Нормативы могут быть специфичны для:<br>- типа актора (<code>WAREHOUSE</code>, <code>COURIER</code>).<br>- конкретного статуса (<code>Принял</code>, <code>Отправил</code>, <code>взял</code>, etc.).<br>- географии (<code>city</code>, <code>region</code>).<br>- типа товара/категории (опционально, для будущих версий).<br>- Нарушение фиксируется, когда переход между двумя последовательными статусами превышает заданное время.<br>- SLA нормативы конфигурируются через UI или API.</p> |
| **State Transitions (Brief)**  | `Normal Operation` → `SLA Exceeded` → `Violation Detected` → `Notification Sent` → `Corrective Action Initiated`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| **Notes / Constraints**        | <p>- Специфика SLA может потребовать сложной логики учета времени (например, исключение нерабочих часов, выходных).<br>- Распределение нагрузки на сервис мониторинга должно быть учтено.<br>- История изменений SLA нормативов должна сохраняться.</p>                                                                                                                                                                                                                                                                                                                                                                  |

#### F1-002 Input / Output / Error (Framework I/O & Error)

| **Type**                       | **Field Name**               | **Data Type (Example)** | **Required** | **Description / Purpose**                                                    | **Format / Constraints** | **Example Value / Notes**                          |
| ------------------------------ | ---------------------------- | ----------------------- | ------------ | ---------------------------------------------------------------------------- | ------------------------ | -------------------------------------------------- |
| Input (SLA Config)             | `sla_id`                     | string                  | Yes          | Уникальный ID норматива.                                                     | UUID                     | "sla-cfg-001"                                      |
| Input (SLA Config)             | `operation`                  | string                  | Yes          | Название операции (e.g., `WAREHOUSE_RECEIVED`, `COURIER_PICKUP`).            | Enum / String            | "WAREHOUSE\_SHIPPED"                               |
| Input (SLA Config)             | `max_duration_minutes`       | integer                 | Yes          | Максимально допустимое время выполнения (в минутах).                         | Positive Integer         | 120                                                |
| Input (SLA Config)             | `effective_from`             | string (ISO 8601)       | Yes          | Дата начала действия норматива.                                              | UTC                      | "2025-05-15T00:00:00Z"                             |
| Input (SLA Config)             | `effective_to`               | string (ISO 8601)       | No           | Дата окончания действия норматива.                                           | UTC                      | null                                               |
| Input (SLA Config)             | `conditions`                 | object                  | No           | Условия применимости норматива (city, region, actor\_type, etc.).            | JSON                     | `{"city": "Moscow", "actor_type": "WAREHOUSE"}`    |
| Input (Violation Event)        | `violation_id`               | string                  | Yes          | Уникальный ID зафиксированного нарушения.                                    | UUID                     | "vio-inst-001"                                     |
| Input (Violation Event)        | `delivery_unit_id`           | string                  | Yes          | ID единицы поставки, где произошло нарушение.                                | UUID                     | "a1b2c3d4-e5f6-7890-1234-567890abcdef"             |
| Input (Violation Event)        | `violation_type`             | string                  | Yes          | Тип нарушения (e.g., "SLA\_EXCEEDED").                                       | Enum                     | "SLA\_EXCEEDED"                                    |
| Input (Violation Event)        | `description`                | string                  | Yes          | Описание нарушения (e.g., "Time to ship exceeded SLA for delivery unit..."). | String                   | "Time to ship exceeded SLA for delivery unit 123." |
| Input (Violation Event)        | `occurred_at`                | string (ISO 8601)       | Yes          | Время фиксации нарушения.                                                    | UTC                      | "2025-05-15T13:00:00Z"                             |
| Input (Violation Event)        | `related_event_ids`          | array                   | No           | ID Событий, по которым зафиксировано нарушение.                              | Array of Strings (UUIDs) | `["event-id-1", "event-id-2"]`                     |
| Input (Violation Event)        | `violated_sla_config_id`     | string                  | Yes          | ID конфигурации SLA, которая была нарушена.                                  | String                   | "sla-cfg-001"                                      |
| Input (Violation Event)        | `original_time_diff_minutes` | integer                 | Yes          | Фактическая разница во времени выполнения в минутах.                         | Integer                  | 150                                                |
| Success Output (Config Update) | `message`                    | string                  | Yes          | Подтверждение обновления конфигурации SLA.                                   |                          | "SLA configuration updated successfully."          |
| Error Output                   | `error_code`                 | string                  | Yes          | Код ошибки.                                                                  | See Error Code Table     | E2001                                              |
| Error Output                   | `message`                    | string                  | Yes          | Понятное сообщение об ошибке.                                                |                          | "Invalid SLA duration."                            |
| Error Output                   | `detail`                     | object                  | No           | Детальная информация.                                                        |                          |                                                    |

**Error Code Examples (Framework)**

| **Error Code** | **Category** | **User Message (Brief)**                | **Dev Notes / Handling**                                                                                           |
| -------------- | ------------ | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| E2001          | Validation   | Invalid SLA duration.                   | Максимальное время должно быть положительным числом. Возврат 400.                                                  |
| E2002          | Validation   | Missing required fields for SLA config. | Необходимы `operation`, `max_duration_minutes`, `effective_from`. Возврат 400.                                     |
| E2003          | Conflict     | Overlapping SLA periods detected.       | При добавлении нового SLA, проверить, что нет пересечений по времени для одинаковых условий. Возврат 409 Conflict. |
| E2004          | System       | Failed to update SLA configuration.     | Ошибка при записи в БД. Возврат 500.                                                                               |
| E2005          | System       | Failed to process event for SLA check.  | Ошибка при чтении логов или применении правила. Возврат 500.                                                       |
| E2006          | Not Found    | SLA Configuration not found.            | При попытке редактирования или просмотра несуществующей конфигурации. Возврат 404.                                 |

#### F1-002 API Contract (Integration Contract — Framework Constraints)

| **Category**               | **Framework Requirements**                                                                                                                                                                                                                                                            |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| API Scope                  | <p>- CRUD (Create, Read, Update, Delete) API для конфигураций SLA (<code>/api/v1/sla-configs</code>).<br>- API для получения списка зафиксированных нарушений (<code>/api/v1/violations</code>).<br>- Internal API/event stream для получения новых событий статусов для анализа.</p> |
| Input Requirements         | <p>- SLA Config API: Строгая валидация полей, типы данных, диапазоны, формат дат.<br>- Event stream Processor: Должен уметь читать события из Message Queue.<br>- Для всех API: аутентификация, авторизация (например, роли Admin/Operator).</p>                                      |
| Output Requirements        | <p>- SLA Config API: Стандартный CRUD response.<br>- Violations API: JSON с массивом нарушений, пагинацией.<br>- Internal API: может быть событийным (stream).</p>                                                                                                                    |
| Error Handling             | <p>- Использовать стандартные HTTP статусы.<br>- Единая система кодов ошибок.<br>- Для Violations API: пагинация и фильтрация (по <code>delivery_unit_id</code>, <code>violation_type</code>, <code>status</code>).</p>                                                               |
| Versioning                 | `v1` для всех API.                                                                                                                                                                                                                                                                    |
| Security                   | <p>- HTTPS, аутентификация, авторизация.<br>- Ограничение доступа к конфигурированию SLA (только Admins).</p>                                                                                                                                                                         |
| Rate Limiting & Throttling | <p>- Применимо к SLA Config API и Violations API, если они доступны извне.<br>- Может отсутствовать для внутреннего event stream.</p>                                                                                                                                                 |
| Dependency Management      | <p>- Зависимость от Notification Service для отправки алертов.<br>- Зависимость от F1-001 для получения событий.</p>                                                                                                                                                                  |
| Observability              | <p>- Логирование, метрики (latency, throughput, error rate) для всех API.<br>- Трассировка для отслеживания потока обработки событий и генерации нарушений.</p>                                                                                                                       |
| Extensibility              | <p>- Возможность добавления новых условий в SLA (<code>product_category</code>, <code>delivery_method</code>).<br>- Возможность определения кастомных триггеров нарушений.</p>                                                                                                        |
| Documentation              | - Swagger/OpenAPI для SLA Config API, Violations API.                                                                                                                                                                                                                                 |

#### F1-002 UI / Interaction Description

| **Category**             | **Content Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Page Structure           | <p><strong>SLA Configuration Page</strong>:<br>- Таблица существующих SLA нормативов с возможностью фильтрации/сортировки.<br>- Форма для добавления/редактирования SLA: поля для операции, времени, условий (город, тип актора), дат действия.<br><strong>Violation List Page</strong>:<br>- Таблица зафиксированных нарушений: <code>delivery_unit_id</code>, <code>status</code>, <code>violation_type</code>, <code>occurred_at</code>, <code>assigned_to</code> (if applicable), <code>status</code> (New, In Progress, Resolved).<br>- Панель фильтрации: по датам, ID доставки, типу нарушения, статусу.<br>- Детализация нарушения при клике.</p> |
| Input Behavior           | <p>- Формы ввода SLA: маски, валидация в реальном времени, селекторы дат.<br>- Фильтры в таблицах: автодополнение, выбор из списка.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Button Behavior          | <p>- "Add SLA", "Edit SLA", "Save SLA", "Delete SLA".<br>- "View Details", "Assign to me", "Mark as In Progress", "Resolve Violation".</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Animations & Effects     | <p>- Плавное появление/скрытие форм.<br>- Индикаторы загрузки при сохранении/фильтрации.<br>- Визуальное выделение просроченных/критичных нарушений в списке.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Response & Notifications | <p>- Toast-уведомления об успешном сохранении SLA или назначении/изменении статуса нарушения.<br>- Inline-сообщения об ошибках валидации.<br>- Алерты для Операционного контролера (см. F2-001).</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Page Navigation Logic    | <p>- Ссылки с таблицы нарушений на детальную страницу единицы поставки (если такая есть/будет).<br>- Переходы между страницами конфигурации и списка нарушений.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Compatibility Notes      | <p>- Адаптивный дизайн для десктопа.<br>- Совместимость с основными браузерами.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Component References     | <p>- Таблицы с сортировкой/пагинацией<br>- Формы с валидацией<br>- Date Picker<br>- Системы уведомлений (Toast)</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |

#### F1-002 Testing Requirements (SLA Monitoring and Violation Detection)

| Test Type             | Description                                                                                                                                                                                                                                                                                                                                                                         |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Functional Testing    | <p>- Создание SLA для склада (e.g., 1 час на "Принял").<br>- Отправка события "Принял" с задержкой 1 час 5 минут.<br>- Проверка, что нарушение фиксируется и пишется в Violations DB.<br>- Создание SLA для курьера (e.g., 30 минут на "взял" → "в работе").<br>- Проверка, что переход между этими статусами отслеживается.<br>- Тест с разными условиями (город, тип актора).</p> |
| Boundary Testing      | <p>- Тест на граничных значениях времени: событие приходит ровно в SLA / на 1 минуту позже.<br>- Тест с очень большими нормативами времени.<br>- Тест с нормативами = 0 минут (если применимо).</p>                                                                                                                                                                                 |
| Exception Testing     | <p>- Отправка некорректных событий (см. F1-001), проверка, что SLA мониторинг работает стабильно и не падает.<br>- Нарушение SLA при недоступности Notification Service.<br>- Попытка создать SLA с перекрывающимися датами.</p>                                                                                                                                                    |
| Security Testing      | <p>- Проверка авторизации: Пользователь без роли Admin не может менять SLAConfig.<br>- Проверка API на уязвимости.</p>                                                                                                                                                                                                                                                              |
| Performance Testing   | <p>- Тест нагрузки на SLA Monitoring Service: сколько одновременных событий/единиц поставки может обрабатывать.<br>- Тест на создание большого количества SLA конфигураций.<br>- Тест скорости генерации уведомлений.</p>                                                                                                                                                           |
| Observability Testing | <p>- Проверка, что метрики latency, throughput, error rate корректно собираются для SLA сервиса.<br>- Проверка дерева трассировки при возникновении нарушения.<br>- Проверка логов на наличие ошибок и информативных записей.</p>                                                                                                                                                   |
| Integration Testing   | <p>- Тест полного цикла: Event ingestion → SLA check logic → Violation recording → Notification sending.<br>- Проверка корректности сбора данных из F1-001.</p>                                                                                                                                                                                                                     |
| Configuration Testing | <p>- Проверка валидации на форме SLA Config.<br>- Тест применения изменений SLA в реальном времени.<br>- Тест временной применимости SLA (<code>effective_from</code>, <code>effective_to</code>).</p>                                                                                                                                                                              |

#### F1-002 Visualization

*   **Data Flow Diagram (DFD) - SLA Monitoring**

    ```mermaid
    graph LR
        A[F1-001: Event DB] --> B(SLA Monitoring Service);
        C[SLA Config DB] --> B;
        B --> D[Violations DB];
        B --> E(Notification Service);
        E --> F[Operator Workspace];
        C --> G[SLA Config Admin UI];
    ```
*   **Sequence Diagram - SLA Violation Detection**

    ```mermaid
    sequenceDiagram
        participant Processor as Event Processor (from F1-001)
        participant SLA_Monitor as SLA Monitoring Service
        participant Violations_DB as Violations Database
        participant SLA_Config_DB as SLA Config DB
        participant Notifier as Notification Service

        Processor->>SLA_Monitor: Publish Event (e.g., "Warehouse Shipped")
        SLA_Monitor->>SLA_Monitor: Get current time and event details
        SLA_Monitor->>SLA_Config_DB: Query relevant SLA Configuration (e.g., for "WAREHOUSE_SHIPPED", current city/actor)
        SLA_Config_DB-->>SLA_Monitor: Return SLA Config (max_duration=120min, effective_from=...)
        SLA_Monitor->>Processor: Get previous event time (e.g., Warehouse Received)
        Processor-->>SLA_Monitor: Previous event time (e.g., 10:00 AM)
        SLA_Monitor->>SLA_Monitor: Calculate duration (Current time - Previous event time)
        alt Duration > max_duration:
            SLA_Monitor->>SLA_Monitor: Generate Violation event data
            SLA_Monitor->>Violations_DB: Store Violation Record
            Violations_DB-->>SLA_Monitor: Record saved confirmation
            SLA_Monitor->>Notifier: Send Alert (Violation Details)
            Notifier-->>SLA_Monitor: Alert sent confirmation
        else Duration <= max_duration:
            SLA_Monitor-->>Processor: OK / No Violation
        end
    ```
*   **State Machine Diagram (Violation Status)**

    ```mermaid
    stateDiagram-v2
        [*] --> New: Violation detected
        New --> InProgress: Operator starts investigation/action
        InProgress --> Resolved: Action taken, issue fixed
        New --> Acknowledged: Operator reviewed, no immediate action
        Acknowledged --> InProgress: Action initiated later
        Acknowledged --> Resolved: Issue resolved
        New --> Closed: System auto-closed (e.g., after X days if no action)
        InProgress --> Closed: System auto-closed
        Resolved --> Closed: Manual closure confirmation
    ```

@@@LOOP\_END@@@

@@@LOOP\_START@@@

**Feature Summary: F1-003 - Operational Controller Workspace**

| **Category**                   | **Content**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Feature ID**                 | F1-003                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **Feature Name**               | Operational Controller Workspace                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| **User Story ID**              | US-004, US-007, US-008, US-009, US-010                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **Short Description**          | Web-приложение (UI) для операционных контролеров, позволяющее мониторить статусы товаров, просматривать нарушения SLA и инициировать корректирующие действия.                                                                                                                                                                                                                                                                                                                                                                                                         |
| **Business Context**           | Предоставляет операционным специалистам инструменты для проактивного управления логистикой, своевременного реагирования на проблемы и обеспечения бесперебойной работы маркетплейса.                                                                                                                                                                                                                                                                                                                                                                                  |
| **Goals / Acceptance Metrics** | <p>- Время загрузки дашборда &#x3C; 5 секунд.<br>- Время отклика на фильтры/поиск &#x3C; 2 секунд.<br>- 100% нарушений SLA видны в интерфейсе с деталями.<br>- Возможность создания задачи/тикета на исправление в ~3 клика.<br>- Успешный экспорт отчетов за 1 минуту.</p>                                                                                                                                                                                                                                                                                           |
| **Dependencies**               | <p>- F1-001: Read API for Journal Data.<br>- F1-002: API for Violations and SLA Config Data.<br>- Backend for Corrective Action Management (new module or integration with existing ticketing system).<br>- Notification Service integration.</p>                                                                                                                                                                                                                                                                                                                     |
| **Business Rules**             | <p>- Отображать список единиц поставки с их последними статусами.<br>- Детализировать путь каждой единицы поставки (история событий).<br>- Фильтровать данные по городу, статусу, дате, ID поставки.<br>- Выделять единицы поставки с активными нарушениями SLA.<br>- Просматривать детали нарушений (тип, время, длительность).<br>- Возможность назначать нарушения себе или коллегам.<br>- Создавать "корректирующие действия" (corrective actions), привязанные к нарушению/единице поставки.<br>- Экспортировать списки нарушений (с фильтрами) в CSV/Excel.</p> |
| **State Transitions (Brief)**  | `Idle` → `Loading Data` → `Displaying Dashboard/Lists` → `Filtering/Searching` → `Viewing Details` → `Creating/Managing Corrective Actions`.                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **Notes / Constraints**        | <p>- UI должен быть отзывчивым (responsive) для использования на различных устройствах.<br>- Должна быть реализована система ролей и прав доступа.<br>- Высокая скорость отклика критична для оперативной работы.</p>                                                                                                                                                                                                                                                                                                                                                 |

#### F1-003 Input / Output / Error (Framework I/O & Error)

| **Type**                              | **Field Name**        | **Data Type (Example)** | **Required** | **Description / Purpose**                               | **Format / Constraints** | **Example Value / Notes**                                                                         |
| ------------------------------------- | --------------------- | ----------------------- | ------------ | ------------------------------------------------------- | ------------------------ | ------------------------------------------------------------------------------------------------- |
| Input (API Request)                   | `filters`             | object                  | No           | Параметры для фильтрации данных (dashboard/lists).      | JSON                     | `{"city": "St. Petersburg", "status": "IN_PROGRESS", "date_range": ["2025-05-01", "2025-05-15"]}` |
| Input (API Request)                   | `pagination`          | object                  | No           | Параметры для пагинации.                                | JSON                     | `{"page": 1, "limit": 50}`                                                                        |
| Input (API Request)                   | `deliveryUnitId`      | string                  | Yes          | ID единицы поставки для получения детальной информации. | UUID                     | "a1b2c3d4-e5f6-7890-1234-567890abcdef"                                                            |
| Input (Corrective Action)             | `violation_id`        | string                  | Yes          | ID нарушения, к которому относится действие.            | String                   | "vio-inst-001"                                                                                    |
| Input (Corrective Action)             | `action_description`  | string                  | Yes          | Описание планируемого действия.                         | Text                     | "Contact courier to expedite delivery."                                                           |
| Input (Corrective Action)             | `assigned_to_user_id` | string                  | No           | ID пользователя, которому назначается действие.         | UUID                     | "user-abcde"                                                                                      |
| Success Output (List/Dashboard Data)  | `total_items`         | integer                 | Yes          | Общее количество найденных элементов.                   |                          | 150                                                                                               |
| Success Output (List/Dashboard Data)  | `items`               | array                   | Yes          | Массив объектов (delivery units / violations).          | JSON Array               | `[ { "deliveryUnitId": "...", "currentStatus": "...", ... }, ... ]`                               |
| Success Output (Delivery Unit Detail) | `deliveryUnitId`      | string                  | Yes          |                                                         |                          | "a1b2c3d4-e5f6-7890-1234-567890abcdef"                                                            |
| Success Output (Delivery Unit Detail) | `statusHistory`       | array                   | Yes          | История статусов.                                       | Array of Objects         | `[ {"status": "взял", "timestamp": "..."}, ... ]`                                                 |
| Success Output (Delivery Unit Detail) | `activeViolations`    | array                   | Yes          | Активные нарушения для этой поставки.                   | Array of Objects         | `[ {"violation_id": "vio-inst-001", ...}, ... ]`                                                  |
| Success Output (Corrective Action)    | `action_id`           | string                  | Yes          | ID созданного действия.                                 | UUID                     | "ca-12345"                                                                                        |
| Success Output (Corrective Action)    | `status`              | string                  | Yes          | Статус действия (e.g., "New", "In Progress").           | Enum                     | "New"                                                                                             |
| Error Output                          | `error_code`          | string                  | Yes          | Код ошибки.                                             | See Error Code Table     | E3001                                                                                             |
| Error Output                          | `message`             | string                  | Yes          | Понятное сообщение об ошибке.                           |                          | "Failed to load delivery units."                                                                  |
| Error Output                          | `detail`              | object                  | No           | Детальная информация.                                   |                          |                                                                                                   |

**Error Code Examples (Framework)**

| **Error Code** | **Category**  | **User Message (Brief)**            | **Dev Notes / Handling**                                      |
| -------------- | ------------- | ----------------------------------- | ------------------------------------------------------------- |
| E3001          | System        | Failed to load delivery units.      | Ошибка при запросе данных из F1-001 Read API. Возврат 500.    |
| E3002          | System        | Failed to load violations.          | Ошибка при запросе данных из F1-002 API. Возврат 500.         |
| E3003          | Validation    | Invalid filter parameters.          | Некорректные значения в запросе фильтрации. Возврат 400.      |
| E3004          | Not Found     | Delivery unit not found.            | Запрошенная `deliveryUnitId` не найдена. Возврат 404.         |
| E3005          | Authorization | Permission denied.                  | Пользователь не имеет прав на просмотр/действие. Возврат 403. |
| E3006          | System        | Failed to create corrective action. | Ошибка при сохранении действия. Возврат 500.                  |
| E3007          | System        | Failed to export report.            | Ошибка при генерации отчета. Возврат 500.                     |

#### F1-003 API Contract (Integration Contract — Framework Constraints)

| **Category**               | **Framework Requirements**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| API Scope                  | <p>- RESTful API для получения списка единиц поставки (<code>GET /api/v1/operational/delivery-units</code>).<br>- RESTful API для получения детальной информации о единице поставки (<code>GET /api/v1/operational/delivery-units/{id}</code>).<br>- RESTful API для получения списка нарушений (<code>GET /api/v1/operational/violations</code>).<br>- RESTful API для создания корректирующего действия (<code>POST /api/v1/operational/corrective-actions</code>).<br>- RESTful API для обновления статуса нарушения/действия (<code>PUT /api/v1/operational/violations/{id}/status</code>, <code>PUT /api/v1/operational/corrective-actions/{id}/status</code>).<br>- RESTful API для экспорта отчетов (<code>GET /api/v1/operational/reports/violations</code>).</p> |
| Input Requirements         | <p>- Все GET запросы должны поддерживать пагинацию, сортировку и фильтрацию.<br>- POST/PUT запросы должны быть защищены от CSRF (если SPA).<br>- Строгая валидация параметров запроса.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Output Requirements        | <p>- Структурированные JSON ответы с пагинацией, фильтрацией.<br>- Успешные статусы <code>200 OK</code>, <code>201 Created</code>.<br>- Ошибки <code>4xx</code>, <code>5xx</code> со стандартным форматом.<br>- Экспорт отчетов может возвращаться как бинарный файл (e.g., <code>Content-Disposition: attachment; filename="violations_report.csv"</code>).</p>                                                                                                                                                                                                                                                                                                                                                                                                          |
| Error Handling             | <p>- Стандартные HTTP статусы, коды ошибок, понятные сообщения.<br>- Обработка различных сценариев: нет данных, ошибки сервера, права доступа.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Versioning                 | `v1`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Security                   | <p>- HTTPS.<br>- Аутентификация (e.g., JWT) и авторизация по ролям (Operator, Admin).<br>- RBAC (Role-Based Access Control).</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Rate Limiting & Throttling | - Ограничение на количество запросов для предотвращения DoS атак и чрезмерной нагрузки на бэкенд.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Dependency Management      | <p>- Интеграция с F1-001 (Read API) и F1-002 (Violations API).<br>- Возможно, интеграция с системой управления задачами (Service Desk, Jira).</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Observability              | <p>- Логирование каждого запроса, метрики (latency, throughput, error rate), кастомные метрики (e.g., <code>active_violations_count</code>, <code>created_corrective_actions_total</code>).<br>- Трассировка запросов через все бэкенд-сервисы.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Extensibility              | <p>- Возможность добавления новых ролей и прав доступа.<br>- Возможность добавления новых типов отчетов.<br>- Возможность интеграции с внешними системами трекинга задач.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Documentation              | - Swagger/OpenAPI для всех операционных API.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

#### F1-003 UI / Interaction Description

| **Category**                   | **Content Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Page: Dashboard**            | <p>- <strong>Header</strong>: Логотип, название маркетплейса, имя пользователя, кнопка выхода.<br>- <strong>Sidebar Navigation</strong>: Ссылки на "Dashboard", "Violations List", "SLA Configuration" (для Admin), "Reports".<br>- <strong>Main Content Area (Dashboard)</strong>:<br>- <strong>Overview Cards</strong>: Общее количество поставок, поставок в работе, нарушений SLA.<br>- <strong>Map View</strong>: Интерактивная карта с индикацией количества нарушений/просрочек по городам.<br>- <strong>Status Distribution Chart</strong>: Круговая диаграмма или гистограмма распределения статусов по всем поставкам.<br>- <strong>Recent Violations Feed</strong>: Список последних 5-10 нарушений.<br>- <strong>Interaction</strong>: Фильтрация по городам (на карте или выпадающий список).</p>         |
| **Page: Delivery Unit List**   | <p>- <strong>Header/Sidebar</strong>: как на Dashboard.<br>- <strong>Content Area</strong>:<br>- <strong>Filter Bar</strong>: Поля для поиска по ID / статусу / дате / городу.<br>- <strong>Table</strong>: Строки с <code>deliveryUnitId</code>, <code>currentStatus</code>, <code>lastUpdate</code>, <code>violationsCount</code>, <code>actions</code>.<br>- <strong>Pagination</strong>: навигация по странице.<br>- <strong>Interaction</strong>: Клик по строке для перехода к деталям <code>deliveryUnitId</code>. Кнопка "View Details" в столбце <code>actions</code>.</p>                                                                                                                                                                                                                                    |
| **Page: Delivery Unit Detail** | <p>- <strong>Header/Sidebar</strong>: как на Dashboard.<br>- <strong>Content Area</strong>:<br>- <strong>Top Section</strong>: <code>deliveryUnitId</code>, <code>currentStatus</code>, <code>customerInfo</code>, <code>destinationAddress</code>.<br>- <strong>Status History Section</strong>: Таблица/лента с историей всех событий (статус, время, актер).<br>- <strong>Active Violations Section</strong>: Список активных нарушений, связанных с этой поставкой, с кнопками "Assign to me", "View Action", "Create Action".<br>- <strong>Corrective Action Section</strong>: Форма для создания новой корректирующей действия или список существующих (с их статусами).<br>- <strong>Interaction</strong>: Клик по названию нарушения для просмотра деталей. Создание/редактирование действия.</p>              |
| **Page: Violations List**      | <p>- <strong>Header/Sidebar</strong>: как на Dashboard.<br>- <strong>Content Area</strong>:<br>- <strong>Filter Bar</strong>: Поиск по <code>violationId</code>, <code>deliveryUnitId</code>, <code>status</code> (New, In Progress, Resolved), <code>created_at</code> range.<br>- <strong>Table</strong>: Строки с <code>violationId</code>, <code>deliveryUnitId</code>, <code>violationType</code>, <code>description</code>, <code>occurredAt</code>, <code>status</code>, <code>assignedTo</code>, <code>actions</code>.<br>- <strong>Pagination</strong>.<br>- <strong>Action Buttons</strong>: "Assign to me", "Mark as In Progress", "Resolve", "Export Report".<br>- <strong>Interaction</strong>: Назначение нарушения, смена статуса. Клик на <code>deliveryUnitId</code> ведет на детальную страницу.</p> |
| Button Behavior                | <p>- <strong>Common Buttons</strong>: "Save", "Cancel", "Apply", "Clear Filters".<br>- <strong>Action Buttons</strong>: "Create Corrective Action", "Assign", "Resolve Violation".<br>- <strong>State Changes</strong>: Кнопки могут быть disabled, показывать loading-индикаторы.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Animations & Effects           | <p>- Плавные переходы страниц, появление модальных окон (например, для создания действия).<br>- Подсветка строк с нарушениями.<br>- Индикаторы загрузки при запросах к API.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Response & Notifications       | <p>- Toast-уведомления об успешных действиях (сохранено, назначено, разрешено).<br>- Inline-сообщения об ошибках валидации или системных ошибках.<br>- Звуковые алерты для критичных нарушений (опционально).</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Page Navigation Logic          | <p>- Прямые ссылки на страницы.<br>- Навигация согласно структуре (Dashboard → List → Detail).<br>- Браузерная история (back/forward buttons).</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Compatibility Notes            | <p>- Адаптивный дизайн для планшетов и ноутбуков.<br>- Тестирование в Chrome, Firefox, Safari, Edge.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Component References           | <p>- Data Table component with sorting, filtering, pagination.<br>- Map component (e.g., Leaflet, Google Maps API).<br>- Charting library (e.g., Chart.js, D3.js).<br>- Form components with validation.<br>- Modal windows.<br>- Toast notifications.</p>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |

#### F1-003 Testing Requirements (Operational Controller Workspace)

| Test Type              | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Functional Testing     | <p>- Проверка загрузки дашборда с актуальными данными.<br>- Применение всех доступных фильтров (по городу, статусу, дате).<br>- Детальный просмотр единицы поставки, проверка корректности истории статусов и нарушений.<br>- Создание корректирующего действия, проверка его сохранения и отображения.<br>- Изменение статуса нарушения (New → In Progress → Resolved).<br>- Экспорт отчета нарушений с примененными фильтрами.<br>- Navigating through pages.</p> |
| Boundary Testing       | <p>- Тест с пустыми фильтрами и очень большим количеством данных (проверка пагинации и производительности).<br>- Тест с граничными датами в фильтрах.<br>- Тест отправки пустого описания действия.</p>                                                                                                                                                                                                                                                             |
| Exception Testing      | <p>- Попытка доступа к страницам с невалидными правами доступа.<br>- Ошибка при загрузке данных (проверка отображения ошибки пользователю).<br>- Ошибка при создании действия (проверка сообщения об ошибке).<br>- Тест при недоступности бэкенд API.</p>                                                                                                                                                                                                           |
| Security Testing       | <p>- Тестирование RBAC: убедиться, что только Операционный контролер видит и может выполнять соответствующие действия.<br>- Проверка на CSRF, XSS.<br>- Тест на SQL injection в параметрах запросов.</p>                                                                                                                                                                                                                                                            |
| Performance Testing    | <p>- Измерение времени загрузки страниц и отклика на действия пользователя.<br>- Тест под нагрузкой (множество одновременных пользователей).<br>- Тест скорости экспорта больших отчетов.</p>                                                                                                                                                                                                                                                                       |
| UI Interaction Testing | <p>- Проверка работы всех элементов управления (кнопки, чекбоксы, радиокнопки, фильтры).<br>- Корректность отображения индикаторов загрузки.<br>- Отзывчивость интерфейса.</p>                                                                                                                                                                                                                                                                                      |
| Usability Testing      | <p>- Оценка простоты использования, интуитивности навигации.<br>- Четкость сообщений об ошибках и уведомлений.<br>- Удобство создания корректирующих действий.</p>                                                                                                                                                                                                                                                                                                  |
| Compatibility Testing  | - Проверка работы интерфейса в разных браузерах и на разных разрешениях экрана (responsive design).                                                                                                                                                                                                                                                                                                                                                                 |
| Data Integrity Testing | <p>- Проверка корректности сопоставления нарушений с единицами поставки, действий с нарушениями.<br>- Проверка, что экспортированные отчеты содержат верные данные.</p>                                                                                                                                                                                                                                                                                             |

#### F1-003 Visualization

*   **Data Flow Diagram (DFD) - Operator Workspace**

    ```mermaid
    graph LR
        A[Operator Browser] --> B(Frontend App);
        B --> C(API Gateway);
        C --> D[F1-001: Read API];
        C --> E[F1-002: Violations API];
        C --> F[F1-003: Operational API];
        D --> G[F1-001: DB];
        E --> H[F1-002: DB];
        F --> I[Corrective Actions Service/DB];
        I --> E;
        F --> E;
    ```
*   **Sequence Diagram - Creating a Corrective Action**

    ```mermaid
    sequenceDiagram
        participant Operator as Operator (UI)
        participant Frontend as Frontend Application
        participant API_GW as API Gateway
        participant Op_API as Operational API (F1-003)
        participant Violations_API as Violations API (F1-002)
        participant CA_Service as Corrective Action Service
        participant Notifier as Notification Service

        Operator->>Frontend: Clicks "Create Action" button for Violation X
        Frontend->>Frontend: Show modal form for Corrective Action
        Operator->>Frontend: Enters description, assigns task
        Frontend->>Frontend: Validates local form data
        Frontend->>API_GW: POST /api/v1/operational/corrective-actions (violation_id, description, assigned_to_user_id)

        API_GW->>Op_API: Forward request
        Op_API->>Op_API: Authenticate & Authorize User
        Op_API->>CA_Service: Create Corrective Action request
        CA_Service->>CA_Service: Validate data, generate CA ID
        CA_Service->>CA_Service: Store CA in its DB (or related DB)
        CA_Service-->>Op_API: Action created confirmation (CA ID, Status)
        Op_API-->>API_GW: Success response (CA ID, Status)
        API_GW-->>Frontend: Success response
        Frontend-->>Operator: Show "Corrective Action Created" toast notification

        Note over CA_Service,Notifier: If assignment is made, trigger notification
        CA_Service->>Notifier: Send notification to assigned user
    ```
* **UI Wireframe Snippets (Conceptual)**
  *   **Dashboard Overview Card**:

      ```
      +---------------------+
      | Critical Violations |
      |        [ 5 ]        |
      |   [View All ->]     |
      +---------------------+
      ```
  *   **Violation List Row**:

      ```
      | ViolID | DeliveryUnitID    | Type     | Status     | Assigned  | Actions          |
      |--------|-------------------|----------|------------|-----------|-----------------|
      | vio_001| du_abc123         | SLA_SHIP | New        | John Doe  | [Assign][View] |
      ```
  *   **Action Creation Modal**:

      ```
      +----------------------------------+
      | Create Corrective Action         |
      +----------------------------------+
      | Violation ID: vio_001            |
      | Delivery Unit ID: du_abc123      |
      | Description:                     |
      | [______________________________] |
      | Assigned To: [Dropdown: John Doe]|
      |                                  |
      | [Save]  [Cancel]                 |
      +----------------------------------+
      ```

@@@LOOP\_END@@@

***

### 5. Page Prototype / UI Wireframes

| Platform | Page Name                      | Page Functionality                                                                             | User Operations                                                    | Input Fields                                            | Output Fields                                                                 | Responsive Design                                                    | Accessibility                                               |
| -------- | ------------------------------ | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------------- | ----------------------------------------------------------- |
| Web      | Dashboard                      | Overview of system state, map-based data, key KPIs                                             | View metrics, filter by city                                       | City filter                                             | Map data, card values, status charts                                          | Desktop-first, responsive to tablet (limited features on mobile web) | Keyboard navigation, ARIA labels, sufficient color contrast |
| Web      | Delivery Unit List             | List of all delivery units, filtering, searching                                               | Filter, sort table, view details                                   | Search bar, status dropdown, date range picker          | Table with `deliveryUnitId`, `currentStatus`, `lastUpdate`, `violationsCount` | Desktop-first, responsive                                            | N/A                                                         |
| Web      | Delivery Unit Detail           | Detailed view of a single delivery unit, status history, active violations, corrective actions | View history, assign violation, create action                      | Action description, assignee selection                  | Status history, violation details, action details                             | Desktop-first, responsive                                            | N/A                                                         |
| Web      | Violations List                | List of all SLA violations, filtering, sorting, actions                                        | Filter violations, assign violations, change status, export report | Violation ID, status, date range, DU ID                 | Table with violation details, status, assignee                                | Desktop-first, responsive                                            | N/A                                                         |
| Web      | SLA Configuration (Admin only) | View, create, edit, delete SLA rules                                                           | Manage SLA policies                                                | SLA parameters (operation, duration, conditions, dates) | Table of SLA rules                                                            | Desktop-first                                                        | N/A                                                         |

***

### 6. Overall Interaction Design Description

| Page                                                | Element                                         | Behavior Description                                                                            | Animation/Transition                            | State                             |
| --------------------------------------------------- | ----------------------------------------------- | ----------------------------------------------------------------------------------------------- | ----------------------------------------------- | --------------------------------- |
| All Main Pages                                      | Navigation Sidebar                              | Collapses/expands on click. Highlight of the active page.                                       | Smooth slide-out/in animation.                  | Expanded / Collapsed              |
| Dashboard                                           | Map View                                        | Hovering over city shows tooltip with violation count. Clicking a city filters list/map data.   | Tooltip appears on hover. Smooth zoom/pan.      | Default view, Filtered view.      |
| Lists (DUs, Violations)                             | Table Rows                                      | Hover highlights row. Click selects row and enables action buttons (Assign, View Details).      | Hover state change.                             | Default, Selected.                |
| Creation/Edit Forms (SLA Config, Corrective Action) | Input Fields                                    | Focused state shows border. On validation error, border turns red, error message appears below. | Error message slides in.                        | Default, Focused, Error, Success. |
| Buttons                                             | Standard Action Buttons (Save, Assign, Resolve) | Default, Hover, Pressed states. Loading indicator appears during async ops.                     | Gentle press animation. Spinner icon in button. | Enabled, Disabled, Loading.       |
| Notifications                                       | Toast Messages                                  | Appear at top-right or bottom-right corner. Disappear after a few seconds or on user dismiss.   | Fade in/out, Slide.                             | Info, Success, Warning, Error.    |

***

### 7. Non-Functional Requirements

| Category                       | Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Performance Requirements**   | <p>- <strong>API Response Time</strong>: 95% of API responses must be &#x3C; 500ms. Critical read operations (DUs, Violations) > 99% &#x3C; 1000ms.<br>- <strong>Page Load Time</strong>: Dashboard and lists &#x3C; 5 seconds.<br>- <strong>Throughput</strong>: Capable of processing 10,000+ events/sec (initial) and scaling to 100,000+ events/sec with infrastructure.<br>- <strong>Concurrency</strong>: Support at least 100 concurrent users on the Operator Workspace.</p>                                        |
| **Scalability**                | <p>- Horizontal scalability for all microservices (Event Processor, SLA Monitor, APIs).<br>- Database scalability (sharding, replication) to handle growing data volume and read/write loads.<br>- Message Queue scalability (e.g., number of partitions).</p>                                                                                                                                                                                                                                                              |
| **Reliability & Availability** | <p>- <strong>Availability</strong>: Target SLA of 99.95% for the Tracking Journal Service and Operator Workspace.<br>- <strong>Fault Tolerance</strong>: No single point of failure. Services should auto-restart in case of crashes.<br>- <strong>Data Durability</strong>: Critical journal and violations data must be durable; use database replication and backups.</p>                                                                                                                                                |
| **Security**                   | <p>- <strong>Authentication</strong>: Secure authentication for all API endpoints and UI access.<br>- <strong>Authorization</strong>: Role-based access control (RBAC) for UI and API operations.<br>- <strong>Data Protection</strong>: HTTPS for all communication. Encryption at rest for sensitive data (if applicable). Regular security audits.<br>- <strong>Input Validation</strong>: Rigorous validation of all incoming data to prevent injection attacks.</p>                                                    |
| **Maintainability**            | <p>- Clean code, adherence to coding standards.<br>- Comprehensive unit and integration tests.<br>- Clear logging and monitoring.<br>- Well-documented codebase and APIs.<br>- Automated CI/CD pipelines.</p>                                                                                                                                                                                                                                                                                                               |
| **Usability**                  | <p>- Intuitive and easy-to-use interface for operational controllers.<br>- Clear and actionable information presentation.<br>- Minimal training required for core operations.</p>                                                                                                                                                                                                                                                                                                                                           |
| **Observability**              | <p>- <strong>Logging</strong>: Structured, centralized logging for all services. Log levels configurable.<br>- <strong>Metrics</strong>: Comprehensive metrics on system performance, resource usage, business KPIs (e.g., event throughput, SLA violation rate).<br>- <strong>Tracing</strong>: End-to-end distributed tracing across microservices for debugging.<br>- <strong>Alerting</strong>: Proactive alerts for system health issues and business-critical events (e.g., high error rates, SLA breach spikes).</p> |

***

### 8. Acceptance Criteria

| Feature Point                       | Acceptance Method                 | Acceptance Standard                                                                                                                                                                                                                                          |
| ----------------------------------- | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **F1-001: Event Ingestion**         | API Testing, Integration Testing  | <p>- All valid events are ingested and recorded in the journal within 500ms.<br>- Idempotent processing prevents duplicate entries.<br>- System handles 10,000+ events/sec without errors.</p>                                                               |
| **F1-002: SLA Violation Detection** | Manual Testing, Automated Tests   | <p>- SLA configurations are correctly applied.<br>- Violations are detected and logged within 1 minute of exceeding SLA.<br>- Notifications are sent for critical violations within 5 minutes.</p>                                                           |
| **F1-003: Operator Workspace**      | Manual Testing, UI Testing        | <p>- Dashboard displays accurate, real-time data.<br>- Users can filter and view delivery units and violations.<br>- Operators can create and manage corrective actions.<br>- Reports can be exported with correct data.<br>- RBAC is strictly enforced.</p> |
| **Highload & Scalability**          | Performance Testing, Load Testing | <p>- The system scales horizontally to handle increased load (e.g., 100k events/sec).<br>- Availability remains high (99.95%) under load.<br>- Latency stays within acceptable limits (&#x3C; 1 sec avg).</p>                                                |
| **Observability**                   | Monitoring Dashboard Review       | <p>- All required metrics and traces are collected.<br>- Alerts are configured and triggered for critical issues.<br>- Logs are accessible and searchable.</p>                                                                                               |

***

### 9. Risks & Boundary Description

#### 9.1 Project Risks

| Risk Category             | Risk Description                                                                       | Impact Level | Probability | Mitigation Strategy                                                                                                                                                          |
| ------------------------- | -------------------------------------------------------------------------------------- | ------------ | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Technical**             | Difficulty in achieving high throughput for event ingestion and SLA monitoring.        | High         | Medium      | Use asynchronous processing (Kafka), choose performant database (Cassandra, distributed PostgreSQL), optimize event processing logic. Performance testing from early stages. |
| **Integration**           | Unreliable or slow data from upstream systems (Warehouse, Courier).                    | High         | Medium      | Implement robust error handling, retries, circuit breakers. Define clear SLAs with upstream teams. Implement data validation at ingress.                                     |
| **Operational**           | Downtime of critical services (Kafka, DB, Journal Service) causes data loss or outage. | High         | Low         | Implement HA for all components, disaster recovery plan, regular backups.                                                                                                    |
| **Scope Creep**           | Addition of complex SLA rules or new types of events/statuses not initially planned.   | Medium       | Medium      | Strict change management process for scope. Prioritize features based on business value. MVP approach.                                                                       |
| **Resource Availability** | Lack of skilled personnel for highload/distributed systems development and operations. | Medium       | Medium      | Training, hiring, knowledge sharing sessions. Clear documentation.                                                                                                           |

#### 9.2 Product Boundaries

| Boundary Type            | Description                                                                                                                                                                                        | Reason                                              | Future Consideration                                                         |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Data Ownership**       | The microservice acts as a journal/log, not the primary source of truth for delivery unit details. Primary master data resides in other services (e.g., Order Service, Delivery Unit Service).     | Focus of this service is tracking events.           | Integrate with a dedicated Delivery Unit Management service for master data. |
| **User Roles**           | This PRD focuses on the Operator Controller. Functionality for other roles (e.g., Warehouse/Courier feedback on journal accuracy) may be out of scope for v1.0.                                    | Limited scope for MVP.                              | Future iterations can include user feedback mechanisms.                      |
| **Corrective Actions**   | The system records and tracks corrective actions. It does not _execute_ actions automatically (e.g., rerouting a courier). Complex workflow automation for corrective actions is out of scope.     | Focus on tracking and enabling manual intervention. | Complex workflow engine integration for auto-resolution.                     |
| **Analytics Beyond SLA** | Advanced analytics (e.g., root cause analysis of delivery failures beyond SLA breaches, carrier performance scoring) may require separate modules or integration with a data warehouse.            | MVP focus on SLA and operational monitoring.        | Dedicated BI/Data Analytics module.                                          |
| **Platform Integration** | The initial integration is with defined Warehouse and Courier backends. Integration with other logistics providers or custom supplier systems will be handled as separate integration initiatives. | Standardized integration for core services first.   | Dedicated integration layer/framework for external partners.                 |
