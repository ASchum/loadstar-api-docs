# LoadStar API - Коды ошибок и обработка

## 📐 Формат ответа с ошибкой

Все ошибки возвращаются в едином формате:

```json
{
  "error": "string",      // Тип ошибки (для программной обработки)
  "message": "string",    // Описание для пользователя
  "details": object|null  // Дополнительная информация (опционально)
}
```

**Content-Type:** `application/json`

---

## 🔴 HTTP статус-коды

### 400 Bad Request
**Когда:** Неправильный формат запроса (невалидный JSON, отсутствуют обязательные поля)

```json
{
  "error": "Bad Request",
  "message": "Неверный формат JSON",
  "details": {
    "line": 5,
    "column": 12
  }
}
```

---

### 401 Unauthorized
**Когда:** Отсутствует или невалидный токен

**Варианты:**
```json
// Токен отсутствует
{
  "error": "Token Missing",
  "message": "Требуется заголовок Authorization",
  "details": null
}

// Токен невалидный
{
  "error": "Token Invalid",
  "message": "Не удалось проверить подпись токена",
  "details": {
    "reason": "signature_invalid"
  }
}

// Токен истёк
{
  "error": "Token Expired",
  "message": "Срок действия токена истёк",
  "details": {
    "expired_at": "2026-02-13T10:00:00Z",
    "now": "2026-02-13T18:00:00Z"
  }
}
```

**Фронтенд действие:**
- `Token Missing` → редирект на /login
- `Token Invalid` → удалить токен, редирект на /login
- `Token Expired` → попытаться обновить через refresh token

---

### 403 Forbidden
**Когда:** Токен валидный, но недостаточно прав

```json
{
  "error": "Forbidden",
  "message": "У вас нет прав для выполнения этой операции",
  "details": {
    "required_role": "Admin",
    "your_role": "Sales"
  }
}
```

**Фронтенд действие:**
- Показать сообщение пользователю
- Скрыть недоступные кнопки/разделы

---

### 404 Not Found
**Когда:** Ресурс не найден

```json
// Объект не найден
{
  "error": "Not Found",
  "message": "Клиент с ID 123 не найден",
  "details": {
    "resource": "client",
    "id": 123
  }
}

// Endpoint не существует
{
  "error": "Route Not Found",
  "message": "Endpoint /api/v2/clients не существует",
  "details": {
    "path": "/api/v2/clients",
    "method": "GET"
  }
}
```

**Фронтенд действие:**
- Показать 404 страницу или сообщение
- Предложить вернуться к списку

---

### 422 Unprocessable Entity
**Когда:** Валидация не прошла

```json
{
  "error": "Validation Error",
  "message": "Переданные данные не прошли валидацию",
  "errors": {
    "email": [
      "Email обязателен для заполнения",
      "Email должен быть корректным адресом"
    ],
    "phone": [
      "Телефон должен быть в формате +7XXXXXXXXXX"
    ],
    "inn": [
      "ИНН должен содержать 10 или 12 цифр"
    ]
  }
}
```

**Особенность:** поле `errors` вместо `details`!

**Фронтенд действие:**
- Подсветить поля с ошибками красным
- Показать текст ошибки под каждым полем
- Не закрывать модальное окно

---

### 500 Internal Server Error
**Когда:** Баг на сервере (деление на ноль, SQL ошибка и т.д.)

```json
// Production (скрываем детали)
{
  "error": "Internal Server Error",
  "message": "Произошла внутренняя ошибка сервера",
  "details": {
    "trace_id": "abc123-def456-789"
  }
}

// Development (показываем стек)
{
  "error": "Internal Server Error",
  "message": "Ошибка БД: таблица или представление не найдено",
  "details": {
    "file": "/var/www/src/controllers/ClientController.php",
    "line": 45,
    "sql_error": "SQLSTATE[42S02]: Base table or view not found",
    "trace": "..."
  }
}
```

**Фронтенд действие:**
- Показать общее сообщение: "Что-то пошло не так"
- Логировать в console
- Предложить повторить попытку

---

## 🔄 Специфичные бизнес-ошибки

### Попытка удалить активный заказ
```json
// DELETE /api/orders/123
// Status: 422

{
  "error": "Cannot Delete Order",
  "message": "Невозможно удалить заказ со статусом 'Approved'",
  "details": {
    "order_id": 123,
    "current_status": "Approved",
    "allowed_statuses": ["New", "Rejected"]
  }
}
```

### Недостаточно средств (гипотетически)
```json
// POST /api/orders/123/financial-records
// Status: 422

{
  "error": "Insufficient Funds",
  "message": "Недостаточно средств на балансе",
  "details": {
    "available": 10000.00,
    "required": 15000.00,
    "currency": "RUB"
  }
}
```

### Конфликт версий (оптимистическая блокировка)
**HTTP Status:** 409 Conflict  
**Когда:** Данные были изменены другим пользователем с момента их загрузки

```json
// PATCH /api/clients/123
// Status: 409 Conflict

{
  "error": "Conflict",
  "message": "Данные были изменены другим пользователем. Загрузите актуальную версию",
  "details": {
    "your_updated_at": "2026-02-13T10:00:00Z",
    "current_updated_at": "2026-02-13T14:30:00Z"
  }
}
```

**Как работает:**
1. Frontend загружает данные: `GET /api/clients/123` → получает `updated_at: "2026-02-13T10:00:00Z"`
2. Пользователь редактирует данные в форме
3. Frontend отправляет: `PATCH /api/clients/123` с `updated_at: "2026-02-13T10:00:00Z"` в теле
4. Backend проверяет: если `updated_at` в БД отличается → возвращает 409 Conflict
5. Frontend показывает диалог с выбором действия

**Фронтенд действие:**
- Показать модальное окно/диалог с сообщением о конфликте
- Показать версии (ваша и текущая) для информации
- Единственная кнопка: **[Обновить страницу]**
- При нажатии: `window.location.reload()` - страница обновляется
- Пользователь видит актуальные данные с изменениями коллеги
- Вносит свои изменения поверх актуальных данных
- Сохраняет с новым `updated_at`

**Применяется к:**
- `PATCH /api/clients/{id}` — обновление клиента
- `PATCH /api/contragents/{id}` — обновление контрагента
- `PATCH /api/requests/{id}` — обновление запроса
- `PATCH /api/users/{id}` — обновление пользователя

---

## 💻 Обработка на фронтенде

### TypeScript типы

```typescript
// types/api.ts
export interface ApiError {
  error: string;
  message: string;
  details?: Record<string, any> | null;
}

export interface ValidationError {
  error: 'Validation Error';
  message: string;
  errors: Record<string, string[]>; // поле → массив ошибок
}

export type ApiErrorResponse = ApiError | ValidationError;
```

### Универсальный обработчик

```typescript
// utils/errorHandler.ts
import { toast } from 'react-toastify'; // или ваша библиотека

export function handleApiError(error: any) {
  // Нет ответа (сеть упала)
  if (!error.response) {
    toast.error('Нет соединения с сервером');
    return;
  }

  const status = error.response.status;
  const data: ApiErrorResponse = error.response.data;

  switch (status) {
    case 401:
      // Token проблемы
      if (data.error === 'Token Expired') {
        // Попытка обновить токен
        return refreshToken();
      }
      // Иначе на логин
      logout();
      window.location.href = '/login';
      break;

    case 403:
      toast.error(data.message || 'У вас нет прав для этой операции');
      break;

    case 404:
      toast.error(data.message || 'Ресурс не найден');
      break;

    case 409:
      // Конфликт версий - данные изменены другим пользователем
      // Показать диалог и предложить обновить страницу
      showConflictDialog({
        yourVersion: data.details?.your_updated_at,
        currentVersion: data.details?.current_updated_at,
        onReload: () => window.location.reload()
      });
      break;

    case 422:
      // Валидация - обрабатываем в компонентах формы
      if ('errors' in data) {
        return data.errors; // Вернём для подсветки полей
      }
      toast.error(data.message);
      break;

    case 500:
      toast.error('Произошла ошибка сервера. Попробуйте позже.');
      console.error('Server error:', data.details);
      break;

    default:
      toast.error(data.message || 'Неизвестная ошибка');
  }
}
```

### В компоненте формы

```typescript
// components/ClientForm.tsx
import { useState } from 'react';
import { handleApiError } from '@/utils/errorHandler';

function ClientForm({ clientId }) {
  const [formData, setFormData] = useState({});
  const [errors, setErrors] = useState<Record<string, string[]>>({});
  const [showConflictDialog, setShowConflictDialog] = useState(false);
  const [conflictDetails, setConflictDetails] = useState(null);

  const saveClient = async (withForce = false) => {
    setErrors({});
    
    try {
      const payload = {
        ...formData,
        updated_at: formData.updated_at,
        handleSubmit = async (e) => {
    e.preventDefault();
    setErrors({});
    
    try {
      await api.put(`/clients/${clientId}`, {
        ...formData,
        updated_at: formData.updated_at
      });
      toast.success('Клиент обновлен');
    } catch (error) {
      if (error.response?.status === 409) {
        // Конфликт версий - показать диалог
        setConflictDetails(error.response.data.details);
        setShowConflictDialog(true);
      } else if (error.response?.status === 422) {
        // Валидация
        setErrors(error.response.data.errors);
      } else {
        handleApiError(error);
      }
    }
  };

  return (
    <>
      <form onSubmit={handleSubmit}>
        <input name="name" value={formData.name} onChange={...} />
        {errors.name && <div className="error">{errors.name[0]}</div>}
        
        <input name="city" value={formData.city} onChange={...} />
        {errors.city && <div className="error">{errors.city[0]}</div>}
        
        <button type="submit">Сохранить</button>
      </form>
      
      {/* Диалог конфликта */}
      {showConflictDialog && (
        <ConflictDialog
          yourVersion={conflictDetails?.your_updated_at}
          currentVersion={conflictDetails?.current_updated_at}
          onReload={() => window.location.reload()}
        />
      )}
    </>
  );
}

// Компонент диалога конфликта
function ConflictDialog({ yourVersion, currentVersion, onReload }) {
  return (
    <div className="modal">
      <div className="modal-content">
        <h3>⚠️ Данные были изменены</h3>
        <p>
          Другой пользователь изменил эту запись с момента 
          открытия формы.
        </p>
        <div className="versions">
          <div>Ваша версия: {formatDate(yourVersion)}</div>
          <div>Текущая версия: {formatDate(currentVersion)}</div>
        </div>
        <div className="actions">
          <button onClick={onReload} className="primary">
            Обновить страницу
          </button>
        </div>
        <small>
          После обновления вы увидите актуальные данные 
          и сможете внести свои изменения.
---

## 🎨 UI паттерны для ошибок

### 1. Toast уведомления (общие ошибки)
```typescript
// 401, 403, 404, 500 → toast
toast.error(message);
```

### 2. Модальное окно (конфликт версий)
```typescript
// 409 → показать диалог с кнопкой "Обновить страницу"
<ConflictDialog 
  onReload={reloadPage} 
/>
```

### 3. Inline ошибки (валидация форм)
```typescript
// 422 → красная подсветка поля + текст под полем
<input className={errors.email ? 'error' : ''} />
{errors.email && <span className="error-text">{errors.email[0]}</span>}
```

### 4. Модальное окно (критичные бизнес-ошибки)
```typescript
// Удаление заблокировано, специфичные бизнес-ошибки
<Modal>
  <h2>{error.error}</h2>
  <p>{error.message}</p>
  <pre>{JSON.stringify(error.details, null, 2)}</pre>
</Modal>
```

### 5. Empty state (404)
```typescript
// Список пуст или объект не найден
<EmptyState 
  icon={<SearchIcon />}
  title="Клиент не найден"
  description="Клиент с ID 123 не существует"
  action={<Button onClick={goBack}>Вернуться к списку</Button>}
/>
```

---

## ✅ Чеклист для фронтендера

При интеграции каждого endpoint:

- [ ] Обработан 401 → редирект на логин
- [ ] Обработан 403 → показано сообщение о правах
- [ ] Обработан 404 → показан empty state
- [ ] Обработан 422 → подсвечены поля формы
- [ ] Обработан 500 → показано общее сообщение
- [ ] Все ошибки залогированы в console.error()
- [ ] Показан loading state до получения ответа
- [ ] Кнопка заблокирована во время запроса

---

## 📚 Дополнительно

### Retry стратегия

```typescript
// axios interceptor с retry для 500 ошибок
axios.interceptors.response.use(
  response => response,
  async error => {
    const config = error.config;
    
    // Retry до 3 раз для 500/502/503
    if ([500, 502, 503].includes(error.response?.status)) {
      config._retry = config._retry || 0;
      
      if (config._retry < 3) {
        config._retry++;
        await new Promise(resolve => setTimeout(resolve, 1000 * config._retry));
        return axios(config);
      }
    }
    
    return Promise.reject(error);
  }
);
```

---

## 🎯 Итог

**Для бэкенда (PHP):**
- Всегда возвращай JSON с полями `error`, `message`, `details`
- Используй правильные HTTP статус-коды
- 422 для валидации → поле `errors` вместо `details`

**Для фронтенда:**
- Создай универсальный `handleApiError`
- TypeScript типы для всех ошибок
- Разные UI паттерны для разных типов ошибок
- Логирование в console

**Документация:**
- Этот файл (`error-codes.md`) выкладывай вместе с OpenAPI
- Обновляй при добавлении новых типов ошибок
