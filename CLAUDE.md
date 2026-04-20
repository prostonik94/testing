# CLAUDE.md — Руководство по проекту

## Обзор проекта

Автоматический pipeline на **n8n** для сбора, обработки и публикации AI/tech новостей из Hacker News с генерацией контента через LLM.

**Расписание:** `0 9,18 * * *` — дважды в день (09:00 и 18:00 МСК)

## Архитектура workflow

```
Schedule Trigger / Manual Trigger
  → Code: генерация 8 поисковых запросов (AI, LLM, GPT, agent, automation, API, startup, open source)
  → HTTP Request: Hacker News Algolia API (points > 10, 50 результатов на запрос)
  → Aggregate: сбор всех ответов в единый массив
  → Code: дедупликация по objectID + топ-5 по points
  → Basic LLM Chain (GPT-4.1-mini): генерация анонса EN + AR (150–200 символов)
  → Code: парсинг JSON из LLM + сборка финального объекта
  → Google Sheets: appendOrUpdate по полю title
  → Telegram: отправка с inline-кнопками
```

## Файлы проекта

| Файл | Назначение |
|------|-----------|
| `тестовое Прохоров Н (2).json` | Экспорт workflow из n8n (credentials заменены на заглушки `ЗАГЛУШКА`) |
| `README.md` | Документация проекта |

## Стек технологий

| Компонент | Решение |
|-----------|---------|
| Автоматизация | n8n (self-hosted) |
| Источник данных | Hacker News Algolia API (бесплатный, без авторизации) |
| LLM | OpenAI GPT-4.1-mini (maxTokens: 300, temperature: 0.7) |
| Хранение | Google Sheets (appendOrUpdate по title) |
| Уведомления | Telegram Bot API (chatId: 195622777) |

## Настройка и деплой

### 1. Импорт workflow
```bash
# Импортируй workflow.json в n8n через UI: Settings → Import from file
```

### 2. Credentials (заменить ЗАГЛУШКА на реальные)
- **OpenAI API** → нода `OpenAI Chat Model`
- **Google Sheets OAuth2** → нода `Append or update row in sheet`
- **Telegram API** → нода `Send a text message`

### 3. Конфигурация нод
- **Google Sheets Document ID:** `1SffwHPqGtNOlnlvygxThUFZORdakivZWQDoaw7qI2o8`
- **Telegram Chat ID:** `195622777`
- **Активировать:** Schedule Trigger после настройки credentials

## Ключевые ноды и логика

### Code in JavaScript2 — генерация запросов
Создаёт 8 поисковых запросов к HN API. При добавлении новых тем — добавлять сюда.

### Code in JavaScript1 — фильтрация
Дедупликация по `objectID`, сортировка по `points DESC`, отбор топ-5. Бросает ошибку если статей нет.

### Basic LLM Chain — промпт
Генерирует строго JSON: `{"text_en": "...", "text_ar": "..."}`. Промпт запрещает любой другой вывод.

### Code in JavaScript — парсинг LLM
Обходит обрыв `pairedItem` через `$('Code in JavaScript1').all()[$itemIndex]`. Нормализует все поля в строки/числа.

## Известные ограничения

- **LLM latency:** при 5 статьях суммарное время 30–60 сек; увеличение batch size может вызвать таймаут
- **pairedItem разрыв:** LangChain-ноды обрывают цепочку — данные берутся через `$('NodeName').all()[$itemIndex]`
- **Дедупликация между запусками:** Google Sheets сравнивает только по `title` — при изменении заголовка возможен дубль
- **Пустой source:** посты типа `Tell HN:` не имеют внешней ссылки — кнопка «Читать статью» нерабочая

## Правила разработки

- При изменении промпта LLM — всегда проверять что выход остаётся валидным JSON
- Новые поля в Google Sheets добавлять в schema ноды `Append or update row in sheet`
- Не трогать `batching.delayBetweenBatches: 1000` — защита от rate limit OpenAI
- Credentials никогда не коммитить; использовать заглушку `ЗАГЛУШКА` в экспортах

## Приоритеты при доработке

1. Добавить фильтр `created_at > now - 24h` для исключения старых постов
2. Вынести LLM-генерацию в отдельный workflow с очередью
3. Добавить Telegram-алерт при падении любого шага
4. Векторная дедупликация через embeddings + pgvector
