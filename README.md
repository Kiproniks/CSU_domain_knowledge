# Intelligent Library (веб-интерфейс только на Python)

Сейчас проект полностью ориентирован на компоненты, работающие на Python:

* Django — backend, API и серверный веб-интерфейс;
* Celery — фоновые задачи;
* PostgreSQL — база данных;
* Redis — брокер сообщений и хранилище результатов;
* aiogram — Telegram-бот;
* Ollama — опционально для работы с LLM.

React/Vite больше не требуется для запуска проекта.

## Что умеет система

1. Загружать книги в форматах FB2 и PDF.
2. Проверять XML и размер файла (максимум 50 МБ).
3. Вычислять SHA-256 хеш книги и использовать общий кэш для уже загруженных книг.
4. Разбивать текст на логические блоки.
5. Группировать блоки по основным темам.
6. Выделять от 2 до 4 подтем для каждой темы.
7. Сохранять упоминания понятий и связывать их с исходными блоками текста.
8. Формировать полное содержание и краткое описание книги.
9. Искать понятия и сравнивать их между книгами пользователя.
10. Экспортировать результаты анализа в PDF, TXT, CSV и JSON.
11. Строить интерактивную карту понятий и подтем с фильтрами и возможностью переходить на более подробный уровень.

## Веб-интерфейс на Python

Основные страницы реализованы с помощью Django Templates:

* `/login/` — вход;
* `/register/` — регистрация;
* `/library/` — библиотека пользователя;
* `/library/books/<book_id>/` — информация о книге;
* `/library/books/<book_id>/blocks/<block_id>/` — просмотр блока книги;
* `/library/concepts/` — список понятий;
* `/library/concepts/<concept_id>/` — информация о понятии;
* `/library/concepts/<concept_id>/compare/` — сравнение понятия;
* `/library/map/` — интерактивная карта.

## Умное семантическое разбиение текста

Pipeline обработки текста работает в два этапа.

### 1. Структурное разбиение

Сначала используются главы и разделы, которые уже есть в исходном документе:

* разделы и заголовки из FB2;
* страницы PDF, если структурные данные недоступны.

### 2. Семантическое разбиение

Внутри каждой главы текст дополнительно разбивается на логические блоки.

Для этого используются:

* embeddings абзацев;
* точки резкого изменения косинусного сходства между соседними абзацами;
* переходные слова и фразы;
* ограничения минимального, целевого и максимального размера блока.

Основные параметры можно настроить через `.env`:

```env
BLOCK_MIN_WORDS
BLOCK_TARGET_WORDS
BLOCK_MAX_WORDS
```

## Основные модели

В проекте используются следующие модели:

* `GlobalBookCache` — общий кэш загруженных книг;
* `UserBook` — книга пользователя;
* `LogicalBlock` — логический блок текста;
* `BookTheme` — тема книги;
* `ThemeSubtopic` — подтема;
* `Concept` — понятие;
* `ConceptMention` — упоминание понятия в тексте;
* `UserConceptEdit` — пользовательское редактирование понятия;
* `BookSummary` — краткое содержание книги.

Старые модели глоссария сохранены только для совместимости с миграциями.

## API

API продолжает работать и включает следующие endpoints.

### Авторизация

```text
POST /api/auth/register/
POST /api/auth/login/
POST /api/auth/logout/
GET  /api/auth/me/
```

### Книги

```text
GET    /api/books/
POST   /api/books/upload/
GET    /api/books/{id}/
DELETE /api/books/{id}/
POST   /api/books/{id}/protect/
POST   /api/books/{id}/reanalyze/
GET    /api/books/{id}/summary/
GET    /api/books/{id}/blocks/
GET    /api/books/{id}/blocks/{block_id}/
GET    /api/books/{id}/concepts/
GET    /api/books/{id}/export/?format=pdf|txt|csv|json
```

### Понятия

```text
GET   /api/concepts/
GET   /api/concepts/search/?q=...
GET   /api/concepts/{concept_id}/
GET   /api/concepts/{concept_id}/compare/
PATCH /api/concepts/mentions/{mention_id}/edit/
POST  /api/concepts/mentions/{mention_id}/reset/
```

### Статистика

```text
GET /api/stats/
```

## Настройка окружения

Скопируйте `.env.example` в `.env` и укажите необходимые значения.

Основные переменные:

```env
DATABASE_URL
POSTGRES_*

REDIS_URL
CELERY_BROKER_URL
CELERY_RESULT_BACKEND

LLM_PROVIDER
OLLAMA_BASE_URL
OLLAMA_MODEL

VECTOR_STORE
CHROMA_PATH
EMBEDDING_MODEL

TELEGRAM_BOT_TOKEN
```

## Запуск с локальными PostgreSQL и Redis

Перейдите в папку backend:

```powershell
cd backend
```

Создайте виртуальное окружение:

```powershell
python -m venv venv
```

Активируйте его:

```powershell
.\venv\Scripts\activate
```

Установите зависимости:

```powershell
pip install -r requirements.txt
```

Примените миграции:

```powershell
python manage.py migrate
```

Запустите Django:

```powershell
python manage.py runserver
```

В отдельном терминале запустите Celery:

```powershell
cd backend
.\venv\Scripts\activate
celery -A config worker -l info --pool=solo
```

После запуска веб-интерфейс доступен по адресу:

```text
http://127.0.0.1:8000/login/
```

## Запуск PostgreSQL и Redis через Docker

Если установлен Docker, PostgreSQL и Redis можно запустить командой:

```powershell
docker compose up -d postgres redis
```

## Ollama

Ollama используется для запуска языковой модели локально.

Например:

```bash
ollama pull qwen2.5:1.5b
```

Использование Ollama является необязательным. Если Ollama недоступна, система использует fallback-логику, поэтому проект продолжает работать.

## Telegram-бот

Для запуска Telegram-бота:

```powershell
cd backend
.\venv\Scripts\activate
python -m apps.telegram_bot.bot
```

Для работы бота необходимо указать `TELEGRAM_BOT_TOKEN` в `.env`.

## Тесты

Для запуска тестов:

```powershell
cd backend
.\venv\Scripts\activate
python manage.py test
```
