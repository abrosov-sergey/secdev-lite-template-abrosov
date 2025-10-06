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