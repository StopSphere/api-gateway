# API Gateway

Central entry point for all client requests in the ShopSphere e-commerce platform. Routes HTTP traffic to microservices using dynamic service discovery from Eureka, load balancing, and intelligent request routing based on path predicates.

## Service Overview

The API Gateway is the single entry point for all client applications. It abstracts the complexity of the distributed microservices architecture by providing a unified interface, handling service discovery via Eureka, load balancing requests across service instances, and routing based on request paths. Clients communicate exclusively with the gateway on port 8083, which then routes to the appropriate backend service (User, Product, Order, Inventory, or Payment Service).

---

## Tech Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| **Language** | Java | 21 |
| **Framework** | Spring Boot | 4.0.6 |
| **Gateway Library** | Spring Cloud Gateway | 2025.1.1 |
| **Load Balancing** | Spring Cloud LoadBalancer | - |
| **Service Registry** | Netflix Eureka Client | - |
| **Reactor Framework** | Project Reactor | - |
| **Testing** | JUnit 5, Reactor Test | - |

---

## Request Flow Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                   API GATEWAY REQUEST FLOW                          │
└─────────────────────────────────────────────────────────────────────┘

CLIENT REQUESTS
┌────────────────────────────────────┐
│  External Client                   │
│  (Mobile, Web, Desktop, API)       │
└────────────────┬────────────────────┘
                 │ HTTP Request
                 ▼
┌────────────────────────────────────────────────────────────────┐
│  API GATEWAY (Port 8083)                                       │
│  ├─ Receives request                                          │
│  ├─ Matches path predicate                                    │
│  ├─ Query Eureka: Where is this service?                      │
│  ├─ Load Balance: Pick instance (Round Robin)                 │
│  └─ Forward request to service                               │
└────────────────┬──────────┬──────────┬──────────┬──────────────┘
                 │          │          │          │
    ┌────────────▼┐ ┌───────▼──────┐ ┌─────────▼─┐ ┌────────▼────┐
    │ USER        │ │ PRODUCT      │ │ ORDER     │ │ INVENTORY   │
    │ SERVICE     │ │ SERVICE      │ │ SERVICE   │ │ SERVICE     │
    │ :8080       │ │ :8081        │ │ :8082     │ │ :8084       │
    └────────────┬┘ └───────┬──────┘ └─────────┬─┘ └────────┬────┘
                 │          │          │          │
    ┌────────────┴──────────┴──────────┴──────────┴────────┐
    │              Gateway Aggregates Responses           │
    │                                                     │
    └────────────────────────┬────────────────────────────┘
                             │ HTTP Response
                             ▼
                    ┌─────────────────────┐
                    │  Response to Client │
                    └─────────────────────┘
```

---

## Request Routes & Predicates

The gateway routes requests based on path patterns. Each route is mapped to a backend service registered in Eureka.

| Path Pattern | Service | Port | Purpose |
|---|---|---|---|
| `/v1/api/users/**` | USER-SERVICE | 8080 | User management, authentication, registration |
| `/v1/api/products/**` | PRODUCT-SERVICE | 8081 | Product catalog, search, details |
| `/v1/api/orders/**` | ORDER-SERVICE | 8082 | Order creation, status, history |
| `/v1/api/inventory/**` | INVENTORY-SERVICE | 8084 | Stock management, reservations |
| `/v1/api/payments/**` | PAYMENT-SERVICE | 8085 | Payment processing (event-driven) |

---

## Gateway Features

### 1. **Dynamic Service Discovery**
- Integrates with Eureka Service Registry
- Queries Eureka to discover registered service instances
- No hardcoded service endpoints required
- Automatic detection when services start/stop

### 2. **Load Balancing**
- Uses `lb://SERVICE-NAME` URI scheme for load balancing
- Distributes requests round-robin across available instances
- Handles service instance failures gracefully
- Supports multiple instances per service

### 3. **Intelligent Routing**
- Path-based predicates: Routes based on URL path patterns
- Supports wildcards: `/v1/api/products/**` matches all product endpoints
- Case-sensitive path matching
- Clean separation of concerns across services

### 4. **Request/Response Handling**
- Transparent HTTP request/response proxying
- Preserves HTTP headers and query parameters
- Supports all HTTP methods (GET, POST, PUT, DELETE, PATCH)
- Async handling using Project Reactor (non-blocking)

---

## Configuration

### application.yml - Full Configuration

```yaml
server:
  port: 8083

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      server:
        webflux:
          routes:
            # User Service Routes
            - id: user-service
              uri: lb://USER-SERVICE
              predicates:
                - Path=/v1/api/users/**

            # Product Service Routes
            - id: product-service
              uri: lb://PRODUCT-SERVICE
              predicates:
                - Path=/v1/api/products/**

            # Order Service Routes
            - id: order-service
              uri: lb://ORDER-SERVICE
              predicates:
                - Path=/v1/api/orders/**

            # Inventory Service Routes
            - id: inventory-service
              uri: lb://INVENTORY-SERVICE
              predicates:
                - Path=/v1/api/inventory/**

            # Payment Service Routes
            - id: payment-service
              uri: lb://PAYMENT-SERVICE
              predicates:
                - Path=/v1/api/payments/**

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
    fetch-registry: true
    register-with-eureka: true
  instance:
    prefer-ip-address: true
    hostname: api-gateway
```

### Key Configuration Details

- **Gateway Port**: `8083` — All client requests arrive here
- **Routes Definition**: WebFlux-based declarative routing
- **Service Discovery**: Eureka integration enabled with `register-with-eureka: true`
- **Registry Fetch**: `fetch-registry: true` ensures local cache of service instances
- **Load Balancer**: Prefix `lb://` enables client-side load balancing

---

## How to Run

### Prerequisites
- **Java 21** installed
- **Docker** (for dependencies)
- **Eureka Discovery Server** running on `localhost:8761`
- **At least one backend service** (User, Product, Order, Inventory, or Payment Service) running

### Start the Gateway

```powershell
# Build the project
.\gradlew.bat clean build

# Run the gateway
.\gradlew.bat bootRun
```

The gateway will:
1. Start on `http://localhost:8083`
2. Connect to Eureka at `http://localhost:8761/eureka`
3. Begin accepting client requests
4. Discover and route to backend services based on path

### Verify Gateway is Running

```powershell
# Check gateway health
curl http://localhost:8083/actuator/health

# Should return 200 OK with health status
```

---

## Usage Examples

### Access User Service (through Gateway)

```bash
# Login endpoint
curl -X POST http://localhost:8083/v1/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password123"}'

# Register endpoint
curl -X POST http://localhost:8083/v1/api/users/register \
  -H "Content-Type: application/json" \
  -d '{"name":"John","email":"john@example.com","password":"pass"}'
```

### Access Product Service (through Gateway)

```bash
# List all products
curl http://localhost:8083/v1/api/products

# Get specific product
curl http://localhost:8083/v1/api/products/{productId}

# Search products
curl http://localhost:8083/v1/api/products/search?keyword=laptop
```

### Access Order Service (through Gateway)

```bash
# Create order (requires JWT)
curl -X POST http://localhost:8083/v1/api/orders \
  -H "Authorization: Bearer <jwt-token>" \
  -H "Content-Type: application/json" \
  -d '{"productId":"550e8400-e29b-41d4-a716-446655440000","quantity":2}'

# Get order by ID
curl http://localhost:8083/v1/api/orders/{orderId} \
  -H "Authorization: Bearer <jwt-token>"

# List orders
curl http://localhost:8083/v1/api/orders \
  -H "Authorization: Bearer <jwt-token>"
```

### Access Inventory Service (through Gateway)

```bash
# Check stock
curl http://localhost:8083/v1/api/inventory/{productId}

# Reserve inventory
curl -X POST http://localhost:8083/v1/api/inventory/reserve \
  -H "Content-Type: application/json" \
  -d '{"productId":"550e8400-e29b-41d4-a716-446655440000","quantity":5}'
```

### Access Payment Service (through Gateway)

```bash
# Note: Payment Service is event-driven, no direct REST calls
# But routes are configured for potential future REST endpoints
curl http://localhost:8083/v1/api/payments/{paymentId}
```

---

## How Routing Works

### Step-by-Step Request Flow

1. **Client Request Arrives**
   - Client makes HTTP request to `http://localhost:8083/v1/api/products/123`

2. **Path Matching**
   - Gateway evaluates predicates for each route
   - `/v1/api/products/**` matches the path
   - `product-service` route is selected

3. **Service Discovery**
   - Gateway queries local Eureka cache
   - Finds all instances of `PRODUCT-SERVICE`
   - Example: `[192.168.1.100:8081, 192.168.1.101:8081]`

4. **Load Balancing**
   - LoadBalancer picks one instance (round-robin)
   - First request → instance 1
   - Second request → instance 2
   - Third request → instance 1 (cycles)

5. **Request Forwarding**
   - Gateway forwards to selected instance
   - Preserves: HTTP method, headers, query params, body
   - URL becomes: `http://192.168.1.100:8081/v1/api/products/123`

6. **Response Handling**
   - Backend service processes request
   - Returns response to gateway
   - Gateway forwards response to client
   - Client receives response as if it called the service directly

### Load Balancing Strategy

```
Multiple instances of Order Service registered:
  - ORDER-SERVICE instance 1: 192.168.1.50:8082
  - ORDER-SERVICE instance 2: 192.168.1.51:8082
  - ORDER-SERVICE instance 3: 192.168.1.52:8082

Incoming requests distributed:
  Request 1 → Instance 1
  Request 2 → Instance 2
  Request 3 → Instance 3
  Request 4 → Instance 1 (cycles back)
  ...
```

---

## Error Handling & Troubleshooting

### Gateway Starts but Routes Fail

**Problem**: `503 Service Unavailable` or `404 Not Found`

**Solution**:
1. Ensure Eureka Discovery Server is running on `localhost:8761`
2. Verify backend services are registered in Eureka
3. Check backend service names match route configuration:
   - `USER-SERVICE` (must be uppercase)
   - `PRODUCT-SERVICE`
   - `ORDER-SERVICE`
   - `INVENTORY-SERVICE`
   - `PAYMENT-SERVICE`

```bash
# Check Eureka status
curl http://localhost:8761/eureka/apps
```

### Request Timeout

**Problem**: Gateway requests to backend services timeout

**Solution**:
1. Verify backend service is running and healthy
2. Check network connectivity between gateway and service
3. Verify service is registered in Eureka

### Path Not Recognized

**Problem**: Request to `/v1/api/users/me` returns 404

**Possible Cause**: Path predicate is case-sensitive or doesn't match pattern

**Solution**:
1. Check exact path in route configuration
2. Use correct path format: `/v1/api/users/**` (note trailing `/**`)
3. Verify no whitespace in path

---

## Architecture Integration

```
┌────────────────────────────────────────────────────────────┐
│           SHOPSPHERE PLATFORM ARCHITECTURE                 │
└────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│  Client Applications                     │
│  ├─ Mobile App (iOS/Android)            │
│  ├─ Web Browser                         │
│  ├─ Desktop App                         │
│  └─ Third-party API Clients             │
└──────────────────┬───────────────────────┘
                   │ All HTTP requests to :8083
                   ▼
         ┌─────────────────────┐
         │  API GATEWAY :8083  │
         └─────────────────────┘
                   │
        ┌──────────┼──────────┬──────────┬──────────┐
        │          │          │          │          │
        ▼          ▼          ▼          ▼          ▼
     ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
     │USER  │  │PROD  │  │ORDER │  │INV   │  │PAY   │
     │:8080 │  │:8081 │  │:8082 │  │:8084 │  │:8085 │
     └──────┘  └──────┘  └──────┘  └──────┘  └──────┘
        │          │          │          │          │
        └──────────┼──────────┼──────────┼──────────┘
                   │
                   ▼
         ┌──────────────────────┐
         │  EUREKA REGISTRY     │
         │  Service Discovery   │
         │  :8761               │
         └──────────────────────┘
                   │
         ┌─────────┼──────────┬─────────┐
         ▼         ▼          ▼         ▼
      ┌─────┐  ┌────┐     ┌────┐   ┌─────┐
      │MySQL│  │Kafka│    │Redis│  │FS  │
      └─────┘  └────┘     └────┘   └─────┘
```

---

## Testing

### Run Tests

```powershell
# Run all gateway tests
.\gradlew.bat test

# Run with verbose output
.\gradlew.bat test --info

# View test report
# Reports generated in: build/reports/tests/test/index.html
```

### Manual Testing with curl

```powershell
# Test gateway is running
curl http://localhost:8083/actuator/health

# Test routing to a service
curl http://localhost:8083/v1/api/products

# Test with headers
curl -H "Authorization: Bearer token123" http://localhost:8083/v1/api/orders

# Test POST request
curl -X POST http://localhost:8083/v1/api/users/register `
  -H "Content-Type: application/json" `
  -d '{"name":"Test","email":"test@example.com","password":"pass"}'
```

---

## Performance Notes

- **Non-Blocking Architecture**: Uses Project Reactor for async request handling
- **Service Discovery Caching**: Local cache of Eureka registry reduces latency
- **Load Distribution**: Round-robin balancing prevents service overload
- **Connection Pooling**: Reuses HTTP connections to backend services
- **Throughput**: Can handle thousands of concurrent requests
- **Latency**: Minimal overhead (~1-5ms per request) for routing and discovery

---

## Project Structure

```
src/main/java/com/shopsphere/api_gateway/
└── (Currently minimal; configuration-driven routing)

src/main/resources/
└── application.yml                       # Route & Eureka configuration

Dockerfile                                # Container image definition
build.gradle                              # Dependencies and build config
```

---

## Status

✅ Route requests to User Service  
✅ Route requests to Product Service  
✅ Route requests to Order Service  
✅ Route requests to Inventory Service  
✅ Route requests to Payment Service  
✅ Eureka service discovery and registration  
✅ Load balancing across service instances  
✅ Dynamic service instance detection  
✅ Transparent request/response proxying  
✅ Health check endpoint (`/actuator/health`)  

---

**Built for ShopSphere E-commerce Platform**

