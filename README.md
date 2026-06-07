# Momo Store Docker

Проектная работа по контейнеризации приложения `momo-store`. В репозитории есть Go backend, Vue frontend, Dockerfile для обеих частей и `docker-compose.yml` для запуска всего приложения.

Снаружи открыты два порта:

- `80` — frontend;
- `8081` — backend API через nginx-балансировщик.

Backend-контейнеры сами порт на хост не публикуют. Это сделано специально, чтобы можно было поднимать несколько backend-реплик без конфликта портов.

## Запуск

Основной вариант:

```bash
docker compose up --build
```

Проверка:

```bash
curl -i http://localhost:8081/health
curl -i http://localhost:80/health
curl -i http://localhost:80/api/health
```

Остановить контейнеры:

```bash
docker compose down
```

Если нужно поменять порты, теги образов или лимиты ресурсов, можно создать локальный `.env` из примера:

```bash
cp .env.example .env
```

Файл `.env` нужен только для локальных переопределений. Секреты и пароли в него добавлять не нужно.

## Dev-профиль

Dev-профиль поднимает backend через `go run`, а frontend через Vue dev server:

```bash
docker compose -f docker-compose.dev.yml --profile dev up --build
```

По умолчанию dev frontend открыт на `http://localhost:3000`, backend API — на `http://localhost:8082`.

Жёсткие CPU/RAM-лимиты в dev-файле не задаются. Основной `docker-compose.yml` остаётся вариантом с production-лимитами.

## Переменные

Основные значения по умолчанию:

- `MOMO_BACKEND_IMAGE=docker-project-backend`
- `MOMO_FRONTEND_IMAGE=docker-project-frontend`
- `MOMO_PROXY_IMAGE=docker-project-backend-lb`
- `MOMO_IMAGE_TAG=latest`
- `MOMO_WEB_PORT=80`
- `MOMO_API_PORT=8081`
- `MOMO_WEB_API_PATH=/api`
- `MOMO_WEB_PUBLIC_BASE=/`
- `MOMO_BACKEND_CPUS=0.50`
- `MOMO_BACKEND_MEMORY=128m`
- `MOMO_PROXY_CPUS=0.25`
- `MOMO_PROXY_MEMORY=64m`
- `MOMO_WEB_CPUS=0.25`
- `MOMO_WEB_MEMORY=64m`
- `MOMO_DEV_WEB_PORT=3000`
- `MOMO_DEV_API_PORT=8082`
- `MOMO_DEV_API_URL=http://localhost:8082`

`MOMO_WEB_API_PATH` передаётся во frontend на этапе сборки и используется как base URL для Axios. По умолчанию frontend ходит в API через `/api`, а nginx внутри frontend-контейнера проксирует эти запросы в `backend-lb`.

## Образы

Backend собирается многоэтапно: сначала Go-бинарник в `golang:1.25-alpine3.22`, затем запуск в минимальном `alpine:3.22`. В финальном образе нет Go toolchain.

Frontend тоже собирается многоэтапно: зависимости и production build выполняются в `node:22-alpine3.22`, а готовая статика попадает в `nginxinc/nginx-unprivileged:1.29-alpine3.22`.

Для backend-балансировщика используется отдельный образ на `nginxinc/nginx-unprivileged:1.29-alpine3.22`.

Локальные размеры после сборки:

- `docker-project-backend:latest` — `31.5MB`;
- `docker-project-backend-lb:latest` — `98.3MB`;
- `docker-project-frontend:latest` — `101MB`.

Команды для отдельной сборки:

```bash
docker build -t docker-project-backend:latest ./backend
docker build -t docker-project-backend-lb:latest ./deploy/nginx
docker build -t docker-project-frontend:latest ./frontend
```

## Docker Compose

В основном compose описаны три сервиса:

- `backend` — Go API во внутренней сети `backend`;
- `backend-lb` — nginx между внешним портом `8081` и backend-репликами;
- `frontend` — nginx со статикой Vue и прокси `/api/` в `backend-lb`.

Backend можно масштабировать так:

```bash
docker compose up -d --scale backend=3
```

`backend-lb` подключён к Docker DNS и обращается к имени сервиса `backend`. При нескольких репликах Docker возвращает несколько адресов, а nginx периодически резолвит это имя заново.

## Данные

В приложении используется fake in-memory store. Постоянной базы данных или файлового хранилища здесь нет, поэтому named volume для бизнес-данных не добавлялся.

Для временных файлов nginx и `/tmp` используются `tmpfs`, потому что контейнеры запускаются с read-only файловой системой.

## Безопасность

Сделано:

- backend запускается не от root, а от пользователя `app`;
- frontend и `backend-lb` используют unprivileged nginx;
- включены `read_only: true`, `tmpfs`, `no-new-privileges:true`;
- у контейнеров сброшены capabilities через `cap_drop: [ALL]`;
- для сервисов задан `pids_limit: 200`;
- наружу опубликованы только `80` и `8081`;
- frontend nginx добавляет базовые security headers;
- финальные образы не содержат инструментов сборки;
- Docker build context сокращён через `.dockerignore`;
- секреты не записываются в Dockerfile, compose или README.

Runtime-секретов у приложения нет. Для публикации образов в GitHub Actions используются repository secrets `DOCKER_USER` и `DOCKER_PASSWORD`, где `DOCKER_PASSWORD` — Docker Hub access token.

## Trivy

Локальная проверка:

```bash
trivy image docker-project-backend:latest
trivy image docker-project-backend-lb:latest
trivy image docker-project-frontend:latest
```

В `.github/workflows/deploy.yaml` добавлен отдельный job со сканированием образов через Trivy. Базовые jobs из исходного пайплайна сохранены.

## Проверка

```bash
docker compose config
docker compose -f docker-compose.dev.yml --profile dev config
docker compose up --build
curl -i http://localhost:8081/health
curl -i http://localhost:80/health
curl -i http://localhost:80/api/health
docker compose up -d --scale backend=3
docker compose ps
docker compose up -d --scale backend=1
```

Backend-тесты можно запустить через Docker, если Go не установлен на хосте:

```bash
docker run --rm -v "$PWD/backend:/src:ro" -w /src golang:1.25-alpine3.22 go test ./...
```
