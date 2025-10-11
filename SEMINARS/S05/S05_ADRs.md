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