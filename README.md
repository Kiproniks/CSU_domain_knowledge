# Личная библиотека с анализом книг FB2 и PDF

Сервис помогает читателям учебной и профессиональной литературы возвращаться к нужным фрагментам книги без повторного чтения целых глав. Пользователь загружает книгу и получает краткое содержание, структуру тем и карту понятий с привязкой к исходному тексту.

## Функциональность

- Загрузка FB2 и PDF с текстовым слоем, проверка формата и размера файла (до 50 МБ).
- Просмотр конспекта, тем, подтем и логических блоков книги.
- Поиск и сравнение понятий в своей библиотеке, просмотр исходных фрагментов и персональные правки к упоминаниям.
- Интерактивная карта тем и понятий, экспорт анализа из веб-интерфейса в PDF, TXT, CSV и JSON.
- Управление библиотекой через веб-интерфейс и основные операции с книгами через Telegram-бота.

## Архитектура и проектирование

- **Модель данных.** `UserBook` хранит владельца, статус и настройки книги. `GlobalBookCache` хранит общий результат анализа по SHA-256 файла: повторная загрузка может использовать готовый результат. `ConceptMention` связывает понятие с исходным блоком, а `UserConceptEdit` хранит правку отдельно от общего результата.
- **Обработка.** После проверки файла Django ставит задачу в Celery. Задача извлекает текст, выделяет логические блоки, строит иерархию тем и подтем и сохраняет конспект. Статус книги отражает ход обработки и её исход. Ollama может участвовать в анализе; предусмотрены эвристики для работы без неё.
- **Интерфейсы.** Django Templates обслуживают веб-интерфейс. REST API использует токены: `/api/auth/` — регистрация и вход, `POST /api/books/upload/` — загрузка, `GET /api/books/{id}/summary/` — результат, `GET /api/concepts/search/` — поиск. Доступ к книгам ограничен библиотекой пользователя.

## Технологии

Python, Django, Django REST Framework, PostgreSQL, Celery, Redis, aiogram. ChromaDB и Sentence Transformers используются для работы с векторами текста, pypdf и lxml — для разбора файлов, Ollama — для локальной языковой модели.

## Запуск

Для локального демо на Windows понадобятся Python 3.12 и uv. Выполните из корня репозитория в PowerShell:

```powershell
uv venv --python 3.12 .venv
uv pip install --python .venv/Scripts/python.exe --only-binary chroma-hnswlib -r backend/requirements.txt
Copy-Item .env.example .env
```

В `.env` установите следующие значения, заменив существующие строки и раскомментировав параметры SQLite и Celery:

```dotenv
USE_SQLITE=True
CELERY_TASK_ALWAYS_EAGER=True
LLM_PROVIDER=none
VECTOR_STORE=none
EMBEDDING_DISABLE_ST=1
```

В этом режиме книга анализируется в процессе Django: для демо не нужны PostgreSQL, Redis, Ollama и загрузка модели эмбеддингов.

```powershell
Set-Location backend
..\.venv\Scripts\python.exe manage.py migrate
..\.venv\Scripts\python.exe manage.py runserver
```

Откройте [страницу регистрации](http://127.0.0.1:8000/register/), затем загрузите свой FB2 или PDF с текстовым слоем. Тесты запускаются из `backend/`: `..\.venv\Scripts\python.exe manage.py test`.

Для фоновой обработки используйте PostgreSQL и Redis из [Docker Compose](docker-compose.yml): из корня запустите `docker compose up -d postgres redis`, а в `.env` отключите SQLite и синхронный режим Celery. Повторите миграции для PostgreSQL и запустите воркер из `backend/` в отдельном терминале: `..\.venv\Scripts\celery.exe -A config worker -l info --pool=solo`.
