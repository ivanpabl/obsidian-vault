# Проект - rag

## Что это

`documents/rag` (bgl-detect): сервис находит аномалии в логах суперкомпьютера BlueGene/L (Loghub BGL). Детекторы отбирают подозрительные окна, а RAG-детектор находит похожие ранее размеченные шаблоны сообщений (FAISS) и спрашивает LLM (Ollama, qwen2.5:3b) — настоящий ли это сбой. Метрики, логи и трейсы отдаются в Prometheus, Loki, Tempo; дашборды в Grafana.

## Как работает

1. Сырой лог BGL.log → шаблоны через Drain3 (296 типов сообщений) → parquet.
2. Детекторы: кандидат — шаблон в 5-минутном окне (всплеск частоты выше среднего + 3σ за сутки, или уровень ERROR/FATAL/SEVERE/FAILURE).
3. RAG-детектор: эмбеддинг шаблона → 5 похожих шаблонов из базы знаний (сходство ≥ 0,7) → промпт → вердикт LLM + категория + объяснение. Ответы кэшируются в SQLite.
4. Сервис: Flask/gunicorn за HTTP (`POST /detect`, `GET /kb/search`, `/metrics`).
5. Оценка: precision/recall/F1 в сравнении с базовой линией TF-IDF без LLM.

## Текущее состояние

- Проверено несколько гипотез организации пайплайна (предыдущие варианты — LangChain, LangGraph-агент, zero-shot, kNN — в теге `v1-experiment`), оставлен самый эффективный вариант: RAG-детектор с поиском похожих шаблонов.
- Итоги в `docs/results.md`: F1 по шаблонам 0,73 у RAG против 0,67 у TF-IDF; без поиска похожих случаев LLM находит лишь 44% аномалий.
- Развитие: анализ в реальном времени и алертинг.

## Структура (важные пути)

- `bgl_detect/config.py` — пути, параметры детекторов, адрес Ollama и модель
- `bgl_detect/data/` — prepare.py, parse.py, templates.py (Drain3)
- `bgl_detect/detection/` — detectors.py, context.py (контекст окна, база знаний)
- `bgl_detect/rag/` — detector.py, prompt.py, llm_cache.py (кэш + очередь к Ollama)
- `bgl_detect/service/` — app.py (Flask), telemetry.py, gunicorn.conf.py
- `bgl_detect/evaluation/` — evaluate.py (CLI), holdout.py, baseline.py (TF-IDF)
- `monitoring/` — docker compose: Prometheus, Grafana, Loki, Promtail, Tempo
- `docs/results.md`, `docs/lab.md` — итоги экспериментов и отчёт по observability

## Когда пригодится

- Новая сессия по проекту rag — дать ассистенту как контекст вместо пересказа.
- Запуск сервиса или оценки качества — см. [[Команда - запуск rag]].
- Возврат к архитектурным выборам — см. [[Решение - rag: RAG-детектор и holdout]].

## Связанные заметки

- [[Команда - запуск rag]]
- [[Решение - rag: RAG-детектор и holdout]]
- [[Источник - материалы проекта rag]]
