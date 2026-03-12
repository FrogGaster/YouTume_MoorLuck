# MoorLuck hosting MVP

Платформа в стиле YouTube для летсплеев по HOI4.

## Что реализовано

- регистрация/логин, JWT
- встроенный вход админа: `admin / admin`
- загрузка видео только для `admin` и пользователей с `can_upload=true`
- лайки, комментарии, подписки
- `watch later`, `continue watching`, подсказки поиска
- аналитика: показы, CTR, удержание
- стриминг через `Range` (`206 Partial Content`)
- переключение качества (если есть FFmpeg: `720p/480p/360p`)
- очередь обработки видео (`processing/ready/failed`)
- HLS playback endpoint (если FFmpeg доступен)
- модерация: жалобы, скрытие/бан, очередь админа
- уведомления: новые видео по подпискам и новые комментарии автору
- рекомендации v2: `home/watch_next/trending/subscriptions` + A/B bucket
- studio dashboard v2: просмотры/watch time по дням, источники, топ роликов

---

## 1) Требования

### Для локальной разработки

- Python 3.11+ (проект проверялся на 3.13)
- Node.js 18+ и npm
- (опционально) FFmpeg для автогенерации качеств видео

Проверка:

```powershell
python --version
node --version
npm --version
ffmpeg -version
```

---

## 2) Быстрый запуск (Windows / PowerShell)

Открой 2 терминала.

### Терминал 1: backend

```powershell
cd "C:\Users\MoorLuck\Desktop\свой ютуб\backend"
python -m venv .venv
.\.venv\Scripts\activate
python -m pip install -r requirements.txt
uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

Проверка:

- [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health)

### Терминал 2: frontend

```powershell
cd "C:\Users\MoorLuck\Desktop\свой ютуб\frontend"
npm install
npm run dev -- --host 127.0.0.1 --port 5173
```

Открыть:

- [http://127.0.0.1:5173](http://127.0.0.1:5173)

---

## 3) Первый вход и роли

### Вход админа (встроенный)

- логин: `admin`
- пароль: `admin`

При первом таком входе админ создается автоматически.

### Выдача доступа на загрузку другим пользователям

1. Залогиниться как админ
2. Перейти в профиль
3. Ввести ID пользователя
4. Нажать `Выдать can_upload`

---

## 4) Конфиг через переменные окружения

### Backend

- `DATABASE_URL`  
  По умолчанию SQLite: `backend/app.db`  
  Можно PostgreSQL, например:
  `postgresql+psycopg2://user:password@localhost:5432/moorluck`

- `JWT_SECRET`  
  Обязательно сменить в проде.

- `STORAGE_BACKEND`  
  `local` (по умолчанию) или `s3`

- `S3_ENDPOINT_URL`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_BUCKET`, `S3_REGION`  
  Нужны при `STORAGE_BACKEND=s3` (MinIO тоже поддерживается).

- `USE_CELERY=1`  
  Включает обработку видео через Celery worker.

- `CELERY_BROKER_URL`, `CELERY_RESULT_BACKEND`  
  Обычно Redis, например `redis://localhost:6379/0`

Пример (PowerShell):

```powershell
$env:DATABASE_URL = "postgresql+psycopg2://user:pass@localhost:5432/moorluck"
$env:JWT_SECRET = "SUPER_SECRET_CHANGE_ME"
uvicorn main:app --host 127.0.0.1 --port 8000
```

### Frontend

- `VITE_API_BASE` (опционально)

Файл `frontend/.env`:

```env
VITE_API_BASE=http://127.0.0.1:8000
```

---

## 5) FFmpeg и качества видео

Если установлен `ffmpeg`, при загрузке бэкенд попробует сделать доп. версии видео:

- `720p`
- `480p`
- `360p`

Если FFmpeg не установлен — доступен только `Оригинал`.

---

## 6) Сборка для продакшена

## Frontend

```powershell
cd "C:\Users\MoorLuck\Desktop\свой ютуб\frontend"
npm ci
npm run build
```

Артефакты будут в `frontend/dist`.

## Backend

```powershell
cd "C:\Users\MoorLuck\Desktop\свой ютуб\backend"
.\.venv\Scripts\activate
python -m pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 2
```

Для продакшена обычно ставят reverse proxy (Nginx/Caddy):

- раздавать `frontend/dist`
- проксировать `/auth`, `/videos`, `/channels`, `/analytics`, `/studio`, `/me`, `/search`, `/media` на backend

### Полный стек через Docker Compose (Postgres + Redis + MinIO)

```powershell
cd "C:\Users\MoorLuck\Desktop\свой ютуб"
docker compose up -d
```

Сервисы:

- Postgres: `localhost:5432`
- Redis: `localhost:6379`
- MinIO API: `localhost:9000`
- MinIO Console: `http://localhost:9001`

---

## 7) Обновление проекта

```powershell
# backend
cd "...\backend"
.\.venv\Scripts\activate
python -m pip install -r requirements.txt
python -m pytest -q

# frontend
cd "...\frontend"
npm install
npm run build
```

После этого перезапусти backend/frontend процессы.

### Миграции Alembic

```powershell
cd "...\backend"
alembic upgrade head
```

Создать новую миграцию после изменений моделей:

```powershell
alembic revision --autogenerate -m "describe change"
alembic upgrade head
```

### Запуск Celery worker (опционально)

```powershell
cd "...\backend"
.\.venv\Scripts\activate
$env:USE_CELERY="1"
$env:CELERY_BROKER_URL="redis://localhost:6379/0"
celery -A celery_app.celery worker --loglevel=info
```

---

## 8) Полезные эндпоинты

- `POST /auth/register`
- `POST /auth/login`
- `GET /auth/me`
- `PATCH /admin/users/{id}/upload-access`
- `POST /videos/upload`
- `DELETE /videos/{id}`
- `GET /videos/{id}/stream?quality=source|720p|480p|360p`
- `GET /videos/{id}/qualities`
- `GET /videos/{id}/processing-status`
- `GET /videos/{id}/hls`
- `POST/DELETE /videos/{id}/watch-later`
- `GET /me/watch-later`
- `GET /me/continue-watching`
- `GET /search/suggestions?q=...`
- `POST /reports/video/{id}`
- `POST /reports/comment/{id}`
- `GET /moderation/reports`
- `PATCH /moderation/reports/{kind}/{id}`
- `GET /me/notifications`
- `POST /me/notifications/{id}/read`
- `GET /studio/dashboard`

---

## 9) Если что-то не работает

- **Открывается не тот адрес**: обязательно `http://127.0.0.1:5173`
- **Frontend не видит backend**: проверь `VITE_API_BASE` и `http://127.0.0.1:8000/health`
- **Порт занят**: смени порт в командах запуска
- **Видео только в оригинале**: проверь наличие `ffmpeg`
- **Сломалась база после экспериментов**: останови бэкенд, удали `backend/app.db`, запусти заново
