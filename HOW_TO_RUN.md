# Drishtiksha: How to Run

This guide explains how to run the complete Drishtiksha application on
Windows using Docker Desktop.

The application has two Docker Compose stacks:

1. **ML server**: Python/FastAPI inference service and its Redis instance.
2. **Main application**: React frontend, Node.js backend, PostgreSQL, and Redis.

Start the ML server first, then start the main application.

## 1. Prerequisites

Install and start:

- Docker Desktop for Windows
- Git, if cloning the project

The Docker-based setup does not require installing Python, CMake, Visual Studio
Build Tools, Node.js, or PostgreSQL on the host machine.

Verify Docker is running:

```powershell
docker info
```

If this command fails, start Docker Desktop and retry.

## 2. Open the project directory


The project root contains:

```text
Backend\
Frontend\
Server\
docker-compose.yml
docker-compose.local.yml
```

## 3. Download the ML model weights

The model weights are not included in the repository. Download the required
files from the project maintainer's model storage and place them in:

```text
Server\models\
```

For the default configuration, these files are required:

```text
Server\models\EyeBlink-CNN-LSTM-v1.pth
Server\models\Face-Landmarks.dat
Server\models\STFT-Spectrogram-CNN-v2.pth
```

Verify the files:

```powershell
Get-ChildItem .\Server\models
```

The ML server cannot start until the configured model files exist. Do not
rename the files unless you also update `Server\configs\config.yaml`.

## 4. Configure the ML server

Create the ML server environment file:

```powershell
Copy-Item .\Server\.env.example .\Server\.env
```

The default configuration activates:

```text
EYEBLINK-CNN-LSTM-V1
STFT-SPECTROGRAM-CNN-V2
```

The ML server Compose configuration supplies these container-specific values:

```text
REDIS_URL=redis://redis:6379
ASSETS_BASE_URL=http://localhost:3000
STORAGE_PATH=/app/storage
```

For a local run, `DEVICE="cpu"` is recommended unless Docker is configured for
GPU access.

## 5. Start the ML server

Run these commands from the project root:

```powershell
cd ".\Server"
docker compose up --build -d
```

The first build can take a long time because it installs PyTorch, dlib, and
other machine-learning dependencies.

Check the ML containers:

```powershell
docker compose ps
```

View the ML application logs:

```powershell
docker compose logs -f app
```

The service is ready when the logs show Uvicorn running on port `8000` and the
configured models have loaded successfully.

ML server address:

```text
http://localhost:8000
```

Stop viewing logs with `Ctrl+C`. This only exits log viewing; it does not stop
the containers.

Return to the project root:

```powershell
cd ..
```

## 6. Configure the main application

Create the root environment file from the backend Docker template:

```powershell
Copy-Item .\Backend\.env.docker.example .\.env
```

For local development, the template defaults are sufficient. Before production
use, change at least:

```text
POSTGRES_PASSWORD
JWT_SECRET
JWT_REFRESH_SECRET
SERVER_API_KEY
```

The backend uses the ML server through:

```text
SERVER_URL=http://host.docker.internal:8000
```

This allows the backend container to reach the ML server running in the
separate ML Compose stack.

## 7. Start the main application

From the project root, run:

```powershell
docker compose -f docker-compose.yml -f docker-compose.local.yml up --build -d
```

This starts:

- Frontend
- Backend API
- Background worker
- PostgreSQL
- Redis

Check the services:

```powershell
docker compose -f docker-compose.yml -f docker-compose.local.yml ps
```

View all main-application logs:

```powershell
docker compose -f docker-compose.yml -f docker-compose.local.yml logs -f
```

View only backend logs:

```powershell
docker compose -f docker-compose.yml -f docker-compose.local.yml logs -f backend
```

## 8. Open the application

| Service | Address |
| --- | --- |
| Frontend | http://localhost:5173 |
| Backend API | http://localhost:3000 |
| ML server | http://localhost:8000 |
| PostgreSQL | localhost:5432 |
| Redis | localhost:6379 |

Open the frontend in a browser:

```text
http://localhost:5173
```

## 9. Optional: Start Prisma Studio

Prisma Studio provides a database GUI:

```powershell
docker compose -f docker-compose.yml -f docker-compose.local.yml --profile studio up -d prisma-studio
```

Open:

```text
http://localhost:5555
```

## 10. Normal shutdown

Stop the main application:

```powershell
docker compose -f docker-compose.yml -f docker-compose.local.yml down
```

Stop the ML server:

```powershell
cd ".\Server"
docker compose down
cd ..
```

## 11. Clean reset

Use this only when you intentionally want to delete local PostgreSQL, Redis,
and application data:

```powershell
docker compose -f docker-compose.yml -f docker-compose.local.yml down -v

cd ".\Server"
docker compose down -v
cd ..
```

The `-v` option deletes Docker volumes. Database data and Redis data stored in
those volumes will be lost.

## 12. Restart after code or configuration changes

For Dockerfile, dependency, Compose, or environment changes:

```powershell
docker compose -f .\Server\docker-compose.yml down
docker compose -f .\Server\docker-compose.yml up --build -d

docker compose -f .\docker-compose.yml -f .\docker-compose.local.yml down
docker compose -f .\docker-compose.yml -f .\docker-compose.local.yml up --build -d
```

Use `--no-cache` only when Docker is reusing a stale build layer:

```powershell
docker compose -f .\Server\docker-compose.yml build --no-cache
```

## 13. Troubleshooting

### `Server\.env not found`

Create it from the template:

```powershell
Copy-Item .\Server\.env.example .\Server\.env
```

### `File path does not exist: models/...`

The model weights are missing. Place the exact files listed in
`Server\configs\config.yaml` inside `Server\models`.

### `ASSETS_BASE_URL Field required`

Ensure `Server\.env` contains:

```text
ASSETS_BASE_URL=http://localhost:3000
STORAGE_PATH=/app/storage
```

The Compose file also supplies these values to the ML container.

### `CMake is not installed`

This occurs when running `uv sync` directly on Windows. The Docker setup
installs CMake and the Linux C++ build tools automatically. Prefer the Docker
commands in this guide.

### `EACCES: permission denied, mkdir '/app/node_modules'`

Rebuild using the current backend Dockerfile:

```powershell
docker compose -f .\docker-compose.yml -f .\docker-compose.local.yml build --no-cache
```

### A port is already allocated

Find the process or container using the port, then stop it. Check Docker
containers with:

```powershell
docker ps
```

The ML Compose stack does not publish its Redis container to the host, so it
does not conflict with the main application's Redis port.

### The backend cannot reach the ML server

Confirm both stacks are running:

```powershell
docker compose -f .\Server\docker-compose.yml ps
docker compose -f .\docker-compose.yml -f .\docker-compose.local.yml ps
```

Confirm the root `.env` contains:

```text
SERVER_URL=http://host.docker.internal:8000
```

## 14. Direct Windows development without Docker

This is optional and more difficult because dlib must compile locally.

Install:

- Python 3.12
- CMake, added to `PATH`
- Visual Studio 2022 Build Tools with **Desktop development with C++**
- Redis
- uv

Verify from Developer PowerShell for VS 2022:

```powershell
cmake --version
cl
```

Then:

```powershell
cd ".\Server"
uv sync
uv run uvicorn src.app.main:app --host 0.0.0.0 --port 8000 --reload
```

Docker is the recommended Windows setup because it provides the required
native build tools inside the Linux image.
