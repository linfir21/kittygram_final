# Kittygram

Социальная сеть для любителей котиков: пользователи регистрируются, загружают фотографии своих питомцев, указывают окрас, год рождения и достижения, а также могут смотреть и оценивать чужих котиков.

**Деплой:** https://linfir-kittygram.ddns.net

![CI/CD workflow](https://github.com/linfir21/kittygram_final/actions/workflows/main.yml/badge.svg)

## Стек технологий

- **Backend:** Python 3.12, Django 5, Django REST Framework, Djoser (аутентификация), PostgreSQL 13, Gunicorn
- **Frontend:** React 17, react-router-dom
- **Инфраструктура:** Docker, Docker Compose, Nginx (шлюз и раздача статики)
- **CI/CD:** GitHub Actions — тесты, сборка и пуш образов на Docker Hub, автодеплой на сервер, уведомление в Telegram

## Архитектура

Проект состоит из четырёх контейнеров, оркестрируемых через Docker Compose:

| Сервис | Описание |
|---|---|
| `db` | PostgreSQL 13, данные в volume `pg_data` |
| `backend` | Django-приложение на Gunicorn (порт 8000), раздаёт `/api/` и `/admin/` |
| `frontend` | Сборка React-приложения, статика копируется в общий volume |
| `gateway` | Nginx-шлюз, принимает внешний трафик на порту **9000** и проксирует: `/api/` и `/admin/` → backend, `/media/` → медиафайлы, всё остальное → статика фронтенда |

## API

Аутентификация — по токену (`/api/token/login/`, `/api/token/logout/`).

- `GET/POST /api/cats/` — список котиков / добавление нового
- `GET/PUT/PATCH/DELETE /api/cats/{id}/` — карточка котика
- `GET/POST /api/achievements/` — достижения
- Документация эндпоинтов Djoser — `/api/`

## Запуск локально

Требования: установленные Docker и Docker Compose.

```bash
# 1. Создайте файл окружения
cp .env.example .env
# отредактируйте .env (для локальной разработки поставьте DEBUG=True)

# 2. Соберите и запустите контейнеры
docker compose up --build -d

# 3. Примените миграции и соберите статику
docker compose exec backend python manage.py migrate
docker compose exec backend python manage.py collectstatic --noinput
```

Сайт будет доступен на http://localhost:9000

### Запуск backend без Docker (для разработки)

```bash
cd backend
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver
```

> В этом случае в `.env` укажите `DB_HOST=localhost` и запустите PostgreSQL самостоятельно.

## Деплой в продакшен

Для прода используется `docker-compose.production.yml` — он вместо сборки образов тянет готовые с Docker Hub (`linfir21/kittygram_backend`, `linfir21/kittygram_frontend`, `linfir21/kittygram_gateway`):

```bash
scp docker-compose.production.yml <user>@<server>:/путь/к/проекту/
# на сервере:
cd /путь/к/проекту
docker compose -f docker-compose.production.yml pull
docker compose -f docker-compose.production.yml down
docker compose -f docker-compose.production.yml up -d
docker compose -f docker-compose.production.yml exec backend python manage.py migrate
docker compose -f docker-compose.production.yml exec backend python manage.py collectstatic --noinput
```

### Переменные окружения

Задаются в файле `.env` (шаблон — `.env.example`):

| Переменная | Описание |
|---|---|
| `POSTGRES_DB` | имя базы данных |
| `POSTGRES_USER` | пользователь базы данных |
| `POSTGRES_PASSWORD` | пароль пользователя базы данных |
| `DB_HOST` | хост базы (`db` внутри Compose, `localhost` — локально) |
| `DB_PORT` | порт базы данных (по умолчанию 5432) |
| `SECRET_KEY` | секретный ключ Django (генерируется через `get_random_secret_key`) |
| `DEBUG` | режим отладки — `True` только для разработки |
| `ALLOWED_HOSTS` | разрешённые хосты/IP через запятую |

> ⚠️ Файл `.env` в репозитории не хранится — только `.env.example` с плейсхолдерами. Никогда не коммитьте реальные секреты.

## CI/CD

Workflow `.github/workflows/main.yml` при каждом пуше в `main`:

1. **tests** — flake8, isort и `manage.py test` на Python 3.12 с PostgreSQL
2. **build and push** — сборка и публикация трёх образов на Docker Hub
3. **deploy** — копирование `docker-compose.production.yml` на сервер, обновление контейнеров, миграции и статика
4. **send_message** — уведомление об успешном деплое в Telegram

Для работы CI в репозитории должны быть настроены секреты:

`DOCKER_USERNAME`, `DOCKER_PASSWORD`, `HOST`, `USER`, `SSH_KEY`, `TELEGRAM_TO`, `TELEGRAM_TOKEN`

## Автор

[linfir21](https://github.com/linfir21) — учебный проект Яндекс Практикума
