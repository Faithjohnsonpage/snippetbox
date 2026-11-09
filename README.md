# Snippetbox

A web application for creating, storing and viewing short text snippets (think a minimal, self-expiring pastebin), built in Go while working through Alex Edwards' *Let's Go*.

The goal of the project is to practise the building blocks of a production-style Go web service using mostly the standard library: routing, middleware, persistence, sessions, authentication, structured logging and secure defaults.

## Features

- Create and view text snippets with configurable expiry (1, 7 or 365 days)
- User signup, login and logout with bcrypt-hashed passwords
- Server-side sessions stored in MySQL, with flash messages
- Form decoding and validation with reusable validator helpers
- CSRF protection on all state-changing routes
- Structured, levelled logging with `log/slog`
- HTTPS-only server with a hardened TLS configuration
- Centralised error handling and panic recovery
- Unit, handler, end-to-end and integration tests (see [Testing](#testing))

## Tech stack

| Concern | Choice |
|---|---|
| Language | Go (1.22+ for method and wildcard routing in `net/http`) |
| Router | Standard library `http.ServeMux` |
| Middleware chaining | `justinas/alice` |
| Database | MySQL via `go-sql-driver/mysql` |
| Sessions | `alexedwards/scs/v2` with the MySQL store |
| CSRF | `justinas/nosurf` |
| Form decoding | `go-playground/form/v4` |
| Password hashing | `golang.org/x/crypto/bcrypt` |
| Logging | `log/slog` |

## Project layout

```
.
├── cmd/web/            # Application entry point, handlers, routes, middleware, helpers
│   ├── main.go
│   ├── handlers.go
│   ├── helpers.go
│   ├── middleware.go
│   ├── routes.go
│   ├── context.go
│   └── *_test.go       # Handler, middleware and helper tests
├── internal/
│   ├── models/         # SnippetModel, UserModel and shared model errors
│   │   └── mocks/      # In-memory model stand-ins for handler tests
│   └── validator/      # Form validation helpers
├── ui/                 # HTML templates and static assets
├── tls/                # Local development certificate and key (not committed)
└── go.mod
```

## Getting started

### Prerequisites

- Go 1.22 or newer
- MySQL 8.x
- OpenSSL or Go's `generate_cert.go` for a local TLS certificate

### 1. Clone and install dependencies

```bash
git clone <your-repo-url>
cd snippetbox
go mod download
```

### 2. Set up the database

```sql
CREATE DATABASE snippetbox CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE snippetbox;

CREATE TABLE snippets (
    id INTEGER NOT NULL PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(100) NOT NULL,
    content TEXT NOT NULL,
    created DATETIME NOT NULL,
    expires DATETIME NOT NULL
);
CREATE INDEX idx_snippets_created ON snippets(created);

CREATE TABLE users (
    id INTEGER NOT NULL PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    hashed_password CHAR(60) NOT NULL,
    created DATETIME NOT NULL
);
ALTER TABLE users ADD CONSTRAINT users_uc_email UNIQUE (email);

CREATE TABLE sessions (
    token CHAR(43) PRIMARY KEY,
    data BLOB NOT NULL,
    expiry TIMESTAMP(6) NOT NULL
);
CREATE INDEX sessions_expiry_idx ON sessions (expiry);
```

Create a dedicated, least-privilege database user for the app rather than connecting as root:

```sql
CREATE USER 'web'@'localhost' IDENTIFIED BY '<password>';
GRANT SELECT, INSERT, UPDATE, DELETE ON snippetbox.* TO 'web'@'localhost';
```

### 3. Generate a local TLS certificate

```bash
mkdir tls && cd tls
go run $(go env GOROOT)/src/crypto/tls/generate_cert.go --rsa-bits=2048 --host=localhost
cd ..
```

This produces a self-signed `cert.pem` and `key.pem`. Browsers will warn about it; that is expected for local development.

### 4. Run the application

```bash
go run ./cmd/web -addr=":4000" -dsn="web:<password>@/snippetbox?parseTime=true"
```

Then open <https://localhost:4000>.

> Flag names and defaults may differ slightly depending on how you configured `main.go`. Run `go run ./cmd/web -help` to see what your build accepts.

## Routes

| Method | Path | Auth required | Description |
|---|---|---|---|
| GET | `/{$}` | No | Home page, latest snippets |
| GET | `/snippet/view/{id}` | No | View a single snippet |
| GET | `/snippet/create` | Yes | Snippet creation form |
| POST | `/snippet/create` | Yes | Create a snippet |
| GET | `/user/signup` | No | Signup form |
| POST | `/user/signup` | No | Register a new user |
| GET | `/user/login` | No | Login form |
| POST | `/user/login` | No | Authenticate a user |
| POST | `/user/logout` | Yes | Log out |
| GET | `/static/` | No | Static assets |

## How it works

### Middleware chain

Routes are split into two groups. Static files bypass all middleware. Dynamic routes go through a per-route chain composed with `alice`:

```go
dynamic := alice.New(app.sessionManager.LoadAndSave, noSurf, app.authenticate)
protected := dynamic.Append(app.requireAuthentication)
```

The whole mux is then wrapped in the standard chain: panic recovery, request logging and secure response headers.

### Sessions

Sessions are managed by `scs` and persisted in the MySQL `sessions` table, so they survive server restarts and work across multiple instances. Key points:

- The `LoadAndSave` middleware loads session data into `r.Context()` and saves it after the handler returns.
- Flash messages are written with `Put` and read once with `PopString`.
- The session token is renewed with `RenewToken` on login and logout to prevent session fixation.
- Cookies are `HttpOnly`, `Secure` and `SameSite=Lax`.
- `Lifetime` and `IdleTimeout` are configured on the session manager in `main.go`.

### Authentication

On successful login the user's ID is stored in the session under `authenticatedUserID`. An `authenticate` middleware then checks once per request that the user still exists in the database and stores the result in the request context under a custom, unexported key type. Handlers and the `requireAuthentication` middleware read that cached value instead of querying the database repeatedly. Deleted users are therefore locked out on their next request.

### Logging

The application uses `log/slog` with a text handler writing to stdout. A single `*slog.Logger` is created in `main.go` and injected into the `application` struct, so every handler and helper logs through the same instance. Request details (remote address, protocol, method, URI) are logged by middleware, and server errors are logged with the request method and URI plus a stack trace where helpful.

### Error handling

Helpers in `helpers.go` (`serverError`, `clientError`, `notFound`) centralise error responses. Panics in handlers are recovered by middleware, which closes the connection and returns a 500 rather than crashing the server.

### CSRF protection

All dynamic routes run through `nosurf`, which sets a `csrf_token` cookie (`HttpOnly`, `Secure`, `Path=/`) and rejects unsafe-method requests that do not carry a matching token, either as a `csrf_token` form field or an `X-CSRF-Token` header.

### Security headers

Every response sets `Content-Security-Policy`, `Referrer-Policy`, `X-Content-Type-Options`, `X-Frame-Options` and `X-XSS-Protection`.

### Dependency injection

Handlers are methods on an `application` struct that holds the logger, models, session manager and form decoder. Dependencies are constructed once in `main()` and injected through the struct, which keeps handlers easy to test and avoids global state.

## Testing the API with curl

Because the server is HTTPS-only with a self-signed certificate, use `-k`, and keep every request on `https://` so the `Secure` session cookie is sent back.

With CSRF protection enabled, fetch a token first and send it with each POST. The token is embedded in the rendered forms, so either read it from the page source or expose it through a small JSON endpoint. Then:

```bash
# Signup
curl -ik -c cookies.txt -b cookies.txt -X POST https://localhost:4000/user/signup \
  -H "X-CSRF-Token: <token>" \
  -d "name=Faith" -d "email=faith@example.com" -d "password=supersecure123"

# Login
curl -ik -c cookies.txt -b cookies.txt -X POST https://localhost:4000/user/login \
  -H "X-CSRF-Token: <token>" \
  -d "email=faith@example.com" -d "password=supersecure123"

# Create a snippet (requires login)
curl -ik -c cookies.txt -b cookies.txt -X POST https://localhost:4000/snippet/create \
  -H "X-CSRF-Token: <token>" \
  -d "title=Test Snippet" -d "content=Some content" -d "expires=365"

# Logout
curl -ik -c cookies.txt -b cookies.txt -X POST https://localhost:4000/user/logout \
  -H "X-CSRF-Token: <token>"
```

## Testing

The project uses Go's built-in `testing` package plus `net/http/httptest`, with no external test framework.

### Running the tests

```bash
# Run everything
go test ./...

# Verbose output
go test -v ./...

# One package, or one test by name
go test -v ./cmd/web
go test -v -run TestSnippetCreate ./cmd/web

# Race detector (recommended: sessions and handlers share state)
go test -race ./...

# Coverage
go test -cover ./...
go test -coverprofile=coverage.out ./... && go tool cover -html=coverage.out

# Skip slower integration tests
go test -short ./...
```

### Test types

| Kind | Where | What it covers |
|---|---|---|
| Unit tests | `cmd/web`, `internal/validator` | Helpers and pure functions, e.g. `humanDate()` and validator rules |
| Table-driven tests | throughout | Many inputs and expected outputs per test, run as sub-tests with `t.Run` |
| HTTP handler tests | `cmd/web` | Handlers called with `httptest.NewRecorder` and `httptest.NewRequest` |
| Middleware tests | `cmd/web` | Security headers, panic recovery and auth checks, using a stub `next` handler |
| End-to-end tests | `cmd/web` | A full app served through `httptest.NewTLSServer`, including session cookies and CSRF tokens |
| Integration tests | `internal/models` | Models against a real MySQL test database |

### Test helpers and mocks

- **`testutils_test.go`**: shared helpers, such as `newTestApplication` (builds an `application` with a discard logger, mock models and a real session manager) and `newTestServer` (wraps `httptest.NewTLSServer` with a cookie jar and disabled redirect-following).
- **Mock models**: `internal/models/mocks` provides in-memory stand-ins for `SnippetModel` and `UserModel`, so handler tests never need a database. This relies on the models being behind interfaces in the `application` struct.
- **Discard logger**: tests use a `slog` handler that writes to `io.Discard`, which keeps test output clean.
- **CSRF in tests**: end-to-end tests first GET a page, extract the `csrf_token` from the response, then send it back with the POST.

### Integration test database

Model tests run against a separate database so they never touch development data.

```sql
CREATE DATABASE test_snippetbox CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'test_web'@'localhost' IDENTIFIED BY '<password>';
GRANT CREATE, DROP, ALTER, INDEX, SELECT, INSERT, UPDATE, DELETE ON test_snippetbox.* TO 'test_web'@'localhost';
```

Each integration test runs a setup script that creates the tables and inserts known fixtures, and a teardown script (registered with `t.Cleanup`) that drops everything afterwards. These tests call `t.Skip` under `-short`.

## Roadmap

- [ ] Raise test coverage and add CI to run `go test -race ./...` on every push
- [ ] Graceful shutdown and explicit server timeouts
- [ ] Configuration via environment variables
- [ ] Dockerfile and docker-compose setup
- [ ] JSON API variant for use with a separate frontend

## Acknowledgements

Built while following [*Let's Go*](https://lets-go.alexedwards.net/) by Alex Edwards. The structure and patterns come from the book; the implementation and any deviations are my own.
