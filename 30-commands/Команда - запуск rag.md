# Команда - запуск rag

## Когда использовать

Запуск мониторинга и сервиса bgl-detect, проверка качества, анализ кода.

## Команды

Окружение (создаётся один раз):

```bash
uv venv --python 3.12 .venv && uv pip install --python .venv/bin/python -r pyproject.toml --extra-index-url https://download.pytorch.org/whl/cpu
```

Данные (один раз из сырого BGL.log, validation кладётся вне проекта в ~/rag-holdout):

```bash
.venv/bin/python -m bgl_detect.data.prepare /путь/к/BGL.log ~/rag-holdout
```

Мониторинг + сервис:

```bash
mkdir -p logs && (cd monitoring && docker compose up -d)
.venv/bin/gunicorn -c bgl_detect/service/gunicorn.conf.py bgl_detect.service.app:app
```

Остановка:

```bash
kill $(cat logs/rag-service.pid)
cd monitoring && docker compose down
```

Оценка качества (каждый прогон дописывает строку в results/experiments.csv):

```bash
.venv/bin/python -m bgl_detect.evaluation.evaluate rag --holdout template --note "что проверяли"
.venv/bin/python -m bgl_detect.evaluation.evaluate tfidf --holdout template
```

Долгий прогон с лимитом памяти (без подкачки):

```bash
systemd-run --user --scope -q -p MemoryMax=4G -p MemorySwapMax=0 .venv/bin/python -m bgl_detect.evaluation.evaluate rag
```

SonarQube (требует ~2 ГБ памяти; первый раз: cp .env.example .env, вход admin/admin):

```bash
docker compose -f docker-compose.sonarqube.yml up -d
scripts/sonar_scan.sh
```

## Успешный результат

- Сервис отвечает на http://localhost:5000 (`/health`, `POST /detect {"candidate_id": N}`).
- Grafana: http://localhost:3000 («RAG-сервис — обзор»), Prometheus :9090, Loki :3100, Tempo :3200.
- Главная метрика оценки — `f1_holdout` в режиме `template`.

## Частые проблемы

- Ollama на CPU: один ответ LLM — 5–40 с; повтор берётся из кэша. Адрес/модель — `OLLAMA_URL`, `RAG_MODEL` (по умолчанию qwen2.5:3b).
- DuckDB ограничен `DUCKDB_MEMORY_LIMIT` (2 ГБ), подкачки нет — долгие прогоны через systemd-run с MemoryMax.
- SonarQube останавливать на время работы с LLM (`docker compose -f docker-compose.sonarqube.yml stop`).

## Связанные заметки

- [[Проект - rag]]
