## Приоритизация рисков (L×I)


| Risk ID | Source (DFD/Row) | Consolidated Description | Threat (S/T/R/I/D/E) | NFR link (ID) | L (1-5) | Rationale-L | I (1-5) | Rationale-I | **Score (=L×I)** | Decision (Top-5?) | ADR candidate |
| ------- | ---------------- | ------------------------ | -------------------- | ------------- | ------: | ----------- | ------: | ----------- | ---------------: | ----------------- | ------------- |
| R-01 | Edge: Internet → API (JWT) | Компрометация токена / захват аккаунта (краденые/перехваченные JWT, brute-force) | S | NFR-202, NFR-305, NFR-101 | 4 | публичная поверхность, типовые векторы (phishing, token leak) | 5 | компрометация аккаунта → PII/неправильные заказы/платежи | 20 | 1 | JWT TTL+Refresh, MFA, rate-limit on auth endpoints |
| R-02 | Node: Service / DB / Logs | Утечка PII в логах или БД (логи с сырой PII, длительное хранение) | I | NFR-103, NFR-303, NFR-106 | 3 | возможен через ошибки/деплой/неправильное логирование | 5 | регуляторные и репутационные последствия, штрафы | 15 | 2 | PII masking, retention policy (≤7d), redact in logs |
| R-03 | Edge: Service → External API (provider) | Отказ/зависание внешнего провайдера → деградация сервиса / cascade failure | D | NFR-102, NFR-104 | 3 | внешний зависимый компонент, подвергается сбоям | 4 | потеря доступности части функционала (расчёт, доступность такси) | 12 | 3 | Timeouts≤2s, retry with jitter, circuit-breaker |
| R-04 | Edge: Internet → API (order input) | Манипуляция/подмена полей заказа (tampering) — изменение точки/цены/флагов | T | NFR-206, NFR-302, NFR-205 | 3 | публичные входы, возможна инъекция/изменение параметров | 4 | неверные заказы/финансовые ошибки, споры с водителем/клиентом | 12 | 4 | Strict input validation, server-side canonicalization, integrity checks |
| R-05 | Edge: Internet → API (mass requests) | Массовое создание заказов/уведомлений → ресурсная деградация / DoS | D | NFR-201, NFR-101, NFR-301 | 4 | автоматизированный абьюз (bots, scripts) возможен | 3 | ухудшение качества сервиса, задержки уведомлений/заказов | 12 | 5 | Global + per-user rate limits, backpressure, quotas |
| R-06 | Edge: API → Client (errors) | Утечка подробных ошибок / стэктрейсов в ответах → информационная утечка | I/R | NFR-105, NFR-304 | 2 | встречается при некорректной обработке исключений | 3 | раскрытие внутренней структуры, помогает атакующим | 6 | 7 | RFC7807 error responses, hide stack traces, correlation_id |
| R-07 | Node: DB (writes) | Нарушение целостности данных (SQL injection / неканоничные данные) | T/I | NFR-205, NFR-306 | 2 | уязвимости в валидации/ORM могут быть эксплуатированы | 4 | потеря целостности заказов/платежей, сложный откат | 8 | 6 | Parametrized queries, normalization, DB constraints |
| R-08 | Edge: Client → Service (payment) | Утечка платежных данных (карты) через логи или API | I | NFR-602, NFR-601 | 3 | Ошибки логирования/валидации возможны | 5 | Нарушение PCI DSS, финансовые потери, регуляторные штрафы | 15 | 2 | PII masking (карты), PCI DSS compliance | 
| R-09 | Node: Payment Service | Неавторизованный доступ к платежным методам | S/E | NFR-603 | 4 | Публичный API, высокий интерес злоумышленников | 4 | Финансовые потери, мошеннические операции | 16 | 1 | RBAC checks, MFA для платежных операций | 
| R-10 | Edge: API → Payment Provider | Отказ платежного провайдера | D | NFR-102 | 3 | Внешняя зависимость | 4 | Невозможность оплаты поездок | 12 | 5 | Circuit breaker, fallback payment methods | 
| R-11 | Node: Rating Service | Накрутка рейтингов (фальшивые отзывы) | T | NFR-503, NFR-502 | 4 | Простота создания фальшивых аккаунтов | 3 | Искажение репутации водителей | 12 | 6 | Rate limiting, phone verification | 
| R-12 | Edge: Client → Service (ratings) | Подмена данных оценки (манипуляция рейтингом) | T | NFR-504 | 3 | Возможна подмена параметров запроса | 3 | Искажение статистики водителей | 9 | 8 | Input validation, digital signatures | 
| R-13 | Node: Ride History Service | Утечка истории поездок (PII) | I | NFR-402 | 3 | Ошибки в ACL возможны | 4 | Нарушение приватности, GDPR штрафы | 12 | 4 | Strict RBAC, PII masking | 
| R-14 | Edge: DB → Backup | Утечка PII из бэкапов | I | NFR-602 | 2 | Низкая вероятность при правильных процессах | 5 | Критическое нарушение GDPR | 10 | 7 | Encryption at rest, access controls 

## Сводка Top-5

* Top-1: R-01 — Компрометация токена / захват аккаунта (S). L=4, I=5, Score=20. ADR candidate: "JWT TTL+Refresh + MFA + auth rate limits".
  - Почему: public auth endpoints + PII and potential financial impact → highest priority.

* Top-2: R-02 — Утечка PII в логах/БД (I). L=3, I=5, Score=15. ADR candidate: "PII Masking & Retention Policy (≤7d), log redaction".
  - Почему: высокие последствия по приватности и соответствию.

* Top-3: R-03 — Зависание внешнего провайдера (D). L=3, I=4, Score=12. ADR candidate: "Timeout/Retry/CB for external APIs".
  - Почему: внешняя зависимость влияет на доступность ключевых функций.

* Top-4: R-04 — Манипуляция данных заказа (T). L=3, I=4, Score=12. ADR candidate: "Strict input validation + server-side canonicalization".
  - Почему: приводит к финансовым/операционным инцидентам; тяготеет к мошенничеству.

* Top-5: R-05 — Массовые запросы / спам заказов (D). L=4, I=3, Score=12. ADR candidate: "Per-user quotas, global rate limits, backpressure".
  - Почему: высока вероятность автоматического абьюза, практический эффект на доступность.
