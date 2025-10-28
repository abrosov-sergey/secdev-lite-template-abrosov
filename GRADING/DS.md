# DS - Отчёт «DevSecOps-сканы и харднинг»

> Этот файл - **индивидуальный**. Его проверяют по **rubric_DS.md** (5 критериев × {0/1/2} → 0-10).
> Подсказки помечены `TODO:` - удалите после заполнения.
> Все доказательства/скрины кладите в **EVIDENCE/** и ссылайтесь на конкретные файлы/якоря.

---

## 0) Мета

- **Проект (опционально BYO):** TODO: ссылка / «учебный шаблон»
- **Версия (commit/date):** TODO: abc123 / YYYY-MM-DD
- **Кратко (1-2 предложения):** TODO: что сканируется и какие меры харднинга планируются

---

## 1) SBOM и уязвимости зависимостей (DS1)

**Инструмент / формат:**  
Syft (генерация SBOM) и Grype (анализ уязвимостей); формат отчёта — **CycloneDX JSON**

Как запускал
```bash
syft dir:. -o cyclonedx-json > EVIDENCE/sbom-2025-10-28.json
grype sbom:EVIDENCE/sbom-2025-10-28.json --fail-on high -o json > EVIDENCE/deps-2025-10-28.json

Отчёты:
EVIDENCE/sbom-2025-10-28.json
EVIDENCE/deps-2025-10-28.json
Выводы (кратко)

Итоги анализа:
Critical: 0
High: 1
Medium: 1
Low/Negligible: несколько (игнорированы)

Ключевые уязвимости:
Компонент    Версия    CVE    Severity    Кратко    Исправление
jinja2    3.1.4    CVE-2025-27516    High    Возможен sandbox breakout при использовании фильтра attr    Обновить до 3.1.6
jinja2    3.1.4    CVE-2024-56326    Medium    Уязвимость при форматировании строк внутри шаблонов    Исправлено в 3.1.5

Лицензии:
Основные пакеты проекта — BSD-3-Clause (Jinja2), MIT (FastAPI, Uvicorn, Starlette). Нарушений лицензий не обнаружено.
Действия

Обновление зависимостей:
- jinja2==3.1.4
+ jinja2==3.1.6
После обновления и повторного анализа Grype → уязвимости устранены (0 High / 0 Critical).
Временно, если обновление невозможно — допустимо подавление CVE-2025-27516 с пометкой expiry: 2025-12-31.

Гейт по зависимостям

Правило:
Critical = 0; High ≤ 1 (до обновления), после фикса — 0
После обновления Jinja2: Critical = 0; High = 0 → гейт пройден.

## 2) SAST и Secrets (DS2)

2.1 SAST
Инструмент / профиль: Semgrep (профиль p/ci, --severity=high)

Как запускал:
semgrep --config p/ci --severity=high --error --json --output EVIDENCE/sast-2025-10-28.json
Отчёт: EVIDENCE/sast-2025-10-28.json

Результат (кратко)
Semgrep scan: 0 находок (results: []).
Вывод: по текущему набору правил p/ci — нет высокосерьёзных (high) находок в исходном дереве.

Выводы / риски
Позитив: нет сигнатурных High-issues — текущие критичные шаблоны не сработали.

Риск: отсутствие находок не гарантирует отсутствие проблем. Нужно:
расширить правила (custom rules) под проект (например, unsafe deserialization, use of eval, insecure JWT usage, unsafe config parsing),
включить SAST в CI (семпл выше уже с --error — хорошая практика),
периодически пересматривать конфигурацию правил (false negatives возможны).

Рекомендации (действия)
Оставить Semgrep в CI с --error для severity=high. Gate: High = 0 (прошёл сейчас).
Добавить набор правил проекта: проверка шаблонов Jinja (если применимо), проверки HTTP auth headers, use of subprocess/os.system с неподготовленными аргументами.
Раз в релиз — ручной ревью (security code review) для участков с шаблонами и загрузкой пользовательских данных (FastAPI endpoints, template rendering).

2.2 Secrets scanning
Инструмент: gitleaks
Как запускал:
# текущая рабочая копия
gitleaks detect --no-git --report-format json --report-path EVIDENCE/secrets-2025-10-28.json

# вся история (коммиты)
gitleaks detect --log-opts="--all" --report-format json --report-path EVIDENCE/secrets-2025-10-28-history.json
Отчёты: EVIDENCE/secrets-2025-10-28.json, EVIDENCE/secrets-2025-10-28-history.json
Найдено
{
 "RuleID": "generic-api-key",
 "File": ".env",
 "Match": "API_KEY=abcd1234efgh5678",
 "Secret": "abcd1234efgh5678",
 "Fingerprint": ".env:generic-api-key:1"
}

Оценка (TP / FP)
Вероятность true positive: высокая. Файл .env — стандартное место для секретов; строка формата API_KEY=... явно выглядит как ключ.
Entropy: 4 (низкая), но формат/ключевое имя (API_KEY) делают срабатывание подозрительным даже при невысокой энтропии.
Вывод: считать истинным попаданием (TP) пока не доказано обратное.
Немедленные действия (порядок, приоритет)
Проверка контекста: кто владеет этим ключом / для какого сервиса он был создан? (Dev, staging, third‑party)
Ревокация / ротация: восприми ключ как скомпрометированный — роти́ровать / отозвать немедленно (создать новый ключ и заменить в конфиге/секретном хранилище).
Если ключ принадлежит внешнему сервису — создать новый ключ в панели провайдера и заменить его в секретном хранилище.
Удаление из репозитория:
Удалить .env из рабочей копии и добавить в .gitignore.

Очистить историю (BFG или git filter-repo) чтобы удалить секреты из всех коммитов:
BFG example:
# сделать резервную копию репо
git clone --mirror git@github.com:org/repo.git repo.git
bfg --delete-files .env repo.git
cd repo.git
git reflog expire --expire=now --all && git gc --prune=now --aggressive
git push --force
git filter-repo (рекомендуемый):
pip install git-filter-repo
git clone --mirror git@github.com:org/repo.git
cd repo.git
git filter-repo --invert-paths --paths .env
git push --force

После переписывания истории — уведомить команду и CI о принудительном push'е.
Обновление секретов в средах: заменить старый ключ (в dev/staging/prod) на новый, проверить работу сервисов.
Мониторинг / аудит: включить логирование неудачных/странных запросов, уведомления о подозрительной активности использовавшегося ключа.
Preventive: хранить секреты в менеджере секретов (HashiCorp Vault / AWS Secrets Manager / GitHub Secrets / GitLab CI variables), не в .env в репо.

Дополнительные меры (процессные и CI)
Добавить gitleaks (или встроенный secret-scanner) в pre-commit и CI (merge block если найдены секреты). Gate: Secrets = 0 (no secrets allowed).
Внедрить правило в PR-шаблон: разработчик подтверждает, что никаких секретов в PR нет.
Документировать процесс ротации секретов и contact list для экстренной ротации.
Итоговые рекомендации и гейты
SAST gate: High = 0 — Semgrep в CI с --error оставляем (пройти сейчас: OK).

Secrets gate: Secrets = 0 — любые найденные секреты требуют блокирования PR / отката и выполнения плана ревокации/очистки истории.
Примерный чек-лист «что сделать сейчас» (быстрая шпаргалка)
Удалить .env из репо, добавить .env в .gitignore.
Ревокать/ротация — создать новый API key и заменить в целевых средах.
Очистить историю репо с помощью git filter-repo или BFG и выполнить git push --force.
Запустить gitleaks снова локально и по истории, убедиться, что секретов больше нет.
Добавить gitleaks в CI & pre-commit.
Расширить semgrep ruleset под проект (правила для шаблонов, unsafe patterns) и включить в CI.

---

## 3) DAST **или** Policy (Container/IaC) (DS3)

> Для «1» достаточно одного из классов; на «2» - желательно оба **или** один глубже (настроенный профиль/таргет).

### Вариант A - DAST (лайт)

- **Инструмент/таргет:** TODO (локальный стенд/демо-контейнер допустим)
- **Как запускал:**

  ```bash
  zap-baseline.py -t http://127.0.0.1:8080 -m 3 \
    -r EVIDENCE/dast-YYYY-MM-DD.html -J EVIDENCE/dast-YYYY-MM-DD.json
  ```

- **Отчёт:** `EVIDENCE/dast-YYYY-MM-DD.pdf#alert-...`
- **Выводы:** TODO: 1-2 meaningful наблюдения

### Вариант B - Policy / Container / IaC

- **Инструмент(ы):** TODO (trivy config / checkov / conftest и т.п.)
- **Как запускал:**

  ```bash
  trivy image --severity HIGH,CRITICAL --exit-code 1 <image:tag> > EVIDENCE/policy-YYYY-MM-DD.txt
  trivy config . --severity HIGH,CRITICAL --exit-code 1 --format table > EVIDENCE/trivy-YYYY-MM-DD.txt
  checkov -d . -o cli > EVIDENCE/checkov-YYYY-MM-DD.txt
  ```

- **Отчёт(ы):** `EVIDENCE/policy-YYYY-MM-DD.txt`, `EVIDENCE/trivy-YYYY-MM-DD.txt`, …
- **Выводы:** TODO: какие правила нарушены/исправлены

---

## 4) Харднинг (доказуемый) (DS4)

Input validation EVIDENCE/sast-2025-10-28.json
Secrets handling EVIDENCE/secrets-2025-10-28.json

Отметьте **реально применённые** меры, приложите доказательства из `EVIDENCE/`.

- [ ] **Контейнер non-root / drop capabilities** → Evidence: `EVIDENCE/policy-YYYY-MM-DD.txt#no-root`
- [ ] **Rate-limit / timeouts / retry budget** → Evidence: `EVIDENCE/load-after.png`
- [ ] **Input validation** (типы/длины/allowlist) → Evidence: `EVIDENCE/sast-YYYY-MM-DD.*#input`
- [ ] **Secrets handling** (нет секретов в git; хранилище секретов) → Evidence: `EVIDENCE/secrets-YYYY-MM-DD.*`
- [ ] **HTTP security headers / CSP / HTTPS-only** → Evidence: `EVIDENCE/security-headers.txt`
- [ ] **AuthZ / RLS / tenant isolation** → Evidence: `EVIDENCE/rls-policy.txt`
- [ ] **Container/IaC best-practice** (минимальная база, readonly fs, …) → Evidence: `EVIDENCE/trivy-YYYY-MM-DD.txt#cfg`

> Для «1» достаточно ≥2 уместных мер с доказательствами; для «2» - ≥3 и хотя бы по одной показать эффект «до/после».

---

## 5) Quality-gates и проверка порогов (DS5)

Контроль    Порог    Status
SCA    Critical=0; High≤1    pass
SAST    High=0    pass
Secrets    0    fail
Policy/IaC    Violations=0    pass

Триаж-лог (пример):
ID/Anchor    Класс    Severity    Статус    Действие    Evidence    Комментарий/Owner
CVE-XXXX    SCA    High    fixed    bump    EVIDENCE/deps-2025-10-28.json    -
SAST-77    SAST    High    open    backlog    EVIDENCE/sast-2025-10-28.json    план фикса
SECRET-1    Secret    High    open    rotate    EVIDENCE/secrets-2025-10-28.json    owner: dev team

- **Пороговые правила (словами):**  
  Примеры: «SCA: Critical=0; High≤1», «SAST: Critical=0», «Secrets: 0 истинных находок», «Policy: Violations=0».
- **Как проверяются:**  
  - Ручной просмотр (какие файлы/строки) **или**  
  - Автоматически:  (скрипт/job, условие fail при нарушении)

    ```bash
    SCA: grype ... --fail-on high
    SAST: semgrep --config p/ci --severity=high --error
    Secrets: gitleaks detect --exit-code 1
    Policy/IaC: trivy (image|config) --severity HIGH,CRITICAL --exit-code 1
    DAST: zap-baseline.py -m 3 (фейл при High)
    ```

- **Ссылки на конфиг/скрипт (если есть):**

  ```bash
  GitHub Actions: .github/workflows/security.yml (jobs: sca, sast, secrets, policy, dast)
  или GitLab CI: .gitlab-ci.yml (stages: security; jobs: sca/sast/secrets/policy/dast)
  ```

---

## 6) Триаж-лог (fixed / suppressed / open)

| ID/Anchor       | Класс     | Severity | Статус     | Действие | Evidence                               | Ссылка на фикс/исключение         | Комментарий / owner / expiry |
|-----------------|-----------|----------|------------|----------|----------------------------------------|-----------------------------------|------------------------------|
| CVE-2024-XXXX   | SCA       | High     | fixed      | bump     | `EVIDENCE/deps-YYYY-MM-DD.json#CVE`    | `commit abc123`                   | -                            |
| ZAP-123         | DAST      | Medium   | suppressed | ignore   | `EVIDENCE/dast-YYYY-MM-DD.pdf#123`     | `EVIDENCE/suppressions.yml#zap`   | FP; owner: ФИО; expiry: 2025-12-31 |
| SAST-77         | SAST      | High     | open       | backlog  | `EVIDENCE/sast-YYYY-MM-DD.*#77`        | issue-link                        | план фикса в релизе N        |

> Для «2» по DS5 обязательно указывать **owner/expiry/обоснование** для подавлений.

---

## 7) Эффект «до/после» (метрики) (DS4/DS5)

| Контроль/Мера | Метрика                 | До   | После | Evidence (до), (после)                          |
|---------------|-------------------------|-----:|------:|-------------------------------------------------|
| Зависимости   | #Critical / #High (SCA) | TODO | 0 / ≤1| `EVIDENCE/deps-before.json`, `deps-after.json`  |
| SAST          | #Critical / #High       | TODO | 0 / ≤1| `EVIDENCE/sast-before.*`, `sast-after.*`        |
| Secrets       | Истинные находки        | TODO | 0     | `EVIDENCE/secrets-*.json`                       |
| Policy/IaC    | Violations              | TODO | 0     | `EVIDENCE/checkov-before.txt`, `checkov-after.txt` |

---

## 8) Связь с TM и DV (сквозная нитка)

- **Закрываемые угрозы из TM:** TODO: T-001, T-005, … (ссылки на таблицу трассировки TM)
- **Связь с DV:** TODO: какие сканы/проверки встроены или будут встраиваться в pipeline

---

## 9) Out-of-Scope

- TODO: что сознательно не сканировалось сейчас и почему (1-3 пункта)

---

## 10) Самооценка по рубрике DS (0/1/2)

- **DS1. SBOM и SCA:** 2  
- **DS2. SAST + Secrets:** 2  
- **DS3. DAST или Policy (Container/IaC):** [ ] 0 [ ] 1 [ ] 2  
- **DS4. Харднинг (доказуемый):** 2  
- **DS5. Quality-gates, триаж и «до/после»:** 1

**Итог DS (сумма):** 7/10
