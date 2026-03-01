# On-Demand Car Wash — High Level Design (HLD)

---

## 1. SYSTEM ARCHITECTURE DIAGRAM

```
                        ┌──────────────────────────────────────┐
                        │           REACT.JS UI                │
                        │  Customer App | Washer App | Admin    │
                        └──────────────────┬───────────────────┘
                                           │ HTTPS / REST
                                           ▼
                        ┌──────────────────────────────────────┐
                        │           API GATEWAY                │
                        │      (Spring Cloud Gateway)          │
                        │  • Routes requests to services       │
                        │  • Validates JWT tokens              │
                        │  • Rate limiting + Logging           │
                        └──────────────────┬───────────────────┘
                                           │
                        ┌──────────────────▼───────────────────┐
                        │          EUREKA SERVER               │
                        │       (Service Discovery)            │
                        │  All microservices register here.    │
                        │  Services find each other via Eureka │
                        └──────────────────┬───────────────────┘
                                           │
     ┌─────────────────────────────────────▼────────────────────────────────────┐
     │                         MICROSERVICES                                    │
     │                                                                          │
     │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────────┐  │
     │  │  Customer   │  │   Washer    │  │    Order    │  │   Payment     │  │
     │  │  Service    │  │   Service   │  │   Service   │  │   Service     │  │
     │  │  :8081      │  │   :8082     │  │   :8083     │  │   :8084       │  │
     │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └───────┬───────┘  │
     │         │                │                │                  │          │
     │       MySQL           MySQL +           MySQL              MySQL        │
     │                       MongoDB                                           │
     │                                                                          │
     │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                     │
     │  │Notification │  │   Review    │  │    Admin    │                     │
     │  │  Service    │  │   Service   │  │   Service   │                     │
     │  │  :8085      │  │   :8086     │  │   :8087     │                     │
     │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                     │
     │         │                │                │                             │
     │       MongoDB          MongoDB           MySQL                          │
     └─────────────────────────────────────────────────────────────────────────┘
                                           │
     ┌─────────────────────────────────────▼────────────────────────────────────┐
     │                       INFRASTRUCTURE / MESSAGING                         │
     │                                                                          │
     │   ┌─────────────────────────────────────────────────────────────────┐   │
     │   │                    RABBITMQ EVENT BUS                           │   │
     │   │  Order Service ──► order.created ──► Notification + Washer     │   │
     │   │  Order Service ──► order.accepted ──► Notification             │   │
     │   │  Payment Service ──► payment.success ──► Notification + Order  │   │
     │   │  Order Service ──► wash.completed ──► Notification + Review    │   │
     │   └─────────────────────────────────────────────────────────────────┘   │
     │                                                                          │
     └─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. TECH STACK

| What | Technology | Why |
|------|-----------|-----|
| Language | Java | Core backend language |
| Framework | Spring Boot | Embedded Tomcat, auto-config, microservice base |
| API Gateway | Spring Cloud Gateway | Single entry, routing, JWT validation |
| Service Discovery | Netflix Eureka | Dynamic service registry, no hardcoded URLs |
| Messaging | RabbitMQ | Async event-driven communication between services |
| SQL Database | MySQL | Relational, ACID transactions, structured data |
| NoSQL Database | MongoDB | Flexible JSON documents, notifications, reviews |
| ORM (SQL) | Hibernate + Spring Data JPA | Object-relational mapping for MySQL |
| ODM (NoSQL) | Spring Data MongoDB | Object-document mapping for MongoDB |
| Security | Spring Security + JWT | Authentication, authorization, role-based access |
| Service Calls | Spring Cloud OpenFeign | Synchronous REST calls between microservices |
| API Docs | Swagger / OpenAPI | Auto-generated API documentation per service |
| Logging | SLF4J + Logback | Code-level logging in every service |
| Testing | JUnit 5 + Mockito | Unit tests for every service layer |
| Static Analysis | Checkstyle / SonarLint | Code quality and style enforcement |
| Build Tool | Maven | Dependency and build management |
| Frontend | React.js | Customer UI, Washer Dashboard, Admin Panel |

---

## 3. MICROSERVICES BREAKDOWN

---

### 3.1 EUREKA SERVER
**Port:** `8761`
**Database:** None
**Purpose:** Every microservice registers itself here on startup. When services need to talk to each other, they ask Eureka for the address — no hardcoded URLs.

**What to create:**
- Standard Spring Boot project
- Add `spring-cloud-starter-netflix-eureka-server`
- Annotate main class with `@EnableEurekaServer`
- `application.yml` config

---

### 3.2 API GATEWAY
**Port:** `8080`
**Database:** None
**Purpose:** The only entry point for all client requests. Routes to correct service, validates JWT, handles CORS.

**Routes configured:**
```
/api/customers/**  →  Customer Service
/api/washers/**    →  Washer Service
/api/orders/**     →  Order Service
/api/payments/**   →  Payment Service
/api/reviews/**    →  Review Service
/api/notify/**     →  Notification Service
/api/admin/**      →  Admin Service
```

**What to create:**
- Spring Boot + `spring-cloud-starter-gateway`
- JWT filter (validates token before forwarding)
- `application.yml` routes config

---

### 3.3 CUSTOMER SERVICE
**Port:** `8081`
**Database:** MySQL
**Tables:** `customers`, `addresses`, `vehicles`

**Purpose:** Handles everything about a customer — register, login, profile, vehicles, payment methods.

#### APIs:

| # | Method | Endpoint | What it does |
|---|--------|----------|-------------|
| 1 | POST | `/api/customers/register` | Register new customer (name, email, password, phone) |
| 2 | POST | `/api/customers/login` | Login → returns JWT token |
| 3 | GET | `/api/customers/{id}` | Get customer profile |
| 4 | PUT | `/api/customers/{id}` | Update customer profile |
| 5 | DELETE | `/api/customers/{id}` | Deactivate account |
| 6 | POST | `/api/customers/{id}/vehicles` | Add a vehicle (make, model, plate, type) |
| 7 | GET | `/api/customers/{id}/vehicles` | List all vehicles of a customer |
| 8 | PUT | `/api/customers/{id}/vehicles/{vid}` | Update vehicle details |
| 9 | DELETE | `/api/customers/{id}/vehicles/{vid}` | Remove a vehicle |

**What to build inside this service:**
- `CustomerController` → `CustomerService` → `CustomerRepository` → MySQL
- `VehicleController` → `VehicleService` → `VehicleRepository` → MySQL
- `Customer` entity, `Vehicle` entity, `Address` entity
- `CustomerDTO`, `VehicleDTO` (DTOs for request/response)
- JWT generation on login (using `io.jsonwebtoken`)
- BCrypt password hashing
- `@ControllerAdvice` for exception handling
- Swagger config
- Loggers (`log.info`, `log.error`) in service layer
- JUnit tests for service layer

---

### 3.4 WASHER SERVICE
**Port:** `8082`
**Database:** MySQL (profile, earnings) + MongoDB (location history)

**Purpose:** Handles washer registration, availability toggle, live GPS location, and earnings.

#### APIs:

| # | Method | Endpoint | What it does |
|---|--------|----------|-------------|
| 1 | POST | `/api/washers/register` | Register new washer |
| 2 | POST | `/api/washers/login` | Login → JWT |
| 3 | GET | `/api/washers/{id}` | Get washer profile |
| 4 | PUT | `/api/washers/{id}` | Update washer profile |
| 5 | PUT | `/api/washers/{id}/status` | Toggle online / offline |
| 6 | POST | `/api/washers/{id}/location` | Update current GPS location (lat, lng) |
| 7 | GET | `/api/washers/nearby?lat=&lng=&radius=` | Find online washers within a given radius |
| 8 | GET | `/api/washers/{id}/earnings` | View earnings summary |
| 9 | GET | `/api/washers/{id}/orders` | View job history |

**What to build inside:**
- `WasherController` → `WasherService` → `WasherRepository` → MySQL
- `LocationController` → `LocationService` → `LocationRepository` → MongoDB
- `Washer` entity (MySQL), `WasherLocation` document (MongoDB)
- Query MongoDB for washers with status `ONLINE` within radius on booking
- Swagger, Loggers, Tests, Exception Handling

---

### 3.5 ORDER SERVICE *(Core — most important)*
**Port:** `8083`
**Database:** MySQL
**Tables:** `orders`, `order_items`, `order_status_history`, `order_images`

**Purpose:** This is the brain. Creates booking, broadcasts to washers, handles accept/reject, tracks order state, manages schedule.

**Order states:**
```
PENDING → ACCEPTED → IN_PROGRESS → COMPLETED → PAID
    ↓
REJECTED → (admin auto-assigns)
    ↓
CANCELLED
```

#### APIs:

| # | Method | Endpoint | What it does |
|---|--------|----------|-------------|
| 1 | POST | `/api/orders` | Create new wash booking (instant or scheduled) |
| 2 | GET | `/api/orders/{id}` | Get order details |
| 3 | PUT | `/api/orders/{id}/status` | Update order status (e.g. IN_PROGRESS, COMPLETED) |
| 4 | POST | `/api/orders/{id}/accept` | Washer accepts an order (first-accept wins, status checked atomically) |
| 5 | POST | `/api/orders/{id}/reject` | Washer rejects an order |
| 6 | POST | `/api/orders/{id}/cancel` | Customer cancels order |
| 7 | GET | `/api/orders/customer/{customerId}` | All orders by customer |
| 8 | GET | `/api/orders/washer/{washerId}` | All orders for washer |
| 9 | GET | `/api/orders/customer/{customerId}/current` | Current (pending/accepted/in-progress) |
| 10 | GET | `/api/orders/customer/{customerId}/past` | Completed / cancelled |
| 11 | POST | `/api/orders/{id}/images` | Upload before/after car images |

**RabbitMQ events published by this service:**
- `order.created` → triggers Notification to alert nearby washers
- `order.accepted` → triggers Notification to inform customer
- `wash.completed` → triggers Notification + Review service

**What to build inside:**
- `OrderController` → `OrderService` → `OrderRepository` → MySQL
- `Order` entity, `OrderStatusHistory` entity
- Optimistic locking via JPA `@Version` on Order entity (prevent double-accept)
- Feign Client to call Washer Service (find nearby)
- Feign Client to call Customer Service (validate customer)
- RabbitMQ publisher (after each state change)
- Swagger, Loggers, Tests, Exception Handling

---

### 3.6 PAYMENT SERVICE
**Port:** `8084`
**Database:** MySQL
**Tables:** `payments`, `receipts`

**Purpose:** Handles all payment operations — initiate payment, store records, generate receipt.

#### APIs:

| # | Method | Endpoint | What it does |
|---|--------|----------|--------------|
| 1 | POST | `/api/payments/initiate` | Initiate payment for an order |
| 2 | POST | `/api/payments/process` | Process and confirm payment |
| 3 | GET | `/api/payments/{id}` | Get payment details |
| 4 | GET | `/api/payments/order/{orderId}` | Get payment by order ID |
| 5 | GET | `/api/payments/receipt/{id}` | Get/download receipt |
| 6 | POST | `/api/payments/{id}/refund` | Initiate refund |

**RabbitMQ events published:**
- `payment.success` → Notification + Order Service
- `payment.failed` → Notification Service

**What to build inside:**
- `PaymentController` → `PaymentService` → `PaymentRepository` → MySQL
- `Payment` entity, `Receipt` entity
- RabbitMQ publisher
- Swagger, Loggers, Tests, Exception Handling

---

### 3.7 NOTIFICATION SERVICE
**Port:** `8085`
**Database:** MongoDB
**Collection:** `notifications`

**Purpose:** Purely event-driven. Consumes RabbitMQ events and sends push/email/SMS notifications. Does NOT receive direct API calls from other services.

**Events it listens to:**

| Event (Queue) | What it does |
|---------------|-------------|
| `order.created` | Push to all nearby washers: "New wash request!" |
| `order.accepted` | Push to customer: "Your washer is on the way!" |
| `order.rejected` | Push to customer: "Looking for another washer..." |
| `order.in_progress` | Push to customer: "Wash started!" |
| `wash.completed` | Push to customer: "Wash done! Leave a review?" |
| `payment.success` | Push to customer: "Payment received. Here is your receipt." |
| `payment.success` | Push to washer: "Customer paid successfully." |
| `wash.scheduled` | Push to washer 2 hours before: "Scheduled job reminder!" |
| `order.cancelled` | Push to washer: "Customer cancelled the order." |

**What to build inside:**
- `RabbitMQ @RabbitListener` consumers (one per event)
- `NotificationService` (log notification + store in MongoDB)
- `Notification` MongoDB document (type, recipientId, message, status, createdAt)
- Loggers, Tests, Exception Handling
- No Swagger needed (no REST endpoints)

---

### 3.8 REVIEW SERVICE
**Port:** `8086`
**Database:** MongoDB
**Collection:** `reviews`

**Purpose:** Customers and washers submit reviews after wash completion. Aggregates ratings.

#### APIs:

| # | Method | Endpoint | What it does |
|---|--------|----------|-------------|
| 1 | POST | `/api/reviews` | Submit a review (rating 1–5, comment, orderId) |
| 2 | GET | `/api/reviews/washer/{washerId}` | All reviews for a washer |
| 3 | GET | `/api/reviews/customer/{customerId}` | All reviews written by customer |
| 4 | GET | `/api/reviews/order/{orderId}` | Review for a specific order |
| 5 | GET | `/api/reviews/washer/{washerId}/avg` | Get average rating of a washer |

**What to build inside:**
- `ReviewController` → `ReviewService` → `ReviewRepository` → MongoDB
- `Review` MongoDB document (reviewId, orderId, customerId, washerId, rating, comment, tags, createdAt)
- After save: Feign call or RabbitMQ event to update Washer average rating
- Swagger, Loggers, Tests, Exception Handling

---

### 3.9 ADMIN SERVICE
**Port:** `8087`
**Database:** MySQL
**Access:** Only `ADMIN` role JWT allowed (enforced at Gateway)

**Purpose:** Admin dashboard operations — manage users, washers, orders, reports, service plans, promo codes.

#### APIs:

| # | Method | Endpoint | What it does |
|---|--------|----------|-------------|
| 1 | GET | `/api/admin/customers` | List all customers |
| 2 | PUT | `/api/admin/customers/{id}/status` | Activate / Deactivate customer |
| 3 | GET | `/api/admin/washers` | List all washers |
| 4 | POST | `/api/admin/washers` | Add a new washer |
| 5 | PUT | `/api/admin/washers/{id}` | Edit washer |
| 6 | PUT | `/api/admin/washers/{id}/status` | Activate / Deactivate washer |
| 7 | GET | `/api/admin/orders` | View all orders (filter by status, date, washer) |
| 8 | PUT | `/api/admin/orders/{id}/assign` | Manually assign washer to pending order |
| 9 | POST | `/api/admin/services` | Create a service plan |
| 10 | PUT | `/api/admin/services/{id}` | Edit service plan |
| 11 | PUT | `/api/admin/services/{id}/status` | Active / Inactive plan |
| 12 | POST | `/api/admin/addons` | Create add-on |
| 13 | PUT | `/api/admin/addons/{id}/status` | Active / Inactive add-on |
| 14 | POST | `/api/admin/promos` | Create promo code |
| 15 | PUT | `/api/admin/promos/{id}` | Edit promo code |
| 16 | GET | `/api/admin/reports/orders` | Filter orders by washer, type, date |
| 17 | GET | `/api/admin/reports/revenue` | Revenue report |
| 18 | GET | `/api/admin/leaderboard` | Water saving leaderboard |

---

## 4. RABBITMQ EVENT MAP

```
WHO PUBLISHES          EVENT KEY              WHO CONSUMES
─────────────────────────────────────────────────────────────────────
Order Service    ──►  order.created       ──►  Notification Service
                                               Washer Service

Order Service    ──►  order.accepted      ──►  Notification Service

Order Service    ──►  order.rejected      ──►  Notification Service

Order Service    ──►  order.inprogress    ──►  Notification Service

Order Service    ──►  wash.completed      ──►  Notification Service
                                               Review Service

Payment Service  ──►  payment.success     ──►  Notification Service
                                               Order Service

Payment Service  ──►  payment.failed      ──►  Notification Service

Order Service    ──►  order.cancelled     ──►  Notification Service

Order Service    ──►  wash.scheduled      ──►  Notification Service
```

---

## 5. DATABASE ASSIGNMENT PER SERVICE

| Service | Database | Tables / Collections |
|---------|----------|---------------------|
| Customer Service | MySQL | `customers`, `addresses`, `vehicles` |
| Washer Service | MySQL | `washers`, `washer_earnings` |
| Washer Service | MongoDB | `washer_locations` |
| Order Service | MySQL | `orders`, `order_status_history`, `order_images` |
| Payment Service | MySQL | `payments`, `receipts` |
| Notification Service | MongoDB | `notifications` |
| Review Service | MongoDB | `reviews` |
| Admin Service | MySQL | `service_plans`, `addons`, `promo_codes` |

---

## 6. WHAT EACH SERVICE MUST IMPLEMENT (Required by Case Study)

Every single microservice must contain these:

| Requirement | Implementation |
|-------------|---------------|
| REST/JSON APIs | `@RestController` with proper `@GetMapping`, `@PostMapping` etc. |
| Spring Boot | Each service is its own Spring Boot project with `@SpringBootApplication` |
| Own Database | Each service connects to its own DB — no cross-DB access |
| Exception Handling | `@ControllerAdvice` + `@ExceptionHandler` — catches all errors, returns clean JSON |
| Loggers | `private static final Logger log = LoggerFactory.getLogger(...)` — log every important event |
| Test Cases | JUnit 5 tests for every service method (use Mockito for mocking) |
| Static Code Analysis | Add Checkstyle or use SonarLint plugin |
| Build Tool | Maven (`pom.xml`) in every service |
| Embedded Tomcat | Spring Boot includes it by default — nothing extra needed |
| RabbitMQ | At minimum Order Service publishes events, Notification Service consumes |
| React.js UI | One React project with 3 modules: Customer, Washer, Admin |

---

## 7. WHERE SPECIFIC TECHNOLOGIES FIT (Cheat Sheet)

| Technology | Lives In | Purpose |
|-----------|----------|---------|
| Spring Cloud Gateway | API Gateway service | Single entry point, JWT check, routing to services |
| Netflix Eureka | Separate Eureka service | All services self-register, dynamic discovery |
| OpenFeign Client | Order, Payment, Admin services | Synchronous REST calls between microservices |
| RabbitMQ | Order + Payment publish; Notification + others consume | Async event-driven communication |
| Spring Security | Every service | Authentication and role-based authorization |
| JWT | Customer + Washer services (issue); API Gateway (validate) | Stateless auth token passed in every request |
| BCrypt | Customer + Washer services | Hash passwords before storing |
| Hibernate + Spring Data JPA | Customer, Washer, Order, Payment, Admin | ORM layer for MySQL |
| Spring Data MongoDB | Washer (location), Notification, Review | ODM layer for MongoDB |
| Swagger / OpenAPI | Every service | Auto-document APIs at `/swagger-ui.html` |
| @ControllerAdvice | Every service | Centralized exception handling → clean JSON errors |
| @EntityListeners (Auditing) | Every JPA entity | Auto-populate createdAt, updatedAt, createdBy |
| JPA @Version | Order entity | Optimistic locking — prevent double-accept on same order |
| SLF4J + Logback | Every service | Structured logging at every layer |
| JUnit 5 + Mockito | Every service test folder | Unit and integration tests |
| Checkstyle / SonarLint | Every service | Static code analysis |

---

## 8. IMPLEMENTATION ORDER (Build in this exact order)

```
PHASE 1 — Foundation
  Step 1: Eureka Server          ← Register all services here
  Step 2: API Gateway            ← Routes + JWT filter

PHASE 2 — Users
  Step 3: Customer Service       ← Register, Login, JWT, Profile, Vehicles
  Step 4: Washer Service         ← Register, Login, Location, Availability

PHASE 3 — Core Business
  Step 5: Order Service          ← CORE — booking, state machine, RabbitMQ publish
  Step 6: Payment Service        ← Payment processing, receipts, RabbitMQ publish

PHASE 4 — Events + Feedback
  Step 7: Notification Service   ← RabbitMQ consumers, store notifications in MongoDB
  Step 8: Review Service         ← Ratings after wash

PHASE 5 — Control
  Step 9: Admin Service          ← Reports, manage users, assign orders

PHASE 6 — Frontend
  Step 10: React.js App          ← Customer module → Washer module → Admin module
```

---

## 9. PROJECT FOLDER STRUCTURE (GitHub)

```
on-demand-carwash/
│
├── eureka-server/                ← Spring Boot Eureka
├── api-gateway/                  ← Spring Cloud Gateway
├── customer-service/             ← Spring Boot + MySQL
├── washer-service/               ← Spring Boot + MySQL + MongoDB
├── order-service/                ← Spring Boot + MySQL + RabbitMQ
├── payment-service/              ← Spring Boot + MySQL + RabbitMQ
├── notification-service/         ← Spring Boot + MongoDB + RabbitMQ
├── review-service/               ← Spring Boot + MongoDB
├── admin-service/                ← Spring Boot + MySQL
│
└── carwash-ui/                   ← React.js project
    ├── src/pages/customer/
    ├── src/pages/washer/
    └── src/pages/admin/
```

---

## 10. INSIDE EACH SPRING BOOT SERVICE (Standard Folder Structure)

Use this structure for EVERY microservice:

```
customer-service/
└── src/
    ├── main/
    │   ├── java/com/carwash/customer/
    │   │   ├── controller/          ← @RestController classes
    │   │   ├── service/             ← @Service classes (business logic)
    │   │   ├── repository/          ← @Repository interfaces (JPA)
    │   │   ├── entity/              ← @Entity classes (DB tables)
    │   │   ├── dto/                 ← Request/Response DTOs
    │   │   ├── exception/           ← Custom exceptions + @ControllerAdvice
    │   │   ├── config/              ← Swagger config, Security config
    │   │   └── CustomerServiceApp.java  ← @SpringBootApplication main class
    │   │
    │   └── resources/
    │       └── application.yml      ← DB config, Eureka config, server port
    │
    └── test/
        └── java/com/carwash/customer/
            └── service/             ← JUnit 5 test classes
```

---

## 11. STANDARD FLOW FOR EVERY API CALL

```
HTTP Request from React.js
        │
        ▼
API Gateway (validates JWT, routes to correct service)
        │
        ▼
Controller (@RestController)       ← receives request, calls service
        │
        ▼
Service (@Service)                 ← business logic, validations, rules
        │
        ▼
Repository (@Repository)           ← talks to database via JPA / Spring Data MongoDB
        │
        ▼
Database (MySQL / MongoDB)         ← stores / fetches data
        │
        ▼ (response travels back up)
Controller returns ResponseEntity<>
        │
        ▼
React.js displays result
```

---

*HLD v1.0 | On-Demand Car Wash | March 2026*
