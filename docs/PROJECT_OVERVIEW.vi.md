# Tổng quan dự án `learn-go-microservices`

Repo này là một ví dụ platform e-commerce theo kiến trúc microservices viết bằng Go (Gin), chạy bằng Docker Compose. Các service giao tiếp theo 2 kiểu:

- HTTP sync: client → `gateway` → các core service
- Event-driven async: Kafka (domain events) và RabbitMQ (notification + delayed jobs)

## Cấu trúc thư mục (high-level)

- `auth-service/`: đăng ký/đăng nhập/JWT + verify email, phát event qua RabbitMQ
- `product-service/`: CRUD/search sản phẩm, publish event qua Kafka, consume event order để trừ/hoàn kho
- `order-service/`: tạo/thu tiền/hủy đơn, idempotency bằng Redis lock, delayed message RabbitMQ nhắc thanh toán + auto-cancel, publish event order qua Kafka
- `mail-service/`: consumer RabbitMQ để gửi mail (dùng MailHog cho dev) và gọi ngược Order service qua HTTP (feign client)
- `gateway/`: API Gateway reverse proxy + middleware auth; route theo prefix `/auth`, `/products`, `/orders`, `/mail`
- `pkg/`: thư viện dùng chung (config, constant, database, dto, httperror, logger, middleware, mq, utils)
- `deployments/`: các file compose dựng hạ tầng + chạy các service
- `infra/`: cấu hình HAProxy cho Postgres/RabbitMQ
- `postman/`: collection + environment

## Luồng request chính

### 1) Client đi qua Gateway

Gateway định tuyến bằng reverse proxy:

- `ANY /auth/*` → Auth service (không yêu cầu auth)
- `ANY /products/*`, `ANY /orders/*`, `ANY /mail/*` → phải qua `AuthMiddleware`

Gateway proxy còn “forward” thông tin user xuống service bằng header nội bộ (ví dụ `X-USER-ID`, `X-EMAIL`).

### 2) Pattern trong từng service

Hầu hết service có layout tương tự:

- `cmd/api/main.go` gọi `cmd/workers.Start()`
- `cmd/workers/root.go` dùng Cobra chạy lệnh `serve-all`
- `internal/config`: đọc `.env` tại working directory bằng Viper
- `internal/server/httpserver.go`: khởi tạo Gin, register middleware/validator, bootstrap route
- `internal/provider`: “wire” dependency (db, redis, kafka, rabbitmq, usecase, controller)
- `internal/controller`: Gin handlers + route grouping
- `internal/repository` + `internal/usecase`: data access + business logic

## Event-driven (Kafka + RabbitMQ)

### Kafka (domain events)

Kafka được dùng cho các event nghiệp vụ kiểu “state change” để service khác phản ứng:

- Product service publish: `product-created`, `product-updated`, `product-deleted`
- Order service publish: `order-created`, `order-cancelled`
- Order service consume product events để sync dữ liệu product phục vụ order
- Product service consume order events để cập nhật tồn kho (reserve/restock)

### RabbitMQ (notifications + delayed jobs)

RabbitMQ được dùng cho:

- Notification: verify email, account verified, order cancel/success… (exchange kiểu `topic`)
- Delayed message: nhắc thanh toán (4h/12h/22h) và auto-cancel sau 24h

Trong `deployments/compose-rabbitmq.yaml`, cluster RabbitMQ bật plugin `rabbitmq_delayed_message_exchange` (x-delayed-message) để support header `x-delay`.

Order service có các producer gửi delayed message (ví dụ `PaymentReminderProducer`, `AutoCancelProducer`) và consumer `AutoCancelConsumer` để tự động hủy đơn quá hạn.

## Data layer

### PostgreSQL HA + pgpool

`deployments/compose-pgpool.yaml` dựng 3 cụm Postgres (auth/product/order), mỗi cụm 3 node repmgr và một `pgpool-*` đứng trước.

Port pgpool expose ra host:

- Auth DB: `localhost:5000`
- Product DB: `localhost:5001`
- Order DB: `localhost:5002`

Root `Makefile` có target `migrateup` chạy migration vào từng DB qua các port này.

### Redis cluster

`deployments/compose-redis.yaml` dựng Redis Cluster (nhiều node) trong network `production`. Order service dùng Redis cluster + `redislock` để:

- Idempotency / chống double-submit (lock theo request)
- Các thao tác cần “distributed lock”

Ví dụ config Redis nằm ở `order-service/internal/config/redis.go` và được map từ `.env`.

## Cách chạy (dev)

### Yêu cầu

- Docker + Docker Compose
- Go (repo đang target Go `>= 1.23.x`)
- `migrate` CLI (được gọi trong `Makefile` root, dùng để chạy DB migrations)

### Chạy tất cả bằng Makefile

Từ root repo:

```bash
make
```

Lệnh này sẽ:

1. Build các service
2. Dựng hạ tầng (pgpool/redis/kafka/rabbitmq)
3. Chạy migrations cho auth/product/order
4. Chạy `compose-api.yaml` để lên `gateway` + các service

Dừng:

```bash
make compose-down
```

### Kiểm tra nhanh

- Gateway: `http://localhost:8000/health`
- MailHog UI: `http://localhost:8025`
- RabbitMQ mgmt qua HAProxy: `http://localhost:15872`

## Config (.env)

Mỗi service đọc file `.env` ngay tại working directory của nó (Viper `SetConfigName(".env")`). Khi chạy qua Dockerfile/compose, `.env` thường được copy vào image hoặc mount (tùy Dockerfile của từng service).

Nếu bạn muốn mình viết thêm một mục “mẫu `.env` tối thiểu cho từng service”, nói mình biết bạn đang chạy theo Docker Compose hay chạy local (go run) để mình đi đúng format.
