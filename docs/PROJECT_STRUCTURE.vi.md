# Giải thích cấu trúc dự án chi tiết

## Tổng quan thư mục gốc

```
learn-go-microservices/
├── auth-service/          # Service quản lý user: đăng ký, login, verify email
├── product-service/       # Service quản lý sản phẩm: CRUD, sync inventory
├── order-service/         # Service quản lý đơn hàng: tạo, thanh toán, hủy
├── mail-service/          # Service gửi email notification
├── gateway/               # API Gateway (reverse proxy + auth middleware)
├── pkg/                   # Shared libraries dùng chung giữa các service
├── deployments/           # Docker Compose files cho infra và services
├── infra/                 # Config HAProxy cho PostgreSQL, RabbitMQ
├── postman/               # Postman collection để test API
├── Makefile               # Automation: build, migrate, compose up/down
└── README.md
```

---

## Chi tiết từng service (auth/product/order có cấu trúc tương tự)

### 1. `auth-service/`

**Vai trò**: Quản lý authentication & authorization. Xử lý đăng ký, login, email verification.

#### Cấu trúc thư mục

```
auth-service/
├── cmd/
│   ├── api/
│   │   └── main.go                 # Entry point: gọi cmd/workers.Start()
│   └── workers/
│       ├── root.go                  # Cobra CLI: khởi tạo config, logger, chạy HTTP/AMQP workers
│       ├── http.go                  # HTTP server worker
│       └── amqp.go                  # RabbitMQ consumer worker
├── db/
│   └── migrations/
│       ├── 000001_create_table_users.up.sql       # Migration tạo bảng users
│       ├── 000001_create_table_users.down.sql
│       ├── 000002_create_table_verification.up.sql  # Bảng verification (token, expiry)
│       └── 000002_create_table_verification.down.sql
├── internal/
│   ├── config/
│   │   ├── config.go                # Đọc .env bằng Viper, trả về struct Config
│   │   ├── app.go                   # AppConfig: environment, bcrypt cost
│   │   ├── httpserver.go            # HttpServerConfig: host, port, timeout
│   │   ├── postgres.go              # PostgresConfig: connection string
│   │   ├── redis.go                 # RedisConfig: cluster addrs, password
│   │   ├── amqp.go                  # AMQPConfig: RabbitMQ URL
│   │   └── logger.go                # LoggerConfig: log level
│   ├── constant/
│   │   ├── amqp.go                  # Constants cho RabbitMQ: exchange, queue, key
│   │   ├── error.go                 # Error messages
│   │   ├── status.go                # User status constants
│   │   └── verification.go          # Verification constants (TTL, cooldown)
│   ├── controller/
│   │   ├── app_controller.go        # Health check, 404/405 handlers
│   │   └── user_controller.go       # HTTP handlers: Login, Register, Verify, ResendVerification
│   ├── dto/
│   │   ├── login_request.go         # Request/Response DTOs
│   │   ├── register_request.go
│   │   ├── verification_request.go
│   │   └── ...
│   ├── entity/
│   │   ├── user.go                  # Entity User (model DB)
│   │   └── verification.go          # Entity Verification
│   ├── httperror/
│   │   ├── invalid_credential_error.go  # Custom HTTP errors
│   │   ├── user_already_exist_error.go
│   │   └── user_not_verified_error.go
│   ├── log/
│   │   └── logger.go                # Wrapper logger (sử dụng pkg/logger)
│   ├── mq/
│   │   ├── send_verification_producer.go   # Publish "send verification email" event
│   │   └── account_verified_producer.go    # Publish "account verified" event
│   ├── provider/
│   │   ├── init.go                  # BootstrapGlobal: khởi tạo DB, Redis, RabbitMQ, dependencies
│   │   ├── http.go                  # BootstrapHttp: wire controllers vào router
│   │   ├── user.go                  # BootstrapUser: wire user repository/usecase/controller
│   │   └── amqp.go                  # BootstrapAMQP: return list AMQP consumers (nếu có)
│   ├── repository/
│   │   ├── datastore.go             # DataStore: wrapper cho transaction (Atomic)
│   │   ├── user_repository.go       # User CRUD với PostgreSQL
│   │   ├── verification_repository.go
│   │   └── redis_repository.go      # Redis operations (cache, rate limit)
│   ├── server/
│   │   └── httpserver.go            # HttpServer: NewHttpServer, Start, Shutdown
│   ├── usecase/
│   │   └── user_usecase.go          # Business logic: Login, Register, Verify, ResendVerification
│   └── utils/
│       └── tokenutils/
│           └── generator.go         # Generate random verification token
├── Dockerfile                       # Multi-stage build Go app
├── go.mod
└── Makefile                         # Local dev: build binary
```

#### Luồng hoạt động

1. **Register**:
    - Client POST `/auth/register` → Gateway → Auth Controller
    - Usecase tạo user (is_verified=false), tạo verification token (TTL 10 phút)
    - Publish event `send.verification` qua RabbitMQ → Mail Service gửi email
    - Return `{"message": "User registered, check your email"}`

2. **Verify**:
    - Client POST `/auth/verify` với `{token}`
    - Usecase check token hợp lệ & chưa hết hạn, set `is_verified=true`
    - Publish event `account.verified` → Mail Service gửi email chúc mừng
    - Return `{"message": "Account verified"}`

3. **Login**:
    - Client POST `/auth/login` với `{email, password}`
    - Usecase check credential, kiểm tra `is_verified`
    - Nếu ok, generate JWT token (chứa userID, email)
    - Return `{"access_token": "..."}`

4. **ResendVerification**:
    - Client POST `/auth/resend-verification` với `{email}`
    - Usecase check rate limit (Redis): 1 phút chỉ gửi 1 lần
    - Tạo token mới, publish event `send.verification`
    - Return `{"message": "Verification email sent"}`

---

### 2. `product-service/`

**Vai trò**: Quản lý sản phẩm (CRUD), sync inventory khi có order/cancel, publish Kafka event khi product thay đổi.

#### Cấu trúc (tương tự auth-service)

```
product-service/
├── cmd/api/main.go
├── cmd/workers/
│   ├── root.go              # Chạy HTTP worker + Kafka consumer workers
│   ├── http.go
│   └── kafka.go
├── db/migrations/
│   ├── 000001_create_table_products.up.sql
│   └── ...
├── internal/
│   ├── config/              # Config giống auth, thêm KafkaConfig
│   ├── constant/
│   │   └── kafka.go         # Topics: product-created, product-updated, product-deleted
│   ├── controller/
│   │   └── product_controller.go   # CRUD endpoints (protected by auth middleware)
│   ├── entity/
│   │   └── product.go       # Product: ID, Name, Description, Price, Quantity
│   ├── mq/
│   │   ├── product_created_producer.go      # Publish Kafka event
│   │   ├── product_updated_producer.go
│   │   ├── product_deleted_producer.go
│   │   ├── order_created_consumer.go        # Consumer: reserve stock khi order created
│   │   └── order_cancelled_consumer.go      # Consumer: restock khi order cancelled
│   ├── provider/
│   │   ├── init.go          # BootstrapGlobal: DB, KafkaAdmin
│   │   ├── product.go       # Wire product usecase/controller + create Kafka topics
│   │   └── kafka.go         # BootstrapKafka: return list Kafka consumers
│   ├── repository/
│   │   └── product_repository.go
│   ├── usecase/
│   │   └── product_usecase.go      # Create/Update/Delete/Search product
│   └── server/httpserver.go
├── Dockerfile
├── go.mod
└── Makefile
```

#### Luồng hoạt động

1. **Create Product** (POST `/products`):
    - Auth middleware xác thực token → extract userID
    - Controller gọi usecase → save product vào DB
    - Publish Kafka event `product-created` (async)
    - Order service consume event này để sync product metadata

2. **Update Product** (PUT `/products/:productId`):
    - Usecase update DB
    - Publish `product-updated` event
    - Order service update cached product info

3. **Delete Product** (DELETE `/products/:productId`):
    - Usecase soft delete hoặc hard delete
    - Publish `product-deleted` event

4. **Order Created Consumer**:
    - Consume Kafka event `order-created` (từ Order service)
    - Payload: `{order_id, items: [{product_id, quantity}, ...]}`
    - Usecase trừ quantity từng product (reserve stock)
    - Nếu không đủ hàng → log error (trong thực tế có thể retry hoặc cancel order)

5. **Order Cancelled Consumer**:
    - Consume `order-cancelled` event
    - Usecase cộng lại quantity (restock)

---

### 3. `order-service/`

**Vai trò**: Quản lý order lifecycle, sử dụng Redis lock (idempotency), publish Kafka event, schedule delayed RabbitMQ message (payment reminder, auto-cancel).

#### Cấu trúc (giống product, thêm Redis + nhiều AMQP producer)

```
order-service/
├── cmd/api/main.go
├── cmd/workers/
│   ├── root.go              # Chạy HTTP + Kafka consumer + AMQP consumer workers
│   ├── http.go
│   ├── kafka.go
│   └── amqp.go
├── db/migrations/
│   ├── 000001_create_table_products.up.sql   # Order service cũng lưu bảng products (sync từ Kafka)
│   ├── 000002_create_table_orders.up.sql
│   └── 000003_create_table_order_items.up.sql
├── internal/
│   ├── config/
│   │   ├── redis.go         # RedisConfig (cluster)
│   │   ├── kafka.go
│   │   └── amqp.go
│   ├── constant/
│   │   ├── kafka.go         # Topics: order-created, order-cancelled
│   │   ├── amqp.go          # Queues: payment-reminder, auto-cancel, cancel-notification, order-success
│   │   ├── order.go         # Order status: PENDING, PAID, CANCELLED, COMPLETED
│   │   └── delay.go         # Payment reminder delays: 4h, 12h, 22h
│   ├── controller/
│   │   └── order_controller.go    # POST /orders, GET /orders, POST /pay, POST /cancel
│   ├── entity/
│   │   ├── order.go         # Order: ID, RequestID, CustomerID, Status, TotalAmount, Items
│   │   ├── order_item.go
│   │   └── product.go       # Cached product từ Kafka
│   ├── mq/
│   │   ├── order_created_producer.go        # Kafka
│   │   ├── order_cancelled_producer.go
│   │   ├── payment_reminder_producer.go     # RabbitMQ delayed (x-delayed-message)
│   │   ├── auto_cancel_producer.go          # RabbitMQ delayed 24h
│   │   ├── cancel_notification_producer.go  # RabbitMQ
│   │   ├── order_success_producer.go        # RabbitMQ
│   │   ├── product_created_consumer.go      # Kafka consumer sync product
│   │   ├── product_updated_consumer.go
│   │   ├── product_deleted_consumer.go
│   │   └── auto_cancel_consumer.go          # AMQP consumer: tự động hủy order sau 24h
│   ├── middleware/
│   │   └── user_middleware.go               # Extract userID/email từ context (Gateway set)
│   ├── provider/
│   │   ├── init.go          # BootstrapGlobal: DB, Redis, RedisLock, RabbitMQ, KafkaAdmin
│   │   ├── order.go         # Wire order usecase (nhiều producer)
│   │   ├── kafka.go         # Return Kafka consumers (product events)
│   │   └── amqp.go          # Return AMQP consumers (auto-cancel)
│   ├── repository/
│   │   ├── order_repository.go
│   │   ├── order_item_repository.go
│   │   ├── product_repository.go    # Cached products
│   │   └── lock_repository.go       # Redis distributed lock
│   ├── usecase/
│   │   └── order_usecase.go
│   │       ├── Save()       # Tạo order: acquire lock, insert DB, publish events
│   │       ├── Pay()        # Cập nhật status=PAID, publish order-success
│   │       └── Cancel()     # Cập nhật status=CANCELLED, publish order-cancelled + cancel-notification
│   └── utils/
│       └── redisutils/
│           └── lock.go      # Helper tạo lock key
├── Dockerfile
├── go.mod
└── Makefile
```

#### Luồng hoạt động

1. **Create Order** (POST `/orders`):
    - Client gửi `{request_id, items: [{product_id, quantity}, ...]}`
    - Usecase:
        1. Acquire Redis lock theo `request_id` + `customer_id` (idempotency: tránh tạo 2 order giống nhau)
        2. Check order đã tồn tại chưa (theo request_id)
        3. Nếu chưa → tạo order (status=PENDING), tính total_amount
        4. Publish Kafka `order-created` event → Product service trừ kho
        5. Publish RabbitMQ delayed messages:
            - `payment_reminder` với delay 4h, 12h, 22h → Mail service gửi email nhắc
            - `auto_cancel` với delay 24h → Order service auto cancel nếu chưa pay
        6. Release lock
    - Return `{"order_id": ...}`

2. **Pay Order** (POST `/orders/pay`):
    - Client gửi `{order_id}`
    - Usecase:
        1. Check order status = PENDING
        2. Update status = PAID
        3. Publish `order-success` event → Mail service gửi email "Order completed"
    - Return `{"message": "Payment successful"}`

3. **Cancel Order** (POST `/orders/cancel`):
    - Client gửi `{order_id}`
    - Usecase:
        1. Check status = PENDING hoặc PAID
        2. Update status = CANCELLED
        3. Publish Kafka `order-cancelled` → Product service restock
        4. Publish `cancel-notification` → Mail service gửi email hủy
    - Return `{"message": "Order cancelled"}`

4. **Auto Cancel Consumer** (RabbitMQ):
    - Sau 24h, consumer nhận message từ queue `auto-cancel`
    - Usecase:
        1. Check order status
        2. Nếu vẫn PENDING → cancel order (giống Cancel Order ở trên)
        3. Publish `cancel-notification` và `order-cancelled`

5. **Product Sync Consumers** (Kafka):
    - Consume `product-created/updated/deleted` events
    - Update bảng products local (denormalize) để không phải gọi Product service mỗi lần create order

---

### 4. `mail-service/`

**Vai trò**: Consumer RabbitMQ events, gửi email thông qua SMTP (dev dùng MailHog). Feign client gọi Order service để lấy thông tin order.

#### Cấu trúc

```
mail-service/
├── cmd/api/main.go
├── cmd/workers/
│   ├── root.go              # Chạy AMQP consumer workers
│   └── amqp.go
├── internal/
│   ├── config/
│   │   ├── amqp.go
│   │   ├── smtp.go          # SMTPConfig: host, port, from email
│   │   └── feign.go         # FeignConfig: OrderURL
│   ├── constant/
│   │   ├── amqp.go          # Queues: verification, verified, reminder, cancel-notification, order-success
│   │   └── email.go         # Email templates (subject, body)
│   ├── dto/
│   │   ├── send_verification_event.go       # Event payloads
│   │   ├── payment_reminder_event.go
│   │   ├── cancel_notification_event.go
│   │   └── order_success_event.go
│   ├── feign/
│   │   └── order.go         # HTTP client gọi Order service GET /orders/:id
│   ├── mq/
│   │   ├── send_verification_consumer.go    # Consumer gửi email verify
│   │   ├── account_verified_consumer.go     # Consumer gửi email chúc mừng
│   │   ├── payment_reminder_consumer.go     # Consumer gửi email nhắc thanh toán
│   │   ├── order_cancel_consumer.go         # Consumer gửi email hủy order
│   │   └── order_success_consumer.go        # Consumer gửi email hoàn thành
│   ├── provider/
│   │   ├── init.go          # BootstrapGlobal: RabbitMQ, Mailer
│   │   └── amqp.go          # BootstrapAMQP: return list AMQP consumers
│   └── server/              # Không có HTTP server (chỉ chạy consumer workers)
├── Dockerfile
├── go.mod
└── Makefile
```

#### Luồng hoạt động

1. **Send Verification Consumer**:
    - Consume queue `verification`
    - Event payload: `{email, token}`
    - Gửi email với link verify: `http://localhost:8000/auth/verify?token=xxx`

2. **Payment Reminder Consumer**:
    - Consume queue `reminder` (delayed messages từ Order service)
    - Event payload: `{order_id, user_id, email}`
    - Gọi Order service GET `/orders/:id` (feign client) để lấy order details
    - Gửi email: "Your order #123 is pending payment, please pay within 24h"

3. **Order Cancel Consumer**:
    - Consume queue `cancel-notification`
    - Gọi Order service kiểm tra order status
    - Nếu status = CANCELLED → gửi email: "Your order #123 has been cancelled"

4. **Order Success Consumer**:
    - Consume queue `order-success`
    - Gửi email: "Your order #123 is completed! Thank you for your purchase"

---

### 5. `gateway/`

**Vai trò**: API Gateway - entry point duy nhất cho client. Reverse proxy request tới các service, auth middleware xác thực JWT.

#### Cấu trúc

```
gateway/
├── cmd/api/main.go
├── cmd/workers/
│   ├── root.go              # Chỉ chạy HTTP worker
│   └── http.go
├── internal/
│   ├── config/
│   │   ├── config.go
│   │   ├── httpserver.go
│   │   ├── service.go       # ServiceConfig: URLs của các service (auth, product, order, mail)
│   │   └── logger.go
│   ├── controller/
│   │   ├── app_controller.go        # Health check
│   │   └── gateway_controller.go    # Route proxy
│   ├── log/
│   │   └── logger.go
│   ├── proxy/
│   │   └── reverse_proxy.go         # NewReverseProxy: httputil.ReverseProxy wrapper
│   ├── provider/
│   │   ├── init.go          # BootstrapGlobal: JwtUtil, AuthMiddleware
│   │   └── http.go          # BootstrapHttp: wire controllers
│   └── server/httpserver.go
├── Dockerfile
├── go.mod
└── Makefile
```

#### Routing

```go
// gateway_controller.go
func (c *GatewayController) Route(r *gin.Engine) {
    authProxy := proxy.NewReverseProxy(c.serviceCfg.AuthURL)
    productProxy := proxy.NewReverseProxy(c.serviceCfg.ProductURL)
    orderProxy := proxy.NewReverseProxy(c.serviceCfg.OrderURL)
    mailProxy := proxy.NewReverseProxy(c.serviceCfg.MailURL)

    // Public routes
    r.Any("/auth/*path", authProxy)

    // Protected routes
    protected := r.Group("", c.authMiddleware.Authorization())
    {
        protected.Any("/products/*path", productProxy)
        protected.Any("/orders/*path", orderProxy)
        protected.Any("/mail/*path", mailProxy)
    }
}
```

Gateway không biết logic nghiệp vụ, chỉ:

1. Parse JWT token từ header `Authorization: Bearer <token>`
2. Extract `userID`, `email` từ claims
3. Set header nội bộ `X-USER-ID`, `X-EMAIL` để forward cho service
4. Proxy request tới service tương ứng

---

## 6. `pkg/` - Shared libraries

Thư mục này chứa code dùng chung giữa các service (để tránh duplicate).

```
pkg/
├── config/
│   ├── config.go            # Helper functions cho Viper config
│   ├── jwt.go               # JWT secret config
│   └── smtp.go              # SMTP config
├── constant/
│   ├── app.go               # App constants
│   ├── ctx.go               # Context keys: CTX_USER_ID, CTX_EMAIL
│   ├── error.go             # Common error messages
│   ├── http_header.go       # Header constants: X_USER_ID, X_EMAIL
│   ├── page.go              # Pagination constants
│   └── response.go          # Response constants
├── database/
│   ├── postgres.go          # NewPostgres: khởi tạo sqlx.DB
│   ├── redis.go             # NewRedisCluster: khởi tạo redis.ClusterClient
│   ├── amqp.go              # NewAMQP: connect RabbitMQ
│   └── kafka.go             # NewKafkaAsyncProducer, NewKafkaSyncProducer, NewKafkaConsumerGroup, NewKafkaAdmin
├── dto/
│   ├── error.go             # ErrorResponse DTO
│   ├── paging.go            # PageMetaData, Links
│   └── web_response.go      # WebResponse[T]
├── httperror/
│   ├── response_error.go    # ResponseError: custom error với status code
│   ├── server_error.go
│   ├── timeout_error.go
│   ├── unauthorized_error.go
│   └── ...
├── logger/
│   ├── logger.go            # Logger interface
│   ├── logrus_logger.go     # Logrus implementation
│   ├── zap_logger.go        # Zap implementation
│   └── zerolog_logger.go    # Zerolog implementation
├── middleware/
│   ├── auth_middleware.go   # AuthMiddleware: parse JWT, set context
│   ├── error_middleware.go  # ErrorHandler: catch errors từ ctx.Error()
│   ├── logger_middleware.go # Logger: log request/response
│   └── timeout_middleware.go# RequestTimeout: timeout context
├── mq/
│   ├── amqp_consumer.go     # AMQPConsumer interface + helper
│   ├── amqp_producer.go     # AMQPProducer interface
│   ├── kafka_consumer.go    # KafkaConsumer interface
│   └── kafka_producer.go    # KafkaProducer interface + metadata
└── utils/
    ├── encryptutils/
    │   └── bcrypt.go        # BcryptHasher: hash & check password
    ├── ginutils/
    │   └── response.go      # ResponseOK, ResponseError helpers
    ├── jwtutils/
    │   └── jwt.go           # JwtUtil: Sign, Parse JWT
    ├── pageutils/
    │   └── page.go          # NewMetadata, NewLinks
    ├── smtputils/
    │   └── mailer.go        # Mailer: SendMail with gomail
    └── validationutils/
        └── validator.go     # Custom validator helpers
```

Tất cả service import từ `github.com/jordanmarcelino/learn-go-microservices/pkg/...` để reuse code.

---

## 7. `deployments/` - Docker Compose files

```
deployments/
├── compose-api.yaml         # Chạy tất cả services (auth, product, order, mail, gateway)
├── compose-kafka.yaml       # Kafka cluster 3 brokers (KRaft mode)
├── compose-pgpool.yaml      # PostgreSQL cluster (3 cụm: auth/product/order, mỗi cụm 3 nodes + pgpool)
├── compose-rabbitmq.yaml    # RabbitMQ cluster 3 nodes + HAProxy + delayed-message plugin
├── compose-redis.yaml       # Redis cluster 6 nodes (3 master + 3 replica)
└── compose-spilo.yaml       # (Không dùng trong Makefile, có thể là alternative Postgres HA)
```

**Thứ tự chạy** (theo `Makefile`):

1. Build tất cả services
2. Create network `production`
3. `docker compose -f compose-pgpool.yaml -f compose-redis.yaml -f compose-kafka.yaml -f compose-rabbitmq.yaml up -d` (infra)
4. `make migrateup` (chạy migrations vào 3 DB)
5. `docker compose -f compose-api.yaml up -d` (services)

---

## 8. `infra/` - HAProxy configs

```
infra/
├── postgres-ha/
│   └── haproxy-postgresql.cfg   # Load balance pgpool (nếu dùng)
└── rabbitmq-ha/
    └── haproxy-rabbitmq.cfg     # Load balance RabbitMQ management UI
```

Các file này được mount vào container HAProxy trong compose file (ví dụ `haproxy-rabbitmq` trong `compose-rabbitmq.yaml`).

---

## Demo luồng thực tế: Register → Verify → Create Order → Pay/Cancel

### Bước 1: Register user

**Request**:

```http
POST http://localhost:8000/auth/register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "Password123!"
}
```

**Flow**:

1. Gateway nhận request → proxy tới Auth service
2. Auth service:
    - Insert user vào DB (is_verified=false)
    - Insert verification token vào bảng verification (expiry 10 phút)
    - Publish RabbitMQ event `send.verification` với payload `{email, token}`
3. Mail service consume event → gửi email verification đến user@example.com
4. Check MailHog UI: `http://localhost:8025` → thấy email với link verify

**Response**:

```json
{
    "message": "User registered successfully, please check your email"
}
```

---

### Bước 2: Verify email

User click link trong email hoặc gọi API:

**Request**:

```http
POST http://localhost:8000/auth/verify
Content-Type: application/json

{
  "token": "abc123xyz"
}
```

**Flow**:

1. Auth service:
    - Check token tồn tại, chưa hết hạn
    - Update user `is_verified=true`
    - Delete token khỏi DB
    - Publish event `account.verified` → Mail service gửi email chúc mừng
2. User nhận email "Your account has been verified"

**Response**:

```json
{
    "message": "Account verified successfully"
}
```

---

### Bước 3: Login

**Request**:

```http
POST http://localhost:8000/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "Password123!"
}
```

**Flow**:

1. Auth service:
    - Check email/password
    - Check is_verified = true
    - Generate JWT token (payload: userID=1, email=user@example.com)

**Response**:

```json
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

Lưu token này để dùng cho các request sau.

---

### Bước 4: Create product (admin/seller)

**Request**:

```http
POST http://localhost:8000/products
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "name": "iPhone 15 Pro",
  "description": "Latest Apple smartphone",
  "price": 999.99,
  "quantity": 50
}
```

**Flow**:

1. Gateway:
    - Parse JWT → extract userID=1, email=user@example.com
    - Set header `X-USER-ID: 1`, `X-EMAIL: user@example.com`
    - Proxy tới Product service
2. Product service:
    - Insert product vào DB
    - Publish Kafka event `product-created` với payload `{id: 1, name: "iPhone 15 Pro", price: 999.99, quantity: 50}`
3. Order service consume event → sync product vào bảng products local

**Response**:

```json
{
    "data": {
        "id": 1,
        "name": "iPhone 15 Pro",
        "price": 999.99,
        "quantity": 50
    }
}
```

---

### Bước 5: Create order

**Request**:

```http
POST http://localhost:8000/orders
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "request_id": "req-12345",
  "description": "My first order",
  "items": [
    {
      "product_id": 1,
      "quantity": 2
    }
  ]
}
```

**Flow**:

1. Gateway parse JWT → forward userID=1
2. Order service:
    - Acquire Redis lock: `lock:req-12345:1` (idempotency)
    - Check order với request_id đã tồn tại chưa → nếu có return order cũ
    - Nếu chưa:
        - Fetch product từ DB local (id=1, price=999.99)
        - Tính total_amount = 2 \* 999.99 = 1999.98
        - Insert order (status=PENDING), insert order_items
        - Publish Kafka `order-created` → Product service trừ quantity: 50 - 2 = 48
        - Publish RabbitMQ delayed messages:
            - `payment_reminder` với delay 4h, 12h, 22h
            - `auto_cancel` với delay 24h
    - Release lock

**Response**:

```json
{
    "data": {
        "id": 100,
        "customer_id": 1,
        "total_amount": 1999.98,
        "description": "My first order",
        "status": "PENDING",
        "items": [{ "product_id": 1, "quantity": 2, "price": 999.99 }]
    }
}
```

---

### Bước 6a: Scenario 1 - User không thanh toán (sau 4h)

**Flow**:

1. Sau 4h, RabbitMQ gửi message tới queue `reminder`
2. Mail service consume → gọi Order service GET `/orders/100`
3. Check status = PENDING → gửi email: "Your order #100 is pending, please pay soon"
4. Tương tự sau 12h, 22h → user nhận thêm 2 email nhắc
5. Sau 24h, message tới queue `auto-cancel`
6. Order service consumer:
    - Check order 100 status vẫn PENDING
    - Update status = CANCELLED
    - Publish Kafka `order-cancelled` → Product service restock: 48 + 2 = 50
    - Publish `cancel-notification` → Mail service gửi email "Order #100 cancelled"

---

### Bước 6b: Scenario 2 - User thanh toán (trước 24h)

**Request**:

```http
POST http://localhost:8000/orders/pay
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "order_id": 100
}
```

**Flow**:

1. Order service:
    - Check order 100 status = PENDING
    - Update status = PAID
    - Publish `order-success` → Mail service gửi email "Order #100 completed!"
2. Khi message `auto-cancel` tới sau 24h → consumer check status = PAID → skip (không cancel)

**Response**:

```json
{
    "message": "Payment successful"
}
```

---

### Bước 6c: Scenario 3 - User muốn hủy thủ công

**Request**:

```http
POST http://localhost:8000/orders/cancel
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "order_id": 100
}
```

**Flow**:

1. Order service:
    - Update status = CANCELLED
    - Publish `order-cancelled` → Product service restock
    - Publish `cancel-notification` → Mail service gửi email

**Response**:

```json
{
    "message": "Order cancelled successfully"
}
```

---

## Tổng kết

- **auth-service**: Register → send verification email → verify → login (JWT)
- **product-service**: CRUD product, publish Kafka events, consume order events để trừ/cộng kho
- **order-service**: Create order (idempotent với Redis lock), publish delayed RabbitMQ messages (payment reminder, auto-cancel), publish Kafka events (order-created/cancelled)
- **mail-service**: Consume RabbitMQ events, gửi email, feign client gọi Order service để lấy order info
- **gateway**: Reverse proxy, auth middleware, forward user info qua header
- **pkg**: Shared code (database, logger, middleware, utils)
- **deployments**: Docker Compose dựng Postgres HA, Redis Cluster, Kafka, RabbitMQ
- **infra**: HAProxy configs

Luồng demo:

1. Register → verify email → login
2. Create product
3. Create order → publish delayed messages
4. Nếu không pay trong 24h → auto cancel + restock
5. Nếu pay → order success
6. Nếu cancel thủ công → restock
