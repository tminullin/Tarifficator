---
icon: timeline-arrow
---

# Business Process Design

### 1. Основной бизнес-процесс приема и журналирования событий статусов

#### 1.1 Процесс приема события статуса от внешних систем

Данный процесс реализует \[US-001], \[US-002], \[US-003] и соответствует \[UC-F-001] из архитектурного документа.

```mermaid
flowchart TD
    START([Внешняя система отправляет событие]) --> RECEIVE_EVENT[Получение HTTP запроса на API Gateway]
    RECEIVE_EVENT --> AUTH_CHECK{Проверка аутентификации?}
    AUTH_CHECK -->|Неуспешно| ERROR_AUTH[Возврат 401 Unauthorized]
    AUTH_CHECK -->|Успешно| VALIDATE_SCHEMA[Валидация схемы события]
    VALIDATE_SCHEMA -->|Невалидно| ERROR_VALIDATION[Возврат 400 Bad Request с деталями ошибки]
    VALIDATE_SCHEMA -->|Валидно| EXTRACT_EVENT_ID[Извлечение event_id из payload]
    EXTRACT_EVENT_ID --> CHECK_DUPLICATE{Проверка дубликата в кэше идемпотентности}
    CHECK_DUPLICATE -->|Дубликат найден| RETURN_ACK_DUPLICATE[Возврат 200 OK с подтверждением]
    CHECK_DUPLICATE -->|Новое событие| ENRICH_METADATA[Обогащение метаданными: timestamp получения, source IP, trace_id]
    ENRICH_METADATA --> STORE_IDEMPOTENCY[Сохранение event_id в кэш идемпотентности с TTL 24ч]
    STORE_IDEMPOTENCY --> PUBLISH_KAFKA[Публикация в Kafka topic events_raw]
    PUBLISH_KAFKA -->|Ошибка Kafka| ERROR_SYSTEM[Возврат 503 Service Unavailable]
    PUBLISH_KAFKA -->|Успешно| RETURN_ACK[Возврат 202 Accepted]
    ERROR_AUTH --> LOG_ERROR_DATA[Запись в лог ошибок]
    ERROR_VALIDATION --> LOG_ERROR_DATA
    ERROR_SYSTEM --> LOG_ERROR_DATA
    RETURN_ACK_DUPLICATE --> LOG_INFO_DATA[Запись в информационный лог]
    RETURN_ACK --> LOG_INFO_DATA
    LOG_ERROR_DATA --> END([Конец процесса])
    LOG_INFO_DATA --> END
```

#### 1.2 Структуры данных для процесса приема событий

```typescript
// Входящее событие статуса
interface IncomingStatusEvent {
  deliveryUnitId: string;        // UUID единицы поставки
  actor: {
    type: 'WAREHOUSE' | 'COURIER' | 'SUPPLIER';
    id: string;                  // ID конкретного склада/курьера
    name?: string;               // Имя для отображения
    location?: {
      city: string;
      region?: string;
      coordinates?: {
        lat: number;
        lng: number;
      };
    };
  };
  status: string;                // "Принял", "Отправил", "взял", "в работе", "отдал"
  timestamp: string;             // ISO 8601 UTC
  event_id: string;              // UUID для идемпотентности
  metadata?: {
    reason_code?: string;        // Код причины (опционально)
    notes?: string;              // Комментарии
    previous_attempt?: boolean;   // Признак повторной попытки
  };
}

// Обогащенное событие для внутренней обработки
interface EnrichedStatusEvent extends IncomingStatusEvent {
  received_at: string;           // Время получения системой
  source_ip: string;             // IP источника
  trace_id: string;              // Идентификатор трассировки
  processing_region: string;     // Регион обработки
}
```

***

### 2. Бизнес-процесс обработки и хранения событий в журнале

#### 2.1 Процесс асинхронной обработки событий

Данный процесс реализует \[US-003] и соответствует \[UC-F-002] из архитектурного документа.

```mermaid
flowchart TD
    START([Событие поступило в Kafka]) --> CONSUME_MESSAGE[Consumer получает сообщение из топика]
    CONSUME_MESSAGE --> DESERIALIZE[Десериализация JSON payload]
    DESERIALIZE -->|Ошибка| ERROR_DESERIALIZE[Отправка в DLQ: dead_letter_events]
    DESERIALIZE -->|Успешно| VALIDATE_BUSINESS_RULES[Проверка бизнес-правил]
    VALIDATE_BUSINESS_RULES -->|Нарушение правил| ERROR_BUSINESS_RULE[Отправка в DLQ: invalid_business_events]
    VALIDATE_BUSINESS_RULES -->|Правила соблюдены| CHECK_DELIVERY_UNIT{Существует ли deliveryUnitId в системе?}
    CHECK_DELIVERY_UNIT -->|Не существует| CREATE_DELIVERY_UNIT[Создание записи единицы поставки]
    CHECK_DELIVERY_UNIT -->|Существует| GET_LAST_STATUS[Получение последнего статуса из БД]
    CREATE_DELIVERY_UNIT --> GET_LAST_STATUS
    GET_LAST_STATUS --> VALIDATE_STATUS_TRANSITION{Валиден ли переход статуса?}
    VALIDATE_STATUS_TRANSITION -->|Невалиден| LOG_WARNING[Запись предупреждения, но сохранение события]
    VALIDATE_STATUS_TRANSITION -->|Валиден| UPDATE_CURRENT_STATUS[Обновление текущего статуса единицы поставки]
    LOG_WARNING --> UPDATE_CURRENT_STATUS
    UPDATE_CURRENT_STATUS --> INSERT_EVENT_RECORD[Вставка записи события в таблицу journal_events]
    INSERT_EVENT_RECORD -->|Ошибка БД| ERROR_DB[Повторная попытка с экспоненциальной задержкой]
    INSERT_EVENT_RECORD -->|Успешно| UPDATE_ANALYTICS_CACHE[Обновление кэша аналитики для реального времени]
    UPDATE_ANALYTICS_CACHE --> PUBLISH_PROCESSED[Публикация в топик events_processed для SLA мониторинга]
    PUBLISH_PROCESSED --> COMMIT_OFFSET[Подтверждение обработки сообщения Kafka]
    ERROR_DESERIALIZE --> UPDATE_ERROR_METRICS[Обновление метрик ошибок]
    ERROR_BUSINESS_RULE --> UPDATE_ERROR_METRICS
    ERROR_DB --> RETRY_LOGIC{Превышен лимит повторов?}
    RETRY_LOGIC -->|Нет| INSERT_EVENT_RECORD
    RETRY_LOGIC -->|Да| ERROR_DLQ[Отправка в DLQ: processing_failed_events]
    ERROR_DLQ --> UPDATE_ERROR_METRICS
    UPDATE_ERROR_METRICS --> END([Конец процесса])
    COMMIT_OFFSET --> END
```

#### 2.2 Структуры данных для журнала событий

```typescript
// Запись единицы поставки
interface DeliveryUnit {
  id: string;                    // UUID
  supplier_id: string;           // ID поставщика
  current_status: string;        // Текущий статус
  current_actor_type: string;    // Тип текущего актора
  current_actor_id: string;      // ID текущего актора
  created_at: string;            // Время создания записи
  updated_at: string;            // Время последнего обновления
  destination_city: string;      // Город назначения
  estimated_delivery: string;    // Плановое время доставки
  metadata: {
    customer_id?: string;
    priority_level?: number;
    special_handling?: boolean;
  };
}

// Запись события в журнале
interface JournalEventRecord {
  id: string;                    // UUID записи
  delivery_unit_id: string;      // Ссылка на единицу поставки
  event_id: string;              // Оригинальный event_id
  actor_type: string;            // WAREHOUSE, COURIER, SUPPLIER
  actor_id: string;              // ID актора
  actor_name: string;            // Имя актора
  status: string;                // Статус события
  timestamp: string;             // Время события (от источника)
  received_at: string;           // Время получения системой
  processed_at: string;          // Время обработки
  trace_id: string;              // Трассировка
  location_data: {
    city: string;
    region?: string;
    coordinates?: {
      lat: number;
      lng: number;
    };
  };
  metadata: object;              // Дополнительные данные
}
```

***

### 3. Бизнес-процесс SLA мониторинга и выявления нарушений

#### 3.1 Процесс анализа соблюдения SLA

Данный процесс реализует \[US-005], \[US-011], \[US-012] и соответствует \[UC-F-003] из архитектурного документа.

```mermaid
flowchart TD
    START([Получено обработанное событие]) --> IDENTIFY_TRANSITION[Определение типа перехода статуса]
    IDENTIFY_TRANSITION --> LOOKUP_SLA_CONFIG[Поиск конфигурации SLA для данного перехода]
    LOOKUP_SLA_CONFIG -->|Конфигурация не найдена| LOG_NO_SLA[Запись в лог отсутствия SLA правила]
    LOOKUP_SLA_CONFIG -->|Конфигурация найдена| GET_PREVIOUS_EVENT[Получение предыдущего события для deliveryUnitId]
    GET_PREVIOUS_EVENT -->|Предыдущее событие не найдено| MARK_INITIAL_STATE[Пометка как начальное состояние]
    GET_PREVIOUS_EVENT -->|Предыдущее событие найдено| CALCULATE_DURATION[Расчет времени выполнения операции]
    CALCULATE_DURATION --> APPLY_BUSINESS_HOURS{Учитывать рабочие часы?}
    APPLY_BUSINESS_HOURS -->|Да| ADJUST_FOR_WORKING_HOURS[Корректировка на рабочие часы и праздники]
    APPLY_BUSINESS_HOURS -->|Нет| COMPARE_WITH_SLA[Сравнение с нормативом SLA]
    ADJUST_FOR_WORKING_HOURS --> COMPARE_WITH_SLA
    COMPARE_WITH_SLA -->|Норматив соблюден| UPDATE_SLA_METRICS_OK[Обновление метрик успешного выполнения]
    COMPARE_WITH_SLA -->|Норматив нарушен| CREATE_VIOLATION_RECORD[Создание записи нарушения]
    CREATE_VIOLATION_RECORD --> DETERMINE_SEVERITY[Определение критичности нарушения]
    DETERMINE_SEVERITY --> CALCULATE_IMPACT[Расчет влияния на клиентский опыт]
    CALCULATE_IMPACT --> STORE_VIOLATION[Сохранение записи нарушения в БД]
    STORE_VIOLATION --> TRIGGER_NOTIFICATION_FLOW[Запуск процесса уведомлений]
    LOG_NO_SLA --> UPDATE_MONITORING_METRICS[Обновление метрик мониторинга]
    MARK_INITIAL_STATE --> UPDATE_MONITORING_METRICS
    UPDATE_SLA_METRICS_OK --> UPDATE_MONITORING_METRICS
    TRIGGER_NOTIFICATION_FLOW --> UPDATE_MONITORING_METRICS
    UPDATE_MONITORING_METRICS --> END([Конец процесса])
```

#### 3.2 Структуры данных для SLA конфигурации и нарушений

```typescript
// Конфигурация SLA
interface SLAConfiguration {
  id: string;                    // UUID конфигурации
  operation_type: string;        // "WAREHOUSE_RECEIVE", "WAREHOUSE_SHIP", "COURIER_PICKUP", etc.
  from_status: string;           // Исходный статус
  to_status: string;             // Целевой статус
  max_duration_minutes: number;  // Максимальное время в минутах
  apply_working_hours: boolean;  // Учитывать рабочие часы
  conditions: {
    actor_type?: string;         // Ограничение по типу актора
    city?: string[];             // Ограничение по городам
    priority_level?: number;     // Ограничение по приоритету
    special_handling?: boolean;  // Особая обработка
  };
  effective_from: string;        // Дата начала действия
  effective_to?: string;         // Дата окончания действия
  created_by: string;            // Создатель правила
  version: number;               // Версия конфигурации
}

// Запись нарушения SLA
interface SLAViolationRecord {
  id: string;                    // UUID нарушения
  delivery_unit_id: string;      // Единица поставки
  sla_config_id: string;         // Конфигурация SLA
  violation_type: 'SLA_EXCEEDED' | 'CRITICAL_DELAY' | 'BUSINESS_HOURS_EXCEEDED';
  from_event_id: string;         // ID исходного события
  to_event_id: string;           // ID целевого события
  expected_duration_minutes: number;  // Ожидаемое время
  actual_duration_minutes: number;    // Фактическое время
  delay_minutes: number;         // Величина задержки
  severity: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  impact_score: number;          // Оценка влияния (0-100)
  occurred_at: string;           // Время выявления нарушения
  detected_at: string;           // Время детектирования системой
  status: 'NEW' | 'ACKNOWLEDGED' | 'IN_PROGRESS' | 'RESOLVED' | 'CLOSED';
  assigned_to?: string;          // Назначено пользователю
  resolution_notes?: string;     // Заметки по решению
  metadata: {
    affected_customer_count?: number;
    estimated_delivery_delay?: number;
    root_cause_category?: string;
  };
}
```

***

### 4. Бизнес-процесс уведомлений и эскалации

#### 4.1 Процесс отправки уведомлений о нарушениях

Данный процесс реализует \[US-007] и соответствует \[UC-F-004] из архитектурного документа.

```mermaid
flowchart TD
    START([Создана запись нарушения SLA]) --> DETERMINE_NOTIFICATION_RULES[Определение правил уведомления по severity]
    DETERMINE_NOTIFICATION_RULES --> GET_RECIPIENT_LIST[Получение списка получателей уведомлений]
    GET_RECIPIENT_LIST --> CHECK_NOTIFICATION_COOLDOWN{Проверка cooldown периода для данного типа нарушения}
    CHECK_NOTIFICATION_COOLDOWN -->|В cooldown| LOG_SUPPRESSED[Запись в лог подавленного уведомления]
    CHECK_NOTIFICATION_COOLDOWN -->|Можно отправлять| PREPARE_NOTIFICATION_CONTEXT[Подготовка контекста уведомления]
    PREPARE_NOTIFICATION_CONTEXT --> ENRICH_WITH_DELIVERY_INFO[Обогащение информацией о доставке]
    ENRICH_WITH_DELIVERY_INFO --> DETERMINE_CHANNELS[Определение каналов уведомления]
    DETERMINE_CHANNELS --> SEND_EMAIL{Отправка Email?}
    SEND_EMAIL -->|Да| SEND_EMAIL_NOTIFICATION[Отправка Email уведомления]
    SEND_EMAIL -->|Нет| SEND_SLACK{Отправка Slack?}
    SEND_EMAIL_NOTIFICATION --> SEND_SLACK
    SEND_SLACK -->|Да| SEND_SLACK_NOTIFICATION[Отправка Slack уведомления]
    SEND_SLACK -->|Нет| SEND_PUSH{Отправка Push?}
    SEND_SLACK_NOTIFICATION --> SEND_PUSH
    SEND_PUSH -->|Да| SEND_PUSH_NOTIFICATION[Отправка Push уведомления]
    SEND_PUSH -->|Нет| CHECK_CRITICAL{Критичное нарушение?}
    SEND_PUSH_NOTIFICATION --> CHECK_CRITICAL
    CHECK_CRITICAL -->|Да| TRIGGER_ESCALATION[Запуск процесса эскалации]
    CHECK_CRITICAL -->|Нет| UPDATE_NOTIFICATION_STATUS[Обновление статуса уведомления]
    TRIGGER_ESCALATION --> SCHEDULE_ESCALATION_TIMER[Установка таймера эскалации]
    SCHEDULE_ESCALATION_TIMER --> UPDATE_NOTIFICATION_STATUS
    LOG_SUPPRESSED --> UPDATE_NOTIFICATION_STATUS
    UPDATE_NOTIFICATION_STATUS --> LOG_NOTIFICATION_RESULT[Запись результата уведомления]
    LOG_NOTIFICATION_RESULT --> END([Конец процесса])
```

#### 4.2 Структуры данных для уведомлений

```typescript
// Правила уведомлений
interface NotificationRule {
  id: string;
  name: string;
  trigger_conditions: {
    violation_severity: string[];
    violation_types: string[];
    actor_types: string[];
    cities: string[];
    time_of_day?: {
      start: string;             // "09:00"
      end: string;               // "18:00"
    };
  };
  recipients: {
    roles: string[];             // ["OPERATIONS_CONTROLLER", "SHIFT_MANAGER"]
    specific_users: string[];    // Конкретные пользователи
    escalation_chain: {
      level: number;
      delay_minutes: number;
      recipients: string[];
    }[];
  };
  channels: {
    email: boolean;
    slack: boolean;
    push: boolean;
    sms: boolean;
  };
  cooldown_minutes: number;      // Минимальный интервал между уведомлениями
  active: boolean;
}

// Контекст уведомления
interface NotificationContext {
  violation_id: string;
  delivery_unit_id: string;
  violation_summary: {
    type: string;
    severity: string;
    delay_minutes: number;
    expected_time: string;
    actual_time: string;
  };
  delivery_info: {
    customer_id?: string;
    destination_city: string;
    current_actor: string;
    estimated_delivery: string;
  };
  suggested_actions: string[];   // Рекомендуемые действия
  deep_link_url: string;         // Ссылка в UI контроллера
  timestamp: string;
}
```

***

### 5. Бизнес-процесс управления корректирующими действиями

#### 5.1 Процесс создания и отслеживания корректирующих действий

Данный процесс реализует \[US-008] и частично \[UC-F-004], \[UC-F-005] из архитектурного документа.

```mermaid
flowchart TD
    START([Операционный контролер инициирует действие]) --> VALIDATE_USER_PERMISSIONS[Проверка прав пользователя]
    VALIDATE_USER_PERMISSIONS -->|Нет прав| ERROR_UNAUTHORIZED[Возврат ошибки авторизации]
    VALIDATE_USER_PERMISSIONS -->|Есть права| VALIDATE_ACTION_REQUEST[Валидация запроса на создание действия]
    VALIDATE_ACTION_REQUEST -->|Невалидный| ERROR_VALIDATION_ACTION[Возврат ошибки валидации]
    VALIDATE_ACTION_REQUEST -->|Валидный| DETERMINE_ACTION_TYPE[Определение типа корректирующего действия]
    DETERMINE_ACTION_TYPE --> GET_VIOLATION_CONTEXT[Получение контекста нарушения]
    GET_VIOLATION_CONTEXT --> SUGGEST_ACTION_TEMPLATES[Предложение шаблонов действий]
    SUGGEST_ACTION_TEMPLATES --> CREATE_ACTION_RECORD[Создание записи корректирующего действия]
    CREATE_ACTION_RECORD --> ASSIGN_ACTION{Назначение исполнителя}
    ASSIGN_ACTION -->|Самому себе| ASSIGN_TO_CREATOR[Назначение создателю]
    ASSIGN_ACTION -->|Другому пользователю| ASSIGN_TO_USER[Назначение указанному пользователю]
    ASSIGN_ACTION -->|Команде| ASSIGN_TO_TEAM[Назначение команде]
    ASSIGN_TO_CREATOR --> DETERMINE_INTEGRATION_TYPE[Определение типа внешней интеграции]
    ASSIGN_TO_USER --> SEND_ASSIGNMENT_NOTIFICATION[Отправка уведомления о назначении]
    ASSIGN_TO_TEAM --> SEND_ASSIGNMENT_NOTIFICATION
    SEND_ASSIGNMENT_NOTIFICATION --> DETERMINE_INTEGRATION_TYPE
    DETERMINE_INTEGRATION_TYPE --> CREATE_EXTERNAL_TICKET{Создание внешнего тикета?}
    CREATE_EXTERNAL_TICKET -->|Да| INTEGRATE_TICKETING_SYSTEM[Интеграция с системой тикетинга]
    CREATE_EXTERNAL_TICKET -->|Нет| SET_INTERNAL_TRACKING[Установка внутреннего отслеживания]
    INTEGRATE_TICKETING_SYSTEM --> STORE_EXTERNAL_REFERENCE[Сохранение ссылки на внешний тикет]
    STORE_EXTERNAL_REFERENCE --> UPDATE_ACTION_STATUS[Обновление статуса действия на "Assigned"]
    SET_INTERNAL_TRACKING --> UPDATE_ACTION_STATUS
    UPDATE_ACTION_STATUS --> SETUP_FOLLOW_UP_SCHEDULE[Настройка расписания контроля]
    SETUP_FOLLOW_UP_SCHEDULE --> RETURN_ACTION_DETAILS[Возврат деталей созданного действия]
    ERROR_UNAUTHORIZED --> LOG_SECURITY_EVENT[Запись события безопасности]
    ERROR_VALIDATION_ACTION --> LOG_VALIDATION_ERROR[Запись ошибки валидации]
    LOG_SECURITY_EVENT --> END([Конец процесса])
    LOG_VALIDATION_ERROR --> END
    RETURN_ACTION_DETAILS --> END
```

#### 5.2 Структуры данных для корректирующих действий

```typescript
// Корректирующее действие
interface CorrectiveAction {
  id: string;                    // UUID действия
  violation_id: string;          // Связанное нарушение
  delivery_unit_id: string;      // Единица поставки
  action_type: 'MANUAL_INTERVENTION' | 'PROCESS_ESCALATION' | 'SYSTEM_ADJUSTMENT' | 'COMMUNICATION' | 'INVESTIGATION';
  title: string;                 // Краткое описание действия
  description: string;           // Подробное описание
  priority: 'LOW' | 'MEDIUM' | 'HIGH' | 'URGENT';
  status: 'NEW' | 'ASSIGNED' | 'IN_PROGRESS' | 'WAITING_EXTERNAL' | 'COMPLETED' | 'CANCELLED';
  created_by: string;            // Создатель действия
  assigned_to?: string;          // Назначено пользователю
  assigned_team?: string;        // Назначено команде
  created_at: string;            // Время создания
  due_date?: string;             // Срок выполнения
  started_at?: string;           // Время начала работы
  completed_at?: string;         // Время завершения
  external_ticket?: {
    system: string;              // "JIRA", "ServiceNow", etc.
    ticket_id: string;           // ID во внешней системе
    ticket_url: string;          // Прямая ссылка
  };
  progress_updates: {
    timestamp: string;
    user_id: string;
    status_change?: string;
    notes: string;
    attachments?: string[];
  }[];
  outcome?: {
    resolution_type: 'RESOLVED' | 'MITIGATED' | 'ESCALATED' | 'NO_ACTION_NEEDED';
    resolution_notes: string;
    lessons_learned?: string;
    preventive_measures?: string[];
  };
}

// Шаблон действия
interface ActionTemplate {
  id: string;
  name: string;
  description: string;
  action_type: string;
  applicable_violation_types: string[];
  applicable_severities: string[];
  template_steps: {
    order: number;
    step_description: string;
    estimated_duration_minutes: number;
    required_permissions: string[];
  }[];
  success_criteria: string[];
  escalation_triggers: {
    condition: string;
    escalation_action: string;
  }[];
}
```

***

### 6. Бизнес-процесс отчетности и аналитики

#### 6.1 Процесс генерации отчетов по нарушениям

Данный процесс реализует \[US-009] и соответствует \[UC-F-005] из архитектурного документа.

```mermaid
flowchart TD
    START([Запрос на генерацию отчета]) --> VALIDATE_REPORT_REQUEST[Проверка прав доступа к отчетам]
    VALIDATE_REPORT_REQUEST -->|Нет прав| ERROR_ACCESS_DENIED[Возврат ошибки доступа]
    VALIDATE_REPORT_REQUEST -->|Есть права| PARSE_REPORT_PARAMETERS[Парсинг параметров отчета]
    PARSE_REPORT_PARAMETERS --> VALIDATE_DATE_RANGE[Валидация временного диапазона]
    VALIDATE_DATE_RANGE -->|Невалидный| ERROR_INVALID_RANGE[Возврат ошибки диапазона]
    VALIDATE_DATE_RANGE -->|Валидный| ESTIMATE_REPORT_SIZE[Оценка размера отчета]
    ESTIMATE_REPORT_SIZE --> CHECK_SIZE_LIMITS{Превышены лимиты размера?}
    CHECK_SIZE_LIMITS -->|Да| SUGGEST_OPTIMIZATION[Предложение оптимизации параметров]
    CHECK_SIZE_LIMITS -->|Нет| BUILD_DATA_QUERY[Построение SQL запроса]
    BUILD_DATA_QUERY --> APPLY_FILTERS[Применение фильтров]
    APPLY_FILTERS --> EXECUTE_AGGREGATION_QUERY[Выполнение агрегационного запроса]
    EXECUTE_AGGREGATION_QUERY --> FETCH_DETAILED_DATA[Получение детализированных данных]
    FETCH_DETAILED_DATA --> ENRICH_WITH_METADATA[Обогащение метаданными]
    ENRICH_WITH_METADATA --> DETERMINE_OUTPUT_FORMAT{Определение формата вывода}
    DETERMINE_OUTPUT_FORMAT -->|CSV| GENERATE_CSV[Генерация CSV файла]
    DETERMINE_OUTPUT_FORMAT -->|Excel| GENERATE_EXCEL[Генерация Excel файла]
    DETERMINE_OUTPUT_FORMAT -->|PDF| GENERATE_PDF[Генерация PDF отчета]
    DETERMINE_OUTPUT_FORMAT -->|JSON| GENERATE_JSON[Генерация JSON ответа]
    GENERATE_CSV --> STORE_REPORT_FILE[Сохранение файла отчета]
    GENERATE_EXCEL --> STORE_REPORT_FILE
    GENERATE_PDF --> STORE_REPORT_FILE
    GENERATE_JSON --> RETURN_JSON_RESPONSE[Возврат JSON ответа]
    STORE_REPORT_FILE --> GENERATE_DOWNLOAD_LINK[Генерация ссылки для скачивания]
    GENERATE_DOWNLOAD_LINK --> SCHEDULE_FILE_CLEANUP[Планирование очистки файла]
    SCHEDULE_FILE_CLEANUP --> RETURN_DOWNLOAD_RESPONSE[Возврат ссылки на скачивание]
    ERROR_ACCESS_DENIED --> LOG_ACCESS_ATTEMPT[Запись попытки доступа]
    ERROR_INVALID_RANGE --> LOG_VALIDATION_ERROR_REPORT[Запись ошибки валидации]
    SUGGEST_OPTIMIZATION --> LOG_SIZE_WARNING[Запись предупреждения о размере]
    LOG_ACCESS_ATTEMPT --> END([Конец процесса])
    LOG_VALIDATION_ERROR_REPORT --> END
    LOG_SIZE_WARNING --> END
    RETURN_JSON_RESPONSE --> END
    RETURN_DOWNLOAD_RESPONSE --> END
```

#### 6.2 Структуры данных для отчетности

```typescript
// Параметры запроса отчета
interface ReportRequest {
  report_type: 'VIOLATIONS_SUMMARY' | 'DETAILED_VIOLATIONS' | 'SLA_PERFORMANCE' | 'ACTOR_PERFORMANCE';
  date_range: {
    from: string;                // ISO 8601
    to: string;                  // ISO 8601
  };
  filters: {
    cities?: string[];
    violation_types?: string[];
    severities?: string[];
    actor_types?: string[];
    actor_ids?: string[];
    delivery_unit_ids?: string[];
    status?: string[];
  };
  grouping?: {
    by_city: boolean;
    by_actor: boolean;
    by_day: boolean;
    by_hour: boolean;
  };
  output_format: 'CSV' | 'EXCEL' | 'PDF' | 'JSON';
  include_metadata: boolean;
  max_records?: number;
}

// Данные отчета по нарушениям
interface ViolationReportData {
  summary: {
    total_violations: number;
    violations_by_severity: Record<string, number>;
    violations_by_type: Record<string, number>;
    average_delay_minutes: number;
    most_affected_cities: {
      city: string;
      violation_count: number;
    }[];
    worst_performing_actors: {
      actor_id: string;
      actor_name: string;
      violation_count: number;
      average_delay: number;
    }[];
  };
  detailed_violations: {
    violation_id: string;
    delivery_unit_id: string;
    occurred_at: string;
    severity: string;
    delay_minutes: number;
    actor_name: string;
    city: string;
    status: string;
    resolution_time_minutes?: number;
  }[];
  trends: {
    daily_violation_counts: {
      date: string;
      count: number;
    }[];
    hourly_patterns: {
      hour: number;
      average_violations: number;
    }[];
  };
  metadata: {
    generated_at: string;
    generated_by: string;
    report_parameters: ReportRequest;
    data_freshness: string;      // Насколько свежие данные
  };
}
```

***

### 7. Процесс обработки ошибок и исключительных ситуаций

#### 7.1 Единый процесс обработки системных ошибок

Данный процесс обеспечивает надежность всех вышеперечисленных бизнес-процессов.

```mermaid
flowchart TD
    START([Возникла ошибка в системе]) --> CLASSIFY_ERROR_TYPE[Классификация типа ошибки]
    CLASSIFY_ERROR_TYPE --> IS_VALIDATION_ERROR{Ошибка валидации?}
    IS_VALIDATION_ERROR -->|Да| LOG_VALIDATION_ERROR_FLOW[Запись ошибки валидации]
    IS_VALIDATION_ERROR -->|Нет| IS_SYSTEM_ERROR{Системная ошибка?}
    IS_SYSTEM_ERROR -->|Да| CHECK_RETRY_ELIGIBLE{Возможен retry?}
    IS_SYSTEM_ERROR -->|Нет| IS_BUSINESS_ERROR{Бизнес-ошибка?}
    CHECK_RETRY_ELIGIBLE -->|Да| IMPLEMENT_RETRY_LOGIC[Реализация retry с экспоненциальной задержкой]
    CHECK_RETRY_ELIGIBLE -->|Нет| LOG_SYSTEM_ERROR[Запись системной ошибки]
    IMPLEMENT_RETRY_LOGIC --> CHECK_RETRY_COUNT{Превышен лимит попыток?}
    CHECK_RETRY_COUNT -->|Нет| RETRY_OPERATION[Повторная попытка операции]
    CHECK_RETRY_COUNT -->|Да| SEND_TO_DLQ[Отправка в Dead Letter Queue]
    IS_BUSINESS_ERROR -->|Да| LOG_BUSINESS_ERROR[Запись бизнес-ошибки]
    IS_BUSINESS_ERROR -->|Нет| LOG_UNKNOWN_ERROR[Запись неизвестной ошибки]
    LOG_VALIDATION_ERROR_FLOW --> UPDATE_ERROR_METRICS_FLOW[Обновление метрик ошибок]
    LOG_SYSTEM_ERROR --> CHECK_ALERT_THRESHOLD{Превышен порог алертов?}
    LOG_BUSINESS_ERROR --> UPDATE_ERROR_METRICS_FLOW
    LOG_UNKNOWN_ERROR --> TRIGGER_CRITICAL_ALERT[Запуск критического алерта]
    SEND_TO_DLQ --> UPDATE_DLQ_METRICS[Обновление метрик DLQ]
    CHECK_ALERT_THRESHOLD -->|Да| TRIGGER_SYSTEM_ALERT[Запуск системного алерта]
    CHECK_ALERT_THRESHOLD -->|Нет| UPDATE_ERROR_METRICS_FLOW
    TRIGGER_SYSTEM_ALERT --> UPDATE_ERROR_METRICS_FLOW
    TRIGGER_CRITICAL_ALERT --> UPDATE_ERROR_METRICS_FLOW
    UPDATE_DLQ_METRICS --> UPDATE_ERROR_METRICS_FLOW
    UPDATE_ERROR_METRICS_FLOW --> GENERATE_ERROR_RESPONSE[Генерация ответа об ошибке]
    GENERATE_ERROR_RESPONSE --> END([Конец процесса])
    RETRY_OPERATION --> END
```

#### 7.2 Структуры данных для обработки ошибок

```typescript
// Запись об ошибке
interface ErrorRecord {
  error_id: string;              // UUID ошибки
  error_type: 'VALIDATION' | 'SYSTEM' | 'BUSINESS' | 'INTEGRATION' | 'SECURITY';
  error_code: string;            // Код ошибки (E1001, E2002, etc.)
  severity: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  service_component: string;     // Компонент, где произошла ошибка
  operation_context: string;     // Контекст операции
  error_message: string;         // Сообщение об ошибке
  error_details: {
    stack_trace?: string;
    input_data?: object;
    system_state?: object;
    correlation_id?: string;
  };
  occurred_at: string;           // Время возникновения
  resolved_at?: string;          // Время решения
  retry_count: number;           // Количество попыток
  resolution_action?: string;    // Действие по решению
  affected_resources: string[];  // Затронутые ресурсы
}

// Конфигурация retry логики
interface RetryConfiguration {
  operation_type: string;
  max_retry_attempts: number;
  initial_delay_ms: number;
  max_delay_ms: number;
  backoff_multiplier: number;
  retry_on_errors: string[];     // Коды ошибок для retry
  circuit_breaker?: {
    failure_threshold: number;
    recovery_timeout_ms: number;
    half_open_max_calls: number;
  };
}
```

***

### 8. Процесс управления жизненным циклом сессий и контекста

#### 8.1 Процесс управления пользовательскими сессиями для UI контроллера

Данный процесс обеспечивает \[US-004], \[US-008], \[US-009], \[US-010] и соответствует \[UC-F-005].

```mermaid
flowchart TD
    START([Пользователь открывает рабочее место]) --> CHECK_AUTHENTICATION[Проверка аутентификации]
    CHECK_AUTHENTICATION -->|Не аутентифицирован| REDIRECT_TO_LOGIN[Перенаправление на страницу входа]
    CHECK_AUTHENTICATION -->|Аутентифицирован| VALIDATE_SESSION[Проверка валидности сессии]
    VALIDATE_SESSION -->|Сессия истекла| REFRESH_TOKEN_ATTEMPT[Попытка обновления токена]
    VALIDATE_SESSION -->|Сессия валидна| LOAD_USER_PREFERENCES[Загрузка пользовательских настроек]
    REFRESH_TOKEN_ATTEMPT -->|Неуспешно| REDIRECT_TO_LOGIN
    REFRESH_TOKEN_ATTEMPT -->|Успешно| LOAD_USER_PREFERENCES
    LOAD_USER_PREFERENCES --> INITIALIZE_WORKSPACE_CONTEXT[Инициализация контекста рабочего места]
    INITIALIZE_WORKSPACE_CONTEXT --> LOAD_ACTIVE_VIOLATIONS[Загрузка активных нарушений]
    LOAD_ACTIVE_VIOLATIONS --> LOAD_USER_ASSIGNMENTS[Загрузка назначений пользователя]
    LOAD_USER_ASSIGNMENTS --> SETUP_REAL_TIME_UPDATES[Настройка real-time обновлений]
    SETUP_REAL_TIME_UPDATES --> ESTABLISH_WEBSOCKET[Установка WebSocket соединения]
    ESTABLISH_WEBSOCKET --> START_HEARTBEAT[Запуск heartbeat]
    START_HEARTBEAT --> WORKSPACE_READY[Рабочее место готово]
    WORKSPACE_READY --> MONITOR_USER_ACTIVITY[Мониторинг активности пользователя]
    MONITOR_USER_ACTIVITY --> CHECK_INACTIVITY_TIMEOUT{Превышен таймаут неактивности?}
    CHECK_INACTIVITY_TIMEOUT -->|Нет| MONITOR_USER_ACTIVITY
    CHECK_INACTIVITY_TIMEOUT -->|Да| WARN_SESSION_EXPIRY[Предупреждение об истечении сессии]
    WARN_SESSION_EXPIRY --> USER_EXTENDS_SESSION{Пользователь продлевает сессию?}
    USER_EXTENDS_SESSION -->|Да| EXTEND_SESSION[Продление сессии]
    USER_EXTENDS_SESSION -->|Нет| CLEANUP_SESSION[Очистка сессии]
    EXTEND_SESSION --> MONITOR_USER_ACTIVITY
    CLEANUP_SESSION --> SAVE_USER_STATE[Сохранение состояния пользователя]
    SAVE_USER_STATE --> CLOSE_WEBSOCKET[Закрытие WebSocket]
    CLOSE_WEBSOCKET --> LOGOUT_USER[Выход пользователя]
    REDIRECT_TO_LOGIN --> END([Конец процесса])
    LOGOUT_USER --> END
```

#### 8.2 Структуры данных для управления сессиями

```typescript
// Контекст пользовательской сессии
interface UserSessionContext {
  session_id: string;            // UUID сессии
  user_id: string;               // ID пользователя
  user_role: string;             // Роль пользователя
  permissions: string[];         // Разрешения
  workspace_preferences: {
    default_city_filter?: string[];
    default_severity_filter?: string[];
    refresh_interval_seconds: number;
    notifications_enabled: boolean;
    dashboard_layout: object;
  };
  active_assignments: {
    violation_id: string;
    assigned_at: string;
    priority: string;
  }[];
  current_filters: {
    cities: string[];
    date_range: {
      from: string;
      to: string;
    };
    severities: string[];
    statuses: string[];
  };
  websocket_connection_id?: string;
  last_activity: string;
  session_expires_at: string;
  created_at: string;
}

// Состояние рабочего места
interface WorkspaceState {
  user_session: UserSessionContext;
  dashboard_data: {
    total_active_violations: number;
    my_assigned_violations: number;
    critical_violations: number;
    violations_trend: {
      timestamp: string;
      count: number;
    }[];
    top_affected_cities: {
      city: string;
      violation_count: number;
    }[];
  };
  real_time_updates: {
    last_update: string;
    pending_notifications: {
      type: string;
      message: string;
      timestamp: string;
    }[];
  };
  ui_state: {
    active_tab: string;
    selected_violation_id?: string;
    modal_states: Record<string, boolean>;
    table_sort_order: Record<string, 'asc' | 'desc'>;
    pagination_state: Record<string, {
      page: number;
      page_size: number;
    }>;
  };
}
```

***

### Cross-Document References и версионирование

**Связь с документами:**

* \[PRD-2.1] — соответствие функциональным требованиям
* \[US-001…US-015] — реализация пользовательских историй
* \[ARCH-2.1] — архитектурная поддержка бизнес-процессов

**Версионность:**

* Версия 1.0: Базовые процессы для MVP
* Версия 1.1: Добавление процессов эскалации и интеграции с внешними системами
* Версия 1.2: Расширение аналитики и отчетности

**Следующие документы в цепочке:**

* API Design Document \[API-2.x]
* Database Design Document \[DB-2.x]
* Development Plan \[PLAN-2.x]
