<div align="center">

# 🍲 Foodgram

**Сервис рецептов: публикация, избранное, подписки на авторов и автоматический список покупок**

![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![Django](https://img.shields.io/badge/Django-6.0-092E20?logo=django&logoColor=white)
![DRF](https://img.shields.io/badge/DRF-3.17-A30000?logo=django&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker_Compose-4_services-2496ED?logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-reverse_proxy-009639?logo=nginx&logoColor=white)

</div>

## О проекте

Foodgram — веб-приложение, где пользователи публикуют рецепты, добавляют чужие в избранное, подписываются на авторов и собирают **список покупок**: сервис суммирует ингредиенты всех выбранных рецептов и отдаёт готовый файл для скачивания.

Архитектура — SPA на React + REST API на Django REST Framework. Всё приложение упаковано в **4 Docker-контейнера** и разворачивается одной командой.

## Возможности

- Регистрация и аутентификация по токену (Djoser), профиль пользователя с аватаром (загрузка изображения в Base64).
- Рецепты: создание и редактирование с картинкой, тегами и ингредиентами; редактировать и удалять может только автор.
- Фильтрация рецептов по тегам, автору, избранному и списку покупок; пагинация (6 рецептов на страницу).
- Поиск ингредиентов по началу названия — для автодополнения в форме рецепта.
- Избранное и список покупок; выгрузка списка покупок в `.txt` с агрегацией одинаковых ингредиентов на стороне БД (`annotate(Sum)`).
- Подписки на авторов с выдачей их последних рецептов.
- Справочник из **2186 ингредиентов** загружается management-командой.
- Админ-панель с поиском по пользователям и рецептам.

## Стек

| Слой | Технологии |
|---|---|
| Backend | Python 3.12, Django 6.0, Django REST Framework 3.17, Djoser, django-filter, Pillow |
| База данных | PostgreSQL 15 |
| Сервер приложений | Gunicorn |
| Инфраструктура | Docker, Docker Compose, Nginx (reverse proxy, раздача статики и медиа) |
| Frontend | React (SPA) |

## Архитектура

```
                ┌──────────────────────── Docker Compose ─────────────────────────┐
  браузер ──80──► nginx ──/api/, /admin/──► backend (Django + Gunicorn) ──► db (PostgreSQL)
                │   │                                                              │
                │   └── /, /static/, /media/ ── сборка frontend (React), volumes    │
                └──────────────────────────────────────────────────────────────────┘
```

| Контейнер | Назначение |
|---|---|
| `db` | PostgreSQL 15, данные в именованном volume |
| `backend` | Django + Gunicorn; при старте применяет миграции и собирает статику |
| `frontend` | собирает React-приложение и отдаёт сборку через общий volume |
| `nginx` | единая точка входа: проксирует `/api/` и `/admin/`, раздаёт SPA, статику и медиа |

## Что сделано

- Спроектированы модели: `User` (вход по email), `Recipe`, `Tag`, `Ingredient`, `RecipeIngredient`, `Favorite`, `ShoppingCart`, `Subscription` — с ограничениями уникальности на уровне БД.
- Разделены сериализаторы чтения и записи рецепта; вложенное создание ингредиентов; поле `Base64ImageField` для картинок.
- Собственный permission `IsAuthorOrReadOnly`, общие action-методы для избранного и корзины без дублирования кода.
- Написаны `Dockerfile` бэкенда, `docker-compose.yml` на 4 сервиса и конфигурация Nginx; доработан фронтенд (фильтрация по тегам в избранном, кнопки подписки и избранного).
- Проект был развёрнут на облачном сервере Ubuntu; API проверен Postman-коллекцией из `postman_collection/`.

## Запуск через Docker

**Требования:** Docker 24+ и Docker Compose v2.

```bash
git clone https://github.com/vadim-guseinov/foodgram.git
cd foodgram/infra

cp .env.example .env        # заполните POSTGRES_PASSWORD и SECRET_KEY
docker compose up -d --build

docker compose exec backend python manage.py load_ingredients   # справочник ингредиентов
docker compose exec backend python manage.py createsuperuser
```

После запуска:

- приложение — http://localhost/
- админ-панель — http://localhost/admin/
- спецификация API — [`docs/openapi-schema.yml`](docs/openapi-schema.yml)

### Переменные окружения (`infra/.env`)

| Переменная | Назначение |
|---|---|
| `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | параметры базы данных |
| `DB_HOST`, `DB_PORT` | адрес PostgreSQL (`db:5432` внутри Compose) |
| `SECRET_KEY` | секретный ключ Django |
| `DEBUG` | режим отладки (`True` / `False`) |
| `ALLOWED_HOSTS` | разрешённые хосты через пробел |
| `CORS_ALLOWED_ORIGINS`, `CSRF_TRUSTED_ORIGINS` | доверенные источники через пробел |

## Примеры запросов к API

Получить токен:

```bash
curl -X POST http://localhost/api/auth/token/login/ \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "strongpassword"}'
# {"auth_token": "<token>"}
```

Рецепты с тегами 1 и 2, только из избранного:

```bash
curl "http://localhost/api/recipes/?tags=1,2&is_favorited=1&page=1" \
  -H "Authorization: Token <token>"
```

Создать рецепт:

```bash
curl -X POST http://localhost/api/recipes/ \
  -H "Authorization: Token <token>" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Омлет",
        "text": "Взбить яйца с молоком и жарить 5 минут",
        "cooking_time": 10,
        "tags": [1],
        "ingredients": [{"id": 1123, "amount": 3}],
        "image": "data:image/png;base64,iVBORw0KGgo..."
      }'
```

Скачать список покупок:

```bash
curl http://localhost/api/recipes/download_shopping_cart/ \
  -H "Authorization: Token <token>" -o shopping_list.txt
```

## Автор

**Вадим Гусейнов** — Python-разработчик (Backend) · [Telegram](https://t.me/eviliing)
