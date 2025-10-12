# **ADR: JWT TTL + Refresh + Отзыв токенов**
Status: Proposed

## **Context**
**Risk:** R-01 — Компрометация токена / захват аккаунта (L=4, I=5, Score=20)  
**DFD:** Edge: Internet → API (auth/login)  
**NFR:** NFR-202, NFR-305, NFR-101 (Security-AuthN, регистрация/аутентификация, RateLimiting)  
**Assumptions:** публичный API-gateway; стейтлесс сервисы; отсутствие корпоративного PKI

## **Decision**
Внедрить короткоживущие access-токены + refresh-поток с ротацией и серверным отзывом:
- Access token TTL = 15 минут (для пользовательских токенов)
- Refresh token TTL = 7 дней (ротация refresh-token при каждом обмене)
- Хранить список отозванных refresh-token или использовать версию токена в профиле пользователя; отзывать при logout/смене пароля
- Обязать HTTPS, secure/httponly cookie или Authorization header; применить лимиты аутентификации на gateway (per-IP, per-account)  
**Scope:** endpoints /api/auth/*, все write-эндпоинты требуют валидный access token.

## **Alternatives**
- O2: MFA для чувствительных действий — отложено для админов/важных аккаунтов (UX/cost).  
- O3: Привязка устройства + жёсткие лимиты — дополняет O1, внедрять позже.

## **Consequences**
+ Снижает окно атаки, позволяет отзывать сессии, интегрируется с rate limits  
- Требует реализации refresh-flow и хранения/ревокации; возможны изменения на клиенте

## **DoD / Acceptance (G-W-T)**
**Given** истёкший access token  
**When** POST /api/orders (защищённый write-эндпоинт)  
**Then** ответ = 401 (application/problem+json) и запрос логируется с correlation_id  
**Checks:**
- **test:** интеграционный тест, что обмен refresh даёт новый access с обновлённым exp и старый refresh ротаируется  
- **log:** событие отзыва появляется при logout/смене пароля  
- **metric:** доля запросов с истёкшими токенами подтверждает ожидаемую работу

## **Rollback / Fallback**
Feature flag для нового auth-flow; возможность отката конфигурации auth-сервера; мониторинг всплеска 401.

## **Trace**
- **DFD:** Edge Internet→API (auth)  
- **STRIDE:** S (аутентификация)  
- **Risk scoring:** R-01 (ранг 1)  
- **NFR:** NFR-202, NFR-305, NFR-101  
- **Issues:** TBD

**Owner:** team-taxi/auth lead  
**Date created:** YYYY-MM-DD

---

# **ADR: Маскирование PII + Политика хранения (логи и БД)**
Status: Proposed

## **Context**
**Risk:** R-02 — Утечка PII в логах/БД (L=3, I=5, Score=15)  
**DFD:** Node: Service / DB / Logs  
**NFR:** NFR-103, NFR-303, NFR-106 (Privacy/PII, ретеншн, валидация)  
**Assumptions:** централизованная лог-пайплайн; операционная политика бэкапов

## **Decision**
1) Маскировать PII в логах на уровне логгера (allowlist безопасных полей).  
2) Ввести политику хранения: raw PII TTL = 7 дней; реализовать purge-job для удаления старых данных и учесть бэкапы.  
**Scope:** все сервисные логи и поля PII в БД (телефон, e-mail, адрес).

## **Alternatives**
- O3: шифрование полей в БД — отложено (высокая сложность).  
- O2: только ретеншн — недостаточно без маскирования логов.

## **Consequences**
+ Снижает риск утечек через логи и бэкапы; помогает соответствовать требованиям  
- Меньше данных для отладки; требует тестов purge

## **DoD / Acceptance (G-W-T)**
**Given** событие с PII  
**When** событие логируется и данные сохраняются  
**Then** в логах маскированные поля PII и purge удаляет raw PII старше 7 дней  
**Checks:**
- **test:** интеграционный тест логирования проверяет маскирование полей  
- **log:** выборка по логам не показывает raw PII за последние 30 дней (кроме допустимых исключений)  
- **policy/scan:** запуск purge-job показывает отсутствие записей старше TTL  
- **metric:** отсутствие алертов от PII-сканера логов

## **Rollback / Fallback**
Feature flag для маскирования; откат путём переключения флага; мониторинг для обнаружения потери дебаг-данных.

## **Trace**
- **DFD:** Service → DB и Logs  
- **STRIDE:** I (логи/БД)  
- **Risk scoring:** R-02 (ранг 2)  
- **NFR:** NFR-103, NFR-303, NFR-106

**Owner:** team-taxi/data-privacy lead  
**Date created:** YYYY-MM-DD

---

# **ADR: Timeouts + Retry + Circuit Breaker for External Provider**
Status: Proposed

## **Context**
**Risk:** R-03 — Зависание внешнего провайдера → деградация (L=3, I=4, Score=12)  
**DFD:** Edge: Service → External API (provider)  
**NFR:** NFR-102, NFR-104 (Timeouts/Retry, Performance)  
**Assumptions:** у провайдера переменная латентность и периодические ошибки

## **Decision**
Реализовать устойчивость на стороне клиента:
- Таймаут outbound = 2s на запрос (HTTP клиент сервиса)  
- Повторные попытки ≤ 3 с экспоненциальным backoff + jitter  
- Circuit breaker: открывать при error-rate ≥ 50% за 1 минуту; half-open проба после cooldown  
- Экспорт метрик: external_call_latency, external_call_errors, cb_state  
**Scope:** запросы от сервиса к endpoint'ам провайдера (расчёт стоимости, список доступных такси).

## **Alternatives**
- O2: Асинхронный fallback (202 + очередь) — отложено для критичных non-blocking сценариев.  
- O3: Кэш с коротким TTL — дополняет, но не заменяет O1.

## **Consequences**
+ Предотвращает каскадные отказы, сохраняет отклик сервиса  
- Требует настройки порогов; retries могут добавить нагрузку

## **DoD / Acceptance (G-W-T)**
**Given** провайдер медленен/недоступен  
**When** сервис вызывает endpoint провайдера  
**Then** retries ≤3 и circuit-breaker переходит в open при достижении порога; клиент получает 503 или cached/fallback ответ в зависимости от сценария  
**Checks:**
- **test:** интеграционный тест с симуляцией падения провайдера проверяет retries и открытие CB  
- **metric:** CB открывается при всплеске external_call_errors; метрика latency отслеживается  
- **log:** события смены состояния CB и retries логируются с correlation_id

## **Rollback / Fallback**
Включение/выключение через feature flag; откат к предыдущим настройкам клиента при ложных срабатываниях.

## **Trace**
- **DFD:** Service → External Provider  
- **STRIDE:** D (внешние вызовы)  
- **Risk scoring:** R-03 (ранг 3)  
- **NFR:** NFR-102, NFR-104

**Owner:** team-taxi/backend lead  
**Date created:** YYYY-MM-DD

---

# ADR: RBAC + PII Masking для истории поездок
Status: Proposed

## Context
Risk: R-13 — Утечка истории поездок (PII) (L=3, I=4, Score=12)  
DFD: Node: Ride History Service  
NFR: NFR-401, NFR-402, NFR-403, NFR-404, NFR-405 (Security-AuthZ/RBAC, Privacy/PII, Performance, API-Contract/Errors, Data-Integrity)  
Assumptions: централизованная аутентификация; чувствительные данные водителей требуют защиты

## Decision
Внедрить строгую авторизацию и маскирование PII для эндпоинтов истории поездок:
- RBAC проверка: пользователь может видеть только свои поездки (WHERE user_id = :current_user_id)
- PII маскирование: телефон/email водителя → только имя и рейтинг; адреса → обезличенная форма
- Пагинация: limit ≤ 100, offset ≥ 0, сортировка по дате создания DESC
- Производительность: кэширование на 5 минут, индексы по (user_id, created_at)  
Scope: GET /api/rides/history, GET /api/rides/{id}/details

## Alternatives
- O2: Полное шифрование PII полей — избыточная сложность для данного контекста
- O3: Без маскирования — неприемлемый риск утечки PII

## Consequences
+ Соответствие GDPR, защита приватности пользователей
- Меньше деталей для отладки; дополнительные индексы в БД

## DoD / Acceptance (G-W-T)
Given два пользователя A и B с историями поездок  
When пользователь A запрашивает GET /api/rides/history  
Then ответ содержит только поездки A, PII водителя маскировано, пагинация соблюдает limit=100  
Checks:
- test: интеграционный тест проверяет изоляцию данных между пользователями
- log: аудит-лог содержит user_id при доступе к истории
- scan: статический анализ подтверждает WHERE user_id = :current_user_id
- metric: P95 времени ответа ≤ 300 мс при 50 RPS

## Rollback / Fallback
Feature flag для маскирования PII; откат конфигурации пагинации; мониторинг 403 ошибок

## Trace
- DFD: Node: Ride History Service
- STRIDE: I (утечка PII)
- Risk scoring: R-13 (ранг 4)
- NFR: NFR-401, NFR-402, NFR-403, NFR-404, NFR-405

Owner: team-taxi/history lead  
Date created: YYYY-MM-DD

---

# ADR: Rate Limiting + AuthZ для системы рейтингов
Status: Proposed

## Context
Risk: R-11 — Накрутка рейтингов (L=4, I=3, Score=12)  
DFD: Edge: Client → Rating Service  
NFR: NFR-501, NFR-502, NFR-503, NFR-504, NFR-505 (Data-Integrity, Security-AuthZ/RBAC, RateLimiting, API-Contract/Errors, Auditability)  
Assumptions: система рейтингов влияет на репутацию водителей; возможны попытки манипуляции

## Decision
Реализовать защиту от накрутки рейтингов и обеспечить целостность данных:
- Rate limiting: ≤ 10 оценок в час на аккаунт (на уровне API gateway)
- AuthZ проверка: только пассажир поездки может оставить отзыв
- Валидация: рейтинг 1-5, текст ≤ 1000 символов
- Идемпотентность: один отзыв на поездку, повторная оценка перезаписывает предыдущую  
Scope: POST /api/rides/{id}/rating, PUT /api/rides/{id}/feedback

## Alternatives
- O2: Верификация телефона для оценок — излишняя сложность для MVP
- O3: Модерация всех отзывов — высокая операционная нагрузка

## Consequences
+ Защита от спама и манипуляций; целостность рейтинговой системы
- Дополнительные проверки на критическом пути

## DoD / Acceptance (G-W-T)
Given пользователь не участвовал в поездке  
When POST /api/rides/{id}/rating  
Then ответ 403; при 11 оценке за час — 429; невалидные данные — 400  
Checks:
- test: e2e тест проверяет rate limiting и авторизацию участника поездки
- log: все операции с рейтингами логируются с user_id, ride_id, timestamp
- metric: отслеживание 429/403 ошибок на эндпоинтах оценок
- scan: проверка уникальности отзывов на пару (user_id, ride_id)

## Rollback / Fallback
Настройка rate limit через конфиг; возможность временного отключения лимитов; мониторинг всплеска оценок

## Trace
- DFD: Edge: Client → Rating Service
- STRIDE: T (накрутка рейтингов)
- Risk scoring: R-11 (ранг 6)
- NFR: NFR-501, NFR-502, NFR-503, NFR-504, NFR-505

Owner: team-taxi/ratings lead  
Date created: YYYY-MM-DD

---

# ADR: PCI DSS Compliance для платежных методов
Status: Proposed

## Context
Risk: R-08 — Утечка платежных данных (L=3, I=5, Score=15), R-09 — Неавторизованный доступ (L=4, I=4, Score=16)  
DFD: Edge: Client → Payment Service, Node: Payment Service  
NFR: NFR-601, NFR-602, NFR-603, NFR-604, NFR-605 (Security-Secrets, Privacy/PII, Security-AuthZ/RBAC, API-Contract/Errors, Auditability)  
Assumptions: интеграция с PCI DSS провайдером; строгие требования к защите платежных данных

## Decision
Обеспечить соответствие PCI DSS стандартам для управления платежными методами:
- Токенизация: данные карт хранятся только у провайдера, в БД — reference token
- PII маскирование: номера карт в логах как ****1234
- RBAC: только владелец аккаунта может управлять своими платежными методами
- Валидация: проверка номера карты, срока действия, CVC на стороне провайдера  
Scope: GET/POST/DELETE /api/payment-methods**

## Alternatives
- O2: Локальное шифрование карт — не соответствует PCI DSS без сертификации
- O3: Хранение масок в основной БД — недостаточная безопасность

## Consequences
+ Соответствие PCI DSS; снижение регуляторных рисков
- Зависимость от внешнего провайдера; сложность отладки

## DoD / Acceptance (G-W-T)
Given пользователь добавляет платежную карту  
When проверяется БД и логи  
Then в БД только reference token, в логах маскированные номера; пользователь B не может удалить методы пользователя A  
Checks:
- test: интеграционный тест проверяет токенизацию и RBAC изоляцию
- log: все операции с платежными методами аудируются; нет raw номеров карт
- scan: PCI DSS сканирование подтверждает отсутствие данных карт в системе
- policy: проверка что провайдер имеет PCI DSS сертификацию

## Rollback / Fallback
Feature flag для переключения на legacy платежную систему (если есть); мониторинг ошибок токенизации

## Trace
- DFD: Edge: Client → Payment Service, Node: Payment Service
- STRIDE: I (утечка данных), S/E (несанкционированный доступ)
- Risk scoring: R-08 (ранг 2), R-09 (ранг 1)
- NFR: NFR-601, NFR-602, NFR-603, NFR-604, NFR-605

Owner: team-taxi/payment lead  
Date created: YYYY-MM-DD
