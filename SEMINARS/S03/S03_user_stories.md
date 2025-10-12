# US-021 - Уведомление о приезде такси  
## Роль-Цель-Ценность:  
Как пассажир, я хочу получать уведомление о приезде такси, чтобы вовремя выйти к машине.  

## Кратко:
Отправка push/email/SMS уведомления, когда водитель подъехал к месту подачи.  

## API/Endpoints:  
POST /api/notifications/ride-arrival  

## NFR hooks:
RateLimiting, Timeouts/Retry, Privacy/PII, Performance, API-Contract/Errors, Security-InputValidation

---

# US-022 - Заказ такси  
## Роль-Цель-Ценность:  
Как пользователь, я хочу заказать такси через приложение, чтобы быстро добраться до нужного адреса.

## Кратко:
Оформление заказа такси с указанием точки отправления и назначения, расчёт стоимости, назначение водителя.

## API/Endpoints:  
POST /api/orders  
GET /api/orders/{id}/status  

## NFR hooks:
RateLimiting, Security-AuthN, Performance, API-Contract/Errors, Data-Integrity, Security-InputValidation

---

# US-023 - Регистрация пользователя  
## Роль-Цель-Ценность:  
Как новый пользователь, я хочу зарегистрировать учётную запись, чтобы пользоваться сервисом заказа такси.

## Кратко:
Регистрация с подтверждением e-mail или телефона, валидация данных, создание профиля.

## API/Endpoints:  
POST /api/auth/register  
POST /api/auth/verify-email  

## NFR hooks:
RateLimiting, Security-InputValidation, Privacy/PII, API-Contract/Errors, Security-AuthN, Data-Integrity

---

# US-024 - Просмотр истории поездок
Как пользователь, я хочу видеть историю своих поездок, чтобы отслеживать расходы и повторять маршруты.

## Кратко:
Список завершённых поездок с детализацией (дата, маршрут, стоимость, водитель).

## API/Endpoints:
GET /api/rides/history
GET /api/rides/{id}/details

## NFR hooks:
Security-AuthZ/RBAC, Privacy/PII, Performance, API-Contract/Errors, Data-Integrity

---

# US-025 - Оценка поездки и водителя
Как пассажир, я хочу оценить поездку и водителя, чтобы поддерживать качество сервиса.

## Кратко: 
Рейтинг (1-5 звёзд) и текстовый отзыв после завершения поездки.

## API/Endpoints:
POST /api/rides/{id}/rating
PUT /api/rides/{id}/feedback

## NFR hooks:
Data-Integrity, Security-AuthZ/RBAC, RateLimiting, API-Contract/Errors, Auditability

---

# US-026 - Управление способами оплаты
Как пользователь, я хочу управлять привязанными картами, чтобы быстро оплачивать поездки.

## Кратко:
Добавление, просмотр и удаление платёжных методов через PCI-совместимый провайдер.

## API/Endpoints:
GET /api/payment-methods
POST /api/payment-methods
DELETE /api/payment-methods/{id}

## NFR hooks:
Security-Secrets, Privacy/PII, Security-AuthZ/RBAC, API-Contract/Errors, Auditability
