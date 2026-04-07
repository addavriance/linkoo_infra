# linkoo_infra

Конфигурационные файлы инфраструктуры сайта [linkoo.dev](https://linkoo.dev).

## Структура

```
cloud/    — конфиг облачного сервера-прокси
server/   — основной сервер приложения
```

## cloud

Отдельный публичный сервер, который принимает входящий трафик и проксирует его на основной сервер через Tailscale (IP `100.105.255.110:80`). Конфигурация максимально простая.

- `nginx.conf` — базовый конфиг nginx с подключением модуля geoip2 для определения страны клиента
- `default` — виртуальные хосты: редирект HTTP → HTTPS, проксирование `linkoo.dev` и `*.linkoo.dev` на основной сервер. Поддомены пробрасываются через заголовок `X-Subdomain`, страна — через `CF-IPCountry`. Для `/api/auth/max` включена поддержка WebSocket.

## server

Основной сервер с приложением. Всё поднимается через `docker-compose.yml`.

### Контейнеры

- **mongodb** — MongoDB 4.4 (версия без требования AVX), данные хранятся в named volume
- **redis** — Redis 7, ограничение памяти 128mb с политикой `allkeys-lru`
- **backend** — Node.js API, собирается из репозитория `linkoo_backend`. Переменные окружения: JWT, OAuth (Google, VK, Discord, GitHub), ЮКасса, S3
- **frontend** — SPA, собирается из репозитория `linkoo`. При сборке передаётся `VITE_API_URL`
- **nginx** — nginx:alpine, слушает порты 80 и 443, монтирует конфиг из `./nginx/nginx.conf`

### server/nginx/nginx.conf

Nginx внутри Docker-сети. Два upstream: `backend:5000` и `frontend:80`. Маршрутизация:

- `/api/` → backend (с поддержкой WebSocket)
- `/api-docs/` → backend
- `/{короткая ссылка}` → backend если бот (для OG-превью), иначе → frontend
- `/` → frontend

SSL не терминируется на уровне этого nginx — он работает по HTTP, TLS снимает облако.

### .env

Переменные окружения берутся из `.env` файла в папке `server`. Пример — `.env.example`.

## Деплой

Деплой автоматический через GitHub Actions (`.github/workflows/deploy.yml`). Срабатывает при пуше в `main` с изменениями в папке `server/`, либо вручную через `workflow_dispatch`.

Шаги:
1. Подключение к Tailscale (OAuth-токен из секретов)
2. Копирование `docker-compose.yml` и `nginx/nginx.conf` на сервер по SCP (через Tailscale IP)
3. `docker compose up -d --no-build` + `docker compose restart nginx`
4. Уведомление в Telegram о старте и результате деплоя
