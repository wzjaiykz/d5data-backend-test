# d5data-backend-test

[![CI](https://github.com/wzjaiykz/d5data-backend-test/actions/workflows/ci.yml/badge.svg)](https://github.com/wzjaiykz/d5data-backend-test/actions/workflows/ci.yml)

Spring Boot 3.2.0 written test: a `Hello World` endpoint protected by Spring
Security, plus a username/password login endpoint that returns a JWT.

## Requirements

- Java 17
- Maven 3.9+

## Run

```bash
mvn spring-boot:run
# or
mvn package && java -jar target/demo-0.0.1-SNAPSHOT.jar
```

The app listens on `http://localhost:8080`.

## Endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/login` | public | `{"username":"test","password":"123456"}` → `{"token":"...","tokenType":"Bearer","expiresIn":1800}` |
| GET | `/api/hello` | Bearer token | Returns `{"message":"Hello World"}` |

Credentials for the test: **test / 123456** (in-memory user, BCrypt-hashed at startup).

## Try it

```bash
# 1. Unauthenticated request is rejected
curl -i http://localhost:8080/api/hello
# HTTP/1.1 401

# 2. Log in and capture the token
TOKEN=$(curl -s -X POST http://localhost:8080/api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"test","password":"123456"}' | jq -r .token)

# 3. Call the protected endpoint
curl -s http://localhost:8080/api/hello -H "Authorization: Bearer $TOKEN"
# {"message":"Hello World"}

# 4. A wrong password is rejected
curl -i -X POST http://localhost:8080/api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"test","password":"wrong"}'
# HTTP/1.1 401
```

## How it works

- `SecurityConfig` — stateless filter chain: `/api/login` is public, everything
  else requires authentication; API clients receive **401 instead of a redirect**
  to a login page, which is what a JSON client expects.
- `JwtAuthFilter` — reads `Authorization: Bearer <token>`, verifies it and fills
  the security context when valid; invalid tokens simply stay anonymous.
- `JwtService` — issues and verifies HMAC-signed tokens (30 min TTL).
- `AuthController` — delegates the credential check to Spring Security's
  `AuthenticationManager` instead of comparing passwords by hand, so the BCrypt
  encoder and user store stay the single source of truth.

## Tests

```bash
mvn test
```

Five MockMvc tests cover: unauthenticated access, successful login, wrong
password, valid token access, and a tampered token.

## Known limitations

- User store is in memory; no database or registration flow (out of scope for
  the task).
- No refresh token, rate limiting or account lockout.
- The JWT secret lives in `application.yml` for the test; in production it would
  come from the environment.
