# LoadStar - Workflow и Бизнес-логика
**Дата:** 12.02.2026  
**Версия:** 2.0 (после консолидации схемы)

---

## Итоговая структура БД
- **Всего таблиц:** 17
- **Объединенные таблицы:** 
  - `client` (лиды + клиенты, разделение через status)
  - `request` (запросы + заказы, разделение через status)
- **Удаленные таблицы:**
  - `leads` (объединена с client)
  - `orders` (объединена с request)
  - `request_change_requests` (заменено на изменение статуса)
  - `request_invoice_client_items`, `request_invoice_counterparty_items` (удалены, счета упрощены)

---

## Статусы запросов/заказов (request.status)

### Список статусов
1. **New** — новый запрос создан
2. **Negotiating** — согласование маршрута с логистом
3. **Approved** — одобрен, стал заказом, выполняется
4. **Completed** — завершен
5. **Rejected** — отклонен

### Переходы между статусами

```
New ──────────────────────────→ Approved
 │                                  │
 │                                  │
 ├──→ Negotiating ──→ Approved ────┤
 │                                  │
 │                                  ↓
 └──────────────────────────→ Rejected
                                   ↑
         Approved ←─────────→ Negotiating (для изменения маршрута)
                                   │
                                   ↓
                              Completed
```

### Права на изменение статусов

| Переход | Кто может выполнить | Примечание |
|---------|---------------------|------------|
| New → Approved | Sales/Ops сами | Пропуск согласования, если маршрут готов |
| New → Negotiating | Sales/Ops | Отправка на согласование с логистом |
| Negotiating → Approved | Инициатор сам | После согласования маршрута |
| Approved → Negotiating | **Только Supervisor** | Для изменения маршрута выполняющегося заказа |
| Approved → Completed | **Только Supervisor** | Завершение заказа |
| Любой → Rejected | Sales/Ops/Supervisor | Отклонение запроса/заказа |

---

## Статусы клиентов (client.status)

### Список статусов
1. **Lead** — потенциальный клиент (лид)
2. **Active** — активный клиент
3. **Inactive** — неактивный клиент (бывший)

### Переходы
- **Lead → Active**: Автоматически при переходе первого запроса в статус **Approved** (заполняется `client.converted_at`)
- **Active → Inactive**: Вручную (клиент перестал с нами работать)
- **Inactive клиенты**: Не показываются в списках при создании нового запроса

### Метрика "не стали клиентами"
Лиды без успешных запросов: `client.status='Lead' AND NOT EXISTS (SELECT 1 FROM request WHERE client_id=client.id AND status IN ('Approved','Completed'))`

---

## Обязательные поля

### request.client_id
- **NOT NULL** — запрос всегда привязан к клиенту или лиду
- Один клиент/лид может иметь **множество** запросов одновременно (связь 1:N)

### request.is_customs_by_us
- **NOT NULL** — обязательное поле
- `true` = таможня нашими силами
- `false` = силами импортера

---

## Маршрут и согласование

### Заполнение route_stages
- **Может заполняться уже в статусе New** — Sales/Ops могут скопировать готовый маршрут или создать с нуля
- Логист не всегда нужен, если маршрут стандартный

### Согласование маршрута
- **Статус Negotiating** = идет согласование через `route_negotiations` (чат)
- **Маршрут считается согласованным** при переходе в статус **Approved**
- **Удалено поле** `route_negotiation_status` (boolean) — согласование = статус

### Изменение маршрута выполняющегося заказа
1. Supervisor устанавливает `request.status = 'Negotiating'`
2. Sales/Ops редактируют `route_stages`  
3. После согласования переводят обратно в `Approved`
- **Удалено поле** `is_route_final` — не нужно
- **Удалена таблица** `request_change_requests` — заменено изменением статуса

---

## Роли и основные права

### Sales (Менеджер по продажам)
- Работает с **лидами** (client.status='Lead')
- Создает запросы для лидов
- Может пропустить согласование (New → Approved) если маршрут готов
- Может отправить на согласование (New → Negotiating)
- После согласования сам переводит в Approved
- Не может завершать заказы (Approved → Completed)

### Ops (Оператор)
- Работает с **клиентами** (client.status='Active')
- Создает запросы для клиентов
- Те же права на статусы, что и Sales
- В статусе Approved вводит фактические даты этапов (actual_*_date в route_stages)
- Не может завершать заказы (Approved → Completed)

### Logist (Логист)
- Участвует в согласовании через `route_negotiations` (чат)
- Помогает построить оптимальный маршрут
- НЕ меняет статусы запросов

### Supervisor (Супервизор)
- **Только он** может:
  - Завершить заказ (Approved → Completed)
  - Вернуть заказ на доработку (Approved → Negotiating)
- Полный доступ ко всем запросам/заказам (не только своим)

### Admin (Администратор)
- Управление пользователями и справочниками
- НЕ работает с запросами/заказами

---

## Финансы и документы (только для status >= Approved)

### Таблицы, заполняемые только для заказов:
- `request_contracts` — договоры
- `request_documents` — документы
- `request_invoices_to_client` + items — счета клиентам
- `request_invoices_to_counterparty` + items — счета контрагентам
- `request_financial_records` — фактические доходы/расходы

---

## Ключевые удаленные элементы

### Удаленные поля
1. **request.request_type** — определяется через `JOIN` (user.role + client.status)
2. **request.is_active** — дубликат status
3. **request.order_completion_date** — дубликат completed_at
4. **request.route_negotiation_status** — заменено на проверку статуса
5. **request.is_route_final** — заменено на статус Negotiating
6. **client.category** (A/B/C/D) — не было в спецификации
7. **client.rejected_at, rejection_reason** — лиды не "отклоняются", просто не конвертируются

### Удаленные статусы
1. **Draft** из request — запрос создается сразу в New, данные всегда полные
2. **In_Progress** из request — объединено с Approved (в Approved уже идет выполнение)

### Удаленные таблицы
1. **leads** — объединена с clients (разделение через status)
2. **orders** — объединена с requests (разделение через status)
3. **request_change_requests** — изменение маршрута через статус Negotiating

---

## Workflow: Типичный сценарий с лидом

1. **Sales создает лида**: INSERT INTO clients (name, status='Lead', ...)
2. **Sales создает запрос**: INSERT INTO requests (client_id=лид, status='New', ...)
3. **Sales заполняет route_stages** (копирует или создает)
4. **Варианты развития:**
   - **A. Быстрый путь**: Sales сразу переводит New → Approved (если уверен в маршруте)
   - **B. Через согласование**: Sales переводит New → Negotiating → пишет в route_negotiations → Logist отвечает → Sales переводит в Approved
5. **При первом переходе в Approved**: Триггер обновляет `client.status='Lead' → 'Active'` + заполняет `converted_at`
6. **Ops вводит фактические даты**: UPDATE route_stages SET actual_departure_date=..., actual_arrival_date=...
7. **Supervisor завершает**: Approved → Completed

---

## Workflow: Типичный сценарий с клиентом

1. **Ops выбирает клиента**: clients.status='Active'
2. **Ops создает запрос**: INSERT INTO requests (client_id=клиент, status='New', ...)
3. **Ops заполняет route_stages**
4. **Ops переводит**: New → Approved (чаще без согласования, типовые маршруты)
5. **Ops ведет заказ**: вводит даты, загружает документы, вводит финансы
6. **Supervisor завершает**: Approved → Completed

---

## Workflow: Изменение маршрута выполняющегося заказа

1. **Заказ в статусе Approved**, груз в пути
2. **Нужно изменить маршрут** (форс-мажор, смена условий)
3. **Supervisor** возвращает: Approved → Negotiating
4. **Ops редактирует** route_stages (удаляет/добавляет/изменяет этапы)
5. **Ops/Sales** согласовывают через route_negotiations (при необходимости)
6. **Supervisor** одобряет: Negotiating → Approved

---

## Индексы и производительность

### Критичные индексы
- `requests(client_id)` — частые JOIN с clients
- `requests(status)` — фильтрация по статусу
- `requests(initiator_id)` — фильтр "мои запросы"
- `clients(status)` — фильтр Lead/Active/Inactive
- `route_stages(request_id, stage_order)` — UNIQUE, сортировка этапов

---

## Валидация на уровне приложения

### При создании запроса
- `client_id` NOT NULL (обязательно выбрать лида/клиента)
- `is_customs_by_us` NOT NULL (обязательно указать)
- `initiator_id` = текущий пользователь

### При переводе New → Approved
- route_stages должна содержать хотя бы 1 этап
- request_transport_units должна содержать хотя бы 1 тип ТС

### При переводе Approved → Completed
- Только Supervisor
- Все этапы должны иметь actual_*_date (опционально)

### При смене статуса Lead → Active
- Автоматически при первом Approved
- Заполнить converted_at = NOW()
- Проверить наличие обязательных полей клиента (bank_name, inn и т.д.)

---

## Конец документа
Этот документ является single source of truth для бизнес-логики системы LoadStar после консолидации схемы БД.
