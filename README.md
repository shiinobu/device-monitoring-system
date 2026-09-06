# Device Monitoring System (DMS)

A realtime Device Monitoring System built to monitor device availability through periodic heartbeats and WebSocket status updates.

The project demonstrates a practical backend architecture with **Go, PostgreSQL, JWT authentication, background monitoring, WebSocket, and Docker Compose**, integrated with a **Next.js dashboard**.

## Highlights

- JWT-based admin authentication
- Device registration and CRUD management
- Device heartbeat every 10 seconds
- Automatic `ONLINE` / `OFFLINE` detection
- Realtime status updates through WebSocket
- Browser notifications for status transitions
- Monitoring summary and CSV export
- Five-device simulator for local testing
- Docker Compose setup for the complete stack
- Layered backend architecture: Handler → Service → Repository

## Architecture

```text
┌───────────────┐
│ Device /      │
│ Simulator     │
└───────┬───────┘
        │ Heartbeat / REST API
        ▼
┌──────────────────────┐
│ Go Backend           │
│ Gin + JWT            │
│ Handler → Service    │
│ → Repository         │
└──────┬───────┬──────┘
       │       │
       ▼       ▼
┌──────────┐  ┌────────────────┐
│PostgreSQL│  │ Status Monitor │
└──────────┘  └───────┬────────┘
                      │
                      ▼
               ┌──────────────┐
               │  WebSocket   │
               └──────┬───────┘
                      ▼
               ┌──────────────┐
               │ Next.js      │
               │ Dashboard    │
               └──────────────┘
```

### Realtime flow

1. A device sends a heartbeat every 10 seconds.
2. The backend records `last_seen` and marks the device `ONLINE`.
3. A background monitor checks device activity every 5 seconds.
4. A device becomes `OFFLINE` when no heartbeat is received for more than 30 seconds.
5. Status transitions are broadcast to connected dashboards through WebSocket.
6. The frontend updates the device list and monitoring statistics without a page refresh.

## Tech Stack

### Backend

- Go 1.26+
- Gin
- pgxpool
- JWT
- bcrypt
- Gorilla WebSocket

### Frontend

- Next.js 16
- React 19
- TypeScript
- Tailwind CSS

### Database & Infrastructure

- PostgreSQL 16+
- Docker
- Docker Compose

## Project Structure

```text
device-monitoring-system/
├── backend/
│   ├── cmd/
│   │   ├── server/
│   │   └── seed-admin/
│   ├── internal/
│   │   ├── auth/
│   │   ├── config/
│   │   ├── database/
│   │   ├── handlers/
│   │   ├── middleware/
│   │   ├── models/
│   │   ├── repositories/
│   │   ├── services/
│   │   └── websocket/
│   └── migrations/
├── frontend/
│   └── src/
│       ├── app/
│       ├── components/
│       ├── lib/
│       └── types/
├── simulator/
├── docker-compose.yml
└── README.md
```

## Requirements

For local development:

- Go 1.26+
- Node.js 22+
- npm
- PostgreSQL 16+
- Git

For the easiest setup, use Docker Desktop with Docker Compose.

## Installation

### Option 1 — Docker Compose

From the project root:

```powershell
docker compose build
docker compose up -d
docker compose ps
```

Or build and start in one command:

```powershell
docker compose up -d --build
```

Expected services:

```text
postgres
backend
frontend
simulator
```

Open the dashboard:

```text
http://localhost:3000
```

Check the backend:

```text
http://localhost:8080/health
```

Stop the stack:

```powershell
docker compose down
```

### Option 2 — Local Development

Clone the repository:

```powershell
git clone https://github.com/shiinobu/device-monitoring-system.git
cd device-monitoring-system
```

Install backend dependencies:

```powershell
cd backend
go mod download
```

Install frontend dependencies:

```powershell
cd ..\frontend
npm install
```

Install simulator dependencies:

```powershell
cd ..\simulator
go mod download
```

Create `backend/.env` from `backend/.env.example` and configure PostgreSQL and application secrets for your local environment.

For local development, the database URL normally uses:

```env
DATABASE_URL=postgres://postgres:postgres@localhost:5432/dms?sslmode=disable
```

Run the database migrations in numerical order, then start the services:

```powershell
# Backend
cd backend
go run ./cmd/server

# Frontend
cd ..\frontend
npm run dev

# Simulator
cd ..\simulator
go run .
```

## Configuration

Backend environment variables:

```env
APP_ENV=development
APP_PORT=8080
DATABASE_URL=postgres://postgres:postgres@postgres:5432/dms?sslmode=disable
JWT_SECRET=change-this-secret
JWT_EXPIRATION=24h
HEARTBEAT_TIMEOUT=30s
STATUS_CHECK_INTERVAL=5s
ADMIN_USERNAME=admin
ADMIN_PASSWORD=change-this-password
CORS_ORIGIN=http://localhost:3000
```

Frontend:

```env
NEXT_PUBLIC_API_URL=http://localhost:8080/api/v1
NEXT_PUBLIC_WS_URL=ws://localhost:8080/api/v1/ws
```

When the backend runs inside Docker, containers communicate using Docker service names. Browser-side `NEXT_PUBLIC_*` URLs should continue to use `localhost` when the browser accesses the application from the host machine.

> Never commit real `.env` files, passwords, database credentials, or production JWT secrets. Use `backend/.env.example` as the configuration template.

## Database

Create the database if you are running PostgreSQL locally:

```sql
CREATE DATABASE dms;
```

Run migrations from `backend/migrations/` in numerical order.

The system tracks device information such as:

- Device ID
- Device name
- Serial number
- OS version
- IP address
- Location
- Status
- Last heartbeat
- Last online/offline timestamps
- Created/updated timestamps

## Admin User

For development, create the initial admin account with:

```powershell
cd backend
go run ./cmd/seed-admin
```

Use the credentials configured through environment variables. Do not use development credentials in production.

## Simulator

The simulator represents five devices:

```text
DMS-001  Office PC 001
DMS-002  Office PC 002
DMS-003  Office PC 003
DMS-004  Warehouse PC 001
DMS-005  IT Administrator PC
```

On startup it authenticates, checks whether the devices already exist, registers missing devices, and starts the heartbeat loop.

Heartbeat interval:

```text
10 seconds
```

Stop the simulator with `Ctrl + C`. After heartbeats stop, the backend will eventually mark the devices as `OFFLINE`.

## Device Status

```text
                  heartbeat
OFFLINE ─────────────────────────► ONLINE
   ▲                                  │
   │                                  │
   └──── no heartbeat for > 30s ──────┘
```

The status monitor runs every 5 seconds. The device is considered offline when:

```text
current_time - last_seen > 30 seconds
```

A normal heartbeat that keeps an already-online device online does not create a status-change notification.

## WebSocket

WebSocket endpoint:

```text
ws://localhost:8080/api/v1/ws
```

Status events use the `DEVICE_STATUS_CHANGED` event type.

Example payload:

```json
{
  "type": "DEVICE_STATUS_CHANGED",
  "device_id": "DMS-003",
  "status": "OFFLINE",
  "device": {
    "device_id": "DMS-003",
    "device_name": "Office PC 003",
    "status": "OFFLINE"
  }
}
```

The dashboard uses these events to update device state and monitoring statistics in realtime.

## API Reference

Base URL:

```text
http://localhost:8080/api/v1
```

### Authentication

```http
POST /auth/login
```

### Device Management

```http
POST   /devices
GET    /devices
GET    /devices/:device_id
PUT    /devices/:device_id
DELETE /devices/:device_id
```

Protected endpoints require:

```http
Authorization: Bearer <JWT>
```

### Heartbeat

```http
POST /devices/:device_id/heartbeat
```

### Reports

```http
GET /reports/summary
GET /reports/devices/export
```

The report endpoints provide monitoring statistics and CSV export functionality.

## Demo

A video demonstration is available here:

```text
https://shorturl.at/7wM7z
```

## Security Notes

- Environment-specific configuration must stay outside version control.
- JWT secrets must be replaced for each environment.
- Development credentials are examples only.
- If a secret has previously been committed, rotate it and remove the secret from repository history where appropriate.

## Current Status

This repository is a portfolio/demo project focused on realtime device monitoring and backend engineering practices.

- Automated backend unit tests are included for authentication and configuration behavior.
- GitHub Actions validates backend tests, `go vet`, and the frontend production build on pushes and pull requests to `main`.
- Docker Compose provides a reproducible local stack for PostgreSQL, backend, frontend, and simulator.
- Screenshots can be added later to make the portfolio presentation more visual.

## License

This project is licensed under the **MIT License**. See [LICENSE](LICENSE) for details.
