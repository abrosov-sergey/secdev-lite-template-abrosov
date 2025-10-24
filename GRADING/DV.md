# DV - Мини-проект «DevOps-конвейер»

> Все доказательства/скрины кладите в **EVIDENCE/** и ссылайтесь на конкретные файлы/якоря.

---

## 0) Мета

- **Проект (опционально BYO):** «учебный шаблон»
- **Версия (commit/date):** 2025-10-24
- **Кратко (1-2 предложения):** «DevOps-конвейер»

---

## 1) Воспроизводимость локальной сборки и тестов (DV1)

- **Одна команда для сборки/тестов:**

  ```bash
  make ci-s06
  ```

- **Версии инструментов (фиксация):**

  ```bash
  python --version
  pip freeze > EVIDENCE/pip-freeze.txt
  ```

- **Описание шагов (кратко):**

```bash
python -m venv .venv
. .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python scripts/init_db.py
uvicorn app.main:app --reload
```


- **Выполненные тесты:**

S06-02 - SQL Injection (search LIKE):


**GWT**
Gherkin
```Gherkin
Сценарий: Инъекция в LIKE не возвращает все записи
Given:   Предзаполненная таблица items
When:    GET /search?q=' OR '1'='1'
Then:    Количество результатов не больше, чем для "шумового" запроса ("zzzzzzzzz")
Artifacts: EVIDENCE/S06/test-report.xml
Связано с: DV3, DV4

Сценарий: Слишком длинный запрос возвращает 400
Given:   Сервис поиска развёрнут
When:    GET /search?q=<очень длинная строка>
Then:    Код 400
Artifacts: EVIDENCE/S06/test-report.xml
Связано с: DV3, DV4

Сценарий: Позитивный поиск возвращает 200
Given:   В таблице items есть запись с name, содержащим "apple"
When:    GET /search?q=apple
Then:    Код 200
Artifacts: EVIDENCE/S06/test-report.xml
Связано с: DV3
```

**pytest**
```python
def test_search_should_not_return_all_on_injection():
    resp_noise = client.get("/search", params={"q": "zzzzzzzzz"}).json()
    inj = client.get("/search", params={"q": "' OR '1'='1"}).json()
    assert len(inj["items"]) <= len(
        resp_noise["items"]
    ), "Инъекция в LIKE не должна приводить к выдаче всех элементов"


def test_search_long_query():
    resp_long = client.get(
        "/search",
        params={
            "q": "zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz"
        },
    )
    assert resp_long.status_code == 400, "Слишком длинный запрос"


def test_search_query():
    # позитивный тест
    resp_long = client.get(
        "/search",
        params={
            "q": "apple"
        },
    )
    assert resp_long.status_code == 200
```

- **Выполненные тесты:** S06-05 - Ошибки и логи без утечек

**GWT**
Gherkin
```Gherkin
Сценарий: Внутренняя ошибка не раскрывает детали
Given:   Обработчик ошибок включён
When:    Инициируем исключение (спровоцировать контролируемо)
Then:    Код 500 и "безопасное" тело без стек-трейса/секретов
Artifacts: EVIDENCE/S06/logs/app.log (если есть), test-report.xml
Связано с: DV3, DV4, DV5
```

**pytest**
```python
def test_logging_hide_password():
    payload = {"username": "usr", "password": "pwd"}
    resp = client.post("/login_w_error", json=payload)
    print(resp.status_code)
    assert resp.status_code == 500

    with open('app.log', 'r') as f:
        file = f.read()
        assert "pwd" not in file
```


Логи, Скриншоты с подтверждениями, версии пакетов:

https://github.com/CepbluKot/secdev-lite-template/tree/main/EVIDENCE/S06

---

## 2) Контейнеризация (DV2)

- **Dockerfile:** TODO: `Dockerfile` (база, multi-stage? минимальный образ?)
- **Сборка/запуск локально:**

  ```bash
  docker build -t secdev-seed:latest .
  docker run --rm -p 8000:8000 --name seed secdev-seed:latest

  docker inspect seed | jq '.[0].State.Health' > EVIDENCE/S07/health.json

  ``

- **(Опционально) docker-compose:** `docker-compose.yml` - содержит 1 сервис "web", который запускае контейнер с нашим учебным приложением.

Сделал следующие hardenings:

1) S07-10 - Лимиты ресурсов контейнера (лайт)

в docker-compose
```yaml
# сделал hardening S07-10 - Лимиты ресурсов контейнера (лайт)
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
```

в docker

```
docker run --memory=512m --cpus="1.0" --rm -p 8000:8000 secdev-seed:latest
# memory и cpu для hardening S07-10 - Лимиты ресурсов контейнера (лайт)
```

2) S07-3 - Non-root user (запуск не от root)


в docker
```yaml
# hardening  S07-3 - Non-root user (запуск не от root)
RUN useradd -m -u 10001 appuser && chown -R appuser:appuser /app
USER appuser
```


Логи с подтверждениями:

https://github.com/CepbluKot/secdev-lite-template/tree/main/EVIDENCE/S07

---

## 3) CI: базовый pipeline и стабильный прогон (DV3)

- **Платформа CI:** TODO: GitHub Actions / GitLab CI / другое
- **Файл конфига CI:** TODO: путь (напр., `.github/workflows/ci.yml`)
- **Стадии (минимум):** checkout → deps → **build** → **test** → (package)
- **Фрагмент конфигурации (ключевые шаги):**

  ```yaml
  # TODO: укоротите под себя
  jobs:
  build_test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - name: Cache deps
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: pip-${{ hashFiles('**/requirements*.txt') }}
      - run: pip install -r requirements.txt
      - run: pytest -q

  ```

- **Стабильность:** TODO: последние N запусков зелёные? краткий комментарий
- **Ссылка/копия лога прогона:** `EVIDENCE/ci-YYYY-MM-DD-build.txt`

---

## 4) Артефакты и логи конвейера (DV4)

_Сложите файлы в `/EVIDENCE/` и подпишите их назначение._

| Артефакт/лог                    | Путь в `EVIDENCE/`            | Комментарий                                  |
|---------------------------------|-------------------------------|----------------------------------------------|
| Лог успешной сборки/тестов (CI) | `ci-YYYY-MM-DD-build.txt`     | ключевые шаги/время                          |
| Локальный лог сборки (опц.)     | `local-build-YYYY-MM-DD.txt`  | для сверки                                   |
| Описание результата сборки      | `package-notes.txt`           | какой образ/wheel/архив получился            |
| Freeze/версии инструментов      | `pip-freeze.txt` (или аналог) | воспроизводимость окружения                  |

---

## 5) Секреты и переменные окружения (DV5 - гигиена, без сканеров)

- **Шаблон окружения:** добавлен файл `/.env.example` со списком переменных (без значений), например:
  - `REG_USER=`
  - `REG_PASS=`
  - `API_TOKEN=`
- **Хранение и передача в CI:**  
  - секреты лежат в настройках репозитория/организации (**masked**),  
  - в pipeline они **не печатаются** в явном виде.
- **Пример использования секрета в job (адаптируйте):**

  ```yaml
  - name: Login to registry (masked)
    env:
      REG_USER: ${{ secrets.REG_USER }}
      REG_PASS: ${{ secrets.REG_PASS }}
    run: |
      echo "::add-mask::$REG_PASS"
      echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin registry.example.com
  ```

- **Быстрая проверка отсутствия секретов в коде (любой простой способ):**

  ```bash
  # пример: поиск популярных паттернов
  git grep -nE 'AKIA|SECRET|token=|password=' || true
  ```

  _Сохраните вывод в `EVIDENCE/grep-secrets.txt`._
- **Памятка по ротации:** TODO: кто/как меняет secrets при утечке/ревоке токена.

---

## 6) Индекс артефактов DV

_Чтобы преподаватель быстро сверил файлы._

| Тип     | Файл в `EVIDENCE/`            | Дата/время         | Коммит/версия | Runner/OS    |
|---------|--------------------------------|--------------------|---------------|--------------|
| CI-лог  | `ci-YYYY-MM-DD-build.txt`      | `YYYY-MM-DD hh:mm` | `abc123`      | `gha-ubuntu` |
| Лок.лог | `local-build-YYYY-MM-DD.txt`   | …                  | `abc123`      | `local`      |
| Package | `package-notes.txt`            | …                  | `abc123`      | -            |
| Freeze  | `pip-freeze.txt` (или аналог)  | …                  | `abc123`      | -            |
| Grep    | `grep-secrets.txt`             | …                  | `abc123`      | -            |

---

## 7) Связь с TM и DS (hook)

- **TM:** этот конвейер обслуживает риски процесса сборки/поставки (например, культура работы с секретами, воспроизводимость).  
- **DS:** сканы/гейты/триаж будут оформлены в `DS.md` с артефактами в `EVIDENCE/`.

---

## 8) Самооценка по рубрике DV (0/1/2)

- **DV1. Воспроизводимость локальной сборки и тестов:** [ ] 0 [ ] 1 [ ] 2  
- **DV2. Контейнеризация (Docker/Compose):** [ ] 0 [ ] 1 [ ] 2  
- **DV3. CI: базовый pipeline и стабильный прогон:** [ ] 0 [ ] 1 [ ] 2  
- **DV4. Артефакты и логи конвейера:** [ ] 0 [ ] 1 [ ] 2  
- **DV5. Секреты и конфигурация окружения (гигиена):** [ ] 0 [ ] 1 [ ] 2  

**Итог DV (сумма):** __/10
