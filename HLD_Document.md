over structure like this # HIGH LEVEL DESIGN DOCUMENT
## On-Demand Car Wash System

---

**Project Name:** On-Demand Car Wash Platform  
**Document Type:** High Level Design (HLD)  
**Document Version:** 1.0  
**Date:** March 1, 2026  
**Prepared By:** Ayush Pandey  

---

## DOCUMENT APPROVAL

| Name | Department | Role | Date |
|------|------------|------|------|
| | | Solution Architect | |
| | | Tech Lead | |
| | | Product Owner | |

---

## DOCUMENT CHANGE HISTORY

| Version | Author | Date | Description |
|---------|--------|------|-------------|
| 1.0 | Ayush Pandey | 01/03/2026 | Initial HLD for On-Demand Car Wash System |

---

## TABLE OF CONTENTS

1. [Document Purpose](#1-document-purpose)
2. [Intended Audience](#2-intended-audience)
3. [Project Background & Objectives](#3-project-background--objectives)
4. [System Overview](#4-system-overview)
5. [Architecture Style](#5-architecture-style)
6. [Technology Stack](#6-technology-stack)
7. [Microservices Breakdown](#7-microservices-breakdown)
8. [System Architecture Diagram](#8-system-architecture-diagram)
9. [Database Design Strategy](#9-database-design-strategy)
10. [API Gateway & Service Discovery](#10-api-gateway--service-discovery)
11. [Messaging Architecture (RabbitMQ)](#11-messaging-architecture-rabbitmq)
12. [Core Business Flows](#12-core-business-flows)
13. [Cross-Cutting Concerns](#13-cross-cutting-concerns)
14. [Security Architecture](#14-security-architecture)
15. [Deployment Architecture](#15-deployment-architecture)
16. [Non-Functional Requirements](#16-non-functional-requirements)
17. [Inter-Service Communication](#17-inter-service-communication)
18. [API Surface Summary](#18-api-surface-summary)

---

## 1. Document Purpose

This document describes the High Level Design (HLD) of the **On-Demand Car Wash System** — a microservices-based platform that connects customers requiring car wash services with nearby professional washers, similar in model to Uber/Ola but for car washing.

This document covers:
- Overall system architecture
- Microservices breakdown with responsibilities
- Technology choices and justifications
- Database strategy per service
- Communication patterns (synchronous and asynchronous)
- Security, logging, and deployment approach

---

## 2. Intended Audience

| Role | Engagement |
|------|------------|
| Product Owners / SME | Ensure architecture aligns with business goals |
| Business Analysts | Informed on key architectural decisions |
| Enterprise Architects | Validate alignment with enterprise architecture guidelines |
| Solution Architects | Ensure design aligns with requirements and constraints |
| Developers | Use as guiding document for detail design and implementation |

---

## 3. Project Background & Objectives

### 3.1 Project Background

The On-Demand Car Wash System is a platform enabling customers to book professional car wash services at their location. The system connects three key actors:

- **Customers** — book instant or scheduled wash services
- **Washers** — mobile service providers who accept and fulfill wash requests
- **Admin** — system operator managing users, services, pricing, and analytics

The platform operates on a real-time matching model: a customer submits a request, nearby available washers receive the request, and the first washer to accept serves the customer.

### 3.2 Project Objectives

- Build a **scalable, production-grade microservices platform** using Spring Boot
- Each microservice must expose **REST/JSON APIs** and own its database
- Implement **real-time washer matching** using geospatial queries
- Support **instant and scheduled** wash bookings
- Integrate **payment gateway** (Braintree) for transactions
- Use **RabbitMQ** for all asynchronous event-driven communication
- Maintain full **audit trail** of system actions
- Provide a **React/Angular frontend** for customers, washers, and admin

---

## 4. System Overview

### 4.1 Actors

| Actor | Description |
|-------|-------------|
| **Customer** | End user who requests car wash services |
| **Washer** | Service provider who fulfills wash requests |
| **Admin** | System operator who manages the entire platform |

### 4.2 High-Level Actor Capabilities

**Customer**
- Register / Login
- Add vehicle details and payment information
- Book instant wash (Wash Now) or schedule future wash
- Track washer in real-time
- Make payment via gateway
- Receive digital receipt with car images
- Rate and review washer
- View current and past orders
- Receive notifications (washer accepted, service started, completed)
- View leaderboard (water saving stats)

**Washer**
- Login
- Toggle online/offline availability
- Receive nearby wash requests
- Accept or reject requests
- Navigate to customer via Google Maps
- Upload before/after car images
- Generate invoice
- View order history and earnings
- Maintain profile and service ratings

**Admin**
- Manage customers and washers
- Create/manage service plans and add-ons
- Create promo codes
- Manual order assignment
- View reports and analytics
- Manage leaderboard
- View system logs and audit trails

---

## 5. Architecture Style

### 5.1 Pattern: Microservices Architecture

The system follows a **Microservices Architecture** where the application is divided into small, independently deployable services, each responsible for a specific business capability.

```
                     MONOLITH (Not used)
  ┌──────────────────────────────────────────┐
  │  CustomerLogic + OrderLogic + PayLogic   │ ← Hard to scale, single point of failure
  └──────────────────────────────────────────┘

                  MICROSERVICES (Our approach)
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │ Customer MS │  │  Order MS   │  │ Payment MS  │  ← Independent, scalable
  └─────────────┘  └─────────────┘  └─────────────┘
```

### 5.2 Core Microservices Principles Applied

| Principle | Implementation |
|-----------|---------------|
| Single Responsibility | Each service owns exactly one business domain |
| Database per Service | No shared databases between services |
| API-first | All inter-service and client communication via REST APIs |
| Event-driven | Asynchronous communication via RabbitMQ |
| Independently deployable | Each service runs in its own process |
| Service Discovery | All services register with Eureka |

---

## 6. Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Backend Framework | Java 17 + Spring Boot 3.x | Core microservices |
| API Gateway | Spring Cloud Gateway | Single entry point, routing, auth |
| Service Discovery | Netflix Eureka | Dynamic service registry |
| Messaging | RabbitMQ | Asynchronous event communication |
| Primary Database | MySQL 8 | Relational data (Orders, Payments, Customers) |
| Secondary Database | MongoDB | Document data (Locations, Logs, Reviews) |
| Caching / Real-time | Redis | Washer live location, distributed locking |
| ORM | Hibernate + Spring Data JPA | MySQL object-relational mapping |
| Document Mapper | Spring Data MongoDB | MongoDB object-document mapping |
| Authentication | JWT (JSON Web Tokens) | Stateless auth across services |
| API Documentation | Swagger / OpenAPI 3 | Per-service API documentation |
| Logging | SLF4J + Logback + ELK Stack | Code-level and centralized logging |
| Monitoring | Spring Boot Actuator + Prometheus + Grafana | Health and metrics |
| Payment | Braintree Gateway | Payment processing |
| Notifications | Firebase Cloud Messaging (FCM) | Push notifications |
| Frontend | Angular / React | Customer, Washer, Admin UIs |
| Build Tool | Maven | Dependency and build management |
| Containerization | Docker | Service packaging |

---

## 7. Microservices Breakdown

### 7.1 Service Registry (Eureka Server)

| Property | Value |
|----------|-------|
| Port | 8761 |
| Role | Central registry where all services self-register |
| Database | None |
| Communication | All services connect to this server on startup |

**Purpose:** Solves dynamic service discovery. Services do not hardcode each other's URLs. They register with Eureka and query it when they need to communicate.

---

### 7.2 API Gateway Service

| Property | Value |
|----------|-------|
| Port | 8080 |
| Technology | Spring Cloud Gateway |
| Role | Single entry point for all client requests |
| Database | None |

**Responsibilities:**
- Route incoming requests to appropriate microservices
- Validate and forward JWT tokens
- Rate limiting
- CORS handling
- Request/Response logging

**Routing Rules:**

| Path Prefix | Routes To |
|-------------|-----------|
| `/api/customers/**` | Customer Service |
| `/api/washers/**` | Washer Service |
| `/api/orders/**` | Order Service |
| `/api/payments/**` | Payment Service |
| `/api/reviews/**` | Review Service |
| `/api/notifications/**` | Notification Service |
| `/api/admin/**` | Admin Service |

---

### 7.3 Customer Service

| Property | Value |
|----------|-------|
| Port | 8081 |
| Database | MySQL |
| ORM | Hibernate + Spring Data JPA |
| Pattern | Controller → Service → Repository → MySQL |

**Responsibilities:**
- Customer registration and login (JWT issuance)
- Profile management (name, email, phone, address)
- Vehicle management (car make, model, plate)
- Payment method storage
- Customer account settings

**Key Entities:**
- `Customer` (id, firstName, lastName, email, passwordHash, phone)
- `Address` (addressId, customerId, street, city, pincode)
- `Vehicle` (vehicleId, customerId, make, model, plateNumber, type)

**Key APIs:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/customers/register` | Register new customer |
| POST | `/api/customers/login` | Customer login → JWT |
| GET | `/api/customers/{id}` | Get customer profile |
| PUT | `/api/customers/{id}` | Update profile |
| POST | `/api/customers/{id}/vehicles` | Add vehicle |
| GET | `/api/customers/{id}/vehicles` | List vehicles |
| DELETE | `/api/customers/{id}` | Deactivate account |

---

### 7.4 Washer Service

| Property | Value |
|----------|-------|
| Port | 8082 |
| Database | MySQL + MongoDB |
| ORM | Spring Data JPA + Spring Data MongoDB |
| Pattern | Controller → Service → Repository → MySQL/MongoDB |

**Responsibilities:**
- Washer registration, login, and profile management
- Washer document verification tracking
- Availability toggle (online/offline)
- **Real-time location updates** (every 5 seconds → stored in MongoDB/Redis)
- Washer rating and review aggregation
- Earnings history

**Key Entities:**
- `Washer` (MySQL): id, name, email, phone, vehicle, status, avgRating
- `WasherLocation` (MongoDB): washerId, lat, lng, timestamp, isOnline
- `WasherEarning` (MySQL): earningId, washerId, orderId, amount, date

**Key APIs:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/washers/register` | Register washer |
| POST | `/api/washers/login` | Login → JWT |
| PUT | `/api/washers/{id}/status` | Toggle online/offline |
| POST | `/api/washers/{id}/location` | Update live location |
| GET | `/api/washers/nearby` | Find washers within radius |
| GET | `/api/washers/{id}/earnings` | View earnings |

---

### 7.5 Order Service *(Core Service)*

| Property | Value |
|----------|-------|
| Port | 8083 |
| Database | MySQL |
| ORM | Spring Data JPA |
| Pattern | Controller → Service → Repository → MySQL |

**Responsibilities:**
- Create new wash order (instant or scheduled)
- Broadcast order request to nearby washers (via RabbitMQ)
- Handle washer acceptance/rejection with **distributed lock** (Redis)
- Auto-assign washer if no acceptance within timeout (Admin fallback)
- Track order state machine (PENDING → ACCEPTED → IN_PROGRESS → COMPLETED → CANCELLED)
- Store before/after car images (references only)

**Order State Machine:**
```
PENDING → ACCEPTED → IN_PROGRESS → COMPLETED
        ↓                                   ↓
    REJECTED / NO_RESPONSE             RATED (Review given)
        ↓
    CANCELLED
```

**Key Entities:**
- `Order` (id, customerId, washerId, vehicleId, serviceType, status, scheduledAt, totalAmount)
- `OrderImage` (imageId, orderId, imageUrl, type: BEFORE/AFTER)
- `OrderStatusHistory` (historyId, orderId, status, changedAt)

**Key APIs:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/orders` | Create new wash booking |
| POST | `/api/orders/{id}/accept` | Washer accepts order |
| POST | `/api/orders/{id}/reject` | Washer rejects order |
| PUT | `/api/orders/{id}/status` | Update order status |
| GET | `/api/orders/{id}` | Get order details |
| GET | `/api/orders/customer/{customerId}` | Customer's order history |
| GET | `/api/orders/washer/{washerId}` | Washer's job history |

---

### 7.6 Payment Service

| Property | Value |
|----------|-------|
| Port | 8084 |
| Database | MySQL |
| ORM | Spring Data JPA |
| External | Braintree Payment Gateway |

**Responsibilities:**
- Generate payment token for client (Braintree drop-in UI)
- Process payment after order completion
- Store payment records
- Generate and store digital receipts
- Handle refunds (if applicable)
- Publish `PAYMENT_SUCCESS` / `PAYMENT_FAILED` events to RabbitMQ

**Key Entities:**
- `Payment` (id, orderId, customerId, amount, status, transactionId, gateway, paidAt)
- `Receipt` (id, paymentId, receiptUrl, generatedAt)

**Key APIs:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/payments/token` | Get Braintree client token |
| POST | `/api/payments/process` | Process payment |
| GET | `/api/payments/{id}` | Get payment details |
| GET | `/api/payments/order/{orderId}` | Payment by order |
| GET | `/api/payments/receipt/{id}` | Download receipt |

---

### 7.7 Notification Service *(Event-Driven)*

| Property | Value |
|----------|-------|
| Port | 8085 |
| Database | MongoDB |
| Communication | RabbitMQ consumer only (no sync inbound API calls) |

**Responsibilities:**
- Consume events from RabbitMQ queues
- Send push notifications via Firebase Cloud Messaging (FCM)
- Send email notifications (via SMTP or SendGrid)
- Send SMS notifications (via Twilio)
- Store notification history in MongoDB

**Events Consumed:**

| Event | Action |
|-------|--------|
| `ORDER_CREATED` | Notify nearby washers of new request |
| `ORDER_ACCEPTED` | Notify customer that washer is coming |
| `ORDER_REJECTED` | Notify customer to wait for other washer |
| `SERVICE_STARTED` | Notify customer that wash has started |
| `PAYMENT_SUCCESS` | Send payment receipt notification |
| `WASH_COMPLETED` | Notify customer, prompt for rating |

---

### 7.8 Review & Rating Service

| Property | Value |
|----------|-------|
| Port | 8086 |
| Database | MongoDB |
| Document Mapper | Spring Data MongoDB |

**Responsibilities:**
- Accept reviews after wash completion
- Calculate and aggregate washer average rating
- Store review content (stars, comments, tags)
- Trigger rating update on Washer Service

**Key Document:**
```json
{
  "reviewId": "uuid",
  "orderId": "1001",
  "customerId": "501",
  "washerId": "301",
  "rating": 4.5,
  "comment": "Great job, very thorough.",
  "tags": ["punctual", "professional"],
  "createdAt": "2026-03-01T10:30:00"
}
```

**Key APIs:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/reviews` | Submit review after wash |
| GET | `/api/reviews/washer/{washerId}` | All reviews for a washer |
| GET | `/api/reviews/customer/{customerId}` | Reviews by customer |
| GET | `/api/reviews/order/{orderId}` | Review for specific order |

---

### 7.9 Admin Service

| Property | Value |
|----------|-------|
| Port | 8087 |
| Database | MySQL |
| Access | Admin JWT role only (enforced at Gateway) |

**Responsibilities:**
- View and manage all customers and washers
- Manually assign washers to orders
- Create and manage service plans and add-ons
- Create and manage promo codes
- Generate reports (revenue, orders, washer performance)
- View system audit logs
- Manage leaderboard (water saving statistics)

**Key APIs:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/admin/customers` | List all customers |
| GET | `/api/admin/washers` | List all washers |
| POST | `/api/admin/orders/{id}/assign` | Manually assign washer |
| POST | `/api/admin/services` | Create service plan |
| POST | `/api/admin/promos` | Create promo code |
| GET | `/api/admin/reports/revenue` | Revenue report |
| GET | `/api/admin/reports/orders` | Orders report |

---

## 8. System Architecture Diagram

```
                    ┌────────────────────────────────┐
                    │     CLIENT APPLICATIONS        │
                    │  (Angular / React / Mobile)    │
                    │  Customer | Washer | Admin UI  │
                    └────────────────┬───────────────┘
                                     │ HTTPS
                                     ▼
                    ┌────────────────────────────────┐
                    │         API GATEWAY            │
                    │   (Spring Cloud Gateway :8080) │
                    │  • Routing  • Auth  • Logging  │
                    └────────────────┬───────────────┘
                                     │
                    ┌────────────────▼───────────────┐
                    │        EUREKA SERVER           │
                    │  (Service Discovery :8761)     │
                    └────────────────┬───────────────┘
                                     │
     ┌───────────────────────────────▼──────────────────────────────┐
     │                     MICROSERVICES LAYER                      │
     │                                                              │
     │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
     │  │Customer  │ │ Washer   │ │  Order   │ │  Payment     │   │
     │  │  :8081   │ │  :8082   │ │  :8083   │ │   :8084      │   │
     │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────┬───────┘   │
     │       │            │            │              │            │
     │  ┌────▼─────┐ ┌────▼─────┐ ┌────▼─────┐ ┌──────▼───────┐   │
     │  │  MySQL   │ │MySQL +   │ │  MySQL   │ │    MySQL     │   │
     │  │          │ │MongoDB   │ │          │ │  + Braintree │   │
     │  └──────────┘ └──────────┘ └──────────┘ └──────────────┘   │
     │                                                              │
     │  ┌──────────┐ ┌──────────┐ ┌──────────┐                    │
     │  │Notific.  │ │ Review   │ │  Admin   │                    │
     │  │  :8085   │ │  :8086   │ │  :8087   │                    │
     │  └────┬─────┘ └────┬─────┘ └────┬─────┘                    │
     │       │            │            │                           │
     │  ┌────▼─────┐ ┌────▼─────┐ ┌────▼─────┐                    │
     │  │ MongoDB  │ │ MongoDB  │ │  MySQL   │                    │
     │  └──────────┘ └──────────┘ └──────────┘                    │
     └───────────────────────┬──────────────────────────────────────┘
                             │
     ┌───────────────────────▼──────────────────────────────────────┐
     │                    INFRASTRUCTURE LAYER                      │
     │                                                              │
     │  ┌──────────────────┐      ┌──────────────────────────────┐  │
     │  │    RABBITMQ      │      │           REDIS              │  │
     │  │  Message Broker  │      │  Live Location + Locks       │  │
     │  └──────────────────┘      └──────────────────────────────┘  │
     │                                                              │
     │  ┌──────────────────────────────────────────────────────┐   │
     │  │              ELK STACK (Centralized Logging)         │   │
     │  │  Elasticsearch + Logstash + Kibana                   │   │
     │  └──────────────────────────────────────────────────────┘   │
     └──────────────────────────────────────────────────────────────┘
```

---

## 9. Database Design Strategy

The system uses **Polyglot Persistence** — choosing the right database for each service based on data characteristics.

### 9.1 MySQL (Relational) — Used For:

| Service | Reason |
|---------|--------|
| Customer Service | Structured user data, address relationships, ACID compliance needed |
| Order Service | Financial transactions, strong consistency, relational order-customer-washer ties |
| Payment Service | Financial records — ACID compliance is mandatory |
| Admin Service | Reports, structured analytics queries |
| Washer Service (profile) | Structured washer profile, earnings history |

### 9.2 MongoDB (Document) — Used For:

| Service | Reason |
|---------|--------|
| Washer Service (location) | Frequent writes, flexible location JSON, Geospatial queries |
| Notification Service | Flexible notification documents, no fixed schema |
| Review Service | Flexible review content with varying fields and tags |

### 9.3 Redis — Used For:

| Use Case | Mechanism |
|----------|-----------|
| Live washer location tracking | `GEOADD`, `GEORADIUS` commands — updated every 5 seconds |
| Distributed locking (prevent double-accept) | `SETNX` atomic lock on order ID |
| Washer online/offline TTL | Auto-expire location if no ping for 15 seconds |
| Session caching | Cache JWT validation results |

### 9.4 Database Design Principle

> Each microservice owns its database exclusively.  
> **No service queries another service's database directly.**  
> Cross-service data access only happens via REST API or RabbitMQ events.

---

## 10. API Gateway & Service Discovery

### 10.1 API Gateway (Spring Cloud Gateway)

```
Client Request
     │
     ▼
┌──────────────────────────────────────────┐
│              API GATEWAY                 │
│                                          │
│  1. Extract JWT from Authorization header│
│  2. Validate token                       │
│  3. Extract role (CUSTOMER/WASHER/ADMIN) │
│  4. Route to correct microservice        │
│  5. Forward token in request header      │
└──────────────────────────────────────────┘
     │
     ▼
Correct Microservice
```

### 10.2 Eureka (Service Discovery)

```
Startup → Customer MS registers: CUSTOMER-SERVICE @ localhost:8081
Startup → Order MS registers: ORDER-SERVICE @ localhost:8083

When Order MS needs Customer MS:
Order MS → asks Eureka → Eureka returns CUSTOMER-SERVICE address
Order MS → calls Customer MS directly
```

No hardcoded service URLs. Services are discovered dynamically.

---

## 11. Messaging Architecture (RabbitMQ)

### 11.1 Why RabbitMQ?

Without RabbitMQ (tight coupling):
```
Payment Service → directly calls → Notification Service
Payment Service → directly calls → Order Service
Payment Service → directly calls → Audit Service
❌ Payment Service breaks if any downstream service is down
```

With RabbitMQ (loose coupling):
```
Payment Service → publishes event → RabbitMQ Queue
RabbitMQ → delivers to → Notification Service (async)
RabbitMQ → delivers to → Order Service (async)
RabbitMQ → delivers to → Audit Service (async)
✅ Payment Service doesn't know or care who is listening
```

### 11.2 Event Catalog

| Exchange | Routing Key | Publisher | Consumers |
|----------|-------------|-----------|-----------|
| `order.exchange` | `order.created` | Order Service | Notification Service, Washer Service |
| `order.exchange` | `order.accepted` | Order Service | Notification Service |
| `order.exchange` | `order.completed` | Order Service | Notification Service, Review Service |
| `payment.exchange` | `payment.success` | Payment Service | Notification Service, Order Service |
| `payment.exchange` | `payment.failed` | Payment Service | Notification Service, Order Service |
| `washer.exchange` | `washer.rejected` | Washer Service | Order Service, Notification Service |

### 11.3 RabbitMQ Flow — Order Booking

```
Order Service publishes → order.created
            │
            ├──► Notification Service: "Nearby washers notified"
            └──► Washer Service: "Match washers to request"
```

---

## 12. Core Business Flows

### 12.1 Customer Books Instant Wash

```
1. Customer submits wash request
   └── POST /api/orders → API Gateway → Order Service

2. Order Service creates order (status: PENDING)
   └── Saves to MySQL

3. Order Service publishes event: ORDER_CREATED
   └── RabbitMQ → Notification Service
                 → Sends push notification to nearby washers

4. Washer taps "Accept"
   └── POST /api/orders/{id}/accept → Order Service
   └── Redis distributed lock checks: first accept wins
   └── Status updated: ACCEPTED

5. Order Service publishes: ORDER_ACCEPTED
   └── RabbitMQ → Notification Service → Notifies customer

6. Washer travels and starts service
   └── PUT /api/orders/{id}/status → IN_PROGRESS

7. Wash completed
   └── PUT /api/orders/{id}/status → COMPLETED

8. Payment triggered
   └── POST /api/payments/process → Payment Service → Braintree

9. Payment Service publishes: PAYMENT_SUCCESS
   └── RabbitMQ → Notification Service (receipt notification)
              → Order Service (update order to PAID)

10. Customer submits review
    └── POST /api/reviews → Review Service
```

### 12.2 No Washer Accepts (Auto-Assignment)

```
ORDER_CREATED published
     │
     └── No washer accepts within N minutes
          └── Order Service timeout trigger
               └── Admin Service auto-assigns nearest washer
                    └── Notification sent to assigned washer
```

---

## 13. Cross-Cutting Concerns

### 13.1 Logging (Two Levels)

**Code-Level Logging (LLD — inside each service)**
```java
log.info("Order created with ID: {}", order.getId());
log.error("Payment failed for orderId: {}", orderId);
```
- Tool: SLF4J + Logback
- Applied in: Controller, Service, Repository
- Format: Structured JSON logs

**Centralized Logging (HLD — system-wide)**
```
All Microservices
      │ (Log output)
      ▼
Logstash (collects + transforms)
      ▼
Elasticsearch (indexes and stores)
      ▼
Kibana (visualize, search, alert)
```

### 13.2 Auditing

Implemented at **entity level** inside each service using Spring Data Auditing:

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Order {
    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    private String createdBy;

    @LastModifiedBy
    private String lastModifiedBy;
}
```

Every database record tracks **who created it, when, and who last changed it**.

### 13.3 Swagger / OpenAPI Documentation

Each microservice exposes its own Swagger UI:

| Service | Swagger URL |
|---------|------------|
| Customer | `http://localhost:8081/swagger-ui.html` |
| Order | `http://localhost:8083/swagger-ui.html` |
| Payment | `http://localhost:8084/swagger-ui.html` |
| Washer | `http://localhost:8082/swagger-ui.html` |

Configuration dependency per service:
```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.x</version>
</dependency>
```

### 13.4 Exception Handling

Centralized exception handling inside each service using `@ControllerAdvice`:

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<?> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse(ex.getMessage()));
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<?> handleValidation(ValidationException ex) {
        return ResponseEntity.status(400)
            .body(new ErrorResponse(ex.getMessage()));
    }
}
```

### 13.5 Monitoring

| Tool | Purpose |
|------|---------|
| Spring Boot Actuator | Exposes `/actuator/health`, `/actuator/metrics` per service |
| Prometheus | Scrapes metrics from all services |
| Grafana | Dashboards for real-time monitoring |

---

## 14. Security Architecture

### 14.1 Authentication Flow (JWT)

```
1. Customer/Washer hits login API
2. Customer Service / Washer Service validates credentials
3. JWT token generated with payload:
   { userId, email, role: CUSTOMER/WASHER/ADMIN, expiry }
4. Token returned to client

5. Every subsequent request:
   Client → sends JWT in Authorization: Bearer <token> header
   API Gateway → validates token
   If valid → forwards request to service
   If invalid → returns 401 Unauthorized
```

### 14.2 Role-Based Access Control (RBAC)

| Endpoint Pattern | Allowed Roles |
|-----------------|--------------|
| `/api/customers/**` | CUSTOMER |
| `/api/washers/**` | WASHER |
| `/api/admin/**` | ADMIN |
| `/api/orders/**` | CUSTOMER, WASHER |
| `/api/payments/**` | CUSTOMER |
| `/api/reviews/**` | CUSTOMER |

### 14.3 Password Security

- Passwords stored as **BCrypt hashed values** — never plain text
- Password never returned in API responses

---

## 15. Deployment Architecture

### 15.1 Service Port Mapping

| Service | Port |
|---------|------|
| Eureka Server | 8761 |
| API Gateway | 8080 |
| Customer Service | 8081 |
| Washer Service | 8082 |
| Order Service | 8083 |
| Payment Service | 8084 |
| Notification Service | 8085 |
| Review Service | 8086 |
| Admin Service | 8087 |
| MySQL | 3306 |
| MongoDB | 27017 |
| Redis | 6379 |
| RabbitMQ | 5672 / 15672 (Management UI) |

### 15.2 Docker Compose Overview

All services, databases, and infrastructure components are containerized and run via Docker Compose for local development.

```yaml
# Conceptual structure
services:
  eureka-server:      # Port 8761
  api-gateway:        # Port 8080
  customer-service:   # Port 8081
  washer-service:     # Port 8082
  order-service:      # Port 8083
  payment-service:    # Port 8084
  notification-service: # Port 8085
  review-service:     # Port 8086
  admin-service:      # Port 8087
  mysql:              # Port 3306
  mongodb:            # Port 27017
  redis:              # Port 6379
  rabbitmq:           # Port 5672 + 15672
```

---

## 16. Non-Functional Requirements

| Category | Requirement | Approach |
|----------|-------------|----------|
| Scalability | Services must independently scale | Microservices + Docker |
| Availability | Core services (Order, Payment) must be highly available | Redundant instances |
| Performance | Washer matching < 200ms | Redis geospatial queries |
| Consistency | Payment and Order data must be strongly consistent | MySQL ACID transactions |
| Eventual Consistency | Notifications may have slight delay | RabbitMQ async |
| Security | All endpoints secured | JWT + RBAC at Gateway |
| Observability | All services monitored | Actuator + Prometheus + Grafana |
| Audit | All data changes tracked | Spring Data Auditing |
| Documentation | All APIs documented | Swagger/OpenAPI per service |

---

## 17. Inter-Service Communication

### 17.1 Synchronous (REST via Feign Client + Eureka)

Used when a service **needs an immediate response** to proceed.

```
Order Service needs Customer name:
Order Service (Feign Client) → Eureka → Customer Service
                                               │
                                               ▼
                                      Returns Customer data
```

| Caller | Calls | For |
|--------|-------|-----|
| Order Service | Customer Service | Validate customer exists |
| Order Service | Washer Service | Get nearby washers |
| Payment Service | Order Service | Verify order before payment |
| Admin Service | All Services | Reports, management |

### 17.2 Asynchronous (RabbitMQ Events)

Used when the caller **does not need to wait** for a result.

```
Payment Service publishes: PAYMENT_SUCCESS
     │
     ├──► Notification Service (send receipt) — runs independently
     ├──► Order Service (mark order PAID) — runs independently
     └──► Audit Service (log action) — runs independently
```

**Rule:**
- Use REST for queries (GET data)
- Use RabbitMQ for state change notifications (events)

---

## 18. API Surface Summary

### Standard HTTP Status Codes Used

| Code | Meaning | Used When |
|------|---------|-----------|
| 200 | OK | Successful GET, PUT |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Validation failure |
| 401 | Unauthorized | Missing/invalid JWT |
| 403 | Forbidden | Wrong role |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource |
| 500 | Internal Server Error | Unexpected server error |

### Standard Response Envelope

**Success Response:**
```json
{
  "status": "success",
  "data": { },
  "message": "Customer created successfully",
  "timestamp": "2026-03-01T10:00:00"
}
```

**Error Response:**
```json
{
  "status": "error",
  "error": "RESOURCE_NOT_FOUND",
  "message": "Customer with ID 501 not found",
  "timestamp": "2026-03-01T10:00:00"
}
```

---

## Summary

This HLD defines a complete, production-grade microservices platform for an On-Demand Car Wash system. The design covers:

- **9 microservices** each with a clear single responsibility
- **Polyglot persistence** (MySQL for transactional, MongoDB for documents, Redis for real-time)
- **Event-driven architecture** using RabbitMQ for loose coupling
- **API Gateway + Eureka** for routing and dynamic service discovery
- **JWT-based security** with role-based access control
- **Cross-cutting concerns** (Swagger, Logging, Auditing, Exception Handling, Monitoring) applied consistently
- **Docker-ready** deployment via Docker Compose

The next step is to produce individual **Low Level Design (LLD) documents** for each microservice using this HLD as the architectural foundation.

---
*Document Version 1.0 | On-Demand Car Wash System | March 2026*
