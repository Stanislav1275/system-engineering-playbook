# Интерфейсы ICD-lite

<iframe src="../swagger/swagger.html" width="100%" height="1500px" style="border:0;" allowfullscreen="allowfullscreen"></iframe>

---

## 5.1. GET /services

| Параметр | Описание |
|---|---|
| Назначение | Получение списка доступных услуг |
| Метод и путь | `GET /api/v1/services` |
| Query-параметры | `departmentId`, `activeOnly` |
| Ответ 200 | Массив услуг: `serviceId`, `name`, `durationMinutes`, `description`, `isActive` |
| Ошибки | `400 INVALID_QUERY`, `500 INTERNAL_ERROR` |
| Идемпотентность | Не требуется, операция чтения |
| Версия | `/api/v1` |

## 5.2. GET /slots

| Параметр | Описание |
|---|---|
| Назначение | Получение доступных слотов для услуги, подразделения и даты |
| Метод и путь | `GET /api/v1/slots` |
| Query-параметры | `serviceId`, `departmentId`, `date` |
| Ответ 200 | Массив слотов: `slotId`, `windowId`, `startAt`, `endAt`, `status` |
| Ошибки | `400 INVALID_DATE`, `404 SERVICE_NOT_FOUND`, `409 SCHEDULE_NOT_CONFIGURED` |
| Ретраи | Клиент может повторить запрос при `503` |
| Идемпотентность | Не требуется, операция чтения |
| Версия | `/api/v1` |

## 5.3. POST /appointments

| Параметр | Описание |
|---|---|
| Назначение | Создание записи на приём |
| Метод и путь | `POST /api/v1/appointments` |
| Заголовки | `Idempotency-Key`, `Content-Type: application/json` |
| Авторизация | Не требуется для заявителей; токен сессии передаётся через cookie или query-параметр при наличии личного кабинета |
| Тело запроса | `applicant`, `serviceId`, `departmentId`, `windowId`, `startAt`, `notificationChannel` |
| Ответ 201 | `appointmentId`, `status`, `serviceId`, `departmentId`, `startAt`, `endAt` |
| Ошибки | `400 INVALID_REQUEST`, `409 SLOT_ALREADY_BOOKED`, `422 SLOT_NOT_AVAILABLE`, `503 SERVICE_UNAVAILABLE` |
| Ретраи | Допустимы только с тем же `Idempotency-Key` |
| Идемпотентность | Обязательна |
| Версия | `/api/v1` |

## 5.4. POST /appointments/{appointmentId}/cancellations

| Параметр | Описание |
|---|---|
| Назначение | Отмена записи заявителем |
| Метод и путь | `POST /api/v1/appointments/{appointmentId}/cancellations` |
| Тело запроса | `reason`, `requestedBy` |
| Ответ 200 | `appointmentId`, `status: cancelled`, `cancelledAt` |
| Ошибки | `404 APPOINTMENT_NOT_FOUND`, `409 CANCELLATION_NOT_ALLOWED`, `403 FORBIDDEN` |
| Ретраи | Повторная отмена должна возвращать безопасный результат, если запись уже отменена |
| Идемпотентность | Желательна |
| Версия | `/api/v1` |

## 5.5. PATCH /registrar/appointments/{appointmentId}/status

| Параметр | Описание |
|---|---|
| Назначение | Изменение статуса обслуживания регистратором |
| Метод и путь | `PATCH /api/v1/registrar/appointments/{appointmentId}/status` |
| Авторизация | Bearer JWT сотрудника, роль `registrar` |
| Тело запроса | `status`, `comment` |
| Ответ 200 | `appointmentId`, `oldStatus`, `newStatus`, `changedAt` |
| Ошибки | `401 UNAUTHORIZED`, `403 FORBIDDEN`, `404 APPOINTMENT_NOT_FOUND`, `409 INVALID_STATUS_TRANSITION` |
| Ретраи | Нежелательны без проверки текущего статуса |
| Идемпотентность | Частичная: повтор того же статуса не должен создавать некорректную историю |
| Версия | `/api/v1` |

## 5.6. POST /queue/tickets

| Параметр | Описание |
|---|---|
| Назначение | Создание талона электронной очереди через терминал |
| Метод и путь | `POST /api/v1/queue/tickets` |
| Тело запроса | `departmentId`, `serviceId`, `appointmentCode`, `source: terminal` |
| Ответ 201 | `ticketId`, `ticketNumber`, `status`, `createdAt` |
| Ошибки | `400 INVALID_REQUEST`, `404 APPOINTMENT_NOT_FOUND`, `409 TOO_EARLY_OR_TOO_LATE`, `503 QUEUE_UNAVAILABLE` |
| Ретраи | Допустимы с терминальным requestId |
| Идемпотентность | Желательна по `terminalRequestId` |
| Версия | `/api/v1` |

## 5.7. Событие AppointmentCreated

| Поле | Тип | Описание |
|---|---|---|
| eventId | UUID | Уникальный идентификатор события |
| eventType | string | `AppointmentCreated` |
| occurredAt | datetime | Время возникновения события |
| appointmentId | UUID | Идентификатор записи |
| applicantId | UUID | Идентификатор заявителя |
| serviceId | UUID | Идентификатор услуги |
| departmentId | UUID | Идентификатор подразделения |
| startAt | datetime | Время начала приёма |
| endAt | datetime | Время окончания приёма |
| notificationChannel | string | Канал уведомления |
| schemaVersion | string | Версия схемы события |

**Ошибки обработки:** событие не удаляется из очереди до успешной обработки; при повторных ошибках переносится в dead-letter queue.
