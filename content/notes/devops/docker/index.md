---
title: Docker
weight: 210
menu:
  notes:
    name: Docker
    identifier: notes-docker
    parent: notes-devops
    weight: 10
---

<!-- Simple -->
{{< note title="Simple" >}}

```dockerfile
FROM golang:1.24-alpine

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/server

# Expose the port
EXPOSE 8080

# Command to run the application
CMD ["./server"]
```

{{< /note >}}

<!-- Multi-Stage -->
{{< note title="Multi-Stage" >}}

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/server

FROM alpine:3.19

RUN apk --no-cache add ca-certificates bash

WORKDIR /app

COPY --from=builder /app/server .

RUN mkdir -p /app/configs
COPY configs/config.yaml /app/configs/config.yaml

EXPOSE 8080

CMD ["./server"]
```

{{< /note >}}

<!-- Production -->
{{< note title="Production" >}}

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags="-s -w" -o /app/server ./cmd/server

FROM alpine:3.19

RUN apk --no-cache add ca-certificates

RUN adduser -D appuser

WORKDIR /app

COPY --from=builder /app/server .

RUN mkdir -p /app/configs
COPY configs/config.yaml /app/configs/config.yaml

RUN chown -R appuser:appuser /app

EXPOSE 80

CMD ["./server"]
```

{{< /note >}}

<!-- Docker Compose -->
{{< note title="Docker Compose" >}}

```yaml
version: '3.8'

services:
  app-service:
    build:
      context: .
      dockerfile: Dockerfile.production
    container_name: app-service
    ports:
      - "8080:8080"
    environment:
      - SERVICE_NAME=app-service
      - DB_HOST=database
      - DB_USER=user
      - DB_PASSWORD=password
      - DB_NAME=mydb
      - REDIS_HOST=cache
    networks:
      - app-network
    restart: always
    depends_on:
      - database
      - cache
    healthcheck:
      test: [ "CMD", "curl", "-f", "http://localhost:8080/health" ]
      interval: 30s
      timeout: 10s
      retries: 3

  database:
    image: postgres:15-alpine
    container_name: app-database
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network
    restart: always
    healthcheck:
      test: [ "CMD", "pg_isready", "-U", "user" ]
      interval: 30s
      timeout: 10s
      retries: 5

  cache:
    image: redis:7-alpine
    container_name: app-redis
    volumes:
      - redis-data:/data
    networks:
      - app-network
    restart: always
    healthcheck:
      test: [ "CMD", "redis-cli", "ping" ]
      interval: 30s
      timeout: 10s
      retries: 5

  prometheus:
    image: prom/prometheus:v2.44.0
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - "9090:9090"
    networks:
      - app-network
    restart: always

  grafana:
    image: grafana/grafana:9.5.1
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - app-network
    restart: always

volumes:
  postgres-data:
  redis-data:
  grafana-data:

networks:
  app-network:
    driver: bridge
```

{{< /note >}}