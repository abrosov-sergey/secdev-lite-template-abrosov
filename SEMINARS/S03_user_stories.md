# US-021 - Уведомление о приезде такси  
## Роль-Цель-Ценность:  
Как пассажир, я хочу получать уведомление о приезде такси, чтобы вовремя выйти к машине.  

## Кратко:
Отправка push/email/SMS уведомления, когда водитель подъехал к месту подачи.  

## API/Endpoints:  
POST /api/notifications/ride-arrival  

## NFR hooks:
RateLimiting, Timeouts/Retry, Privacy/PII, Performance, API-Contract/Errors, Security-InputValidation