# Advanced Accommodation Search Filters – Accommodation Service

Spring Boot + PostgreSQL service for accommodation search.

## Prerequisites

- Java 21
- Maven 3.9+
- Docker (for Postgres local dev)

## Local database

Example (Docker):

```bash
docker run --rm -p 5432:5432 \
  -e POSTGRES_DB=accommodation \
  -e POSTGRES_USER=accommodation \
  -e POSTGRES_PASSWORD=accommodation \
  postgres:16
```

## Run

```bash
mvn test
mvn spring-boot:run
```
