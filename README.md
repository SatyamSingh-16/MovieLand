# 🎬 MovieLand

A production-ready RESTful JSON API for managing a movie catalogue, built with **Go**. MovieLand features user registration and activation, token-based authentication, permission-based authorization, rate limiting, CORS support, graceful shutdown, and automatic database migrations.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [API Endpoints](#api-endpoints)
- [Authentication](#authentication)
- [Permissions](#permissions)
- [Filtering, Sorting & Pagination](#filtering-sorting--pagination)
- [Rate Limiting](#rate-limiting)
- [Database Migrations](#database-migrations)
- [Graceful Shutdown](#graceful-shutdown)

---

## Features

- **Full CRUD** for movies (create, read, update, delete)
- **User registration** with email activation workflow
- **Token-based authentication** using Bearer tokens (stateful, DB-backed)
- **Permission-based authorization** (`movies:read`, `movies:write`)
- **Input validation** with detailed error responses
- **Full-text search** on movie titles (PostgreSQL `tsvector`)
- **Filtering & sorting** with pagination metadata
- **Per-client IP rate limiting** with automatic cleanup
- **CORS** with configurable trusted origins
- **Optimistic concurrency control** (versioned updates to prevent data races)
- **Automatic database migrations** on startup
- **Graceful shutdown** handling `SIGINT` / `SIGTERM`
- **Application metrics** exposed via `expvar` (`/debug/vars`)
- **Structured logging** with `log/slog`
- **SMTP email integration** for user activation tokens

---

## Tech Stack

| Layer           | Technology                                                                 |
| --------------- | -------------------------------------------------------------------------- |
| Language        | Go 1.26+                                                                   |
| Router          | [httprouter](https://github.com/julienschmidt/httprouter)                  |
| Database        | PostgreSQL                                                                 |
| Driver          | [lib/pq](https://github.com/lib/pq)                                       |
| Migrations      | [golang-migrate](https://github.com/golang-migrate/migrate)               |
| Rate Limiting   | [golang.org/x/time/rate](https://pkg.go.dev/golang.org/x/time/rate)       |
| Password Hashing| [golang.org/x/crypto/bcrypt](https://pkg.go.dev/golang.org/x/crypto/bcrypt)|
| Email           | [go-mail](https://github.com/wneessen/go-mail)                            |
| Real IP         | [tomasen/realip](https://github.com/tomasen/realip)                       |

---

## Project Structure

```
MovieLand/
├── cmd/
│   ├── api/                    # Application entry point & HTTP layer
│   │   ├── main.go             # Bootstrap, config, DB connection, migrations
│   │   ├── server.go           # HTTP server with graceful shutdown
│   │   ├── routes.go           # Route definitions & middleware chain
│   │   ├── movies.go           # Movie CRUD handlers
│   │   ├── users.go            # User registration & activation handlers
│   │   ├── tokens.go           # Authentication token handler
│   │   ├── middleware.go       # Rate limiter, auth, CORS, panic recovery, metrics
│   │   ├── errors.go           # Centralized error response helpers
│   │   ├── helpers.go          # JSON read/write, query param parsing
│   │   ├── healthcheck.go      # Health check endpoint
│   │   └── context.go          # Request-scoped user context helpers
│   └── examples/
│       └── cors/               # CORS testing example
├── internal/
│   ├── data/                   # Data models & database access layer
│   │   ├── models.go           # Model registry
│   │   ├── movies.go           # Movie model & CRUD queries
│   │   ├── users.go            # User model, password hashing, queries
│   │   ├── tokens.go           # Token generation & validation
│   │   ├── permissions.go      # Permission model & user-permission queries
│   │   ├── runtime.go          # Custom Runtime type (e.g. "102 mins")
│   │   └── filter.go           # Pagination, sorting & metadata helpers
│   ├── mailer/                 # SMTP email sending
│   │   ├── mailer.go           # Mailer setup
│   │   └── templates/          # Email templates
│   └── validator/
│       └── validator.go        # Reusable validation helpers
├── migrations/                 # SQL migration files (up & down)
│   ├── 000001_create_movies_table
│   ├── 000002_add_movies_check_constraints
│   ├── 000003_add_movies_indexes
│   ├── 000004_create_users_table
│   ├── 000005_create_tokens_table
│   └── 000006_add_permissions
├── go.mod
└── go.sum
```

---

## Prerequisites

- **Go** 1.26 or later
- **PostgreSQL** 13 or later (with the `citext` extension enabled)
- An **SMTP server** for sending activation emails (e.g. [Mailtrap](https://mailtrap.io) for development)

---

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/your-username/MovieLand.git
cd MovieLand
```

### 2. Set up the database

```sql
-- Create the database
CREATE DATABASE movieland;

-- Connect to the database and enable citext
\c movieland
CREATE EXTENSION IF NOT EXISTS citext;
```

### 3. Set the environment variable

```bash
export MOVIELAND_DB_DSN='postgres://username:password@localhost/movieland?sslmode=disable'
```

### 4. Run the application

```bash
go run ./cmd/api
```

The server starts on **port 4000** by default. Migrations are applied automatically on startup.

### 5. Verify it's running

```bash
curl localhost:4000/v1/healthcheck
```

Expected response:

```json
{
  "status": "available",
  "system_info": {
    "environment": "development",
    "version": "1.0.0"
  }
}
```

---

## Configuration

All settings are configurable via command-line flags:

| Flag                    | Default                          | Description                                 |
| ----------------------- | -------------------------------- | ------------------------------------------- |
| `-port`                 | `4000`                           | API server port                             |
| `-env`                  | `development`                    | Environment (`development\|staging\|production`) |
| `-db-dsn`               | `$MOVIELAND_DB_DSN`              | PostgreSQL connection string                |
| `-db-max-open-conns`    | `25`                             | Max open DB connections                     |
| `-db-max-idle-conns`    | `25`                             | Max idle DB connections                     |
| `-db-max-idle-time`     | `15m`                            | Max idle time per connection                |
| `-limiter-rps`          | `2`                              | Rate limiter — requests per second          |
| `-limiter-burst`        | `4`                              | Rate limiter — max burst size               |
| `-limiter-enabled`      | `true`                           | Enable/disable rate limiter                 |
| `-smtp-host`            | `sandbox.smtp.mailtrap.io`       | SMTP server host                            |
| `-smtp-port`            | `25`                             | SMTP server port                            |
| `-smtp-username`        | `""`                             | SMTP username                               |
| `-smtp-password`        | `""`                             | SMTP password                               |
| `-smtp-sender`          | `Greenlight <no-reply@...>`      | SMTP sender address                         |
| `-cors-trusted-origins` | `""`                             | Trusted CORS origins (space-separated)      |

**Example:**

```bash
go run ./cmd/api \
  -port=8080 \
  -env=production \
  -db-dsn='postgres://user:pass@localhost/movieland?sslmode=disable' \
  -limiter-rps=10 \
  -limiter-burst=20 \
  -smtp-username=your_username \
  -smtp-password=your_password \
  -cors-trusted-origins="https://example.com https://app.example.com"
```

---

## API Endpoints

### Health Check

| Method | URL                 | Description              | Auth Required |
| ------ | ------------------- | ------------------------ | ------------- |
| `GET`  | `/v1/healthcheck`   | Show API status & version| No            |

### Movies

| Method   | URL                | Description        | Permission Required |
| -------- | ------------------ | ------------------ | ------------------- |
| `GET`    | `/v1/movies`       | List all movies    | `movies:read`       |
| `POST`   | `/v1/movies`       | Create a movie     | `movies:write`      |
| `GET`    | `/v1/movies/:id`   | Show a movie       | `movies:read`       |
| `PATCH`  | `/v1/movies/:id`   | Update a movie     | `movies:write`      |
| `DELETE` | `/v1/movies/:id`   | Delete a movie     | `movies:write`      |

### Users

| Method | URL                     | Description          | Auth Required |
| ------ | ----------------------- | -------------------- | ------------- |
| `POST` | `/v1/users`             | Register a new user  | No            |
| `PUT`  | `/v1/users/activated`   | Activate a user      | No            |

### Tokens

| Method | URL                            | Description                    | Auth Required |
| ------ | ------------------------------ | ------------------------------ | ------------- |
| `POST` | `/v1/tokens/authentication`    | Create an authentication token | No            |

### Debug

| Method | URL            | Description               | Auth Required |
| ------ | -------------- | ------------------------- | ------------- |
| `GET`  | `/debug/vars`  | Application metrics       | No            |

---

## Authentication

MovieLand uses **stateful token-based authentication**. Tokens are stored as SHA-256 hashes in the database.

### 1. Register a user

```bash
curl -X POST localhost:4000/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com", "password": "pa55word123"}'
```

### 2. Activate the user

After receiving the activation token (via email), activate the account:

```bash
curl -X PUT localhost:4000/v1/users/activated \
  -H "Content-Type: application/json" \
  -d '{"token": "ACTIVATION_TOKEN_HERE"}'
```

### 3. Obtain an authentication token

```bash
curl -X POST localhost:4000/v1/tokens/authentication \
  -H "Content-Type: application/json" \
  -d '{"email": "alice@example.com", "password": "pa55word123"}'
```

### 4. Use the token

Include the token in the `Authorization` header for protected endpoints:

```bash
curl localhost:4000/v1/movies \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

---

## Permissions

Users are assigned permissions upon registration:

| Permission      | Grants Access To                    |
| --------------- | ----------------------------------- |
| `movies:read`   | `GET /v1/movies`, `GET /v1/movies/:id` |
| `movies:write`  | `POST`, `PATCH`, `DELETE` on `/v1/movies` |

New users automatically receive the `movies:read` permission. The `movies:write` permission must be granted manually via the database.

---

## Filtering, Sorting & Pagination

The `GET /v1/movies` endpoint supports query parameters:

| Parameter   | Example                     | Description                          |
| ----------- | --------------------------- | ------------------------------------ |
| `title`     | `?title=godfather`          | Full-text search on title            |
| `genres`    | `?genres=drama,crime`       | Filter by genre(s)                   |
| `sort`      | `?sort=-year`               | Sort field (prefix `-` for descending) |
| `page`      | `?page=2`                   | Page number (default: 1)             |
| `page_size` | `?page_size=10`             | Results per page (default: 20)       |

**Sortable fields:** `id`, `title`, `year`, `runtime` (and their descending variants with `-` prefix).

**Example:**

```bash
curl "localhost:4000/v1/movies?title=batman&sort=-year&page=1&page_size=5" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

---

## Rate Limiting

- **Per-client IP** rate limiting using a token bucket algorithm
- Default: **2 requests/second** with a burst of **4**
- Stale client entries are automatically cleaned up after **3 minutes** of inactivity
- Can be disabled with `-limiter-enabled=false`

When the rate limit is exceeded, the API responds with `429 Too Many Requests`.

---

## Database Migrations

Migrations are stored in the `migrations/` directory and are **applied automatically** when the application starts. The project uses [golang-migrate](https://github.com/golang-migrate/migrate) with the following migrations:

| #   | Migration                         | Description                           |
| --- | --------------------------------- | ------------------------------------- |
| 1   | `create_movies_table`             | Movies table with core fields         |
| 2   | `add_movies_check_constraints`    | Validation constraints on movies      |
| 3   | `add_movies_indexes`              | GIN index for full-text search        |
| 4   | `create_users_table`              | Users table with `citext` email       |
| 5   | `create_tokens_table`             | Tokens table for authentication       |
| 6   | `add_permissions`                 | Permissions & user-permissions tables  |

---

## Graceful Shutdown

The server listens for `SIGINT` and `SIGTERM` signals. On receipt:

1. Stops accepting new connections
2. Waits for in-flight requests to complete (up to 30s timeout)
3. Waits for background tasks (e.g. email sending) to finish
4. Exits cleanly

---

## License

This project is part of [GoLang-Journey](https://github.com/your-username/GoLang-Journey) — a learning journey through Go.
