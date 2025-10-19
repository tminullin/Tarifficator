---
icon: gears
---

# System Architecture Design

### 1. System Overview

**System Name:** Tracking Journal Microservice (ARCH-2.1)\
**System Function Summary:** Централизованный высоконагруженный микросервис для приёма, хранения, анализа и визуализации статусов движения товаров, обеспечивающий контроль SLA складов и курьеров для Операционного контролера.

**Main Modules:**

1. **Сбор событий статусов (Ingestion Service)** — \[UC-F-001], \[US-001], \[US-002], \[US-003]
2. **Хранилище журнала и API чтения (Journal Store & Query API)** — \[UC-F-002], \[US-004], \[US-010]
3. **Модуль SLA-мониторинга и нарушений (SLA Monitor)** — \[UC-F-003], \[US-005], \[US-011], \[US-012]
4. **Уведомления и оркестрация действий (Notification & Actions)** — \[UC-F-004], \[US-007], \[US-008], \[US-009]
5. **Рабочее место Операционного контролера (Controller UI)** — \[UC-F-005], \[US-004], \[US-008], \[US-009], \[US-010]
6. **Подсистема Observability & Highload (Observability Platform)** — \[UC-F-006], \[US-006], \[US-013], \[US-014], \[US-015]

**Ключевые сценарии:**

* Регистрация статуса склада или курьера и запись в журнал \[US-001], \[US-002]
* Идентификация нарушения SLA и уведомление ответственных \[US-005], \[US-007], \[US-012]
* Операционный контроллер анализирует цепочку поставки и инициирует корректирующее действие \[US-004], \[US-008], \[US-010]
* Экспорт отчета по нарушениям для аудита \[US-009]
* Масштабирование системы при росте событий и пользователей \[US-006], \[US-013], \[US-014], \[US-015]

**Характеристики системы:**

* Высокая пропускная способность (10K–100K событий/сек) с горизонтальным масштабированием
* Идемпотентная обработка событий и консистентное хранение
* SLA-мониторинг в реальном времени с задержкой < 1 мин.
* Наблюдаемость уровня enterprise (метрики, логи, трассировка)
* Безопасность: RBAC, шифрование, аудит действий

***

### 2. Overall Architecture

#### 2.1 System Structure Diagram (God's View)

```mermaid
graph TD
    subgraph "Внешние сервисы"
        ORDERS_MS[ORDERS_MS]
        WAREHOUSE_MS[WAREHOUSE_MS]
        COURIER_MS[COURIER_MS]
        TICKETING_MS[TICKETING_MS]
    end
    subgraph "Слой доступа"
        API_GATEWAY[API_GATEWAY]
        EVENT_GATEWAY[EVENT_GATEWAY]
    end
    subgraph "Сервисы сбора статусов"
        EVENT_INGESTION[E_EVENT_INGESTION]
        EVENT_CACHE[E_EVENT_CACHE]
        JOURNAL_STORE[JOURNAL_STORE]
        QUERY_API[QUERY_API]
        SLA_MONITOR[SLA_MONITOR]
        VIOLATION_SERVICE[VIOLATION_SERVICE]
        ACTION_ORCHESTRATOR[ACTION_ORCHESTRATOR]
        CONTROLLER_UI[CONTROLLER_UI]
    end
    subgraph "Вспомогательные компоненты"
        KAFKA_CLUSTER[KAFKA_CLUSTER]
        DB_CLUSTER[DB_CLUSTER]
        ANALYTICS_CACHE[ANALYTICS_CACHE]
        NOTIFICATION_HUB[NOTIFICATION_HUB]
        OBSERVABILITY_STACK[OBSERVABILITY_STACK]
        IDP[IDP]
    end

    ORDERS_MS --> API_GATEWAY
    WAREHOUSE_MS --> API_GATEWAY
    COURIER_MS --> API_GATEWAY
    API_GATEWAY --> EVENT_GATEWAY
    EVENT_GATEWAY --> EVENT_INGESTION
    EVENT_INGESTION --> KAFKA_CLUSTER
    EVENT_INGESTION --> EVENT_CACHE
    KAFKA_CLUSTER --> SLA_MONITOR
    KAFKA_CLUSTER --> JOURNAL_STORE
    JOURNAL_STORE --> DB_CLUSTER
    SLA_MONITOR --> VIOLATION_SERVICE
    VIOLATION_SERVICE --> DB_CLUSTER
    VIOLATION_SERVICE --> NOTIFICATION_HUB
    NOTIFICATION_HUB --> CONTROLLER_UI
    ACTION_ORCHESTRATOR --> TICKETING_MS
    QUERY_API --> CONTROLLER_UI
    CONTROLLER_UI --> ACTION_ORCHESTRATOR
    CONTROLLER_UI --> QUERY_API
    SLA_MONITOR --> ANALYTICS_CACHE
    OBSERVABILITY_STACK --> EVENT_INGESTION
    OBSERVABILITY_STACK --> SLA_MONITOR
    OBSERVABILITY_STACK --> QUERY_API
    IDP --> CONTROLLER_UI
    IDP --> API_GATEWAY
```

#### 2.2 Functional Module Diagram

```mermaid
graph TD
    subgraph "功能模块"
        subgraph "数据采集"
            UC_F_001[UC-F-001 EVENT INGESTION LAYER]
        end
        subgraph "数据存储"
            UC_F_002[UC-F-002 JOURNAL STORE & QUERY]
        end
        subgraph "业务逻辑"
            UC_F_003[UC-F-003 SLA MONITORING]
            UC_F_004[UC-F-004 NOTIFICATION & ACTIONS]
        end
        subgraph "用户界面"
            UC_F_005[UC-F-005 CONTROLLER WORKBENCH]
        end
        subgraph "平台能力"
            UC_F_006[UC-F-006 OBSERVABILITY & HIGHLOAD]
        end
    end

    UC_F_001 --> UC_F_002
    UC_F_001 --> UC_F_003
    UC_F_002 --> UC_F_005
    UC_F_003 --> UC_F_004
    UC_F_004 --> UC_F_005
    UC_F_006 --> UC_F_001
    UC_F_006 --> UC_F_002
    UC_F_006 --> UC_F_003
    UC_F_006 --> UC_F_004
    UC_F_006 --> UC_F_005
```

***

### 3. Business Process Design

#### 3.1 Процесс обработки события статуса \[US-001], \[US-002], \[UC-F-001]

```mermaid
flowchart TD
    START(START) --> RECEIVE_EVENT[RECEIVE_EVENT]
    RECEIVE_EVENT --> VALIDATE_SCHEMA[VALIDATE_SCHEMA]
    VALIDATE_SCHEMA -->|OK| CHECK_IDEMPOTENCY[CHECK_IDEMPOTENCY]
    VALIDATE_SCHEMA -->|ERROR| RETURN_4XX[RETURN_4XX]
    CHECK_IDEMPOTENCY -->|DUPLICATE| ACK_DUPLICATE[ACK_DUPLICATE]
    CHECK_IDEMPOTENCY -->|NEW| ENQUEUE_EVENT[ENQUEUE_EVENT]
    ENQUEUE_EVENT --> PERSIST_WAL[PERSIST_WAL]
    PERSIST_WAL --> ASYNC_PUBLISH[ASYNC_PUBLISH_KAFKA]
    ASYNC_PUBLISH --> ACK_SOURCE[ACK_SOURCE]
    ACK_SOURCE --> END(END)
```

#### 3.2 Процесс реагирования на нарушение SLA \[US-005], \[US-007], \[US-012], \[UC-F-003], \[UC-F-004]

```mermaid
flowchart TD
    START(START) --> CONSUME_EVENT[CONSUME_EVENT_STREAM]
    CONSUME_EVENT --> FIND_PREV_STATUS[FIND_PREVIOUS_STATUS]
    FIND_PREV_STATUS --> CALCULATE_DELTA[CALCULATE_TIME_DELTA]
    CALCULATE_DELTA --> SLA_LOOKUP[SLA_CONFIG_LOOKUP]
    SLA_LOOKUP -->|NO_RULE| LOG_WARNING[LOG_RULE_MISS]
    SLA_LOOKUP -->|RULE_FOUND| CHECK_BREACH[CHECK_SLA_BREACH]
    CHECK_BREACH -->|NO| UPDATE_METRICS[UPDATE_METRICS_OK]
    CHECK_BREACH -->|YES| CREATE_VIOLATION[CREATE_VIOLATION_RECORD]
    CREATE_VIOLATION --> NOTIFY_CHANNELS[NOTIFY_CHANNELS]
    NOTIFY_CHANNELS --> ASSIGN_ACTION[ASSIGN_ACTION_QUEUE]
    UPDATE_METRICS --> END_OK(END)
    LOG_WARNING --> END_WARN(END)
    ASSIGN_ACTION --> END_BREACH(END)
```

***

### 4. State Transition Design

#### 4.1 Статусы единицы поставки \[US-001—US-004], \[UC-F-002], \[UC-F-003]

```mermaid
stateDiagram-v2
    [*] --> SUPPLIER_DISPATCHED
    SUPPLIER_DISPATCHED --> WAREHOUSE_RECEIVED : STATUS=Принял
    WAREHOUSE_RECEIVED --> WAREHOUSE_SHIPPED : STATUS=Отправил
    WAREHOUSE_SHIPPED --> COURIER_PICKED : STATUS=взял
    COURIER_PICKED --> COURIER_IN_PROGRESS : STATUS=в работе
    COURIER_IN_PROGRESS --> COURIER_DELIVERED : STATUS=отдал
    COURIER_IN_PROGRESS --> DELIVERY_FAILED : SLA_BREACH / FAILURE
    COURIER_DELIVERED --> [*]
    DELIVERY_FAILED --> [*]
```

***

### 5. Sequence Diagram Design

#### 5.1 Обработка нарушения и создание корректирующего действия \[US-007], \[US-008], \[UC-F-003], \[UC-F-004], \[UC-F-005]

```mermaid
sequenceDiagram
    participant WMS as WAREHOUSE_MS
    participant API as API_GATEWAY
    participant ING as EVENT_INGESTION
    participant KAF as KAFKA_CLUSTER
    participant SLA as SLA_MONITOR
    participant VIO as VIOLATION_SERVICE
    participant NOT as NOTIFICATION_HUB
    participant UI as CONTROLLER_UI
    participant ACT as ACTION_ORCHESTRATOR
    participant TKT as TICKETING_MS

    WMS->>API: POST STATUS_EVENT
    API->>ING: FORWARD EVENT
    ING->>KAF: PUBLISH EVENT_TOPIC
    KAF->>SLA: DELIVER EVENT_RECORD
    SLA->>SLA: CALCULATE SLA BREACH
    SLA->>VIO: CREATE VIOLATION
    VIO->>NOT: TRIGGER ALERT
    NOT-->>UI: PUSH ALERT
    UI->>UI: CONTROLLER REVIEWS VIOLATION
    UI->>ACT: CREATE CORRECTIVE_ACTION
    ACT->>TKT: OPEN INCIDENT
    TKT-->>ACT: INCIDENT_ID
    ACT-->>UI: CONFIRM ACTION
```

***

### 6. Data Flow Design

#### 6.1 System Data Flow Diagram \[PRD-2.1], \[US-001—US-015]

```mermaid
graph TD
    subgraph "事件源"
        SUPPLIER_SOURCE[SUPPLIER_SOURCE]
        WAREHOUSE_SOURCE[WAREHOUSE_SOURCE]
        COURIER_SOURCE[COURIER_SOURCE]
    end
    subgraph "管道层"
        API_LAYER[API_LAYER]
        QUEUE_LAYER[QUEUE_LAYER]
    end
    subgraph "处理层"
        INGEST_PROCESSOR[INGEST_PROCESSOR]
        SLA_PROCESSOR[SLA_PROCESSOR]
        ACTION_ENGINE[ACTION_ENGINE]
    end
    subgraph "存储层"
        JOURNAL_DB[JOURNAL_DB]
        VIOLATION_DB[VIOLATION_DB]
        CACHE_CLUSTER[CACHE_CLUSTER]
    end
    subgraph "消费层"
        REPORT_SERVICE[REPORT_SERVICE]
        CONTROLLER_APP[CONTROLLER_APP]
        ANALYTICS_SERVICE[ANALYTICS_SERVICE]
    end

    SUPPLIER_SOURCE --> API_LAYER
    WAREHOUSE_SOURCE --> API_LAYER
    COURIER_SOURCE --> API_LAYER
    API_LAYER --> QUEUE_LAYER
    QUEUE_LAYER --> INGEST_PROCESSOR
    INGEST_PROCESSOR --> JOURNAL_DB
    INGEST_PROCESSOR --> CACHE_CLUSTER
    QUEUE_LAYER --> SLA_PROCESSOR
    SLA_PROCESSOR --> VIOLATION_DB
    SLA_PROCESSOR --> ACTION_ENGINE
    ACTION_ENGINE --> CONTROLLER_APP
    JOURNAL_DB --> REPORT_SERVICE
    JOURNAL_DB --> CONTROLLER_APP
    VIOLATION_DB --> CONTROLLER_APP
    CACHE_CLUSTER --> ANALYTICS_SERVICE
```

***

### 7. Error Handling Design

#### 7.1 System Error Handling Process Diagram \[UC-F-001—UC-F-004]

```mermaid
flowchart TD
    START(START) --> DETECT_ERROR[DETECT_ERROR]
    DETECT_ERROR --> CLASSIFY_ERROR[CLASSIFY_ERROR]
    CLASSIFY_ERROR -->|VALIDATION| RETURN_4XX[RETURN_4XX_WITH_CODE]
    CLASSIFY_ERROR -->|IDEMPOTENT| RETURN_200_DUP[RETURN_OK_DUPLICATE]
    CLASSIFY_ERROR -->|SYSTEM| TRIGGER_RETRY[TRIGGER_RETRY]
    CLASSIFY_ERROR -->|QUEUE_BACKLOG| SHED_LOAD[SHED_LOAD]
    TRIGGER_RETRY --> UPDATE_METRIC_FAIL[UPDATE_FAILURE_METRIC]
    SHED_LOAD --> ISSUE_ALERT[ISSUE_LOAD_ALERT]
    RETURN_4XX --> LOG_CLIENT_ERROR[LOG_CLIENT_ERROR]
    RETURN_200_DUP --> LOG_DUP_EVENT[LOG_DUP_EVENT]
    UPDATE_METRIC_FAIL --> ISSUE_ALERT
    ISSUE_ALERT --> STORE_AUDIT[STORE_AUDIT]
    LOG_CLIENT_ERROR --> STORE_AUDIT
    LOG_DUP_EVENT --> STORE_AUDIT
    STORE_AUDIT --> END(END)
```

***

### 8. Deployment Architecture Design

#### 8.1 Kubernetes / Multi-Region Deployment \[UC-F-006], \[US-013], \[US-014]

```mermaid
graph TD
    subgraph "区域-A"
        subgraph "K8S集群-A"
            API_POD_A[API_GATEWAY_POD]
            INGEST_POD_A[INGESTION_POD]
            SLA_POD_A[SLA_MONITOR_POD]
            QUERY_POD_A[QUERY_API_POD]
            UI_POD_A[CONTROLLER_UI_POD]
            OBS_POD_A[OBSERVABILITY_AGENT]
        end
        subgraph "数据层-A"
            KAFKA_A[KAFKA_BROKER_A]
            DB_A[JOURNAL_DB_A]
            CACHE_A[CACHE_NODE_A]
        end
    end
    subgraph "区域-B"
        subgraph "K8S集群-B"
            API_POD_B[API_GATEWAY_POD]
            INGEST_POD_B[INGESTION_POD]
            SLA_POD_B[SLA_MONITOR_POD]
            QUERY_POD_B[QUERY_API_POD]
            UI_POD_B[CONTROLLER_UI_POD]
            OBS_POD_B[OBSERVABILITY_AGENT]
        end
        subgraph "数据层-B"
            KAFKA_B[KAFKA_BROKER_B]
            DB_B[JOURNAL_DB_B]
            CACHE_B[CACHE_NODE_B]
        end
    end
    GLOBAL_LB[GLOBAL_LOAD_BALANCER] --> API_POD_A
    GLOBAL_LB --> API_POD_B
    API_POD_A --> INGEST_POD_A
    API_POD_B --> INGEST_POD_B
    INGEST_POD_A --> KAFKA_A
    INGEST_POD_B --> KAFKA_B
    KAFKA_A --> DB_A
    KAFKA_B --> DB_B
    SLA_POD_A --> KAFKA_A
    SLA_POD_B --> KAFKA_B
    QUERY_POD_A --> DB_A
    QUERY_POD_B --> DB_B
    UI_POD_A --> QUERY_POD_A
    UI_POD_B --> QUERY_POD_B
    OBS_POD_A --> OBSERVABILITY_CORE[OBSERVABILITY_CORE]
    OBS_POD_B --> OBSERVABILITY_CORE
    KAFKA_A <---> KAFKA_B
    DB_A <---> DB_B
    CACHE_A <---> CACHE_B
```

***

### 9. Security and Permissions Design

#### 9.1 Permission Control Architecture Diagram \[UC-F-005], \[US-007], \[US-008]

```mermaid
graph TD
    subgraph "身份与授权"
        IDP_SERVICE[IDP_SERVICE]
        RBAC_ENGINE[RBAC_ENGINE]
        AUDIT_LOGGER[AUDIT_LOGGER]
    end
    subgraph "用户层"
        CONTROLLER_USER[ROLE_CONTROLLER]
        ADMIN_USER[ROLE_ADMIN]
        VIEWER_USER[ROLE_VIEWER]
    end
    subgraph "资源层"
        API_SCOPE[API_SCOPE]
        UI_SCOPE[UI_SCOPE]
        SLA_SCOPE[SLA_SCOPE]
        ACTION_SCOPE[ACTION_SCOPE]
    end

    CONTROLLER_USER --> IDP_SERVICE
    ADMIN_USER --> IDP_SERVICE
    VIEWER_USER --> IDP_SERVICE
    IDP_SERVICE --> RBAC_ENGINE
    RBAC_ENGINE --> API_SCOPE
    RBAC_ENGINE --> UI_SCOPE
    RBAC_ENGINE --> SLA_SCOPE
    RBAC_ENGINE --> ACTION_SCOPE
    API_SCOPE --> AUDIT_LOGGER
    UI_SCOPE --> AUDIT_LOGGER
    SLA_SCOPE --> AUDIT_LOGGER
    ACTION_SCOPE --> AUDIT_LOGGER
```

***

### 10. Operations and Monitoring Design

#### 10.1 System Monitoring Architecture Diagram \[UC-F-006], \[US-015]

```mermaid
graph TD
    subgraph "采集端"
        METRIC_AGENT[METRIC_AGENT]
        LOG_SHIPPER[LOG_SHIPPER]
        TRACE_EXPORTER[TRACE_EXPORTER]
    end
    subgraph "监控核心"
        PROMETHEUS[PROMETHEUS_CLUSTER]
        LOKI[LOKI_STACK]
        TEMPO[TEMPO_STACK]
        ALERT_MANAGER[ALERT_MANAGER]
    end
    subgraph "展示端"
        GRAFANA[GRAFANA_PORTAL]
        ALERT_CHANNELS[ALERT_CHANNELS]
    end

    METRIC_AGENT --> PROMETHEUS
    LOG_SHIPPER --> LOKI
    TRACE_EXPORTER --> TEMPO
    PROMETHEUS --> ALERT_MANAGER
    ALERT_MANAGER --> ALERT_CHANNELS
    PROMETHEUS --> GRAFANA
    LOKI --> GRAFANA
    TEMPO --> GRAFANA
```

***

### 11. Architecture Diagram Index

| #  | Diagram Title                                     | Section | Mermaid Identifier              |
| -- | ------------------------------------------------- | ------- | ------------------------------- |
| 1  | System Structure Diagram                          | 2.1     | graph TD (Overall Architecture) |
| 2  | Functional Module Diagram                         | 2.2     | graph TD (Modules)              |
| 3  | Process: Обработка события статуса                | 3.1     | flowchart TD                    |
| 4  | Process: Реагирование на нарушение SLA            | 3.2     | flowchart TD                    |
| 5  | State Diagram: Статусы единицы поставки           | 4.1     | stateDiagram-v2                 |
| 6  | Sequence: Нарушение SLA и корректирующее действие | 5.1     | sequenceDiagram                 |
| 7  | Data Flow Diagram                                 | 6.1     | graph TD                        |
| 8  | Error Handling Flow                               | 7.1     | flowchart TD                    |
| 9  | Deployment Architecture (Kubernetes)              | 8.1     | graph TD                        |
| 10 | Security & Permissions Architecture               | 9.1     | graph TD                        |
| 11 | Monitoring & Observability Architecture           | 10.1    | graph TD                        |

***

#### Дополнительные замечания и рекомендации

1. **Технологический стек (предварительно):**
   * API Gateway: NGINX Ingress + Kong/Apigee
   * Event Stream: Apache Kafka (с 6+ партициями на ключи `deliveryUnitId`)
   * Хранилище журнала: PostgreSQL Citus / Apache Cassandra (зависит от требований консистентности), плюс Elasticsearch для полнотекстовых запросов UI
   * Кэш и аналитика: Redis Cluster / Aerospike
   * SLA Processing: Flink / Kafka Streams для real-time расчётов
   * Контроллер UI: React + TypeScript, SSR через Next.js по необходимости
   * Observability: Prometheus, Grafana, Loki, Tempo, OpenTelemetry SDK
   * Безопасность: OAuth2/OIDC через Keycloak/Okta, mTLS внутри сети
2. **Нефункциональные параметры:**
   * SLA сервиса: 99.95% доступности
   * RPO ≤ 5 минут для журнала, RTO ≤ 15 минут
   * Среднее время обработки события: < 200 мс (ингест), < 60 секунд до выявления нарушения SLA
   * Гарантия идемпотентности через хранение `event_id` и TTL-кэш на 24ч
3. **План расширения и устойчивости:**
   * Гео-распределённые кластеры (Active/Active) с CDC-репликацией журналов

* Авто-масштабирование ingestion и SLA-подов по нагрузке
* Категоризация нарушений и приоритизация уведомлений

4. **Трейсабилити:**
   * Модули и диаграммы покрывают требования \[PRD-2.1] и \[US-001…US-015].
   * UC-F-001…UC-F-006 соответствуют функциональным блокам PRD, создавая связку PRD → User Stories → Architecture.
