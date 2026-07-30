# Lanner Aircraft — Telegram Mini App: магазин футболок

Telegram Mini App (WebApp) интернет-магазин на полном стеке:
**FastAPI + PostgreSQL + Redis + aiogram + React (WebApp) + React (Admin)**.

Оплата проходит нативно внутри Telegram через `sendInvoice`, покупатель
работает прямо в мессенджере, администратор управляет магазином из отдельной
веб-панели.

---

## Содержание
- [Стек технологий](#стек-технологий)
- [Возможности](#возможности)
- [Архитектура](#архитектура)
- [Требования](#требования)
- [Переменные окружения](#переменные-окружения)
- [Быстрый старт (локально)](#быстрый-старт-локально)
- [Локальный тест WebApp через туннель](#локальный-тест-webapp-через-туннель)
- [Продакшн-деплой](#продакшн-деплой)
- [Первичные данные (seed)](#первичные-данные-seed)
- [Полезные команды](#полезные-команды)
- [Структура проекта](#структура-проекта)


---

## Стек технологий

| Слой         | Технологии                                             |
|--------------|--------------------------------------------------------|
| Backend      | Python, FastAPI, SQLAlchemy, Alembic, Pydantic         |
| База данных  | PostgreSQL                                             |
| Кэш / очереди| Redis                                                  |
| Telegram-бот | aiogram                                                |
| WebApp       | React + TypeScript + Vite                              |
| Admin        | React + TypeScript + Vite                              |
| Инфраструктура | Docker, Docker Compose, Nginx (edge), Let's Encrypt  |

---

## Возможности

**Магазин (WebApp):**
- Каталог с категориями, поиском и баннерами
- Карточка товара: галерея, выбор цвета/размера с учётом остатков
- Корзина, избранное, оформление заказа
- Доставка: ПВЗ / курьер / СДЭК
- Промокоды
- Оплата через Telegram Invoice (`sendInvoice`)

**Бот (aiogram):**
- Команда `/start`, кнопка запуска WebApp
- Обработка платежей (`pre_checkout`, `successful_payment`)
- Уведомления о статусах заказа

**Admin-панель:**
- Товары (создание, редактирование, остатки, изображения)
- Заказы: смена статусов, трек-номера
- Промокоды
- Баннеры
- Дашборд с ключевыми показателями

---


---

## Требования

- **Docker** и **Docker Compose**
- Домен с DNS-записями `shop.example.com` и `admin.shop.example.com`
- Бот, созданный в [@BotFather](https://t.me/BotFather), с настроенным WebApp URL
- Платёжный провайдер, подключённый в @BotFather (для боевых платежей)
- Для локальной разработки без Docker: Node.js 18+ и Python 3.11+

---

## Переменные окружения

Все настройки задаются в файле `.env` (создаётся из `.env.example`).
Ключевые параметры:

| Переменная            | Описание                                          | Пример                              |
|-----------------------|---------------------------------------------------|-------------------------------------|
| `BOT_TOKEN`           | Токен бота из @BotFather                           | `123456:AA...`                      |
| `BOT_WEBHOOK_SECRET`  | Секрет для проверки вебхука Telegram               | `s3cr3t`                            |
| `WEBAPP_URL`          | Публичный HTTPS-URL WebApp                         | `https://shop.example.com`          |
| `PAYMENT_PROVIDER_TOKEN` | Токен платёжного провайдера                     | `284685063:TEST:...`                |
| `POSTGRES_USER`       | Пользователь БД                                    | `teeshop`                           |
| `POSTGRES_PASSWORD`   | Пароль БД                                          | `changeme`                          |
| `POSTGRES_DB`         | Имя БД                                             | `teeshop`                           |
| `REDIS_URL`           | Строка подключения к Redis                         | `redis://redis:6379/0`              |
| `ADMIN_EMAIL`         | Email супер-админа (создаётся при seed)            | `admin@example.com`                 |
| `ADMIN_PASSWORD`      | Пароль супер-админа                                | `strong-password`                   |
| `SECRET_KEY`          | Ключ для подписи JWT/сессий                        | случайная длинная строка            |

> ⚠️ Никогда не коммитьте реальный `.env` в репозиторий. В комплекте только
> `.env.example` с плейсхолдерами.

---

## Быстрый старт (локально)

```bash
cp .env.example .env
# отредактируйте .env (BOT_TOKEN, пароли, ADMIN_EMAIL/ADMIN_PASSWORD и т.д.)

docker compose up --build

После старта:

Backend: http://localhost:8000
API-документация (Swagger): http://localhost:8000/docs
WebApp (dev): http://localhost:5173 (см. вывод контейнера frontend)
Admin (dev): http://localhost:5174 (см. вывод контейнера admin)

```

---

## Локальный тест WebApp через туннель


Telegram требует HTTPS для WebApp, поэтому для локальной проверки поднимите
туннель через cloudflared (или ngrok):

Bash

cloudflared tunnel --url http://localhost:8000
Скопируйте выданный HTTPS-URL.
Пропишите его в .env → WEBAPP_URL.
Укажите тот же URL в @BotFather:
/setmenubutton → WebApp URL.
Перезапустите бота, откройте чат с ним и нажмите кнопку запуска.

---

## Продакшн-деплой

Настройте DNS: shop.example.com и admin.shop.example.com → IP сервера.
Заполните .env реальными значениями (домены, токены, пароли).
Получите TLS-сертификаты (Let's Encrypt) и положите в nginx/certs/:
Bash

certbot certonly --standalone \
  -d shop.example.com -d admin.shop.example.com
cp /etc/letsencrypt/live/shop.example.com/fullchain.pem nginx/certs/
cp /etc/letsencrypt/live/shop.example.com/privkey.pem   nginx/certs/
Запустите:
Bash

docker compose -f docker-compose.prod.yml --env-file .env up --build -d
Установите webhook бота (если используется webhook-режим):
Bash

curl -F "url=https://shop.example.com/webhook/telegram" \
     -F "secret_token=$BOT_WEBHOOK_SECRET" \
     "https://api.telegram.org/bot$BOT_TOKEN/setWebhook"
В @BotFather:
/setmenubutton → WebApp URL: https://shop.example.com
Подключите платёжного провайдера (/mybots → Payments).

---

## Первичные данные (seed)

При первом запуске backend автоматически:

применяет миграции (alembic upgrade head);
выполняет seed: создаёт супер-админа из .env, а также базовые цвета, размеры и категории.
Вход в админку: ADMIN_EMAIL / ADMIN_PASSWORD
(URL: https://admin.shop.example.com).

---

## Полезные команды

Bash

# Логи
docker compose -f docker-compose.prod.yml logs -f backend
docker compose -f docker-compose.prod.yml logs -f bot

# Миграции вручную
docker compose exec backend alembic revision --autogenerate -m "msg"
docker compose exec backend alembic upgrade head

# Пересборка одного сервиса
docker compose -f docker-compose.prod.yml up -d --build frontend

# Бэкап БД
docker compose exec postgres pg_dump -U teeshop teeshop > backup.sql

# Восстановление БД из бэкапа
cat backup.sql | docker compose exec -T postgres psql -U teeshop teeshop

# Проверка webhook бота
curl "https://api.telegram.org/bot$BOT_TOKEN/getWebhookInfo"

---

## Структура проекта


teeshop/
├── backend/    # FastAPI: API, модели, миграции, seed, telegram-verify
├── bot/        # aiogram: команды, платежи, уведомления
├── frontend/   # React WebApp (магазин)
├── admin/      # React Admin (панель управления)
├── nginx/      # edge-конфиг + сертификаты
├── .env.example
└── docker-compose*.yml
