# Microservices Architecture: A Deep Dive

## Table of Contents
1. [Introduction & Monolithic vs Microservices](#introduction)
2. [Core Concepts](#core-concepts)
3. [Scalability](#scalability)
4. [Database Strategy](#database-strategy)
5. [Communication Patterns](#communication-patterns)
6. [API Gateway](#api-gateway)
7. [Service Discovery](#service-discovery)
8. [Logging & Monitoring](#logging-monitoring)
9. [12-Factor Apps](#12-factor-apps)
10. [Case Studies](#case-studies)

---

## 1. Introduction & Monolithic vs Microservices {#introduction}

### What is Monolithic Architecture?

**Definition:** A monolithic application is a single, tightly-integrated codebase where all business logic resides in one application.

**Characteristics:**
- Single database for all features
- All code deployed together
- Shared memory space
- Single technology stack
- Scales vertically (add more CPU/RAM)

**Example - E-Commerce Monolith:**
```
┌─────────────────────────────────────────┐
│         Monolithic App                   │
├─────────────────────────────────────────┤
│ ┌──────────────────────────────────────┐ │
│ │ Authentication Module                │ │
│ ├──────────────────────────────────────┤ │
│ │ Product Catalog Module               │ │
│ ├──────────────────────────────────────┤ │
│ │ Order Processing Module              │ │
│ ├──────────────────────────────────────┤ │
│ │ Payment Module                       │ │
│ ├──────────────────────────────────────┤ │
│ │ Notification Module                  │ │
│ └──────────────────────────────────────┘ │
│              (Spring Boot App)            │
└─────────────────────────────────────────┘
         ↓
    PostgreSQL Database (Single)
```

**Problems with Monolith:**
- Bug in one module → entire app crashes
- Scaling one feature requires scaling entire app
- Technology locked: can't use Node.js for one feature and Java for another
- Large codebase becomes difficult to maintain
- Deployment risk: any change needs full app restart
- Team bottleneck: everyone modifying same codebase

### What is Microservices Architecture?

**Definition:** Breaking a monolithic application into smaller, independent services, each handling a specific business capability and communicating via APIs.

**Characteristics:**
- Database per service (polyglot persistence)
- Independent deployment
- Service-to-service communication via APIs
- Mixed technology stacks allowed
- Horizontal scaling (add more instances)
- Loose coupling, high cohesion

**Example - E-Commerce Microservices:**
```
┌────────────────┐
│   API Gateway  │
└────────┬───────┘
         │
    ┌────┴────┬──────────┬──────────┬──────────┐
    │          │          │          │          │
┌───▼──┐  ┌───▼──┐  ┌───▼──┐  ┌───▼──┐  ┌───▼──┐
│Auth  │  │Prod  │  │Order │  │Pay   │  │Notif │
│Svc   │  │Svc   │  │Svc   │  │Svc   │  │Svc   │
└───┬──┘  └───┬──┘  └───┬──┘  └───┬──┘  └───┬──┘
    │         │         │         │         │
┌───▼──┐ ┌───▼──┐ ┌───▼──┐ ┌───▼──┐ ┌───▼──┐
│Auth  │ │Prod  │ │Order │ │Pay   │ │Notif │
│DB    │ │DB    │ │DB    │ │DB    │ │DB    │
└──────┘ └──────┘ └──────┘ └──────┘ └──────┘
```

**Advantages:**
- Independent scaling: if orders surge, scale only Order Service
- Technology flexibility: use Python for ML, Java for payments
- Fault isolation: payment service crash doesn't affect products
- Faster deployment: deploy Auth Service without affecting others
- Team autonomy: different teams own different services

---

## 2. Core Concepts {#core-concepts}

### 2.1 DNS (Domain Name System)

**What is DNS?**
DNS translates human-readable domain names into IP addresses that computers understand.

**How DNS Works:**

```
1. User types: myecommerce.com in browser
2. Browser queries DNS Resolver (usually ISP)
3. DNS Resolver queries Root Nameserver
   → "Where is .com?"
4. Root responds: "Ask TLD server at 192.0.34.161"
5. Resolver queries TLD Server
   → "Where is myecommerce.com?"
6. TLD responds: "Ask authoritative server at 198.51.44.4"
7. Resolver queries Authoritative Server
8. Authoritative Server responds: "54.12.32.18"
9. Resolver returns IP to browser
10. Browser connects to 54.12.32.18
```

**Real-World DNS Setup (AWS Route 53):**

**Example:** Suppose you own `shopnow.com`

```
shopnow.com → 52.1.24.89  (API Gateway in us-east-1)
www.shopnow.com → 52.1.24.89
api.shopnow.com → 34.5.67.89 (Different region)
cdn.shopnow.com → CloudFront Distribution
```

**DNS Features Used in Microservices:**

| Feature | Purpose |
|---------|---------|
| A Records | Map domain to IPv4 |
| CNAME | Alias one domain to another |
| Load Balancing | Route 53 routes based on latency, geolocation |
| Failover | If primary datacenter down, switch to secondary |
| Health Checks | Remove unhealthy endpoints from DNS responses |

**Why DNS Matters in Microservices:**
- Clients don't care about individual service IPs
- DNS abstracts infrastructure complexity
- Enables geographic distribution & failover
- Dynamic service discovery integration

---

### 2.2 Load Balancers

**What is a Load Balancer?**

A Load Balancer (LB) distributes incoming network traffic across multiple backend servers to ensure:
- **High Availability:** If one server fails, others handle traffic
- **Scalability:** Handle more concurrent users
- **Performance:** Distribute load evenly

**Visual Flow:**
```
                    User Requests
                          │
                          ▼
                  ┌───────────────┐
                  │ Load Balancer │
                  └───────┬───────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
    ┌────────┐        ┌────────┐        ┌────────┐
    │Server 1│        │Server 2│        │Server 3│
    │(50ms)  │        │(50ms)  │        │(50ms)  │
    └────────┘        └────────┘        └────────┘
```

**Types of Load Balancers:**

#### Layer 4 (Transport Layer) - NLB
- Operates at TCP/UDP level
- Ultra-high performance, millions of requests/sec
- Doesn't inspect content, just forwards packets
- Best for: Gaming, IoT, non-HTTP protocols

**Example Use Case:**
```
Network Load Balancer (TCP 443)
    │
    ├─ Instance 1: MySQL (Port 3306)
    ├─ Instance 2: MySQL (Port 3306)
    └─ Instance 3: MySQL (Port 3306)

Client connects → NLB assigns to one instance (round-robin)
```

#### Layer 7 (Application Layer) - ALB
- Inspects HTTP headers, URLs, hostnames
- Route based on URL paths or hostnames
- Slightly higher latency but intelligent routing
- Best for: Web apps, microservices, RESTful APIs

**Example Use Case:**
```
Application Load Balancer (HTTP/HTTPS)

/api/products → Product Service (3 instances)
/api/orders   → Order Service (5 instances)
/api/auth     → Auth Service (2 instances)
/images/*     → CloudFront CDN

Client Request → ALB inspects URL path → Routes to correct service
```

**Real-World Example - Netflix Architecture:**

```
User Request to netflix.com
    ↓
Route 53 (DNS)
    ↓
Global Accelerator (optimizes routing)
    ↓
Network Load Balancer (TCP layer)
    ↓
Application Load Balancer (HTTP layer)
    ↓
┌─────────────────────────────────────────┐
│        Microservices Cluster            │
├─────────────────────────────────────────┤
│ Auth Service  │  Recommendation Service │
│ (2 instances) │  (10 instances)         │
│ Video Service │  User Profile Service   │
│ (15 instances)│  (5 instances)          │
└─────────────────────────────────────────┘
```

**Load Balancer Features:**

| Feature | Description | Example |
|---------|-------------|---------|
| **Health Checks** | Periodically test backend health | HTTP GET /health every 5s |
| **Sticky Sessions** | Route client to same server | User session persistence |
| **SSL Termination** | LB handles encryption/decryption | Reduces CPU load on servers |
| **Auto-Scaling** | Automatically add/remove instances | Scales from 5 to 20 servers on Black Friday |
| **Cross-Zone** | Distribute across availability zones | Survivable if entire AZ fails |

**Health Check Example:**

```
Load Balancer every 30 seconds:
    ├─ Instance 1 → GET /health → 200 OK ✓ (Healthy)
    ├─ Instance 2 → GET /health → 200 OK ✓ (Healthy)
    ├─ Instance 3 → GET /health → Connection Timeout ✗ (Unhealthy)
    └─ Instance 4 → GET /health → 503 Error ✗ (Unhealthy)

New traffic routing:
    Instance 1: 50% (1/2)
    Instance 2: 50% (1/2)
    Instance 3: 0% (marked unhealthy)
    Instance 4: 0% (marked unhealthy)
```

---

### 2.3 Authentication vs Authorization

These are often confused but are different:

#### Authentication (AuthN)
**Definition:** Verifying who you are (identity verification)

**Process:**
```
User → Enter credentials (email + password)
    ↓
Auth Service → Check in database
    ↓
Match? → YES → Issue JWT Token
    ↓
Return: {
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user_id": 12345,
  "name": "John Doe"
}
```

**Example:**
- Login page: "Who are you?" → "I'm john@example.com, password is 'secret123'"
- Facebook Login: Grant access with OAuth2

#### Authorization (AuthZ)
**Definition:** Determining what authenticated user can do (permission verification)

**Process:**
```
User (authenticated with token) → Request resource
    ↓
Check Token → Extract user_id
    ↓
Lookup user roles: [admin, user, moderator]
    ↓
Check if user has permission for resource
    ↓
YES → 200 OK, return resource
NO  → 403 Forbidden
```

**Example:**
- Authenticated as John: Can delete only your own posts
- Authenticated as Admin: Can delete any post
- Authenticated as Viewer: Cannot delete anything

**Real-World Scenario:**

```
Facebook Login (AuthN):
├─ You log in with email + password
├─ Facebook verifies credentials
└─ Issues you a session token

Post Operations (AuthZ):
├─ Edit your post → Allowed (you own it)
├─ Delete friend's post → Denied (you don't own it)
├─ Delete post from 3 years ago → Allowed (you own it)
└─ Admin Delete any post → Allowed (admin role)
```

**Implementation in Microservices:**

```
API Gateway (Central Authentication Point)
    ├─ Check if request has valid JWT token
    ├─ If no token → 401 Unauthorized
    ├─ If invalid → 401 Unauthorized
    ├─ If valid → Extract user_id, roles
    └─ Pass to microservice with Authorization header

Microservice (Check Permissions)
    ├─ Extract user_id from header
    ├─ Check if user has permission for operation
    ├─ If YES → Proceed
    └─ If NO → 403 Forbidden
```

---

## 3. Scalability {#scalability}

### Three Dimensions of Scalability (Scale Cube)

The Scale Cube model describes three independent ways to scale applications:

#### 1. Horizontal Scaling (X-Axis) - Scale by Cloning
**Definition:** Add more instances of the same service

**Scenario:**
```
Peak Traffic (Black Friday Sale)

Before:
    Load Balancer
        ├─ Product Service Instance 1 (CPU 95%)
        ├─ Product Service Instance 2 (CPU 92%)
        └─ Product Service Instance 3 (CPU 98%)
    → Cannot handle more requests, 50% requests time out

After (Auto-scale to 10 instances):
    Load Balancer
        ├─ Product Service Instance 1 (CPU 15%)
        ├─ Product Service Instance 2 (CPU 18%)
        ├─ ... (5 more instances)
        └─ Product Service Instance 10 (CPU 20%)
    → Handles 10x more traffic, CPU healthy
```

**Challenges:**
- Multiple instances need to be stateless
- Session data must be stored externally (Redis)
- Database becomes bottleneck

#### 2. Vertical Scaling (Z-Axis) - Scale by Size
**Definition:** Increase resources (CPU, RAM) of single server

**Scenario:**
```
Before:
    Order Service: 4 CPU, 8 GB RAM
    Processing: 100 orders/sec, CPU 85%

After (Upgrade):
    Order Service: 16 CPU, 64 GB RAM
    Processing: 400 orders/sec, CPU 40%
```

**Limitations:**
- Hardware has physical limits
- Expensive (16 CPU server >> 4x cost of 4 CPU server)
- Doesn't help with availability (single point of failure)
- Can't fix architectural problems

#### 3. Data Partitioning (Y-Axis) - Scale by Splitting
**Definition:** Partition data by function or data range

**Scenario A - Function-Based (Database per Service):**
```
Monolithic Database (Bottleneck):
    Users Table → 500M rows
    Orders Table → 200M rows
    Products Table → 5M rows
    Reviews Table → 1B rows

Partitioned (Database per Service):
    Auth Service DB: Users (500M rows)
    Order Service DB: Orders (200M rows)
    Product Service DB: Products (5M rows) + Reviews (1B rows)
    → Each DB is smaller, queries faster
```

**Scenario B - Data-Based (Sharding):**
```
Users Table (1B rows) - TOO BIG for single database

Sharded by User ID:
    Shard 1 (user_id 0-99M) → MySQL Instance 1
    Shard 2 (user_id 100M-199M) → MySQL Instance 2
    Shard 3 (user_id 200M-299M) → MySQL Instance 3
    Shard 4 (user_id 300M-1B) → MySQL Instance 4

Find User 50M: Goes to Shard 1 (fast lookup)
Find User 250M: Goes to Shard 3 (fast lookup)
```

**Example - Amazon's Approach:**

```
Amazon Prime Video Platform

Horizontal Scaling (X-Axis):
    Peak hours (8 PM): 500k concurrent users
    ├─ Video Streaming: 200 instances
    ├─ Recommendation: 100 instances
    ├─ Search: 50 instances
    └─ Payment: 30 instances

Vertical Scaling (Z-Axis):
    Recommendation Service needs ML computations
    ├─ Instance Type: GPU-optimized (p3.8xlarge)
    ├─ 8 NVIDIA V100 GPUs
    └─ 256 GB RAM

Data Partitioning (Y-Axis):
    User Viewing History (10B records)
    ├─ Shard by country → Different DynamoDB instances
    ├─ Shard by region → Geo-distributed
    └─ Shard by user_id → Within region
```

**Comparison Table:**

| Dimension | Method | Cost | Complexity | Limits |
|-----------|--------|------|-----------|--------|
| X-Axis | Add instances | Low | Low | Network I/O |
| Z-Axis | Bigger servers | High | Low | Physical limits |
| Y-Axis | Partition data | Medium | High | Complexity |

---

### 3.1 Multiple Instances & Data Consistency

**The Challenge: Two Instances, One Database**

When you add multiple instances of the same service, a critical question arises:
> "If I add data in Instance 1, will Instance 2 see it?"

**Answer: YES, but with considerations**

```
Scenario: Order Service with 2 Instances

┌──────────────────────────────────────────────────────┐
│ Order Service Instance 1        Order Service Instance 2 │
│ (In New York)                   (In California)       │
└──┬───────────────────────────────────────────────┬──┘
   │                                               │
   └─────────────────┬─────────────────────────────┘
                     │
              ┌──────▼──────┐
              │  PostgreSQL │
              │  (Database) │
              └─────────────┘
```

**Real-World Scenario: Create Order**

```
Time 1 (08:00:00):
    Client sends POST /orders {user_id: 123, items: [...]}
        ↓ (Load Balancer routes to Instance 1)
    Instance 1 executes:
        1. Validate order
        2. INSERT into orders table
        3. UPDATE inventory
        4. Return {order_id: 5001}

Database State:
    orders table: [... 5000 rows ..., {id: 5001, user_id: 123, ...}]

Time 2 (08:00:05):
    Same client sends GET /orders/5001
        ↓ (Load Balancer routes to Instance 2)
    Instance 2 executes:
        1. SELECT * FROM orders WHERE id = 5001
        2. Return {id: 5001, user_id: 123, ...}
        ✓ Instance 2 sees the data created by Instance 1

Result: ✓ Data IS visible across instances
```

**Why This Works:**
- All instances share SAME database
- Database is single source of truth
- Any instance can read/write
- Consistency maintained at database level

**Deep Dive: Session Data Scenario**

```
❌ WRONG: Each Instance Has Local Session Cache
┌─────────────────────────────────────────┐
│ Instance 1 (Memory)    Instance 2 (Memory)  │
│ user_123: {           user_123: {          │
│   token: abc123        token: xyz789        │
│   logged_in: true      logged_in: false     │
│   cart: [...]          cart: [...]          │
│ }                      }                    │
└─────────────────────────────────────────┘

Problem:
  1. User logs in via Instance 1 → Sets user_123 in Instance 1 memory
  2. User refreshes browser → Load Balancer routes to Instance 2
  3. Instance 2 checks memory → No user_123 data → Logs out!
  4. User must log in again 😞

✓ CORRECT: Use External Session Store (Redis)
┌─────────────────────────────────────────┐
│ Instance 1           Instance 2           │
│ (Stateless)          (Stateless)          │
│ Processes request    Processes request    │
│ Stores session → Redis   ← Reads session  │
└──────────┬──────────────────┬────────────┘
           │                  │
           └────────┬─────────┘
                    │
              ┌─────▼──────┐
              │   Redis    │
              │  (In-mem)  │
              └────────────┘

Flow:
  1. User logs in via Instance 1
  2. Instance 1 → SET user_123: {token, cart, ...} in Redis
  3. User refreshes → Load Balancer routes to Instance 2
  4. Instance 2 → GET user_123 from Redis ✓
  5. User still logged in 👍
```

**Consistency Guarantees:**

| Scenario | Guarantee | Explanation |
|----------|-----------|-------------|
| Read after write (same instance) | Strong | MySQL ensures durability |
| Read after write (different instance) | Strong* | All instances read same DB |
| Concurrent writes | ACID with transactions | Database level locks |
| Session data (local memory) | Weak | Lost on instance restart |
| Session data (Redis) | Strong | Persisted in external store |

**Critical Insight for Microservices:**
```
Never store critical data in instance memory!

❌ Bad:
    let orders = [];
    app.post('/order', (req, res) => {
        orders.push(req.body);  // Lost on restart!
    });

✓ Good:
    app.post('/order', (req, res) => {
        await db.orders.insert(req.body);  // Persisted
        res.json({success: true});
    });
```

---

## 4. Database Strategy {#database-strategy}

### 4.1 Why One Database Per Service?

#### Problem with Shared Database

```
Monolithic Shared Database Approach:

┌─────────────────────────────────────────┐
│     All Microservices                   │
├─────────────────────────────────────────┤
│ Auth | Orders | Products | Payments     │
│ Inventory | Reviews | Shipping          │
└──────────────┬──────────────────────────┘
               │
         ┌─────▼──────┐
         │PostgreSQL  │
         │ Shared DB  │
         └────────────┘
```

**Issues:**

1. **Tight Coupling**
   ```
   Product Service schema change
   → Might break Orders Service
   → Inventory Service might fail
   → Deploy requires coordination
   ```

2. **Scalability Bottleneck**
   ```
   During Black Friday:
   - Product DB queries: 10,000 qps
   - Order DB queries: 5,000 qps
   - Total: 15,000 qps
   
   Single database can handle 50,000 qps
   But schema lock during migration blocks everything!
   
   Problem: Can't scale Product Service separately
   ```

3. **Technology Lock-In**
   ```
   What if we want:
   - PostgreSQL for Orders (transactions)
   - MongoDB for Products (flexible schema)
   - DynamoDB for Sessions (fast key-value)
   
   With shared DB → Forced to use one DB for all
   ```

4. **Operational Issues**
   ```
   Database maintenance:
   - Backup fails → All services down
   - Slow query from Orders Service
     → Locks tables → Products Service crawls
   - Migration on Products table
     → Affects all dependent services
   ```

#### Solution: Database Per Service

```
Microservices with Independent Databases:

┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│Auth Svc  │  │Order Svc │  │Prod Svc  │  │Pay Svc   │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │             │
┌────▼─────┐  ┌───▼─────┐  ┌───▼─────┐  ┌───▼─────┐
│Auth DB   │  │Order DB  │  │Prod DB  │  │Pay DB   │
│Postgres  │  │Postgres  │  │MongoDB  │  │Cassandra│
└──────────┘  └──────────┘  └─────────┘  └────────┘
```

**Benefits:**

1. **Independent Scaling**
   ```
   Black Friday surge:
   - Order Service needs 10 instances
   - Product Service needs 5 instances
   - Each with their own DB
   
   Scale Order DB:
   - Read replicas
   - Sharding
   - Without affecting Products
   ```

2. **Technology Freedom**
   ```
   Auth Service: PostgreSQL (ACID transactions)
   Products: MongoDB (flexible schema)
   Cache: Redis (fast in-memory)
   Analytics: Elasticsearch (full-text search)
   ```

3. **Loose Coupling**
   ```
   Modify Order Service schema
   → No impact on other services
   → Deploy independently
   → Rollback only affects Order Service
   ```

4. **Resilience**
   ```
   Order DB goes down
   → Users can still browse products
   → Can't checkout (expected)
   → Better than entire app being down
   ```

---

### 4.2 What is Redis?

**Definition:** Redis (Remote Dictionary Server) is an in-memory data structure store.

**Key Characteristics:**
- **In-Memory:** Data stored in RAM (fast, ~1 microsecond access)
- **Key-Value:** Simple key → value mapping
- **Data Structures:** Strings, Lists, Sets, Sorted Sets, Hashes, Streams
- **Persistence:** Optional disk backup
- **Single-threaded:** But very fast (100,000 operations/sec per core)

**Real-World Analogy:**
```
Database (PostgreSQL): Bookshelf in a warehouse
- Storage: Huge (100TB)
- Access Time: Slow (need to fetch from warehouse)
- Use for: Long-term storage

Redis: Desk on your office desk
- Storage: Small (32GB)
- Access Time: Fast (reach and grab)
- Use for: Frequently accessed items
```

**Common Redis Use Cases:**

#### 1. Session Storage
```
User Login Flow:
1. User provides credentials
2. Server verifies, creates session
3. SET user_123:session {
    "token": "abc123",
    "logged_in": true,
    "cart": [product1, product2],
    "last_activity": "2024-10-30T10:30:00Z"
}
4. Set expiry: EXPIRE 3600 (1 hour)
5. Return session_id to client

Subsequent Requests:
GET user_123:session → Returns session in microseconds
```

**Speed Comparison:**
```
PostgreSQL lookup: 10ms
MongoDB lookup: 5ms
Redis lookup: 0.1ms (100x faster!)
```

#### 2. Caching
```
E-commerce Product Page:

First Request:
1. GET /products/5001
2. Not in Redis
3. Query PostgreSQL → 50ms
4. SET products:5001 {name, price, ...}
5. EXPIRE 3600
6. Return to user

Second Request (within 1 hour):
1. GET /products/5001
2. Found in Redis → 0.1ms
3. Return immediately

Result: 500x faster!
```

#### 3. Real-Time Analytics
```
Track trending products:
ZADD trending_products 1 "iPhone"
ZADD trending_products 2 "iPad"
ZADD trending_products 5 "AirPods"
ZADD trending_products 3 "MacBook"

ZREVRANGE trending_products 0 2
→ ["AirPods", "MacBook", "iPad"]

Update counts in real-time without hitting DB
```

#### 4. Rate Limiting
```
API Rate Limit: 100 requests per minute per user

User makes request:
1. GET user_123:rate_limit → 45 (current count)
2. If count < 100:
   - INCR user_123:rate_limit → 46
   - Allow request
3. Else:
   - Deny request (429 Too Many Requests)

EXPIRE user_123:rate_limit 60 (reset every minute)
```

#### 5. Pub/Sub (Real-Time Notifications)
```
Notification System:

Publisher (Order Service):
    PUBLISH order_notifications {
        "order_id": 5001,
        "status": "shipped"
    }

Subscribers (Listening):
    Notification Service
    User's Browser (WebSocket)
    Analytics Service
    
All receive notification in milliseconds
```

**Real-World Example - Amazon Prime Checkout:**

```
User clicks "Buy Now"
↓
API Gateway → Payment Service Instance 1
↓
Payment Service checks rate limit:
    GET user_123:requests_per_min → 89
    INCR user_123:requests_per_min → 90
    EXPIRE 60
↓
Payment Service fetches user session:
    GET user_123:session → {
        "user_id": 123,
        "address_id": 5,
        "payment_method": "card_4521"
    } (from Redis)
↓
Retrieve cached product:
    GET product:apple_watch → {
        "price": 399.99,
        "stock": 1000
    } (from Redis)
↓
Process payment
↓
Update Redis cache:
    INCR user_123:orders_count
    DEL user_123:cart (clear cart)
↓
Return confirmation
```

---

### 4.3 SQL vs NoSQL Databases

#### SQL (Relational) Databases

**Examples:** PostgreSQL, MySQL, Oracle, SQL Server

**Structure:**
```
Users Table:
┌─────────┬──────────────┬────────────┬──────────┐
│ id (PK) │ email        │ name       │ age      │
├─────────┼──────────────┼────────────┼──────────┤
│ 1       │ john@ex.com  │ John Doe   │ 30       │
│ 2       │ jane@ex.com  │ Jane Smith │ 28       │
└─────────┴──────────────┴────────────┴──────────┘

Orders Table:
┌──────────┬────────────┬─────────┬──────────────┐
│ id (PK)  │ user_id(FK)│ total   │ created_at   │
├──────────┼────────────┼─────────┼──────────────┤
│ 5001     │ 1          │ 399.99  │ 2024-10-30   │
│ 5002     │ 2          │ 199.99  │ 2024-10-30   │
└──────────┴────────────┴─────────┴──────────────┘
```

**Key Features:**

1. **ACID Transactions**
   ```
   Transfer Money Between Accounts:
   BEGIN TRANSACTION
       Account A -= $100 (UPDATE account SET balance = balance - 100)
       Account B += $100 (UPDATE account SET balance = balance + 100)
   COMMIT
   
   Atomicity: Both succeed or both fail (not partial)
   Consistency: Total money remains same
   Isolation: Concurrent transfers don't interfere
   Durability: Data survives system crash
   ```

2. **Fixed Schema**
   ```
   BEFORE INSERT:
   Table structure: id, email, name, age
   
   Invalid: INSERT (id, email, name, age, phone)
   → ERROR: column "phone" doesn't exist
   
   MUST ALTER TABLE first:
   ALTER TABLE users ADD COLUMN phone VARCHAR(20);
   ```

3. **Complex Queries with Joins**
   ```
   Find all orders by users over 25 years old:
   
   SELECT u.name, o.id, o.total
   FROM users u
   JOIN orders o ON u.id = o.user_id
   WHERE u.age > 25
   AND o.created_at > '2024-01-01'
   ORDER BY o.total DESC
   ```

4. **Vertical Scaling**
   ```
   10M rows: 4 CPU, 16GB RAM
   100M rows: 16 CPU, 128GB RAM
   1B rows: Too big for single machine
   
   Solution: Upgrade server (expensive)
   ```

**Use Cases:**
- Financial systems (bank transfers)
- E-commerce orders (consistency critical)
- Inventory management
- User management with complex queries

---

#### NoSQL (Non-Relational) Databases

**Examples:** MongoDB, DynamoDB, Cassandra, Redis

**Types of NoSQL:**

##### 1. Document Databases (MongoDB)
```
Collection: Users

{
  "_id": 1,
  "email": "john@ex.com",
  "name": "John Doe",
  "age": 30,
  "address": {
    "street": "123 Main St",
    "city": "New York"
  },
  "orders": [5001, 5002, 5003],
  "preferences": {
    "theme": "dark",
    "notifications": true
  }
}

Flexible Schema: Can add fields without ALTER TABLE
```

##### 2. Key-Value Stores (Redis, DynamoDB)
```
Key → Value

user:123 → {name: "John", age: 30}
product:456 → {name: "iPhone", price: 999}
session:abc123 → {token: "xyz789", expires: 3600}

Direct lookup: O(1) time
No joins, no complex queries
Ultra-fast
```

##### 3. Wide-Column (Cassandra, HBase)
```
User ID    | Email         | Name       | Age    | City
-----------|---------------|------------|--------|--------
1          | john@ex.com   | John Doe   | 30     | NYC
2          | jane@ex.com   | Jane Smith | 28     | LA
1000000    | bob@ex.com    | Bob Johnson| 45     | Chicago

Optimized for:
- Time-series data
- Analytics queries
- Horizontal scaling
```

##### 4. Search Engines (Elasticsearch)
```
Used for: Full-text search, logging, analytics

Document:
{
  "id": 123,
  "title": "Best iPhone Cases",
  "body": "Protect your iPhone...",
  "tags": ["iphone", "cases", "protection"]
}

Query: "Best iPhone" → Instantly finds relevant documents
Inverted index makes full-text search fast
```

**NoSQL Advantages:**

1. **Horizontal Scaling**
   ```
   1B documents: 1 server
   10B documents: Add 10 servers (each handles 1B)
   
   Data automatically sharded across servers
   ```

2. **Flexible Schema**
   ```
   MongoDB document 1:
   {id: 1, name: "John", age: 30}
   
   MongoDB document 2:
   {id: 2, name: "Jane", age: 28, phone: "555-1234"}
   
   ✓ Both valid! No schema enforcement
   ```

3. **High Performance**
   ```
   DynamoDB: 100,000+ writes/sec
   Cassandra: 1,000,000+ writes/sec
   
   No complex joins (fast queries)
   No transactions (eventual consistency)
   ```

**Comparison Table:**

| Feature | SQL | NoSQL |
|---------|-----|-------|
| Schema | Fixed | Flexible |
| Joins | Yes, powerful | No |
| Transactions | ACID | Eventual consistency |
| Scaling | Vertical | Horizontal |
| Use Case | Structured data | Unstructured, high-scale |
| Learning curve | Medium | Easy for key-value |
| Examples | Banking, ERP | Social media, IoT, logs |

---

### 4.4 Polyglot Persistence (Using Multiple Database Types)

**Definition:** Using different database technologies for different services based on their needs.

**Real-World Microservices Stack:**

```
┌────────────────────────────────────────────┐
│         E-Commerce Platform                │
└────────────────────────────────────────────┘

Auth Service
    ├─ Use: PostgreSQL
    ├─ Why: ACID transactions, user credentials critical
    └─ Query: SELECT * FROM users WHERE email = ?

Product Service
    ├─ Use: MongoDB
    ├─ Why: Flexible schema (different product types)
    └─ Query: db.products.find({category: "electronics"})

Order Service
    ├─ Use: PostgreSQL
    ├─ Why: Transactions, inventory tracking
    └─ Query: Complex JOINs for order reports

Search Service
    ├─ Use: Elasticsearch
    ├─ Why: Full-text search, faceting
    └─ Query: "iPhone 15 Pro" → Returns relevant products

Session Service
    ├─ Use: Redis
    ├─ Why: In-memory, fast, expiring data
    └─ Query: GET user_123:session

Analytics Service
    ├─ Use: BigQuery / Snowflake
    ├─ Why: OLAP, huge volumes, complex analysis
    └─ Query: SELECT date, SUM(revenue) GROUP BY date

Notification Service
    ├─ Use: DynamoDB
    ├─ Why: Fast key-value, serverless
    └─ Query: Query user notifications by timestamp
```

**Advantages:**
- Each service optimized for its workload
- Auth Service needs ACID → PostgreSQL
- Search needs full-text → Elasticsearch
- Sessions need speed → Redis
- Analytics needs scale → BigQuery

**Challenges:**
- Operational complexity (managing 5 databases)
- Team skill diversity needed
- Cross-service data consistency (eventual consistency)

---

## 5. Communication Patterns {#communication-patterns}

### 5.1 Synchronous Communication (Request-Response)

**Definition:** Caller waits for response before continuing.

```
Request Timeline:
┌────────────┐                          ┌─────────────┐
│   Client   │                          │  Service B  │
└────────┬───┘                          └──────┬──────┘
         │                                     │
         │──── 1. POST /order ────────────────>│
         │   Wait... (blocked)                 │
         │                                     │
         │  2. Validate stock                  │
         │  3. Reserve inventory               │
         │  4. Process payment                 │
         │                                     │
         │<──── 3. 200 OK, Order#5001 ─────────│
         │   (0.5 seconds later)               │
         │
    Can now use order_id in next call
```

**Real-World Scenario: Book Purchase**

```
Customer clicks "Buy Now"
    ↓
API Gateway (1ms)
    ↓
Order Service - Create order record (2ms)
    ↓ (RPC call - synchronous)
Inventory Service - Check stock (3ms)
    ↓ (RPC call - synchronous)
Payment Service - Process card (100ms - network I/O)
    ↓ (RPC call - synchronous)
Shipping Service - Schedule pickup (5ms)
    ↓
Return order confirmation
    ↓
Total time: ~110ms

User sees "Order confirmed!" after 110ms
```

**Implementation:**

```
REST API (HTTP/JSON):
POST /orders
{
  "user_id": 123,
  "items": [
    {"product_id": 456, "quantity": 1}
  ]
}

Service receives → Processes → Responds
HTTP Status Codes:
- 200 OK: Success
- 400 Bad Request: Validation failed
- 500 Internal Server Error: Service crashed
```

**Advantages:**
- Simple to understand and implement
- Real-time feedback
- Good for user-facing operations
- Built-in error handling via HTTP status

**Disadvantages:**
- **Tight Coupling:** Order Service must know Inventory Service location
- **Latency:** Waits for slowest service
- **Cascading Failures:** If Payment Service is down, entire order fails
- **Scalability Issues:** Network calls add latency

**Cascading Failure Example:**

```
Scenario: Payment Service degrades (100ms → 5 seconds)

┌─────────────────────────────────────────┐
│ Black Friday - 10,000 users/sec         │
└─────────────────────────────────────────┘

Without Payment issue:
  - Order Service processes: 10,000 orders/sec
  - Response time: 100ms
  - Works fine

With Payment Service slow (5 seconds):
  - Order Service processes: 10,000 orders/sec
  - Each order waits 5 seconds for payment
  - Order Service threads fill up (can handle ~100 concurrent)
  - After 100 orders, Order Service queue full
  - Remaining orders timeout or are rejected
  - Users see 50% failure rate

Cascade: 1 slow service → Multiple services collapse
```

---

### 5.2 Asynchronous Communication (Event-Driven)

**Definition:** Caller sends message and continues without waiting for response.

```
Timeline:
┌────────────┐                    ┌──────────────┐
│   Client   │                    │ Event Broker │
└────────┬───┘                    └──────┬───────┘
         │                               │
         │─ 1. Order Created ──────────>│ (1ms)
         │ (Message: {order_id, user_id, │
         │  items, payment_method})      │
         │                               │
    Returns 202 Accepted               │
    (Order saved, not processed yet)    │
                                        │
    Order Service → Message Broker:
    Inventory Service ← Listens
    Payment Service ← Listens
    Shipping Service ← Listens
```

**Real-World Scenario: Book Purchase (Async)**

```
Customer clicks "Buy Now"
    ↓
Order Service creates order in database
    ↓
Order Service publishes "OrderCreated" event
    ↓
Returns 202 Accepted to customer (10ms)
"Order received! We'll process it soon."
    ↓ (Parallel processing)
    ├─ Inventory Service sees "OrderCreated"
    │  → Reserves stock
    │  → Publishes "StockReserved"
    │
    ├─ Payment Service sees "OrderCreated"
    │  → Processes payment (100ms)
    │  → Publishes "PaymentProcessed"
    │
    └─ Shipping Service sees "PaymentProcessed"
       → Schedules pickup
       → Publishes "ShippingScheduled"
    
Customer gets email: "Payment confirmed, ships tomorrow"
(Later, in background)
```

**Two Asynchronous Patterns:**

#### Pattern 1: Publish-Subscribe (Pub-Sub)

**Structure:**
```
Publisher (Order Service)
    │
    └─── publish("order.created") ──────┐
                                        │
                    ┌───────────────────┴────────────────┐
                    │                                    │
             ┌──────▼─────┐                    ┌──────────▼───┐
             │ Subscriber  │                    │  Subscriber  │
             │ Inventory   │                    │  Payment     │
             └─────────────┘                    └──────────────┘

Event: "order.created"
{
  "order_id": 5001,
  "user_id": 123,
  "items": [...]
}

Multiple subscribers receive simultaneously
No guaranteed order of processing
```

**Real Implementation (Python with Redis Pub-Sub):**

```python
# Publisher (Order Service)
import redis

broker = redis.Redis()

def create_order(user_id, items):
    order = save_order_to_db(user_id, items)
    
    # Publish event
    broker.publish("order.channel", json.dumps({
        "event": "order.created",
        "order_id": order.id,
        "user_id": user_id,
        "items": items
    }))
    
    return {"order_id": order.id}

# Subscriber (Inventory Service)
pubsub = broker.pubsub()
pubsub.subscribe("order.channel")

for message in pubsub.listen():
    if message['type'] == 'message':
        event = json.loads(message['data'])
        if event['event'] == 'order.created':
            reserve_inventory(event)
```

**Characteristics:**
- Multiple consumers can process simultaneously
- No guaranteed delivery order
- Fire-and-forget
- Decoupled: Publisher doesn't know subscribers

#### Pattern 2: Queue Model (Point-to-Point)

**Structure:**
```
Publisher (Order Service)
    │
    └─── Enqueue Message ──┐
                           │
                    ┌──────▼──────┐
                    │   Queue     │
                    │ [msg1, msg2] │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ Consumer    │
                    │ (Inventory) │
                    └─────────────┘

Queue: [msg1, msg2, msg3, ...]
  ├─ msg1 → Consumer 1 (removed after processing)
  ├─ msg2 → Consumer 2 (removed after processing)
  └─ msg3 → Waiting...
```

**Real Implementation (Python with RabbitMQ):**

```python
# Publisher (Order Service)
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue='order_queue')

def create_order(user_id, items):
    order = save_order_to_db(user_id, items)
    
    # Send message to queue
    channel.basic_publish(
        exchange='',
        routing_key='order_queue',
        body=json.dumps({
            "order_id": order.id,
            "user_id": user_id,
            "items": items
        })
    )
    
    return {"order_id": order.id}

# Consumer (Inventory Service Instance 1)
def callback(ch, method, properties, body):
    event = json.loads(body)
    reserve_inventory(event)
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.queue_declare(queue='order_queue')
channel.basic_consume(queue='order_queue', on_message_callback=callback)
channel.start_consuming()

# Consumer (Inventory Service Instance 2)
# Same code - both instances compete for messages
```

**Characteristics:**
- Single message handled by one consumer
- Guaranteed delivery (message stored until ack)
- Load balanced: If 2 instances, each gets ~50% messages
- FIFO: Messages processed in order

**Comparison:**

| Aspect | Pub-Sub | Queue |
|--------|---------|-------|
| **Multiple Consumers** | All receive copy | One consumes |
| **Delivery** | No guarantee | Guaranteed |
| **Use Case** | Notifications, events | Task distribution |
| **Example** | Stock price updates | Payment processing |
| **Fault Handling** | Message lost if no subscriber | Retried if failed |

---

### 5.3 When to Use Synchronous vs Asynchronous

**Decision Matrix:**

| Scenario | Use | Reason |
|----------|-----|--------|
| Login | Sync | Need immediate response |
| Payment | Sync | User waits on confirmation page |
| Send Email | Async | User doesn't need to wait |
| Generate Report | Async | Takes time, email when done |
| Check Stock | Sync | Real-time inventory check |
| Send Notification | Async | Background task |
| Create Order | Async* | Accept order, process later |
| Search Products | Sync | Real-time results needed |

**Real Example - Amazon Checkout:**

```
Synchronous (Real-time):
  1. Check payment method validity → Sync (2ms)
  2. Verify address format → Sync (1ms)
  3. Check stock availability → Sync (5ms)
  4. Process payment → Sync (100ms)
  
  Total: ~108ms (User waits)

Asynchronous (Background):
  1. Send confirmation email → Async (scheduled)
  2. Update analytics → Async (scheduled)
  3. Generate invoice PDF → Async (scheduled)
  4. Schedule shipping → Async (scheduled)
  5. Update inventory cache → Async (scheduled)
  
  User doesn't wait for these
```

---

### 5.4 Eventually Consistent Systems

**Definition:** System doesn't guarantee data is immediately consistent across all services, but will be consistent "eventually."

**Scenario: Email with Different Databases**

```
Time 0 (10:00:00):
Order Service (PostgreSQL):
    INSERT INTO orders VALUES (5001, user_123, 'pending')
    ✓ Visible immediately

Notification Service (MongoDB):
    {order_id: 5001, status: 'pending'} [NOT YET INSERTED]

Time 5 (10:00:05):
Message Broker processes queue
Notification Service consumes message
    {order_id: 5001, status: 'pending'} [NOW INSERTED]
    ✓ Now visible

Timeline:
     10:00:00                    10:00:05
        │                            │
   Order Created          Notification Synced
        │◄────── 5 seconds delay ──►│
        
During those 5 seconds: DATA IS INCONSISTENT
User queries might see order in one service but not another
```

**Real-World Impact:**

```
User checks order status immediately (10:00:01):
  ├─ Order Service: Order found ✓
  ├─ Notification Service: Order NOT found yet ✗
  └─ Different databases show different data!

User checks again 10 seconds later (10:00:10):
  ├─ Order Service: Order found ✓
  ├─ Notification Service: Order found ✓
  └─ Finally consistent ✓
```

**Example: Facebook Like Button**

```
You like a post on Facebook

Immediate:
  ├─ Your local timeline: Shows +1 like (optimistic update)
  └─ Post author's Facebook DB: ??? (not updated yet)

After 1 second:
  ├─ Your timeline: +1 like ✓
  ├─ Post author's DB: +1 like ✓
  ├─ Analytics DB: ??? (not updated yet)
  └─ Notification sent to author?

After 10 seconds:
  ├─ Everything is consistent ✓
  ├─ All databases have +1 like ✓
  ├─ Author was notified ✓
  └─ Analytics updated ✓

Facebook accepted this tradeoff:
  - Immediate feedback to user (better UX)
  - Eventual consistency across services (scalable)
```

**Why Accept Eventual Consistency?**

```
Strong Consistency (Immediate):
  - All databases sync before returning
  - Expensive: Every write must wait for all DB updates
  - Slow: 100ms → 500ms per operation
  - Doesn't scale

Eventual Consistency (Delayed):
  - Return immediately
  - Sync in background
  - Cheap: Async processing
  - Scales to millions of users

Trade-off:
  Slower response time → Better scalability
  vs.
  Immediate response → Limited scalability
  
For likes, comments, notifications: Users accept delay
For payments, orders: Must be strongly consistent
```

**Implementing Eventual Consistency Correctly:**

```
Order Created Event:

┌─────────────────────────────────────────┐
│       Order Service (PostgreSQL)        │
├─────────────────────────────────────────┤
│ BEGIN TRANSACTION                       │
│   INSERT order INTO orders              │
│   INSERT event INTO event_log:          │
│   {                                     │
│     event_id: 10001,                    │
│     type: "order.created",              │
│     data: {...},                        │
│     status: "pending"                   │
│   }                                     │
│ COMMIT                                  │
└─────────────────────────────────────────┘

Event Processor Service:
SELECT * FROM event_log WHERE status = 'pending'
For each event:
  ├─ Publish to Kafka
  ├─ Inventory Service processes
  ├─ Payment Service processes
  ├─ Mark as delivered
  └─ UPDATE event_log SET status = 'delivered'

Retry Logic:
If event fails:
  ├─ Retry after 5 minutes
  ├─ Retry after 15 minutes
  ├─ Alert if fails 5 times
  └─ Eventually succeeds or escalates

Result: Guarantees eventual consistency
```

---

### 5.5 Why Load Balancers Work Differently in Sync vs Async

#### Synchronous Communication: Load Balancer Required

**Scenario:** Checkout page with 3 Payment Service instances

```
Without Load Balancer (WRONG):
┌──────────────────────────────┐
│ Client App has hardcoded URL │
│ "payment-service:3000"       │
└──────────┬───────────────────┘
           │
    But which instance?
    
Instance 1: 192.168.1.10:3000 ✓
Instance 2: 192.168.1.11:3000 ✓
Instance 3: 192.168.1.12:3000 ✓

If client only knows URL, not which IP
→ Can't reach service
→ Fails
```

**With Load Balancer (CORRECT):**

```
┌────────────────────────────────────┐
│    Client                          │
│    POST http://payment-svc.local   │
└────────────┬───────────────────────┘
             │
        ┌────▼────────┐
        │ Load Balancer│ (192.168.1.5)
        │ :3000       │
        └────┬───────┬┴──────────┐
             │       │          │
    Request 1│   Req 2│      Req 3│
             │       │          │
     ┌───────▼┐ ┌───▼───┐ ┌────▼───┐
     │Inst 1  │ │Inst 2 │ │ Inst 3 │
     │:3001   │ │:3002  │ │ :3003  │
     └────────┘ └───────┘ └────────┘

Load Balancer:
  Request 1 → Instance 1 (0% loaded)
  Request 2 → Instance 2 (0% loaded)
  Request 3 → Instance 3 (0% loaded)
  Request 4 → Instance 1 (now 25% loaded)
  
Balanced! ✓
```

**Why Necessary for Sync:**
1. Client needs to know WHERE to send request
2. Multiple instances but client can only target one
3. Need central distribution point
4. Enables health checks and failover

---

#### Asynchronous Communication: Load Balancer NOT Needed

**Scenario:** Order events with 3 Inventory Service instances

```
Architecture:
┌──────────────────┐
│ Order Service    │
│ Publishes        │
│ "order.created"  │
└────────┬─────────┘
         │
    ┌────▼──────────┐
    │ Message Broker│
    │ (Kafka)       │
    │ Topic: orders │
    └────┬──────┬───┴──────┐
         │      │          │
    ┌────▼┐ ┌──▼───┐ ┌───▼───┐
    │Inst1│ │Inst 2│ │ Inst 3│
    │     │ │      │ │       │
    └─────┘ └──────┘ └───────┘

Broker distributes messages:
  Message 1 → Instance 1 gets (consumes, removes from queue)
  Message 2 → Instance 2 gets (consumes, removes from queue)
  Message 3 → Instance 3 gets (consumes, removes from queue)
  Message 4 → Instance 1 gets (Instance 1 is ready again)
  
Automatic balancing! No central LB needed
```

**Key Differences:**

| Sync | Async |
|------|-------|
| **Client initiated** | Message broker initiated |
| **Client must route** | Broker routes automatically |
| **Needs LB** | LB Not needed |
| **Fail-fast** | Retry-friendly |
| **Latency sensitive** | Batch-friendly |

**Why Async Doesn't Need LB:**

```
Queue Model:
  [Message 1, Message 2, Message 3, Message 4]
  
Instance 1 (idle): Gets Message 1
Instance 2 (idle): Gets Message 2
Instance 3 (busy): Waiting for current work
Instance 1 (done): Gets Message 3

Consumers pull as they're ready
Natural load balancing!

vs.

Sync Model:
  Request comes in NOW
  Client expects response in 100ms
  No time to wait for instance to be ready
  MUST have LB to route immediately
```

---

## 6. API Gateway {#api-gateway}

**Definition:** Single entry point for all client requests to microservices.

```
Architecture Without API Gateway:
┌─────────┐         ┌─────────────────────────┐
│ Web App │         │ Mobile App              │
└────┬────┘         └────┬────────────────────┘
     │                   │
     ├──► Auth Service ◄─┤
     │                   │
     ├──► Order Service◄─┤
     │                   │
     ├──► Product Svc◄───┤
     │                   │
     └──► Payment Svc◄───┘

Problems:
  - Auth logic duplicated in app
  - Mobile/Web must know all service URLs
  - Each client implements rate limiting
  - Each client handles authentication
```

**Architecture With API Gateway:**

```
┌──────────────────────────────────────────┐
│ Web App          Mobile App              │
└──────┬───────────────────────┬───────────┘
       │                       │
       └───────────────┬───────┘
                       │
            ┌──────────▼────────────┐
            │   API Gateway        │
            │ ┌──────────────────┐ │
            │ │ Authentication   │ │
            │ │ Rate Limiting    │ │
            │ │ Request Routing  │ │
            │ │ Caching          │ │
            │ │ Logging          │ │
            │ └──────────────────┘ │
            └──────┬──┬──┬──┬──────┘
                   │  │  │  │
            ┌──────▼──▼──▼──▼──┐
            │ Microservices    │
            │ Auth, Order,     │
            │ Product, Payment │
            └─────────────────┘
```

**Responsibilities of API Gateway:**

### 1. Authentication & Authorization

```
Request: GET /api/orders/5001

API Gateway:
1. Check Authorization header
   GET /api/orders/5001
   Header: Authorization: Bearer eyJhbGc...
   
2. Validate JWT token
   Token valid? ✓
   Expired? ✗
   Signature valid? ✓
   
3. Extract user_id from token: user_id = 123
   
4. Pass to Order Service with header:
   X-User-ID: 123
   X-User-Roles: [admin, user]
   
5. Order Service trusts gateway
   → Doesn't re-verify token
   → Uses user_id from header
```

### 2. Routing

```
Request comes to gateway.api.com
┌─────────────────────────────────┐
│ /api/products/* → Product Svc   │
│ /api/orders/* → Order Svc       │
│ /api/auth/* → Auth Svc          │
│ /api/payments/* → Payment Svc   │
│ /health → Health Check Service  │
└─────────────────────────────────┘

GET /api/products/5001
  → Routes to Product Service

POST /api/orders
  → Routes to Order Service

GET /health
  → Routes to Health Check Service
```

### 3. Rate Limiting

```
API Gateway tracks requests per user/IP

User Token: abc123
  Request 1: 8:00:00 ✓ (1/100)
  Request 2: 8:00:01 ✓ (2/100)
  ...
  Request 100: 8:01:00 ✓ (100/100)
  Request 101: 8:01:01 ✗ 429 Too Many Requests

Prevents:
  - DDoS attacks
  - Runaway clients
  - API abuse
```

### 4. Caching

```
GET /api/products/5001?include=details

API Gateway:
1. Check cache: "products:5001:details"
2. If hit → Return cached response (1ms)
3. If miss → Route to Product Service (100ms)
   → Cache response for 1 hour
   → Return to client

Cache hit on repeat requests saves 99ms!
```

### 5. Request/Response Transformation

```
Client sends:
POST /api/orders {
  "product_id": 5001,
  "quantity": 2
}// filepath: /Users/mayankvashisht/Desktop/classes/DevOps & Cloud/mircoservice/Microservices-Deep-Dive.md
# Microservices Architecture: A Deep Dive

## Table of Contents
1. [Introduction & Monolithic vs Microservices](#introduction)
2. [Core Concepts](#core-concepts)
3. [Scalability](#scalability)
4. [Database Strategy](#database-strategy)
5. [Communication Patterns](#communication-patterns)
6. [API Gateway](#api-gateway)
7. [Service Discovery](#service-discovery)
8. [Logging & Monitoring](#logging-monitoring)
9. [12-Factor Apps](#12-factor-apps)
10. [Case Studies](#case-studies)

---

## 1. Introduction & Monolithic vs Microservices {#introduction}

### What is Monolithic Architecture?

**Definition:** A monolithic application is a single, tightly-integrated codebase where all business logic resides in one application.

**Characteristics:**
- Single database for all features
- All code deployed together
- Shared memory space
- Single technology stack
- Scales vertically (add more CPU/RAM)

**Example - E-Commerce Monolith:**
```
┌─────────────────────────────────────────┐
│         Monolithic App                   │
├─────────────────────────────────────────┤
│ ┌──────────────────────────────────────┐ │
│ │ Authentication Module                │ │
│ ├──────────────────────────────────────┤ │
│ │ Product Catalog Module               │ │
│ ├──────────────────────────────────────┤ │
│ │ Order Processing Module              │ │
│ ├──────────────────────────────────────┤ │
│ │ Payment Module                       │ │
│ ├──────────────────────────────────────┤ │
│ │ Notification Module                  │ │
│ └──────────────────────────────────────┘ │
│              (Spring Boot App)            │
└─────────────────────────────────────────┘
         ↓
    PostgreSQL Database (Single)
```

**Problems with Monolith:**
- Bug in one module → entire app crashes
- Scaling one feature requires scaling entire app
- Technology locked: can't use Node.js for one feature and Java for another
- Large codebase becomes difficult to maintain
- Deployment risk: any change needs full app restart
- Team bottleneck: everyone modifying same codebase

### What is Microservices Architecture?

**Definition:** Breaking a monolithic application into smaller, independent services, each handling a specific business capability and communicating via APIs.

**Characteristics:**
- Database per service (polyglot persistence)
- Independent deployment
- Service-to-service communication via APIs
- Mixed technology stacks allowed
- Horizontal scaling (add more instances)
- Loose coupling, high cohesion

**Example - E-Commerce Microservices:**
```
┌────────────────┐
│   API Gateway  │
└────────┬───────┘
         │
    ┌────┴────┬──────────┬──────────┬──────────┐
    │          │          │          │          │
┌───▼──┐  ┌───▼──┐  ┌───▼──┐  ┌───▼──┐  ┌───▼──┐
│Auth  │  │Prod  │  │Order │  │Pay   │  │Notif │
│Svc   │  │Svc   │  │Svc   │  │Svc   │  │Svc   │
└───┬──┘  └───┬──┘  └───┬──┘  └───┬──┘  └───┬──┘
    │         │         │         │         │
┌───▼──┐ ┌───▼──┐ ┌───▼──┐ ┌───▼──┐ ┌───▼──┐
│Auth  │ │Prod  │ │Order │ │Pay   │ │Notif │
│DB    │ │DB    │ │DB    │ │DB    │ │DB    │
└──────┘ └──────┘ └──────┘ └──────┘ └──────┘
```

**Advantages:**
- Independent scaling: if orders surge, scale only Order Service
- Technology flexibility: use Python for ML, Java for payments
- Fault isolation: payment service crash doesn't affect products
- Faster deployment: deploy Auth Service without affecting others
- Team autonomy: different teams own different services

---

## 2. Core Concepts {#core-concepts}

### 2.1 DNS (Domain Name System)

**What is DNS?**
DNS translates human-readable domain names into IP addresses that computers understand.

**How DNS Works:**

```
1. User types: myecommerce.com in browser
2. Browser queries DNS Resolver (usually ISP)
3. DNS Resolver queries Root Nameserver
   → "Where is .com?"
4. Root responds: "Ask TLD server at 192.0.34.161"
5. Resolver queries TLD Server
   → "Where is myecommerce.com?"
6. TLD responds: "Ask authoritative server at 198.51.44.4"
7. Resolver queries Authoritative Server
8. Authoritative Server responds: "54.12.32.18"
9. Resolver returns IP to browser
10. Browser connects to 54.12.32.18
```

**Real-World DNS Setup (AWS Route 53):**

**Example:** Suppose you own `shopnow.com`

```
shopnow.com → 52.1.24.89  (API Gateway in us-east-1)
www.shopnow.com → 52.1.24.89
api.shopnow.com → 34.5.67.89 (Different region)
cdn.shopnow.com → CloudFront Distribution
```

**DNS Features Used in Microservices:**

| Feature | Purpose |
|---------|---------|
| A Records | Map domain to IPv4 |
| CNAME | Alias one domain to another |
| Load Balancing | Route 53 routes based on latency, geolocation |
| Failover | If primary datacenter down, switch to secondary |
| Health Checks | Remove unhealthy endpoints from DNS responses |

**Why DNS Matters in Microservices:**
- Clients don't care about individual service IPs
- DNS abstracts infrastructure complexity
- Enables geographic distribution & failover
- Dynamic service discovery integration

---

### 2.2 Load Balancers

**What is a Load Balancer?**

A Load Balancer (LB) distributes incoming network traffic across multiple backend servers to ensure:
- **High Availability:** If one server fails, others handle traffic
- **Scalability:** Handle more concurrent users
- **Performance:** Distribute load evenly

**Visual Flow:**
```
                    User Requests
                          │
                          ▼
                  ┌───────────────┐
                  │ Load Balancer │
                  └───────┬───────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
    ┌────────┐        ┌────────┐        ┌────────┐
    │Server 1│        │Server 2│        │Server 3│
    │(50ms)  │        │(50ms)  │        │(50ms)  │
    └────────┘        └────────┘        └────────┘
```

**Types of Load Balancers:**

#### Layer 4 (Transport Layer) - NLB
- Operates at TCP/UDP level
- Ultra-high performance, millions of requests/sec
- Doesn't inspect content, just forwards packets
- Best for: Gaming, IoT, non-HTTP protocols

**Example Use Case:**
```
Network Load Balancer (TCP 443)
    │
    ├─ Instance 1: MySQL (Port 3306)
    ├─ Instance 2: MySQL (Port 3306)
    └─ Instance 3: MySQL (Port 3306)

Client connects → NLB assigns to one instance (round-robin)
```

#### Layer 7 (Application Layer) - ALB
- Inspects HTTP headers, URLs, hostnames
- Route based on URL paths or hostnames
- Slightly higher latency but intelligent routing
- Best for: Web apps, microservices, RESTful APIs

**Example Use Case:**
```
Application Load Balancer (HTTP/HTTPS)

/api/products → Product Service (3 instances)
/api/orders   → Order Service (5 instances)
/api/auth     → Auth Service (2 instances)
/images/*     → CloudFront CDN

Client Request → ALB inspects URL path → Routes to correct service
```

**Real-World Example - Netflix Architecture:**

```
User Request to netflix.com
    ↓
Route 53 (DNS)
    ↓
Global Accelerator (optimizes routing)
    ↓
Network Load Balancer (TCP layer)
    ↓
Application Load Balancer (HTTP layer)
    ↓
┌─────────────────────────────────────────┐
│        Microservices Cluster            │
├─────────────────────────────────────────┤
│ Auth Service  │  Recommendation Service │
│ (2 instances) │  (10 instances)         │
│ Video Service │  User Profile Service   │
│ (15 instances)│  (5 instances)          │
└─────────────────────────────────────────┘
```

**Load Balancer Features:**

| Feature | Description | Example |
|---------|-------------|---------|
| **Health Checks** | Periodically test backend health | HTTP GET /health every 5s |
| **Sticky Sessions** | Route client to same server | User session persistence |
| **SSL Termination** | LB handles encryption/decryption | Reduces CPU load on servers |
| **Auto-Scaling** | Automatically add/remove instances | Scales from 5 to 20 servers on Black Friday |
| **Cross-Zone** | Distribute across availability zones | Survivable if entire AZ fails |

**Health Check Example:**

```
Load Balancer every 30 seconds:
    ├─ Instance 1 → GET /health → 200 OK ✓ (Healthy)
    ├─ Instance 2 → GET /health → 200 OK ✓ (Healthy)
    ├─ Instance 3 → GET /health → Connection Timeout ✗ (Unhealthy)
    └─ Instance 4 → GET /health → 503 Error ✗ (Unhealthy)

New traffic routing:
    Instance 1: 50% (1/2)
    Instance 2: 50% (1/2)
    Instance 3: 0% (marked unhealthy)
    Instance 4: 0% (marked unhealthy)
```

---

### 2.3 Authentication vs Authorization

These are often confused but are different:

#### Authentication (AuthN)
**Definition:** Verifying who you are (identity verification)

**Process:**
```
User → Enter credentials (email + password)
    ↓
Auth Service → Check in database
    ↓
Match? → YES → Issue JWT Token
    ↓
Return: {
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user_id": 12345,
  "name": "John Doe"
}
```

**Example:**
- Login page: "Who are you?" → "I'm john@example.com, password is 'secret123'"
- Facebook Login: Grant access with OAuth2

#### Authorization (AuthZ)
**Definition:** Determining what authenticated user can do (permission verification)

**Process:**
```
User (authenticated with token) → Request resource
    ↓
Check Token → Extract user_id
    ↓
Lookup user roles: [admin, user, moderator]
    ↓
Check if user has permission for resource
    ↓
YES → 200 OK, return resource
NO  → 403 Forbidden
```

**Example:**
- Authenticated as John: Can delete only your own posts
- Authenticated as Admin: Can delete any post
- Authenticated as Viewer: Cannot delete anything

**Real-World Scenario:**

```
Facebook Login (AuthN):
├─ You log in with email + password
├─ Facebook verifies credentials
└─ Issues you a session token

Post Operations (AuthZ):
├─ Edit your post → Allowed (you own it)
├─ Delete friend's post → Denied (you don't own it)
├─ Delete post from 3 years ago → Allowed (you own it)
└─ Admin Delete any post → Allowed (admin role)
```

**Implementation in Microservices:**

```
API Gateway (Central Authentication Point)
    ├─ Check if request has valid JWT token
    ├─ If no token → 401 Unauthorized
    ├─ If invalid → 401 Unauthorized
    ├─ If valid → Extract user_id, roles
    └─ Pass to microservice with Authorization header

Microservice (Check Permissions)
    ├─ Extract user_id from header
    ├─ Check if user has permission for operation
    ├─ If YES → Proceed
    └─ If NO → 403 Forbidden
```

---

## 3. Scalability {#scalability}

### Three Dimensions of Scalability (Scale Cube)

The Scale Cube model describes three independent ways to scale applications:

#### 1. Horizontal Scaling (X-Axis) - Scale by Cloning
**Definition:** Add more instances of the same service

**Scenario:**
```
Peak Traffic (Black Friday Sale)

Before:
    Load Balancer
        ├─ Product Service Instance 1 (CPU 95%)
        ├─ Product Service Instance 2 (CPU 92%)
        └─ Product Service Instance 3 (CPU 98%)
    → Cannot handle more requests, 50% requests time out

After (Auto-scale to 10 instances):
    Load Balancer
        ├─ Product Service Instance 1 (CPU 15%)
        ├─ Product Service Instance 2 (CPU 18%)
        ├─ ... (5 more instances)
        └─ Product Service Instance 10 (CPU 20%)
    → Handles 10x more traffic, CPU healthy
```

**Challenges:**
- Multiple instances need to be stateless
- Session data must be stored externally (Redis)
- Database becomes bottleneck

#### 2. Vertical Scaling (Z-Axis) - Scale by Size
**Definition:** Increase resources (CPU, RAM) of single server

**Scenario:**
```
Before:
    Order Service: 4 CPU, 8 GB RAM
    Processing: 100 orders/sec, CPU 85%

After (Upgrade):
    Order Service: 16 CPU, 64 GB RAM
    Processing: 400 orders/sec, CPU 40%
```

**Limitations:**
- Hardware has physical limits
- Expensive (16 CPU server >> 4x cost of 4 CPU server)
- Doesn't help with availability (single point of failure)
- Can't fix architectural problems

#### 3. Data Partitioning (Y-Axis) - Scale by Splitting
**Definition:** Partition data by function or data range

**Scenario A - Function-Based (Database per Service):**
```
Monolithic Database (Bottleneck):
    Users Table → 500M rows
    Orders Table → 200M rows
    Products Table → 5M rows
    Reviews Table → 1B rows

Partitioned (Database per Service):
    Auth Service DB: Users (500M rows)
    Order Service DB: Orders (200M rows)
    Product Service DB: Products (5M rows) + Reviews (1B rows)
    → Each DB is smaller, queries faster
```

**Scenario B - Data-Based (Sharding):**
```
Users Table (1B rows) - TOO BIG for single database

Sharded by User ID:
    Shard 1 (user_id 0-99M) → MySQL Instance 1
    Shard 2 (user_id 100M-199M) → MySQL Instance 2
    Shard 3 (user_id 200M-299M) → MySQL Instance 3
    Shard 4 (user_id 300M-1B) → MySQL Instance 4

Find User 50M: Goes to Shard 1 (fast lookup)
Find User 250M: Goes to Shard 3 (fast lookup)
```

**Example - Amazon's Approach:**

```
Amazon Prime Video Platform

Horizontal Scaling (X-Axis):
    Peak hours (8 PM): 500k concurrent users
    ├─ Video Streaming: 200 instances
    ├─ Recommendation: 100 instances
    ├─ Search: 50 instances
    └─ Payment: 30 instances

Vertical Scaling (Z-Axis):
    Recommendation Service needs ML computations
    ├─ Instance Type: GPU-optimized (p3.8xlarge)
    ├─ 8 NVIDIA V100 GPUs
    └─ 256 GB RAM

Data Partitioning (Y-Axis):
    User Viewing History (10B records)
    ├─ Shard by country → Different DynamoDB instances
    ├─ Shard by region → Geo-distributed
    └─ Shard by user_id → Within region
```

**Comparison Table:**

| Dimension | Method | Cost | Complexity | Limits |
|-----------|--------|------|-----------|--------|
| X-Axis | Add instances | Low | Low | Network I/O |
| Z-Axis | Bigger servers | High | Low | Physical limits |
| Y-Axis | Partition data | Medium | High | Complexity |

---

### 3.1 Multiple Instances & Data Consistency

**The Challenge: Two Instances, One Database**

When you add multiple instances of the same service, a critical question arises:
> "If I add data in Instance 1, will Instance 2 see it?"

**Answer: YES, but with considerations**

```
Scenario: Order Service with 2 Instances

┌──────────────────────────────────────────────────────┐
│ Order Service Instance 1        Order Service Instance 2 │
│ (In New York)                   (In California)       │
└──┬───────────────────────────────────────────────┬──┘
   │                                               │
   └─────────────────┬─────────────────────────────┘
                     │
              ┌──────▼──────┐
              │  PostgreSQL │
              │  (Database) │
              └─────────────┘
```

**Real-World Scenario: Create Order**

```
Time 1 (08:00:00):
    Client sends POST /orders {user_id: 123, items: [...]}
        ↓ (Load Balancer routes to Instance 1)
    Instance 1 executes:
        1. Validate order
        2. INSERT into orders table
        3. UPDATE inventory
        4. Return {order_id: 5001}

Database State:
    orders table: [... 5000 rows ..., {id: 5001, user_id: 123, ...}]

Time 2 (08:00:05):
    Same client sends GET /orders/5001
        ↓ (Load Balancer routes to Instance 2)
    Instance 2 executes:
        1. SELECT * FROM orders WHERE id = 5001
        2. Return {id: 5001, user_id: 123, ...}
        ✓ Instance 2 sees the data created by Instance 1

Result: ✓ Data IS visible across instances
```

**Why This Works:**
- All instances share SAME database
- Database is single source of truth
- Any instance can read/write
- Consistency maintained at database level

**Deep Dive: Session Data Scenario**

```
❌ WRONG: Each Instance Has Local Session Cache
┌─────────────────────────────────────────┐
│ Instance 1 (Memory)    Instance 2 (Memory)  │
│ user_123: {           user_123: {          │
│   token: abc123        token: xyz789        │
│   logged_in: true      logged_in: false     │
│   cart: [...]          cart: [...]          │
│ }                      }                    │
└─────────────────────────────────────────┘

Problem:
  1. User logs in via Instance 1 → Sets user_123 in Instance 1 memory
  2. User refreshes browser → Load Balancer routes to Instance 2
  3. Instance 2 checks memory → No user_123 data → Logs out!
  4. User must log in again 😞

✓ CORRECT: Use External Session Store (Redis)
┌─────────────────────────────────────────┐
│ Instance 1           Instance 2           │
│ (Stateless)          (Stateless)          │
│ Processes request    Processes request    │
│ Stores session → Redis   ← Reads session  │
└──────────┬──────────────────┬────────────┘
           │                  │
           └────────┬─────────┘
                    │
              ┌─────▼──────┐
              │   Redis    │
              │  (In-mem)  │
              └────────────┘

Flow:
  1. User logs in via Instance 1
  2. Instance 1 → SET user_123: {token, cart, ...} in Redis
  3. User refreshes → Load Balancer routes to Instance 2
  4. Instance 2 → GET user_123 from Redis ✓
  5. User still logged in 👍
```

**Consistency Guarantees:**

| Scenario | Guarantee | Explanation |
|----------|-----------|-------------|
| Read after write (same instance) | Strong | MySQL ensures durability |
| Read after write (different instance) | Strong* | All instances read same DB |
| Concurrent writes | ACID with transactions | Database level locks |
| Session data (local memory) | Weak | Lost on instance restart |
| Session data (Redis) | Strong | Persisted in external store |

**Critical Insight for Microservices:**
```
Never store critical data in instance memory!

❌ Bad:
    let orders = [];
    app.post('/order', (req, res) => {
        orders.push(req.body);  // Lost on restart!
    });

✓ Good:
    app.post('/order', (req, res) => {
        await db.orders.insert(req.body);  // Persisted
        res.json({success: true});
    });
```

---

## 4. Database Strategy {#database-strategy}

### 4.1 Why One Database Per Service?

#### Problem with Shared Database

```
Monolithic Shared Database Approach:

┌─────────────────────────────────────────┐
│     All Microservices                   │
├─────────────────────────────────────────┤
│ Auth | Orders | Products | Payments     │
│ Inventory | Reviews | Shipping          │
└──────────────┬──────────────────────────┘
               │
         ┌─────▼──────┐
         │PostgreSQL  │
         │ Shared DB  │
         └────────────┘
```

**Issues:**

1. **Tight Coupling**
   ```
   Product Service schema change
   → Might break Orders Service
   → Inventory Service might fail
   → Deploy requires coordination
   ```

2. **Scalability Bottleneck**
   ```
   During Black Friday:
   - Product DB queries: 10,000 qps
   - Order DB queries: 5,000 qps
   - Total: 15,000 qps
   
   Single database can handle 50,000 qps
   But schema lock during migration blocks everything!
   
   Problem: Can't scale Product Service separately
   ```

3. **Technology Lock-In**
   ```
   What if we want:
   - PostgreSQL for Orders (transactions)
   - MongoDB for Products (flexible schema)
   - DynamoDB for Sessions (fast key-value)
   
   With shared DB → Forced to use one DB for all
   ```

4. **Operational Issues**
   ```
   Database maintenance:
   - Backup fails → All services down
   - Slow query from Orders Service
     → Locks tables → Products Service crawls
   - Migration on Products table
     → Affects all dependent services
   ```

#### Solution: Database Per Service

```
Microservices with Independent Databases:

┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│Auth Svc  │  │Order Svc │  │Prod Svc  │  │Pay Svc   │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │             │
┌────▼─────┐  ┌───▼─────┐  ┌───▼─────┐  ┌───▼─────┐
│Auth DB   │  │Order DB  │  │Prod DB  │  │Pay DB   │
│Postgres  │  │Postgres  │  │MongoDB  │  │Cassandra│
└──────────┘  └──────────┘  └─────────┘  └────────┘
```

**Benefits:**

1. **Independent Scaling**
   ```
   Black Friday surge:
   - Order Service needs 10 instances
   - Product Service needs 5 instances
   - Each with their own DB
   
   Scale Order DB:
   - Read replicas
   - Sharding
   - Without affecting Products
   ```

2. **Technology Freedom**
   ```
   Auth Service: PostgreSQL (ACID transactions)
   Products: MongoDB (flexible schema)
   Cache: Redis (fast in-memory)
   Analytics: Elasticsearch (full-text search)
   ```

3. **Loose Coupling**
   ```
   Modify Order Service schema
   → No impact on other services
   → Deploy independently
   → Rollback only affects Order Service
   ```

4. **Resilience**
   ```
   Order DB goes down
   → Users can still browse products
   → Can't checkout (expected)
   → Better than entire app being down
   ```

---

### 4.2 What is Redis?

**Definition:** Redis (Remote Dictionary Server) is an in-memory data structure store.

**Key Characteristics:**
- **In-Memory:** Data stored in RAM (fast, ~1 microsecond access)
- **Key-Value:** Simple key → value mapping
- **Data Structures:** Strings, Lists, Sets, Sorted Sets, Hashes, Streams
- **Persistence:** Optional disk backup
- **Single-threaded:** But very fast (100,000 operations/sec per core)

**Real-World Analogy:**
```
Database (PostgreSQL): Bookshelf in a warehouse
- Storage: Huge (100TB)
- Access Time: Slow (need to fetch from warehouse)
- Use for: Long-term storage

Redis: Desk on your office desk
- Storage: Small (32GB)
- Access Time: Fast (reach and grab)
- Use for: Frequently accessed items
```

**Common Redis Use Cases:**

#### 1. Session Storage
```
User Login Flow:
1. User provides credentials
2. Server verifies, creates session
3. SET user_123:session {
    "token": "abc123",
    "logged_in": true,
    "cart": [product1, product2],
    "last_activity": "2024-10-30T10:30:00Z"
}
4. Set expiry: EXPIRE 3600 (1 hour)
5. Return session_id to client

Subsequent Requests:
GET user_123:session → Returns session in microseconds
```

**Speed Comparison:**
```
PostgreSQL lookup: 10ms
MongoDB lookup: 5ms
Redis lookup: 0.1ms (100x faster!)
```

#### 2. Caching
```
E-commerce Product Page:

First Request:
1. GET /products/5001
2. Not in Redis
3. Query PostgreSQL → 50ms
4. SET products:5001 {name, price, ...}
5. EXPIRE 3600
6. Return to user

Second Request (within 1 hour):
1. GET /products/5001
2. Found in Redis → 0.1ms
3. Return immediately

Result: 500x faster!
```

#### 3. Real-Time Analytics
```
Track trending products:
ZADD trending_products 1 "iPhone"
ZADD trending_products 2 "iPad"
ZADD trending_products 5 "AirPods"
ZADD trending_products 3 "MacBook"

ZREVRANGE trending_products 0 2
→ ["AirPods", "MacBook", "iPad"]

Update counts in real-time without hitting DB
```

#### 4. Rate Limiting
```
API Rate Limit: 100 requests per minute per user

User makes request:
1. GET user_123:rate_limit → 45 (current count)
2. If count < 100:
   - INCR user_123:rate_limit → 46
   - Allow request
3. Else:
   - Deny request (429 Too Many Requests)

EXPIRE user_123:rate_limit 60 (reset every minute)
```

#### 5. Pub/Sub (Real-Time Notifications)
```
Notification System:

Publisher (Order Service):
    PUBLISH order_notifications {
        "order_id": 5001,
        "status": "shipped"
    }

Subscribers (Listening):
    Notification Service
    User's Browser (WebSocket)
    Analytics Service
    
All receive notification in milliseconds
```

**Real-World Example - Amazon Prime Checkout:**

```
User clicks "Buy Now"
↓
API Gateway → Payment Service Instance 1
↓
Payment Service checks rate limit:
    GET user_123:requests_per_min → 89
    INCR user_123:requests_per_min → 90
    EXPIRE 60
↓
Payment Service fetches user session:
    GET user_123:session → {
        "user_id": 123,
        "address_id": 5,
        "payment_method": "card_4521"
    } (from Redis)
↓
Retrieve cached product:
    GET product:apple_watch → {
        "price": 399.99,
        "stock": 1000
    } (from Redis)
↓
Process payment
↓
Update Redis cache:
    INCR user_123:orders_count
    DEL user_123:cart (clear cart)
↓
Return confirmation
```

---

### 4.3 SQL vs NoSQL Databases

#### SQL (Relational) Databases

**Examples:** PostgreSQL, MySQL, Oracle, SQL Server

**Structure:**
```
Users Table:
┌─────────┬──────────────┬────────────┬──────────┐
│ id (PK) │ email        │ name       │ age      │
├─────────┼──────────────┼────────────┼──────────┤
│ 1       │ john@ex.com  │ John Doe   │ 30       │
│ 2       │ jane@ex.com  │ Jane Smith │ 28       │
└─────────┴──────────────┴────────────┴──────────┘

Orders Table:
┌──────────┬────────────┬─────────┬──────────────┐
│ id (PK)  │ user_id(FK)│ total   │ created_at   │
├──────────┼────────────┼─────────┼──────────────┤
│ 5001     │ 1          │ 399.99  │ 2024-10-30   │
│ 5002     │ 2          │ 199.99  │ 2024-10-30   │
└──────────┴────────────┴─────────┴──────────────┘
```

**Key Features:**

1. **ACID Transactions**
   ```
   Transfer Money Between Accounts:
   BEGIN TRANSACTION
       Account A -= $100 (UPDATE account SET balance = balance - 100)
       Account B += $100 (UPDATE account SET balance = balance + 100)
   COMMIT
   
   Atomicity: Both succeed or both fail (not partial)
   Consistency: Total money remains same
   Isolation: Concurrent transfers don't interfere
   Durability: Data survives system crash
   ```

2. **Fixed Schema**
   ```
   BEFORE INSERT:
   Table structure: id, email, name, age
   
   Invalid: INSERT (id, email, name, age, phone)
   → ERROR: column "phone" doesn't exist
   
   MUST ALTER TABLE first:
   ALTER TABLE users ADD COLUMN phone VARCHAR(20);
   ```

3. **Complex Queries with Joins**
   ```
   Find all orders by users over 25 years old:
   
   SELECT u.name, o.id, o.total
   FROM users u
   JOIN orders o ON u.id = o.user_id
   WHERE u.age > 25
   AND o.created_at > '2024-01-01'
   ORDER BY o.total DESC
   ```

4. **Vertical Scaling**
   ```
   10M rows: 4 CPU, 16GB RAM
   100M rows: 16 CPU, 128GB RAM
   1B rows: Too big for single machine
   
   Solution: Upgrade server (expensive)
   ```

**Use Cases:**
- Financial systems (bank transfers)
- E-commerce orders (consistency critical)
- Inventory management
- User management with complex queries

---

#### NoSQL (Non-Relational) Databases

**Examples:** MongoDB, DynamoDB, Cassandra, Redis

**Types of NoSQL:**

##### 1. Document Databases (MongoDB)
```
Collection: Users

{
  "_id": 1,
  "email": "john@ex.com",
  "name": "John Doe",
  "age": 30,
  "address": {
    "street": "123 Main St",
    "city": "New York"
  },
  "orders": [5001, 5002, 5003],
  "preferences": {
    "theme": "dark",
    "notifications": true
  }
}

Flexible Schema: Can add fields without ALTER TABLE
```

##### 2. Key-Value Stores (Redis, DynamoDB)
```
Key → Value

user:123 → {name: "John", age: 30}
product:456 → {name: "iPhone", price: 999}
session:abc123 → {token: "xyz789", expires: 3600}

Direct lookup: O(1) time
No joins, no complex queries
Ultra-fast
```

##### 3. Wide-Column (Cassandra, HBase)
```
User ID    | Email         | Name       | Age    | City
-----------|---------------|------------|--------|--------
1          | john@ex.com   | John Doe   | 30     | NYC
2          | jane@ex.com   | Jane Smith | 28     | LA
1000000    | bob@ex.com    | Bob Johnson| 45     | Chicago

Optimized for:
- Time-series data
- Analytics queries
- Horizontal scaling
```

##### 4. Search Engines (Elasticsearch)
```
Used for: Full-text search, logging, analytics

Document:
{
  "id": 123,
  "title": "Best iPhone Cases",
  "body": "Protect your iPhone...",
  "tags": ["iphone", "cases", "protection"]
}

Query: "Best iPhone" → Instantly finds relevant documents
Inverted index makes full-text search fast
```

**NoSQL Advantages:**

1. **Horizontal Scaling**
   ```
   1B documents: 1 server
   10B documents: Add 10 servers (each handles 1B)
   
   Data automatically sharded across servers
   ```

2. **Flexible Schema**
   ```
   MongoDB document 1:
   {id: 1, name: "John", age: 30}
   
   MongoDB document 2:
   {id: 2, name: "Jane", age: 28, phone: "555-1234"}
   
   ✓ Both valid! No schema enforcement
   ```

3. **High Performance**
   ```
   DynamoDB: 100,000+ writes/sec
   Cassandra: 1,000,000+ writes/sec
   
   No complex joins (fast queries)
   No transactions (eventual consistency)
   ```

**Comparison Table:**

| Feature | SQL | NoSQL |
|---------|-----|-------|
| Schema | Fixed | Flexible |
| Joins | Yes, powerful | No |
| Transactions | ACID | Eventual consistency |
| Scaling | Vertical | Horizontal |
| Use Case | Structured data | Unstructured, high-scale |
| Learning curve | Medium | Easy for key-value |
| Examples | Banking, ERP | Social media, IoT, logs |

---

### 4.4 Polyglot Persistence (Using Multiple Database Types)

**Definition:** Using different database technologies for different services based on their needs.

**Real-World Microservices Stack:**

```
┌────────────────────────────────────────────┐
│         E-Commerce Platform                │
└────────────────────────────────────────────┘

Auth Service
    ├─ Use: PostgreSQL
    ├─ Why: ACID transactions, user credentials critical
    └─ Query: SELECT * FROM users WHERE email = ?

Product Service
    ├─ Use: MongoDB
    ├─ Why: Flexible schema (different product types)
    └─ Query: db.products.find({category: "electronics"})

Order Service
    ├─ Use: PostgreSQL
    ├─ Why: Transactions, inventory tracking
    └─ Query: Complex JOINs for order reports

Search Service
    ├─ Use: Elasticsearch
    ├─ Why: Full-text search, faceting
    └─ Query: "iPhone 15 Pro" → Returns relevant products

Session Service
    ├─ Use: Redis
    ├─ Why: In-memory, fast, expiring data
    └─ Query: GET user_123:session

Analytics Service
    ├─ Use: BigQuery / Snowflake
    ├─ Why: OLAP, huge volumes, complex analysis
    └─ Query: SELECT date, SUM(revenue) GROUP BY date

Notification Service
    ├─ Use: DynamoDB
    ├─ Why: Fast key-value, serverless
    └─ Query: Query user notifications by timestamp
```

**Advantages:**
- Each service optimized for its workload
- Auth Service needs ACID → PostgreSQL
- Search needs full-text → Elasticsearch
- Sessions need speed → Redis
- Analytics needs scale → BigQuery

**Challenges:**
- Operational complexity (managing 5 databases)
- Team skill diversity needed
- Cross-service data consistency (eventual consistency)

---

## 5. Communication Patterns {#communication-patterns}

### 5.1 Synchronous Communication (Request-Response)

**Definition:** Caller waits for response before continuing.

```
Request Timeline:
┌────────────┐                          ┌─────────────┐
│   Client   │                          │  Service B  │
└────────┬───┘                          └──────┬──────┘
         │                                     │
         │──── 1. POST /order ────────────────>│
         │   Wait... (blocked)                 │
         │                                     │
         │  2. Validate stock                  │
         │  3. Reserve inventory               │
         │  4. Process payment                 │
         │                                     │
         │<──── 3. 200 OK, Order#5001 ─────────│
         │   (0.5 seconds later)               │
         │
    Can now use order_id in next call
```

**Real-World Scenario: Book Purchase**

```
Customer clicks "Buy Now"
    ↓
API Gateway (1ms)
    ↓
Order Service - Create order record (2ms)
    ↓ (RPC call - synchronous)
Inventory Service - Check stock (3ms)
    ↓ (RPC call - synchronous)
Payment Service - Process card (100ms - network I/O)
    ↓ (RPC call - synchronous)
Shipping Service - Schedule pickup (5ms)
    ↓
Return order confirmation
    ↓
Total time: ~110ms

User sees "Order confirmed!" after 110ms
```

**Implementation:**

```
REST API (HTTP/JSON):
POST /orders
{
  "user_id": 123,
  "items": [
    {"product_id": 456, "quantity": 1}
  ]
}

Service receives → Processes → Responds
HTTP Status Codes:
- 200 OK: Success
- 400 Bad Request: Validation failed
- 500 Internal Server Error: Service crashed
```

**Advantages:**
- Simple to understand and implement
- Real-time feedback
- Good for user-facing operations
- Built-in error handling via HTTP status

**Disadvantages:**
- **Tight Coupling:** Order Service must know Inventory Service location
- **Latency:** Waits for slowest service
- **Cascading Failures:** If Payment Service is down, entire order fails
- **Scalability Issues:** Network calls add latency

**Cascading Failure Example:**

```
Scenario: Payment Service degrades (100ms → 5 seconds)

┌─────────────────────────────────────────┐
│ Black Friday - 10,000 users/sec         │
└─────────────────────────────────────────┘

Without Payment issue:
  - Order Service processes: 10,000 orders/sec
  - Response time: 100ms
  - Works fine

With Payment Service slow (5 seconds):
  - Order Service processes: 10,000 orders/sec
  - Each order waits 5 seconds for payment
  - Order Service threads fill up (can handle ~100 concurrent)
  - After 100 orders, Order Service queue full
  - Remaining orders timeout or are rejected
  - Users see 50% failure rate

Cascade: 1 slow service → Multiple services collapse
```

---

### 5.2 Asynchronous Communication (Event-Driven)

**Definition:** Caller sends message and continues without waiting for response.

```
Timeline:
┌────────────┐                    ┌──────────────┐
│   Client   │                    │ Event Broker │
└────────┬───┘                    └──────┬───────┘
         │                               │
         │─ 1. Order Created ──────────>│ (1ms)
         │ (Message: {order_id, user_id, │
         │  items, payment_method})      │
         │                               │
    Returns 202 Accepted               │
    (Order saved, not processed yet)    │
                                        │
    Order Service → Message Broker:
    Inventory Service ← Listens
    Payment Service ← Listens
    Shipping Service ← Listens
```

**Real-World Scenario: Book Purchase (Async)**

```
Customer clicks "Buy Now"
    ↓
Order Service creates order in database
    ↓
Order Service publishes "OrderCreated" event
    ↓
Returns 202 Accepted to customer (10ms)
"Order received! We'll process it soon."
    ↓ (Parallel processing)
    ├─ Inventory Service sees "OrderCreated"
    │  → Reserves stock
    │  → Publishes "StockReserved"
    │
    ├─ Payment Service sees "OrderCreated"
    │  → Processes payment (100ms)
    │  → Publishes "PaymentProcessed"
    │
    └─ Shipping Service sees "PaymentProcessed"
       → Schedules pickup
       → Publishes "ShippingScheduled"
    
Customer gets email: "Payment confirmed, ships tomorrow"
(Later, in background)
```

**Two Asynchronous Patterns:**

#### Pattern 1: Publish-Subscribe (Pub-Sub)

**Structure:**
```
Publisher (Order Service)
    │
    └─── publish("order.created") ──────┐
                                        │
                    ┌───────────────────┴────────────────┐
                    │                                    │
             ┌──────▼─────┐                    ┌──────────▼───┐
             │ Subscriber  │                    │  Subscriber  │
             │ Inventory   │                    │  Payment     │
             └─────────────┘                    └──────────────┘

Event: "order.created"
{
  "order_id": 5001,
  "user_id": 123,
  "items": [...]
}

Multiple subscribers receive simultaneously
No guaranteed order of processing
```

**Real Implementation (Python with Redis Pub-Sub):**

```python
# Publisher (Order Service)
import redis

broker = redis.Redis()

def create_order(user_id, items):
    order = save_order_to_db(user_id, items)
    
    # Publish event
    broker.publish("order.channel", json.dumps({
        "event": "order.created",
        "order_id": order.id,
        "user_id": user_id,
        "items": items
    }))
    
    return {"order_id": order.id}

# Subscriber (Inventory Service)
pubsub = broker.pubsub()
pubsub.subscribe("order.channel")

for message in pubsub.listen():
    if message['type'] == 'message':
        event = json.loads(message['data'])
        if event['event'] == 'order.created':
            reserve_inventory(event)
```

**Characteristics:**
- Multiple consumers can process simultaneously
- No guaranteed delivery order
- Fire-and-forget
- Decoupled: Publisher doesn't know subscribers

#### Pattern 2: Queue Model (Point-to-Point)

**Structure:**
```
Publisher (Order Service)
    │
    └─── Enqueue Message ──┐
                           │
                    ┌──────▼──────┐
                    │   Queue     │
                    │ [msg1, msg2] │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ Consumer    │
                    │ (Inventory) │
                    └─────────────┘

Queue: [msg1, msg2, msg3, ...]
  ├─ msg1 → Consumer 1 (removed after processing)
  ├─ msg2 → Consumer 2 (removed after processing)
  └─ msg3 → Waiting...
```

**Real Implementation (Python with RabbitMQ):**

```python
# Publisher (Order Service)
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue='order_queue')

def create_order(user_id, items):
    order = save_order_to_db(user_id, items)
    
    # Send message to queue
    channel.basic_publish(
        exchange='',
        routing_key='order_queue',
        body=json.dumps({
            "order_id": order.id,
            "user_id": user_id,
            "items": items
        })
    )
    
    return {"order_id": order.id}

# Consumer (Inventory Service Instance 1)
def callback(ch, method, properties, body):
    event = json.loads(body)
    reserve_inventory(event)
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.queue_declare(queue='order_queue')
channel.basic_consume(queue='order_queue', on_message_callback=callback)
channel.start_consuming()

# Consumer (Inventory Service Instance 2)
# Same code - both instances compete for messages
```

**Characteristics:**
- Single message handled by one consumer
- Guaranteed delivery (message stored until ack)
- Load balanced: If 2 instances, each gets ~50% messages
- FIFO: Messages processed in order

**Comparison:**

| Aspect | Pub-Sub | Queue |
|--------|---------|-------|
| **Multiple Consumers** | All receive copy | One consumes |
| **Delivery** | No guarantee | Guaranteed |
| **Use Case** | Notifications, events | Task distribution |
| **Example** | Stock price updates | Payment processing |
| **Fault Handling** | Message lost if no subscriber | Retried if failed |

---

### 5.3 When to Use Synchronous vs Asynchronous

**Decision Matrix:**

| Scenario | Use | Reason |
|----------|-----|--------|
| Login | Sync | Need immediate response |
| Payment | Sync | User waits on confirmation page |
| Send Email | Async | User doesn't need to wait |
| Generate Report | Async | Takes time, email when done |
| Check Stock | Sync | Real-time inventory check |
| Send Notification | Async | Background task |
| Create Order | Async* | Accept order, process later |
| Search Products | Sync | Real-time results needed |

**Real Example - Amazon Checkout:**

```
Synchronous (Real-time):
  1. Check payment method validity → Sync (2ms)
  2. Verify address format → Sync (1ms)
  3. Check stock availability → Sync (5ms)
  4. Process payment → Sync (100ms)
  
  Total: ~108ms (User waits)

Asynchronous (Background):
  1. Send confirmation email → Async (scheduled)
  2. Update analytics → Async (scheduled)
  3. Generate invoice PDF → Async (scheduled)
  4. Schedule shipping → Async (scheduled)
  5. Update inventory cache → Async (scheduled)
  
  User doesn't wait for these
```

---

### 5.4 Eventually Consistent Systems

**Definition:** System doesn't guarantee data is immediately consistent across all services, but will be consistent "eventually."

**Scenario: Email with Different Databases**

```
Time 0 (10:00:00):
Order Service (PostgreSQL):
    INSERT INTO orders VALUES (5001, user_123, 'pending')
    ✓ Visible immediately

Notification Service (MongoDB):
    {order_id: 5001, status: 'pending'} [NOT YET INSERTED]

Time 5 (10:00:05):
Message Broker processes queue
Notification Service consumes message
    {order_id: 5001, status: 'pending'} [NOW INSERTED]
    ✓ Now visible

Timeline:
     10:00:00                    10:00:05
        │                            │
   Order Created          Notification Synced
        │◄────── 5 seconds delay ──►│
        
During those 5 seconds: DATA IS INCONSISTENT
User queries might see order in one service but not another
```

**Real-World Impact:**

```
User checks order status immediately (10:00:01):
  ├─ Order Service: Order found ✓
  ├─ Notification Service: Order NOT found yet ✗
  └─ Different databases show different data!

User checks again 10 seconds later (10:00:10):
  ├─ Order Service: Order found ✓
  ├─ Notification Service: Order found ✓
  └─ Finally consistent ✓
```

**Example: Facebook Like Button**

```
You like a post on Facebook

Immediate:
  ├─ Your local timeline: Shows +1 like (optimistic update)
  └─ Post author's Facebook DB: ??? (not updated yet)

After 1 second:
  ├─ Your timeline: +1 like ✓
  ├─ Post author's DB: +1 like ✓
  ├─ Analytics DB: ??? (not updated yet)
  └─ Notification sent to author?

After 10 seconds:
  ├─ Everything is consistent ✓
  ├─ All databases have +1 like ✓
  ├─ Author was notified ✓
  └─ Analytics updated ✓

Facebook accepted this tradeoff:
  - Immediate feedback to user (better UX)
  - Eventual consistency across services (scalable)
```

**Why Accept Eventual Consistency?**

```
Strong Consistency (Immediate):
  - All databases sync before returning
  - Expensive: Every write must wait for all DB updates
  - Slow: 100ms → 500ms per operation
  - Doesn't scale

Eventual Consistency (Delayed):
  - Return immediately
  - Sync in background
  - Cheap: Async processing
  - Scales to millions of users

Trade-off:
  Slower response time → Better scalability
  vs.
  Immediate response → Limited scalability
  
For likes, comments, notifications: Users accept delay
For payments, orders: Must be strongly consistent
```

**Implementing Eventual Consistency Correctly:**

```
Order Created Event:

┌─────────────────────────────────────────┐
│       Order Service (PostgreSQL)        │
├─────────────────────────────────────────┤
│ BEGIN TRANSACTION                       │
│   INSERT order INTO orders              │
│   INSERT event INTO event_log:          │
│   {                                     │
│     event_id: 10001,                    │
│     type: "order.created",              │
│     data: {...},                        │
│     status: "pending"                   │
│   }                                     │
│ COMMIT                                  │
└─────────────────────────────────────────┘

Event Processor Service:
SELECT * FROM event_log WHERE status = 'pending'
For each event:
  ├─ Publish to Kafka
  ├─ Inventory Service processes
  ├─ Payment Service processes
  ├─ Mark as delivered
  └─ UPDATE event_log SET status = 'delivered'

Retry Logic:
If event fails:
  ├─ Retry after 5 minutes
  ├─ Retry after 15 minutes
  ├─ Alert if fails 5 times
  └─ Eventually succeeds or escalates

Result: Guarantees eventual consistency
```

---

### 5.5 Why Load Balancers Work Differently in Sync vs Async

#### Synchronous Communication: Load Balancer Required

**Scenario:** Checkout page with 3 Payment Service instances

```
Without Load Balancer (WRONG):
┌──────────────────────────────┐
│ Client App has hardcoded URL │
│ "payment-service:3000"       │
└──────────┬───────────────────┘
           │
    But which instance?
    
Instance 1: 192.168.1.10:3000 ✓
Instance 2: 192.168.1.11:3000 ✓
Instance 3: 192.168.1.12:3000 ✓

If client only knows URL, not which IP
→ Can't reach service
→ Fails
```

**With Load Balancer (CORRECT):**

```
┌────────────────────────────────────┐
│    Client                          │
│    POST http://payment-svc.local   │
└────────────┬───────────────────────┘
             │
        ┌────▼────────┐
        │ Load Balancer│ (192.168.1.5)
        │ :3000       │
        └────┬───────┬┴──────────┐
             │       │          │
    Request 1│   Req 2│      Req 3│
             │       │          │
     ┌───────▼┐ ┌───▼───┐ ┌────▼───┐
     │Inst 1  │ │Inst 2 │ │ Inst 3 │
     │:3001   │ │:3002  │ │ :3003  │
     └────────┘ └───────┘ └────────┘

Load Balancer:
  Request 1 → Instance 1 (0% loaded)
  Request 2 → Instance 2 (0% loaded)
  Request 3 → Instance 3 (0% loaded)
  Request 4 → Instance 1 (now 25% loaded)
  
Balanced! ✓
```

**Why Necessary for Sync:**
1. Client needs to know WHERE to send request
2. Multiple instances but client can only target one
3. Need central distribution point
4. Enables health checks and failover

---

#### Asynchronous Communication: Load Balancer NOT Needed

**Scenario:** Order events with 3 Inventory Service instances

```
Architecture:
┌──────────────────┐
│ Order Service    │
│ Publishes        │
│ "order.created"  │
└────────┬─────────┘
         │
    ┌────▼──────────┐
    │ Message Broker│
    │ (Kafka)       │
    │ Topic: orders │
    └────┬──────┬───┴──────┐
         │      │          │
    ┌────▼┐ ┌──▼───┐ ┌───▼───┐
    │Inst1│ │Inst 2│ │ Inst 3│
    │     │ │      │ │       │
    └─────┘ └──────┘ └───────┘

Broker distributes messages:
  Message 1 → Instance 1 gets (consumes, removes from queue)
  Message 2 → Instance 2 gets (consumes, removes from queue)
  Message 3 → Instance 3 gets (consumes, removes from queue)
  Message 4 → Instance 1 gets (Instance 1 is ready again)
  
Automatic balancing! No central LB needed
```

**Key Differences:**

| Sync | Async |
|------|-------|
| **Client initiated** | Message broker initiated |
| **Client must route** | Broker routes automatically |
| **Needs LB** | LB Not needed |
| **Fail-fast** | Retry-friendly |
| **Latency sensitive** | Batch-friendly |

**Why Async Doesn't Need LB:**

```
Queue Model:
  [Message 1, Message 2, Message 3, Message 4]
  
Instance 1 (idle): Gets Message 1
Instance 2 (idle): Gets Message 2
Instance 3 (busy): Waiting for current work
Instance 1 (done): Gets Message 3

Consumers pull as they're ready
Natural load balancing!

vs.

Sync Model:
  Request comes in NOW
  Client expects response in 100ms
  No time to wait for instance to be ready
  MUST have LB to route immediately
```

---

## 6. API Gateway {#api-gateway}

**Definition:** Single entry point for all client requests to microservices.

```
Architecture Without API Gateway:
┌─────────┐         ┌─────────────────────────┐
│ Web App │         │ Mobile App              │
└────┬────┘         └────┬────────────────────┘
     │                   │
     ├──► Auth Service ◄─┤
     │                   │
     ├──► Order Service◄─┤
     │                   │
     ├──► Product Svc◄───┤
     │                   │
     └──► Payment Svc◄───┘

Problems:
  - Auth logic duplicated in app
  - Mobile/Web must know all service URLs
  - Each client implements rate limiting
  - Each client handles authentication
```

**Architecture With API Gateway:**

```
┌──────────────────────────────────────────┐
│ Web App          Mobile App              │
└──────┬───────────────────────┬───────────┘
       │                       │
       └───────────────┬───────┘
                       │
            ┌──────────▼────────────┐
            │   API Gateway        │
            │ ┌──────────────────┐ │
            │ │ Authentication   │ │
            │ │ Rate Limiting    │ │
            │ │ Request Routing  │ │
            │ │ Caching          │ │
            │ │ Logging          │ │
            │ └──────────────────┘ │
            └──────┬──┬──┬──┬──────┘
                   │  │  │  │
            ┌──────▼──▼──▼──▼──┐
            │ Microservices    │
            │ Auth, Order,     │
            │ Product, Payment │
            └─────────────────┘
```

**Responsibilities of API Gateway:**

### 1. Authentication & Authorization

```
Request: GET /api/orders/5001

API Gateway:
1. Check Authorization header
   GET /api/orders/5001
   Header: Authorization: Bearer eyJhbGc...
   
2. Validate JWT token
   Token valid? ✓
   Expired? ✗
   Signature valid? ✓
   
3. Extract user_id from token: user_id = 123
   
4. Pass to Order Service with header:
   X-User-ID: 123
   X-User-Roles: [admin, user]
   
5. Order Service trusts gateway
   → Doesn't re-verify token
   → Uses user_id from header
```

### 2. Routing

```
Request comes to gateway.api.com
┌─────────────────────────────────┐
│ /api/products/* → Product Svc   │
│ /api/orders/* → Order Svc       │
│ /api/auth/* → Auth Svc          │
│ /api/payments/* → Payment Svc   │
│ /health → Health Check Service  │
└─────────────────────────────────┘

GET /api/products/5001
  → Routes to Product Service

POST /api/orders
  → Routes to Order Service

GET /health
  → Routes to Health Check Service
```

### 3. Rate Limiting

```
API Gateway tracks requests per user/IP

User Token: abc123
  Request 1: 8:00:00 ✓ (1/100)
  Request 2: 8:00:01 ✓ (2/100)
  ...
  Request 100: 8:01:00 ✓ (100/100)
  Request 101: 8:01:01 ✗ 429 Too Many Requests

Prevents:
  - DDoS attacks
  - Runaway clients
  - API abuse
```

### 4. Caching

```
GET /api/products/5001?include=details

API Gateway:
1. Check cache: "products:5001:details"
2. If hit → Return cached response (1ms)
3. If miss → Route to Product Service (100ms)
   → Cache response for 1 hour
   → Return to client

Cache hit on repeat requests saves 99ms!
```

### 5. Request/Response Transformation

```
Client sends:
POST /api/orders {
  "product_id": 5001,
  "quantity": 2
}


**Definition:** A single entry point that sits between clients and microservices, handling cross-cutting concerns.

```
WITHOUT API Gateway (Chaos):
┌─────────────┐
│  Web App    │
└──────┬──────┘
       ├─ Knows Auth Service URL
       ├─ Knows Order Service URL
       ├─ Implements authentication logic
       ├─ Implements rate limiting
       ├─ Implements caching
       ├─ Implements logging
       └─ Implements error handling

┌─────────────┐
│ Mobile App  │
└──────┬──────┘
       ├─ Knows Auth Service URL (different!)
       ├─ Knows Order Service URL (different!)
       ├─ Implements same logic again
       ├─ Different rate limits
       ├─ Different error messages
       └─ Duplicated code everywhere!

┌─────────────┐
│ Desktop App │
└──────┬──────┘
       └─ Same duplication...

Problem: 3x code duplication, hard to maintain

WITH API Gateway (Clean):
┌────────────────────┐
│ All Clients        │
│ (Web/Mobile/etc)   │
└─────────┬──────────┘
          │
    ┌─────▼──────────────┐
    │   API Gateway      │
    │ ┌────────────────┐ │
    │ │ Auth (once)    │ │
    │ │ Rate limit     │ │
    │ │ Cache          │ │
    │ │ Route/log      │ │
    │ └────────────────┘ │
    └─────┬──┬──┬────────┘
          │  │  │
    ┌─────▼──▼──▼────┐
    │ Microservices  │
    └────────────────┘

Benefit: Single point for all cross-cutting concerns
```

### 6.2 Key Responsibilities of API Gateway

#### 1. Authentication & Authorization

**Authentication (Who are you?):**

```
User Login Flow:

Client Request:
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "secret123"
}
    ↓
API Gateway receives request
    ↓
Routes to Auth Service
    ↓
Auth Service:
  1. Find user by email
  2. Hash password, compare
  3. If match → Generate JWT token
  4. Return token to gateway

Auth Service Response:
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzEyMyIsImlhdCI6MTYzNDU2Nzg5MH0.signature...",
  "user_id": 123,
  "name": "John Doe",
  "roles": ["user", "admin"]
}
    ↓
API Gateway returns to client
    ↓
Client stores token in localStorage/cookies
    ↓
Client sends token in future requests:
Authorization: Bearer eyJhbGci...
```

**Authorization (What can you do?):**

```
User tries to access resource:
GET /api/orders/5001
Header: Authorization: Bearer eyJhbGci...

API Gateway:
1. Extract token from Authorization header
2. Verify signature (ensures not tampered)
3. Check expiry (not expired?)
4. Extract claims: {user_id: 123, roles: ["user"]}
5. Pass to Order Service with header:
   X-User-ID: 123
   X-User-Roles: user
   X-Request-ID: abc-123 (for tracing)

Order Service receives request
1. Trust gateway (it validated token)
2. Get user_id from X-User-ID header
3. Retrieve order 5001
4. Check: Does user 123 own order 5001?
5. If YES → Return order
6. If NO → 403 Forbidden

Example:
User 123 tries to view order 5001 (belongs to user 123)
  ✓ Order Service: "You own this order"
  ✓ Return order
  
User 123 tries to view order 5002 (belongs to user 456)
  ✗ Order Service: "You don't own this order"
  ✗ 403 Forbidden
```

**Multiple Authentication Methods:**

```
API Gateway supports:

1. JWT (JSON Web Token)
   Authorization: Bearer eyJhbGci...
   Stateless, scales well, good for APIs
   
2. OAuth2
   For third-party integrations
   "Login with Google", "Login with Facebook"
   
3. API Keys
   Authorization: X-API-Key: sk_live_abc123
   For service-to-service, mobile apps
   
4. mTLS (Mutual TLS)
   Client certificate verification
   Very secure, for internal services
```

**Real-World Implementation (Kong API Gateway):**

```yaml
# Kong Configuration
plugins:
  - name: jwt
    config:
      secret: "your-secret-key"
      key_claim_name: "sub"  # Extract user_id from "sub" claim
      
  - name: acl
    config:
      whitelist:
        - admin
        - user
        
  - name: rate-limiting
    config:
      minute: 100  # 100 requests per minute

routes:
  - name: orders_api
    paths:
      - /api/orders
    methods: 
      - GET
      - POST
    service: order-service
    plugins:
      - jwt
      - rate-limiting
```

---

#### 2. Request Routing

**Path-Based Routing:**

```
API Gateway inspects request path and routes accordingly

GET /api/products/5001
  → Check path
  → Matches /api/products/*
  → Route to Product Service (product-svc.internal:3000)
  → Proxy request to http://product-svc.internal:3000/products/5001

POST /api/orders
  → Check path
  → Matches /api/orders
  → Route to Order Service (order-svc.internal:3000)
  → Proxy request to http://order-svc.internal:3000/orders

GET /api/auth/profile
  → Check path
  → Matches /api/auth/*
  → Route to Auth Service (auth-svc.internal:3000)
  → Proxy request to http://auth-svc.internal:3000/profile
```

**Host-Based Routing:**

```
Different domains route to different services

products.api.shopnow.com
  → Product Service
  
orders.api.shopnow.com
  → Order Service
  
payments.api.shopnow.com
  → Payment Service

Useful for multi-tenant systems
```

**Header-Based Routing:**

```
GET /api/data
X-Version: v1
  → Route to old API Service

GET /api/data
X-Version: v2
  → Route to new API Service

Useful for API versioning and canary deployments
```

**Method-Based Routing:**

```
GET /api/products → Product Service (read)
POST /api/products → Product Service (write)
PUT /api/products/5 → Product Service (update)
DELETE /api/products/5 → Product Service (delete)

Same path, different methods, same service
Or route to different services based on method
```

**Real-World Nginx Configuration:**

```nginx
upstream product_service {
    server product-svc-1:3000 max_fails=3 fail_timeout=30s;
    server product-svc-2:3000 max_fails=3 fail_timeout=30s;
    server product-svc-3:3000 max_fails=3 fail_timeout=30s;
}

upstream order_service {
    server order-svc-1:3000;
    server order-svc-2:3000;
}

server {
    listen 80;
    server_name api.shopnow.com;

    # Product API routes
    location /api/products {
        proxy_pass http://product_service;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-ID $request_id;
    }

    # Order API routes
    location /api/orders {
        proxy_pass http://order_service;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-ID $request_id;
    }

    # Health check endpoint
    location /health {
        return 200 "OK";
    }
}
```

---

#### 3. Rate Limiting & Throttling

**Purpose:** Protect services from overload

```
Scenario: Black Friday without rate limiting

┌─────────────────────────────────────────┐
│ Product Service: 100,000 req/sec        │
│ Can only handle: 50,000 req/sec         │
│                                         │
│ Result: Overload, crashes, 50% fail    │
└─────────────────────────────────────────┘

Solution: Rate limit at gateway

API Gateway enforces:
  Max 100 requests/minute per user
  Max 1000 requests/minute per IP
  Max 500 requests/minute for anonymous users

When limit exceeded:
  Return 429 Too Many Requests
  Include header: Retry-After: 60
```

**Implementation Strategy:**

```
Option 1: Token Bucket Algorithm (Common)

Each user has a "bucket" with tokens
Bucket capacity: 100 tokens
Refill rate: 1 token per second

User makes request:
  ├─ If tokens > 0:
  │  ├─ Remove 1 token
  │  └─ Allow request
  │
  └─ If tokens = 0:
     └─ Deny request (429)

Timeline:
  10:00:00 - User: 100 tokens, makes 5 requests → 95 tokens left
  10:00:01 - Refill: +1 token → 96 tokens
  10:00:02 - Refill: +1 token → 97 tokens
  ...
  10:01:05 - Refill: +1 token → 100 tokens (full)
```

**Real Code (Python with Redis):**

```python
import redis
import time

redis_client = redis.Redis(host='localhost', port=6379)

def rate_limit(user_id, max_requests=100, window=60):
    """
    Rate limit: max_requests per window seconds
    """
    key = f"rate_limit:{user_id}"
    
    current_count = redis_client.get(key)
    
    if current_count is None:
        # First request
        redis_client.setex(key, window, 1)
        return True
    
    current_count = int(current_count)
    
    if current_count < max_requests:
        # Increment
        redis_client.incr(key)
        return True
    else:
        # Over limit
        return False

# Usage in API Gateway
@app.before_request
def apply_rate_limit():
    user_id = get_user_id_from_token()
    
    if not rate_limit(user_id):
        return {
            "error": "Rate limit exceeded",
            "retry_after": 60
        }, 429
```

**Different Rate Limits for Different Users:**

```
Free tier users:
  100 requests per minute

Premium tier users:
  10,000 requests per minute

Enterprise tier users:
  Unlimited (or very high limit)

Admin users:
  Unlimited

Implementation:
  Get user_id from token
  Lookup user tier from database
  Apply appropriate rate limit
```

---

#### 4. Caching

**Purpose:** Reduce backend load by serving cached responses

```
Scenario: Popular product page

GET /api/products/123 (iPhone 15 Pro)

First Request:
1. API Gateway checks cache
2. Not found
3. Routes to Product Service (100ms)
4. Product Service returns:
   {
     "id": 123,
     "name": "iPhone 15 Pro",
     "price": 1199.99,
     "stock": 5000,
     "rating": 4.8
   }
5. API Gateway caches response
   key: "products:123"
   ttl: 3600 (1 hour)
6. Returns to client (100ms)

Second Request (within 1 hour):
1. API Gateway checks cache
2. Found!
3. Returns immediately (1ms)
4. Never hits Product Service

Benefit: 100x faster for repeat requests!
```

**Cache Invalidation Strategies:**

```
1. Time-based (TTL - Most Common)
   Cache for 1 hour, then refresh
   Good for non-critical data
   
   Example: Product catalog
   Prices rarely change, can be stale 1 hour

2. Event-based (Invalidate on change)
   When product updates, clear cache
   Real-time data
   
   Example: Inventory
   User buys item → Inventory decreases
   → Product Service publishes event
   → API Gateway clears products:123 cache
   → Next request gets fresh data

3. Manual (Explicit invalidation)
   Admin clicks "Clear Cache"
   For critical data
   
   Example: Configuration
   Admin changes payment settings
   → Clear cache manually
   → All requests get new settings
```

**Cache Implementation (Nginx):**

```nginx
# Cache GET requests for 1 hour
location /api/products {
    # Cache GET requests only
    if ($request_method != GET) {
        proxy_pass http://product_service;
        proxy_no_cache 1;
    }
    
    proxy_pass http://product_service;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    proxy_cache_valid 200 1h;  # Cache 200 responses for 1 hour
    proxy_cache_valid 404 10m;  # Cache 404s for 10 minutes
    proxy_cache_bypass $http_cache_control;  # Respect Cache-Control header
}

# Don't cache POST/PUT/DELETE
location /api/orders {
    proxy_pass http://order_service;
    proxy_no_cache 1;  # Don't cache writes
}
```

---

#### 5. Request/Response Transformation

**Add Headers:**

```
Client sends:
POST /api/orders
{user_id: 123, items: [...]}

API Gateway:
1. Extract user_id from JWT token: 123
2. Add headers:
   X-User-ID: 123
   X-Request-ID: abc-123-def (for tracing)
   X-Timestamp: 2024-10-30T10:30:00Z
   X-Client-IP: 203.45.67.89
   X-API-Version: v1

Forward to Order Service:
POST /orders
Headers: X-User-ID, X-Request-ID, X-Timestamp, etc.
{user_id: 123, items: [...]}

Order Service benefits:
- No need to parse JWT again
- Has tracing ID for logs
- Knows client IP for analytics
- Knows API version for backwards compatibility
```

**Transform Request Body:**

```
Client sends (public API):
POST /api/orders {
  "product_id": 123,
  "quantity": 2,
  "coupon_code": "SAVE10"
}

API Gateway transforms to (internal API):
POST /orders {
  "product_id": 123,
  "quantity": 2,
  "user_id": 789,              ← Added
  "timestamp": 1634567890,     ← Added
  "request_id": "abc-123",     ← Added
  "coupon_code": "SAVE10",
  "ip_address": "203.45.67.89" ← Added
}

Order Service receives enriched data
No need for service to extract user_id
Service can trust all data came from gateway
```

**Transform Response:**

```
Order Service returns (internal format):
{
  "id": 5001,
  "user_id": 789,
  "status": "order_placed",
  "total_cents": 59999,
  "created_ts": 1634567890,
  "internal_status_code": 200
}

API Gateway transforms to (public format):
{
  "id": 5001,
  "status": "confirmed",
  "total": "$599.99",
  "created_at": "2024-10-30T10:30:00Z"
}

Benefits:
- Hide internal details (timestamps as UNIX, etc)
- Format data consistently
- Convert currencies, dates, etc
- Hide internal status codes
```

---

#### 6. Protocol Translation

```
Modern Gateway can translate between protocols

Client → HTTP/2 with REST
         ↓
    API Gateway
         ↓
Internal Services → gRPC with Protocol Buffers

OR

Client → REST JSON
         ↓
    API Gateway
         ↓
Internal Services → GraphQL
```

**Real-World Example:**

```
Mobile Client (HTTP/2, REST):
GET /api/products/123

API Gateway:
1. Receive HTTP/2 request
2. Convert to gRPC Protocol Buffers
3. Call internal Product Service:
   grpc.products.GetProduct(product_id=123)
4. Receive gRPC response (binary)
5. Convert back to HTTP/2 REST JSON
6. Return to client

Benefits:
- Internal services use fast gRPC
- External clients use familiar REST
- Gateway handles complexity
```

---

#### 7. Logging & Monitoring

**Log Every Request:**

```
API Gateway logs:

{
  "timestamp": "2024-10-30T10:30:00.123Z",
  "request_id": "req-abc-123-def",
  "client_ip": "203.45.67.89",
  "user_id": 123,
  "method": "POST",
  "path": "/api/orders",
  "query_params": "?sort=date",
  "request_headers": {
    "Authorization": "Bearer [REDACTED]",
    "Content-Type": "application/json",
    "User-Agent": "Mozilla/5.0"
  },
  "request_body_size": 256,
  "status_code": 201,
  "response_time_ms": 245,
  "response_size": 512,
  "service_called": "order-service",
  "service_response_time_ms": 100,
  "cache_hit": false,
  "rate_limit_remaining": 87
}
```

**Monitoring Metrics:**

```
API Gateway tracks:
1. Request rate (requests/sec)
2. Error rate (4xx, 5xx %)
3. Response time (p50, p95, p99)
4. Cache hit rate
5. Rate limit violations
6. Upstream service health

Alerts:
- Response time > 1 second
- Error rate > 5%
- Rate limit violations > 100/min
- Service down (no healthy instances)
```

---

### 6.3 Popular API Gateways

| Gateway | Pros | Cons | Best For |
|---------|------|------|----------|
| **Kong** | Open-source, powerful plugins | Complex setup | Production, on-prem |
| **Nginx** | Lightweight, fast | Limited features | Simple routing, cache |
| **AWS API Gateway** | Serverless, managed | Limited control, expensive | AWS ecosystem |
| **Traefik** | Kubernetes-native, dynamic | Kubernetes-only | Kubernetes clusters |
| **Istio** | Service mesh + gateway, powerful | Heavy, complex | Complex microservices |
| **Apigee** | Enterprise features, analytics | Expensive | Large enterprises |

---

## 7. Service Discovery {#service-discovery}

### 7.1 The Problem Service Discovery Solves

**Without Service Discovery (Monolithic Era):**

```
Hardcoded Service Locations:

Order Service:
const PAYMENT_SERVICE_URL = "payment-service.prod.internal:3000"
const INVENTORY_SERVICE_URL = "inventory-service.prod.internal:3001"

Issues:
1. If Payment Service restarts → New IP → Hardcoded URL fails
2. Auto-scaling: New instances added, Order Service doesn't know
3. Maintenance: Must coordinate updates across services
4. Manual: DevOps must update hardcoded URLs
```

**With Service Discovery (Microservices):**

```
Dynamic Service Locations:

Order Service queries Service Registry:
"Where is payment-service?"
    ↓
Registry responds with 3 instances:
  - 192.168.1.10:3000
  - 192.168.1.11:3000
  - 192.168.1.12:3000

Order Service picks one (load balanced)
Makes request to that instance

If instance dies:
  - Stops sending heartbeat
  - Registry removes it
  - Order Service won't route to it anymore
  - Auto-replacement spins up new instance
  - Order Service finds it automatically
```

---

### 7.2 How Service Discovery Works

```
Architecture:

┌────────────────────────────────┐
│   Service Registry             │
│   (Eureka/Consul/etcd)         │
│                                │
│ Services registered:           │
│ {                              │
│   "payment": [                 │
│     {ip: 192.168.1.10, port: 3000},
│     {ip: 192.168.1.11, port: 3000},
│     {ip: 192.168.1.12, port: 3000}
│   ],                           │
│   "inventory": [               │
│     {ip: 192.168.1.20, port: 3001},
│     {ip: 192.168.1.21, port: 3001}
│   ]                            │
│ }                              │
└────────────────────────────────┘
        ▲   ▲   ▲   ▲   ▲
        │   │   │   │   │
    Heartbeat & queries


┌──────────────────────────────────┐
│ Service Instances                │
├──────────────────────────────────┤
│ Payment Service #1: 192.168.1.10 │
│ Payment Service #2: 192.168.1.11 │
│ Payment Service #3: 192.168.1.12 │
│ Inventory Service #1: 192.168.1.20│
│ Inventory Service #2: 192.168.1.21│
└──────────────────────────────────┘
```

### 7.3 Lifecycle: Service Registration & Discovery

**Phase 1: Service Startup & Registration**

```
Payment Service Instance 1 starts:

1. Read config:
   SERVICE_NAME=payment
   PORT=3000
   REGISTRY_URL=http://eureka:8761

2. Get own IP: 192.168.1.10

3. Register with Eureka:
   POST http://eureka:8761/eureka/apps/payment
   {
     "instance": {
       "instanceId": "payment-instance-1",
       "hostName": "payment-1.internal",
       "app": "PAYMENT",
       "ipAddr": "192.168.1.10",
       "status": "UP",
       "port": {"@enabled": true, "$": 3000},
       "healthCheckUrl": "http://192.168.1.10:3000/health",
       "leaseInfo": {
         "renewalIntervalInSecs": 30,
         "durationInSecs": 90
       }
     }
   }

4. Eureka response: 204 No Content (accepted)

5. Service ready to receive requests ✓
```

**Phase 2: Heartbeat (Keep Alive)**

```
Every 30 seconds, Payment Service sends heartbeat:

PUT http://eureka:8761/eureka/apps/payment/payment-instance-1

Eureka receives heartbeat:
  ├─ Check if instance exists
  ├─ If exists: Update "last heartbeat" timestamp
  ├─ If not exists: Register it (recovery)
  └─ Return 200 OK

Timeline:
  10:00:00 - Payment Service registers
  10:00:30 - Sends heartbeat #1
  10:01:00 - Sends heartbeat #2
  10:01:30 - Sends heartbeat #3
  ...

If heartbeat stops:
  10:02:00 - Last heartbeat was 60 seconds ago
  10:02:30 - Eureka marks as DOWN (missed 2 heartbeats)
  10:03:00 - Eureka removes from registry
  → Order Service won't route to it anymore
```

**Phase 3: Client Discovery (Order Service)**

```
Order Service wants to call Payment Service:

1. Query Eureka:
   GET http://eureka:8761/eureka/apps/payment

2. Eureka returns healthy instances:
   {
     "application": {
       "name": "PAYMENT",
       "instance": [
         {
           "instanceId": "payment-instance-1",
           "hostName": "payment-1.internal",
           "ipAddr": "192.168.1.10",
           "port": 3000,
           "status": "UP"
         },
         {
           "instanceId": "payment-instance-2",
           "hostName": "payment-2.internal",
           "ipAddr": "192.168.1.11",
           "port": 3000,
           "status": "UP"
         },
         {
           "instanceId": "payment-instance-3",
           "hostName": "payment-3.internal",
           "ipAddr": "192.168.1.12",
           "port": 3000,
           "status": "UP"
         }
       ]
     }
   }

3. Order Service caches this list (for 30 seconds)

4. Load balancer picks one: 192.168.1.10

5. Make request:
   POST http://192.168.1.10:3000/process-payment

6. If request fails:
   ├─ Retry to another instance
   └─ Or query Eureka for fresh list
```

---

### 7.4 Client-Side vs Server-Side Discovery

#### Client-Side Discovery (Eureka, Consul)

```
Order Service is responsible for discovery

┌─────────────────────────────────────┐
│ Order Service                       │
├─────────────────────────────────────┤
│ 1. Query Service Registry           │
│ 2. Get list of instances            │
│ 3. Implement load balancing logic   │
│ 4. Choose one instance              │
│ 5. Call it directly                 │
└─────────────────────────────────────┘
    ▲
    │ Query/get list
    │
┌───▼────────────────────────────────┐
│ Service Registry                    │
│ payment: [inst1, inst2, inst3]      │
└─────────────────────────────────────┘

Pros:
  - Simple
  - No extra hop
  - Client controls load balancing

Cons:
  - Client must implement logic
  - Language-specific libraries
  - Each client must have same logic
```

**Implementation (Spring Cloud Eureka):**

```java
// Order Service calls Payment Service
@Service
public class OrderService {
    
    @Autowired
    private DiscoveryClient discoveryClient;
    
    @Autowired
    private RestTemplate restTemplate;
    
    public void processOrder(Order order) {
        // Get all Payment Service instances
        List<ServiceInstance> instances = 
            discoveryClient.getInstances("payment-service");
        
        if (instances.isEmpty()) {
            throw new ServiceNotAvailableException("Payment service not found");
        }
        
        // Load balance: pick random instance
        ServiceInstance instance = instances.get(
            new Random().nextInt(instances.size())
        );
        
        // Build URL
        String url = String.format(
            "http://%s:%d/process-payment",
            instance.getHost(),
            instance.getPort()
        );
        
        // Call Payment Service directly
        PaymentResponse response = restTemplate.postForObject(
            url,
            order,
            PaymentResponse.class
        );
    }
}
```

---

#### Server-Side Discovery (Kubernetes, AWS ELB)

```
Load Balancer is responsible for discovery

┌──────────────────────────────┐
│ Order Service                │
│ Calls: payment-service.svc   │
└──────┬───────────────────────┘
       │
   ┌───▼──────────────────────┐
   │ Load Balancer / DNS      │
   │ payment-service.svc      │
   │ → 192.168.1.5 (LB IP)    │
   └───┬──────────────────────┘
       │
   ┌───┴─────┬──────────┬────────┐
   │         │          │        │
┌──▼──┐ ┌──▼──┐ ┌──▼───┐ ┌──▼──┐
│ inst1│ │inst2│ │ inst3│ │inst4│
└─────┘ └─────┘ └──────┘ └─────┘

Pros:
  - Language-agnostic
  - LB handles everything
  - Simple client code

Cons:
  - Extra network hop
  - LB is potential bottleneck
  - LB is single point of failure
```

**Implementation (Kubernetes):**

```yaml
# Service resource in Kubernetes
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
  type: ClusterIP

---

# Order Service deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order
  template:
    metadata:
      labels:
        app: order
    spec:
      containers:
      - name: order
        image: order-service:latest
        env:
          - name: PAYMENT_SERVICE_URL
            value: "http://payment-service:3000"
        # No hardcoded IPs!
```

**Order Service Code (Simple):**

```java
// Very simple - no discovery logic needed
@Service
public class OrderService {
    
    @Value("${payment.service.url}")
    private String paymentServiceUrl;
    
    @Autowired
    private RestTemplate restTemplate;
    
    public void processOrder(Order order) {
        // Just call the URL
        // Kubernetes handles load balancing internally
        PaymentResponse response = restTemplate.postForObject(
            paymentServiceUrl + "/process-payment",
            order,
            PaymentResponse.class
        );
    }
}
```

---

### 7.5 Popular Service Discovery Tools

#### Eureka (Netflix/Spring Cloud)

```
For: On-premises, Java/Spring Boot
Register: Active (services register themselves)
Discovery: Client-side

Architecture:
  Eureka Server (Central Registry)
    ↑ Register/Heartbeat
  All Service Instances

Workflow:
1. Service registers on startup
2. Service sends heartbeat every 30s
3. Eureka marks as DOWN if no heartbeat for 90s
4. Clients query Eureka periodically
5. Clients use Ribbon for load balancing

Example in Spring Boot:
@SpringBootApplication
@EnableEurekaClient
public class PaymentServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
    }
}
```

#### Consul (HashiCorp)

```
For: Multi-datacenter, cross-platform
Register: Both (servers register services)
Discovery: Server-side or client-side

Architecture:
  Consul Server Cluster
    ↑ Register/Health Checks
  All Service Instances

Features:
- Multi-datacenter support (DC1, DC2, DC3)
- Built-in health checks (HTTP, TCP, Exec)
- DNS interface (services.consul)
- Key-value store
- ACLs for security

Example:
Service registers:
  curl -X PUT http://localhost:8500/v1/agent/service/register -d '{
    "ID": "payment-1",
    "Name": "payment",
    "Address": "192.168.1.10",
    "Port": 3000,
    "Check": {
      "HTTP": "http://192.168.1.10:3000/health",
      "Interval": "10s"
    }
  }'

Query:
  DNS: dig payment.service.consul
  → Returns: 192.168.1.10, 192.168.1.11, 192.168.1.12
```

#### Kubernetes Service Discovery

```
For: Kubernetes-only
Register: Automatic (kubelet registers)
Discovery: DNS-based

Architecture:
  Kubernetes API Server
    ↑ Pod lifecycle events
  All Pod Instances

How it works:
1. Pod starts
2. kubelet reports pod status to API Server
3. Kubernetes updates Service endpoints
4. CoreDNS watches Service endpoints
5. Clients query DNS: payment-service.default.svc.cluster.local
6. CoreDNS responds with current pod IPs

Benefits:
- Zero configuration
- Automatic scaling
- Self-healing
- Built into Kubernetes

Example:
payment-service.default.svc.cluster.local
├─ default = namespace
├─ svc = Kubernetes Service
├─ cluster.local = cluster DNS domain
└─ Resolves to service ClusterIP (192.168.1.5)
   Which load balances to pod IPs
```

---

## 8. Logging & Monitoring {#logging-monitoring}

### 8.1 Why Externalize Logs?

**Problem: Logs in Containers**

```
Traditional Monolith:
┌────────────────────────────────┐
│ Single Server                  │
│ /var/log/application.log       │
│ 100GB of logs                  │
│                                │
│ Easy to SSH and debug:         │
│ $ tail -f /var/log/app.log     │
└────────────────────────────────┘

Microservices with Containers:
┌──────────────────────────────────────────────────┐
│ Kubernetes Cluster                               │
├──────────────────────────────────────────────────┤
│ Pod 1 (Order Service)                            │
│ /var/log/app.log - 10GB                          │
│ But where is it physically?                      │
│ It's inside a container!                         │
│                                                  │
│ Pod 2 (Payment Service)                          │
│ /var/log/app.log - 5GB                           │
│                                                  │
│ Pod 3 (Inventory Service)                        │
│ /var/log/app.log - 8GB                           │
│                                                  │
│ Pod 4 (Order Service) - CRASHED                  │
│ /var/log/app.log - DELETED!                      │
│ (Container is gone, logs gone!)                  │
│                                                  │
│ To debug Pod 4's crash:                          │
│ Logs are lost forever                            │
└──────────────────────────────────────────────────┘

Problems:
1. Logs scattered across pods
2. Container dies → logs disappear
3. Scaling: 50 instances → 50 separate log files
4. How to debug cross-service issues?
5. Need to SSH into specific pod
6. By the time you SSH, pod might be restarted
```

**Solution: Externalize Logs**

```
All logs go to centralized system

┌──────────────────────────────────┐
│ Kubernetes Cluster               │
├──────────────────────────────────┤
│ Pod 1: Order Service             │
│ Pod 2: Payment Service           │
│ Pod 3: Inventory Service         │
│ Pod 4: Order Service (crashed)   │
│                                  │
│ All pods log to:                 │
│ STDOUT (container logging)       │
└──────────────┬───────────────────┘
               │
        ┌──────▼──────┐
        │ Log Shipper  │
        │ (Fluentd)    │
        └──────┬───────┘
               │
      ┌────────▼─────────┐
      │ Log Storage       │
      │ (Elasticsearch)   │
      │ 1 TB of logs      │
      │ Searchable        │
      └────────┬──────────┘
               │
      ┌────────▼─────────┐
      │ Log Viewer        │
      │ (Kibana)          │
      │ Web UI            │
      └──────────────────┘

Benefits:
✓ All logs in one place
✓ Logs persist after pod dies
✓ Full-text searchable
✓ Correlate logs across services
✓ Analytics on logs
```

---

### 8.2 ELK Stack (Elasticsearch, Logstash, Kibana)

**Architecture:**

```
Services → Logs → Logstash → Elasticsearch → Kibana

┌────────────────────────────────────────┐
│ Services (log to STDOUT)               │
├────────────────────────────────────────┤
│ Order Service logs to console          │
│ Payment Service logs to console        │
│ Inventory Service logs to console      │
│ Notification Service logs to console   │
└────────────┬────────────────────────────┘
             │ docker logs / kubectl logs
             ▼
┌────────────────────────────────────────┐
│ Log Collector (Fluentd/Logstash)       │
├────────────────────────────────────────┤
│ Collect from all containers            │
│ Parse unstructured text                │
│ Enrich with metadata:                  │
│  - container_name                      │
│  - pod_id                              │
│  - service_name                        │
│  - timestamp                           │
│  - host                                │
│ Convert to JSON                        │
└────────────┬────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ Elasticsearch (Index & Store)          │
├────────────────────────────────────────┤
│ Receives JSON logs                     │
│ Indexes them (full-text search)        │
│ Stores in time-series indices:         │
│  logs-2024.10.30                       │
│  logs-2024.10.31                       │
│ Retention: Keep 30 days                │
│ Delete older logs automatically        │
└────────────┬────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ Kibana (Query & Visualize)             │
├────────────────────────────────────────┤
│ Web interface to search logs           │
│ Query: order_id = 5001                 │
│ Results: All logs mentioning 5001      │
│ Timeline: See flow of events           │
│ Analytics: Count, aggregate, chart     │
└────────────────────────────────────────┘
```

**Real Log Flow:**

```
Time: 10:30:00

Order Service logs:
  2024-10-30T10:30:00.123Z [INFO] Order created: order_id=5001, user_id=123

Console output (pod stdout):
  Order created: order_id=5001, user_id=123

Fluentd reads stdout:
  Parses timestamp, level, message
  Adds metadata: container=order-service-1, pod=order-pod-1

JSON created:
  {
    "timestamp": "2024-10-30T10:30:00.123Z",
    "level": "INFO",
    "message": "Order created",
    "order_id": "5001",
    "user_id": "123",
    "container_name": "order-service",
    "pod_name": "order-pod-1",
    "host": "worker-1"
  }

Elasticsearch indexes:
  Index: logs-2024.10.30
  Doc ID: unique_id_123
  Searchable by: timestamp, level, order_id, container_name, etc

User searches Kibana:
  Query: order_id:5001
  Results:
    10:30:00 - Order Service: Order created
    10:30:01 - Payment Service: Payment processing
    10:30:02 - Payment Service: Payment failed
    10:30:02 - Order Service: Rollback order
    10:30:03 - Notification Service: Sent email
  
  Complete trace! All 50 logs for that order!
```

---

### 8.3 Log Levels & Structured Logging

**Log Levels:**

```
1. DEBUG
   Most verbose, for development
   Example: "Query executed: SELECT * FROM users WHERE id=123"

2. INFO
   Normal operations
   Example: "Order processed: order_id=5001"

3. WARN
   Something unexpected, but handled
   Example: "Payment retry: attempt 2/3"

4. ERROR
   Something went wrong
   Example: "Database connection failed"

5. FATAL
   Critical error, service might crash
   Example: "Out of memory"

Production config:
  ├─ Dev environment: DEBUG
  ├─ Staging environment: INFO
  └─ Production: WARN or ERROR (spam reduction)
```

**Unstructured Logging (Bad):**

```
User logs in and something goes wrong

Log output:
  User john@example.com login attempt
  Verification failed
  User account locked
  
Problems:
- Hard to parse programmatically
- Can't search for specific fields
- Requires human interpretation
- Can't aggregate statistics
```

**Structured Logging (Good):**

```
Same scenario, structured:

Log JSON:
{
  "timestamp": "2024-10-30T10:30:00.123Z",
  "level": "WARN",
  "event": "login_failed",
  "user_id": 123,
  "email": "john@example.com",
  "ip_address": "203.45.67.89",
  "failure_reason": "invalid_password",
  "attempt": 3,
  "service": "auth-service",
  "version": "1.2.3"
}

Benefits:
✓ Easily parsed by machine
✓ Searchable fields:
  - Find all logins from IP 203.45.67.89
  - Find all failed attempts for john@example.com
  - Count failed logins per hour
✓ Aggregatable:
  - Average failed attempts
  - Alert if > 10 failed logins in 1 minute
✓ Alertable: Set up thresholds
```

**Structured Logging Example (Code):**

```javascript
// ❌ Bad - Unstructured
console.log("User login: " + email + " from IP " + ip);

// ✓ Good - Structured
console.log(JSON.stringify({
  timestamp: new Date().toISOString(),
  level: "INFO",
  event: "user_login",
  user_id: 123,
  email: email,
  ip_address: ip,
  session_id: session_id,
  duration_ms: loginTime
}));

// In Elasticsearch, this becomes:
{
  "user_id": 123,
  "email": "john@example.com",
  "ip_address": "203.45.67.89",
  "event": "user_login"
}

// Can query:
// "Find all logins from IP 203.45.67.89"
// "Count logins per user per day"
// "Alert if 100+ logins in 1 minute from same IP"
```

---

### 8.4 Distributed Tracing (Jaeger, Zipkin)

**Problem: Slow Request, Where's the Bottleneck?**

```
User clicks "Buy Now"
App waits 5 seconds
Response comes back

Question: Where was the bottleneck?

Without tracing:
  - Was it the database?
  - Was it the API?
  - Was it the payment service?
  - Was it the network?
  - ???

With distributed tracing:
  10:30:00.000 - API Gateway: receives request (1ms)
  10:30:00.001 - Auth Service call (15ms)
     10:30:00.001 - Auth DB query (10ms)
  10:30:00.016 - Order Service call (100ms)
     10:30:00.016 - Validation (2ms)
     10:30:00.018 - Order DB insert (5ms)
     10:30:00.023 - Inventory Service call (50ms)
        10:30:00.023 - Inventory DB query (30ms)
     10:30:00.073 - Payment Service call (1000ms) ← SLOW!
        10:30:00.073 - External payment API call (950ms) ← BOTTLENECK!
  10:30:01.073 - Response

Total: 1074ms
Bottleneck: External payment API is slow!
```

**Distributed Tracing Architecture:**

```
┌──────────────┐
│ Client       │
│ request_id:  │
│ abc-123      │
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│ API Gateway          │
│ span: gateway (1ms)  │
│ request_id: abc-123  │
├──────────────────────┤
│ Passes header:       │
│ X-Request-ID: abc123 │
└─────┬────────────────┘
      │
      ├─────────────┬──────────────┬─────────────┐
      │             │              │             │
      ▼             ▼              ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│Auth Service  │ │Order Service │ │Payment Svc   │
│span:         │ │span:         │ │span:         │
│auth(15ms)    │ │order(100ms)  │ │payment(1000)│
│request_id:   │ │request_id:   │ │request_id:   │
│abc-123       │ │abc-123       │ │abc-123       │
└──────────────┘ └──────────────┘ └──────────────┘

All traces have same request_id: abc-123
Tracer collects all spans
Builds timeline
Identifies bottleneck
```

**Jaeger UI (Visualization):**

```
Request Timeline:
[████████|██████████████|████████████████████]
  Auth   Order            Payment
  15ms   100ms            1000ms

Waterfall View:
  ├─ Gateway (1ms)
  │  ├─ Auth Service (15ms)
  │  │  └─ Auth DB (10ms)
  │  ├─ Order Service (100ms)
  │  │  ├─ Validation (2ms)
  │  │  ├─ Order DB (5ms)
  │  │  └─ Inventory Service (50ms)
  │  │     └─ Inventory DB (30ms)
  │  └─ Payment Service (1000ms)
  │     └─ External API (950ms) ← RED (slow)
  └─ Return response

Color coding:
  ✓ Green: < 100ms (fast)
  ⚠ Yellow: 100-500ms (ok)
  ✗ Red: > 500ms (slow)
```

**Implementation (OpenTelemetry):**

```python
from opentelemetry import trace, propagate
from opentelemetry.exporters.jaeger import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Set up Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

tracer = trace.get_tracer(__name__)

# In your service
def process_order(order_id, request_id):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order_id", order_id)
        span.set_attribute("request_id", request_id)
        
        # Database call
        with tracer.start_as_current_span("db_query") as db_span:
            db_span.set_attribute("table", "orders")
            order = db.query(f"SELECT * FROM orders WHERE id={order_id}")
        
        # Payment call
        with tracer.start_as_current_span("payment_service") as pay_span:
            pay_span.set_attribute("amount", order.total)
            result = call_payment_service(order)
        
        return result
```

---

### 8.5 Monitoring & Alerting

**Metrics to Monitor:**

```
1. Request Metrics
   - Requests per second (throughput)
   - Response time (p50, p95, p99)
   - Error rate (4xx, 5xx %)
   - Success rate

2. System Metrics
   - CPU usage
   - Memory usage
   - Disk I/O
   - Network I/O

3. Application Metrics
   - Database query time
   - Cache hit rate
   - Queue depth
   - Active connections

4. Business Metrics
   - Orders processed per minute
   - Transactions per hour
   - Revenue per hour
   - Customer conversion rate
```

**Alerting Thresholds:**

```
Alert if:
  - Response time > 1 second
  - Error rate > 5%
  - CPU > 80%
  - Memory > 85%
  - Disk space < 10%
  - Service is down (no healthy instances)
  - Failed deployments
  - High latency on Payment Service (> 500ms)

Example Alert (Prometheus):
alert: HighErrorRate
expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
for: 5m
annotations:
  summary: "High error rate detected"
  description: "Error rate is {{ $value | humanizePercentage }}"
  severity: critical
```

**Real-World Monitoring Stack:**

```
Prometheus (Metrics Collection)
    ↓ (Scrapes every 15s)
Services expose /metrics endpoint
    ↓
Prometheus stores time-series data
    ↓
Grafana (Visualization)
    ├─ Dashboard 1: Overall health
    ├─ Dashboard 2: Per-service metrics
    ├─ Dashboard 3: Business metrics
    └─ Dashboard 4: Infrastructure
    ↓
Alertmanager (Alert Routing)
    ├─ If error rate high → Slack notification
    ├─ If CPU high → PagerDuty alert
    └─ If database down → Phone call
```

---

## 9. 12-Factor Apps {#12-factor-apps}

**Reference:** https://12factor.net/

The 12-Factor App is a methodology for building cloud-native applications.

### Factor 1: Codebase

**Principle:** One codebase tracked in version control, many deploys

```
git repo: github.com/company/order-service

Deploy to:
  dev environment:     git checkout dev branch
  staging environment: git checkout staging branch
  production:          git checkout main branch

Same code, different config = different environments

❌ WRONG:
  dev branch has: ENVIRONMENT=development
  main branch has: ENVIRONMENT=production
  (Code differences!)

✓ RIGHT:
  All branches: const ENV = process.env.ENVIRONMENT
  dev deploy: ENVIRONMENT=development (env var)
  prod deploy: ENVIRONMENT=production (env var)
  (Config differences only!)
```

### Factor 2: Dependencies

**Principle:** Explicitly declare and isolate dependencies

```
✓ RIGHT:
  package.json lists: express@4.18.2, postgres@14.5
  docker build creates image with all deps
  Same image runs everywhere
  
❌ WRONG:
  apt-get install nodejs (version varies)
  npm install (might get different versions)
  Works on dev machine, fails on prod
  "It works on my machine" syndrome
```

### Factor 3: Config

**Principle:** Store config in environment variables

```
✓ RIGHT:
  const DB_HOST = process.env.DB_HOST;
  
  dev: docker run -e DB_HOST=localhost
  prod: docker run -e DB_HOST=prod-db.internal
  
  Same code, different config!

❌ WRONG:
  const DB_HOST = "localhost";  // Hardcoded!
  Must recompile for prod with "prod-db.internal"
  Risk of deploying wrong version
```

### Factor 4: Backing Services

**Principle:** Treat databases, caches, queues as attached resources

```
✓ RIGHT:
  DATABASE_URL=postgresql://user:pass@host:5432/db
  REDIS_URL=redis://host:6379
  AMQP_URL=amqp://host:5672
  
  Can swap databases without code change
  Dev: local database
  Prod: managed RDS

❌ WRONG:
  Hardcoded connection string in code
  Can't easily switch databases
  Must change code and redeploy
```

### Factor 5: Build, Release, Run

**Principle:** Strictly separate build, release, and run stages

```
CORRECT CI/CD Pipeline:

1. BUILD STAGE
   Input: Source code
   ├─ Compile/test
   ├─ Create artifact (Docker image)
   └─ Output: order-service:1.2.3
   
2. RELEASE STAGE
   Input: Docker image + config
   ├─ Tag version: 1.2.3
   ├─ Create release bundle
   └─ Output: Release 1.2.3 (ready to deploy)
   
3. RUN STAGE
   Input: Release 1.2.3
   ├─ Start containers
   ├─ Set environment variables
   └─ Output: Running service

Never mix stages!
```

### Factor 6: Processes

**Principle:** Execute app as one or more stateless processes

```
❌ WRONG: Stateful process
  let sessions = {}  // Stored in memory
  
  Instance 1 restarts → Memory cleared → All sessions lost
  Can't scale: Data in instance 1 not accessible in instance 2

✓ RIGHT: Stateless process
  Store session in Redis (external)
  
  Instance 1: GET session from Redis
  Instance 2: GET session from Redis
  Instance 1 crashes → Session still in Redis
  Easy to scale: Add instance 3, 4, 5 ...
```

### Factor 7: Port Binding

**Principle:** Export HTTP service via port binding

```
✓ RIGHT:
  App listens on port 3000
  No web server needed
  Container port 3000 → host port 3000
  Works anywhere
  
  docker run -p 3000:3000 order-service
  curl http://localhost:3000/health
  ✓ Works!

❌ WRONG:
  App depends on Nginx being installed
  Nginx forwards to app
  Tightly coupled to server
  Doesn't work in containers without installing Nginx
```

### Factor 8: Concurrency

**Principle:** Scale out via process model

```
❌ WRONG: Single process with threads
  java -Xmx4G app.jar
  
  One big server: 4 CPU, 16GB RAM
  Fails → Entire app down
  Can't scale

✓ RIGHT: Multiple lightweight processes
  docker run order-service (Process 1)
  docker run order-service (Process 2)
  docker run order-service (Process 3)
  docker run order-service (Process 4)
  ...
  
  Each process handles one request
  One crashes → Others still running
  Add more processes → Scale horizontally
```

### Factor 9: Disposability

**Principle:** Maximize robustness with fast startup and graceful shutdown

```
❌ WRONG: Slow startup, harsh shutdown
  Startup: Load entire database cache (5 minutes)
  Shutdown: Kill process immediately
  Result: Lost requests, corrupted data

✓ RIGHT: Fast startup, graceful shutdown
  Startup: < 5 seconds
  Shutdown: SIGTERM → Stop accepting requests → Wait for in-flight requests → Close connections → Exit
  
  Implementation:
  process.on('SIGTERM', async () => {
    server.close();  // Stop accepting new requests
    await sleep(10000);  // Wait 10 seconds for existing requests
    process.exit(0);
  });
```

### Factor 10: Dev/Prod Parity

**Principle:** Keep dev and prod as similar as possible

```
❌ WRONG: Different stacks
  Dev: SQLite in-memory
  Prod: PostgreSQL
  
  Dev: No authentication
  Prod: Full OAuth
  
  Dev: Sync processing
  Prod: Async with Kafka
  
  Works in dev, breaks in prod!

✓ RIGHT: Same stack
  Dev: PostgreSQL (local docker-compose)
  Prod: PostgreSQL (managed RDS)
  
  Dev: OAuth enabled (test credentials)
  Prod: OAuth enabled (real credentials)
  
  Dev: Kafka (docker-compose)
  Prod: Kafka (managed MSK)
  
  Same tech, different configs only
```

### Factor 11: Logs

**Principle:** Treat logs as event streams

```
❌ WRONG:
  app.log written to /var/log/
  Requires SSH to view
  Lost when container dies
  Hard to search

✓ RIGHT:
  App writes to stdout
  Container runtime captures it
  Forwarded to centralized logging (ELK)
  Easily searchable
  Persisted forever
```

### Factor 12: Admin Processes

**Principle:** Run admin/maintenance tasks as one-off processes

```
❌ WRONG:
  SSH to server
  Run migration manually
  Run cleanup scripts on prod
  Risky, hard to track

✓ RIGHT:
  Same container image as production
  docker exec order-service npm run migrate
  kubectl create job migrate --image=order-service npm run migrate
  
  Tracked in git
  Logged
  Can replay anytime
  Same environment as production
```

---

## 10. Case Studies {#case-studies}

### Case Study 1: Netflix - From Monolith to Microservices

**Timeline:**

```
2007-2008: Monolithic Era
├─ Single Java app
├─ Single Oracle database
├─ Monthly deployments
├─ Scaling: Add bigger servers ($$$)
└─ Problems: Every bug affected entire system

2009-2010: Transitional Phase
├─ Started using Cassandra (NoSQL)
├─ First service extracted: Recommendation Engine
├─ Streaming infrastructure overhauled
└─ Realized microservices needed

2011-2012: Major Migration
├─ Eureka (service discovery)
├─ Hystrix (resilience/circuit breakers)
├─ Ribbon (client-side load balancing)
├─ Deployed 100+ microservices
└─ Zero downtime deployments achieved

Present Day (Netflix)
├─ 1000+ microservices
├─ 100+ deployments per day
├─ Zero-downtime deployment standard
├─ Global scale (200+ countries)
└─ Self-service platform for developers
```

**Architecture Evolution:**

```
BEFORE (Monolith):
┌─────────────────────────┐
│ Netflix API             │
│ ├─ Streaming           │
│ ├─ Billing             │
│ ├─ Recommendations     │
│ ├─ Search              │
│ ├─ User Management     │
│ └─ Analytics           │
└──────────┬──────────────┘
           │
      ┌────▼──────┐
      │ Oracle DB │
      │ (Single)  │
      └───────────┘

Issues:
- Bug in billing → Entire service down
- Scaling search affects all services
- Can't deploy recommendations without affecting streaming
- Database single point of failure
```

**AFTER (Microservices):**

```
┌────────────────────────────────────────────┐
│ API Gateway (OAuth, Rate Limiting)         │
└────────┬───┬──────┬──────┬─────┬───────────┘
         │   │      │      │     │
    ┌────▼┐ ┌▼──┐ ┌─▼──┐ ┌▼──┐ ┌▼──┐
    │Reco │ │Sear│ │Bill│ │User│ │Ana│
    │mmnd │ │ch  │ │ing │ │Mgmt│ │lyt│
    └────┘ └────┘ └────┘ └────┘ └────┘
     │      │      │      │      │
     ▼      ▼      ▼      ▼      ▼
   Cass  Solr   Psql   Psql   Hbase

Benefits:
✓ Recommendation service down → Others work
✓ Search service scales independently
✓ Billing uses PostgreSQL, recommendations uses Cassandra
✓ Easy to update recommendation algorithm
✓ Deploy 100+ times per day safely
```

**Key Learnings:**

```
1. Infrastructure Must Come First
   ├─ Service discovery (Eureka)
   ├─ Circuit breakers (Hystrix)
   // filepath: /Users/mayankvashisht/Desktop/classes/DevOps & Cloud/mircoservice/Complete-Microservices-Guide.md

# Microservices Architecture: Complete Deep Dive (Sections 6-10)

## Table of Contents
6. [API Gateway](#api-gateway)
7. [Service Discovery](#service-discovery)
8. [Logging & Monitoring](#logging-monitoring)
9. [12-Factor Apps](#12-factor-apps)
10. [Case Studies](#case-studies)

---

## 6. API Gateway {#api-gateway}

### 6.1 What is an API Gateway?

**Definition:** A single entry point that sits between clients and microservices, handling cross-cutting concerns.

```
WITHOUT API Gateway (Chaos):
┌─────────────┐
│  Web App    │
└──────┬──────┘
       ├─ Knows Auth Service URL
       ├─ Knows Order Service URL
       ├─ Implements authentication logic
       ├─ Implements rate limiting
       ├─ Implements caching
       ├─ Implements logging
       └─ Implements error handling

┌─────────────┐
│ Mobile App  │
└──────┬──────┘
       ├─ Knows Auth Service URL (different!)
       ├─ Knows Order Service URL (different!)
       ├─ Implements same logic again
       ├─ Different rate limits
       ├─ Different error messages
       └─ Duplicated code everywhere!

┌─────────────┐
│ Desktop App │
└──────┬──────┘
       └─ Same duplication...

Problem: 3x code duplication, hard to maintain

WITH API Gateway (Clean):
┌────────────────────┐
│ All Clients        │
│ (Web/Mobile/etc)   │
└─────────┬──────────┘
          │
    ┌─────▼──────────────┐
    │   API Gateway      │
    │ ┌────────────────┐ │
    │ │ Auth (once)    │ │
    │ │ Rate limit     │ │
    │ │ Cache          │ │
    │ │ Route/log      │ │
    │ └────────────────┘ │
    └─────┬──┬──┬────────┘
          │  │  │
    ┌─────▼──▼──▼────┐
    │ Microservices  │
    └────────────────┘

Benefit: Single point for all cross-cutting concerns
```

### 6.2 Key Responsibilities of API Gateway

#### 1. Authentication & Authorization

**Authentication (Who are you?):**

```
User Login Flow:

Client Request:
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "secret123"
}
    ↓
API Gateway receives request
    ↓
Routes to Auth Service
    ↓
Auth Service:
  1. Find user by email
  2. Hash password, compare
  3. If match → Generate JWT token
  4. Return token to gateway

Auth Service Response:
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzEyMyIsImlhdCI6MTYzNDU2Nzg5MH0.signature...",
  "user_id": 123,
  "name": "John Doe",
  "roles": ["user", "admin"]
}
    ↓
API Gateway returns to client
    ↓
Client stores token in localStorage/cookies
    ↓
Client sends token in future requests:
Authorization: Bearer eyJhbGci...
```

**Authorization (What can you do?):**

```
User tries to access resource:
GET /api/orders/5001
Header: Authorization: Bearer eyJhbGci...

API Gateway:
1. Extract token from Authorization header
2. Verify signature (ensures not tampered)
3. Check expiry (not expired?)
4. Extract claims: {user_id: 123, roles: ["user"]}
5. Pass to Order Service with header:
   X-User-ID: 123
   X-User-Roles: user
   X-Request-ID: abc-123 (for tracing)

Order Service receives request
1. Trust gateway (it validated token)
2. Get user_id from X-User-ID header
3. Retrieve order 5001
4. Check: Does user 123 own order 5001?
5. If YES → Return order
6. If NO → 403 Forbidden

Example:
User 123 tries to view order 5001 (belongs to user 123)
  ✓ Order Service: "You own this order"
  ✓ Return order
  
User 123 tries to view order 5002 (belongs to user 456)
  ✗ Order Service: "You don't own this order"
  ✗ 403 Forbidden
```

**Multiple Authentication Methods:**

```
API Gateway supports:

1. JWT (JSON Web Token)
   Authorization: Bearer eyJhbGci...
   Stateless, scales well, good for APIs
   
2. OAuth2
   For third-party integrations
   "Login with Google", "Login with Facebook"
   
3. API Keys
   Authorization: X-API-Key: sk_live_abc123
   For service-to-service, mobile apps
   
4. mTLS (Mutual TLS)
   Client certificate verification
   Very secure, for internal services
```

**Real-World Implementation (Kong API Gateway):**

```yaml
# Kong Configuration
plugins:
  - name: jwt
    config:
      secret: "your-secret-key"
      key_claim_name: "sub"  # Extract user_id from "sub" claim
      
  - name: acl
    config:
      whitelist:
        - admin
        - user
        
  - name: rate-limiting
    config:
      minute: 100  # 100 requests per minute

routes:
  - name: orders_api
    paths:
      - /api/orders
    methods: 
      - GET
      - POST
    service: order-service
    plugins:
      - jwt
      - rate-limiting
```

---

#### 2. Request Routing

**Path-Based Routing:**

```
API Gateway inspects request path and routes accordingly

GET /api/products/5001
  → Check path
  → Matches /api/products/*
  → Route to Product Service (product-svc.internal:3000)
  → Proxy request to http://product-svc.internal:3000/products/5001

POST /api/orders
  → Check path
  → Matches /api/orders
  → Route to Order Service (order-svc.internal:3000)
  → Proxy request to http://order-svc.internal:3000/orders

GET /api/auth/profile
  → Check path
  → Matches /api/auth/*
  → Route to Auth Service (auth-svc.internal:3000)
  → Proxy request to http://auth-svc.internal:3000/profile
```

**Host-Based Routing:**

```
Different domains route to different services

products.api.shopnow.com
  → Product Service
  
orders.api.shopnow.com
  → Order Service
  
payments.api.shopnow.com
  → Payment Service

Useful for multi-tenant systems
```

**Header-Based Routing:**

```
GET /api/data
X-Version: v1
  → Route to old API Service

GET /api/data
X-Version: v2
  → Route to new API Service

Useful for API versioning and canary deployments
```

**Method-Based Routing:**

```
GET /api/products → Product Service (read)
POST /api/products → Product Service (write)
PUT /api/products/5 → Product Service (update)
DELETE /api/products/5 → Product Service (delete)

Same path, different methods, same service
Or route to different services based on method
```

**Real-World Nginx Configuration:**

```nginx
upstream product_service {
    server product-svc-1:3000 max_fails=3 fail_timeout=30s;
    server product-svc-2:3000 max_fails=3 fail_timeout=30s;
    server product-svc-3:3000 max_fails=3 fail_timeout=30s;
}

upstream order_service {
    server order-svc-1:3000;
    server order-svc-2:3000;
}

server {
    listen 80;
    server_name api.shopnow.com;

    # Product API routes
    location /api/products {
        proxy_pass http://product_service;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-ID $request_id;
    }

    # Order API routes
    location /api/orders {
        proxy_pass http://order_service;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-ID $request_id;
    }

    # Health check endpoint
    location /health {
        return 200 "OK";
    }
}
```

---

#### 3. Rate Limiting & Throttling

**Purpose:** Protect services from overload

```
Scenario: Black Friday without rate limiting

┌─────────────────────────────────────────┐
│ Product Service: 100,000 req/sec        │
│ Can only handle: 50,000 req/sec         │
│                                         │
│ Result: Overload, crashes, 50% fail    │
└─────────────────────────────────────────┘

Solution: Rate limit at gateway

API Gateway enforces:
  Max 100 requests/minute per user
  Max 1000 requests/minute per IP
  Max 500 requests/minute for anonymous users

When limit exceeded:
  Return 429 Too Many Requests
  Include header: Retry-After: 60
```

**Implementation Strategy:**

```
Option 1: Token Bucket Algorithm (Common)

Each user has a "bucket" with tokens
Bucket capacity: 100 tokens
Refill rate: 1 token per second

User makes request:
  ├─ If tokens > 0:
  │  ├─ Remove 1 token
  │  └─ Allow request
  │
  └─ If tokens = 0:
     └─ Deny request (429)

Timeline:
  10:00:00 - User: 100 tokens, makes 5 requests → 95 tokens left
  10:00:01 - Refill: +1 token → 96 tokens
  10:00:02 - Refill: +1 token → 97 tokens
  ...
  10:01:05 - Refill: +1 token → 100 tokens (full)
```

**Real Code (Python with Redis):**

```python
import redis
import time

redis_client = redis.Redis(host='localhost', port=6379)

def rate_limit(user_id, max_requests=100, window=60):
    """
    Rate limit: max_requests per window seconds
    """
    key = f"rate_limit:{user_id}"
    
    current_count = redis_client.get(key)
    
    if current_count is None:
        # First request
        redis_client.setex(key, window, 1)
        return True
    
    current_count = int(current_count)
    
    if current_count < max_requests:
        # Increment
        redis_client.incr(key)
        return True
    else:
        # Over limit
        return False

# Usage in API Gateway
@app.before_request
def apply_rate_limit():
    user_id = get_user_id_from_token()
    
    if not rate_limit(user_id):
        return {
            "error": "Rate limit exceeded",
            "retry_after": 60
        }, 429
```

**Different Rate Limits for Different Users:**

```
Free tier users:
  100 requests per minute

Premium tier users:
  10,000 requests per minute

Enterprise tier users:
  Unlimited (or very high limit)

Admin users:
  Unlimited

Implementation:
  Get user_id from token
  Lookup user tier from database
  Apply appropriate rate limit
```

---

#### 4. Caching

**Purpose:** Reduce backend load by serving cached responses

```
Scenario: Popular product page

GET /api/products/123 (iPhone 15 Pro)

First Request:
1. API Gateway checks cache
2. Not found
3. Routes to Product Service (100ms)
4. Product Service returns:
   {
     "id": 123,
     "name": "iPhone 15 Pro",
     "price": 1199.99,
     "stock": 5000,
     "rating": 4.8
   }
5. API Gateway caches response
   key: "products:123"
   ttl: 3600 (1 hour)
6. Returns to client (100ms)

Second Request (within 1 hour):
1. API Gateway checks cache
2. Found!
3. Returns immediately (1ms)
4. Never hits Product Service

Benefit: 100x faster for repeat requests!
```

**Cache Invalidation Strategies:**

```
1. Time-based (TTL - Most Common)
   Cache for 1 hour, then refresh
   Good for non-critical data
   
   Example: Product catalog
   Prices rarely change, can be stale 1 hour

2. Event-based (Invalidate on change)
   When product updates, clear cache
   Real-time data
   
   Example: Inventory
   User buys item → Inventory decreases
   → Product Service publishes event
   → API Gateway clears products:123 cache
   → Next request gets fresh data

3. Manual (Explicit invalidation)
   Admin clicks "Clear Cache"
   For critical data
   
   Example: Configuration
   Admin changes payment settings
   → Clear cache manually
   → All requests get new settings
```

**Cache Implementation (Nginx):**

```nginx
# Cache GET requests for 1 hour
location /api/products {
    # Cache GET requests only
    if ($request_method != GET) {
        proxy_pass http://product_service;
        proxy_no_cache 1;
    }
    
    proxy_pass http://product_service;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    proxy_cache_valid 200 1h;  # Cache 200 responses for 1 hour
    proxy_cache_valid 404 10m;  # Cache 404s for 10 minutes
    proxy_cache_bypass $http_cache_control;  # Respect Cache-Control header
}

# Don't cache POST/PUT/DELETE
location /api/orders {
    proxy_pass http://order_service;
    proxy_no_cache 1;  # Don't cache writes
}
```

---

#### 5. Request/Response Transformation

**Add Headers:**

```
Client sends:
POST /api/orders
{user_id: 123, items: [...]}

API Gateway:
1. Extract user_id from JWT token: 123
2. Add headers:
   X-User-ID: 123
   X-Request-ID: abc-123-def (for tracing)
   X-Timestamp: 2024-10-30T10:30:00Z
   X-Client-IP: 203.45.67.89
   X-API-Version: v1

Forward to Order Service:
POST /orders
Headers: X-User-ID, X-Request-ID, X-Timestamp, etc.
{user_id: 123, items: [...]}

Order Service benefits:
- No need to parse JWT again
- Has tracing ID for logs
- Knows client IP for analytics
- Knows API version for backwards compatibility
```

**Transform Request Body:**

```
Client sends (public API):
POST /api/orders {
  "product_id": 123,
  "quantity": 2,
  "coupon_code": "SAVE10"
}

API Gateway transforms to (internal API):
POST /orders {
  "product_id": 123,
  "quantity": 2,
  "user_id": 789,              ← Added
  "timestamp": 1634567890,     ← Added
  "request_id": "abc-123",     ← Added
  "coupon_code": "SAVE10",
  "ip_address": "203.45.67.89" ← Added
}

Order Service receives enriched data
No need for service to extract user_id
Service can trust all data came from gateway
```

**Transform Response:**

```
Order Service returns (internal format):
{
  "id": 5001,
  "user_id": 789,
  "status": "order_placed",
  "total_cents": 59999,
  "created_ts": 1634567890,
  "internal_status_code": 200
}

API Gateway transforms to (public format):
{
  "id": 5001,
  "status": "confirmed",
  "total": "$599.99",
  "created_at": "2024-10-30T10:30:00Z"
}

Benefits:
- Hide internal details (timestamps as UNIX, etc)
- Format data consistently
- Convert currencies, dates, etc
- Hide internal status codes
```

---

#### 6. Protocol Translation

```
Modern Gateway can translate between protocols

Client → HTTP/2 with REST
         ↓
    API Gateway
         ↓
Internal Services → gRPC with Protocol Buffers

OR

Client → REST JSON
         ↓
    API Gateway
         ↓
Internal Services → GraphQL
```

**Real-World Example:**

```
Mobile Client (HTTP/2, REST):
GET /api/products/123

API Gateway:
1. Receive HTTP/2 request
2. Convert to gRPC Protocol Buffers
3. Call internal Product Service:
   grpc.products.GetProduct(product_id=123)
4. Receive gRPC response (binary)
5. Convert back to HTTP/2 REST JSON
6. Return to client

Benefits:
- Internal services use fast gRPC
- External clients use familiar REST
- Gateway handles complexity
```

---

#### 7. Logging & Monitoring

**Log Every Request:**

```
API Gateway logs:

{
  "timestamp": "2024-10-30T10:30:00.123Z",
  "request_id": "req-abc-123-def",
  "client_ip": "203.45.67.89",
  "user_id": 123,
  "method": "POST",
  "path": "/api/orders",
  "query_params": "?sort=date",
  "request_headers": {
    "Authorization": "Bearer [REDACTED]",
    "Content-Type": "application/json",
    "User-Agent": "Mozilla/5.0"
  },
  "request_body_size": 256,
  "status_code": 201,
  "response_time_ms": 245,
  "response_size": 512,
  "service_called": "order-service",
  "service_response_time_ms": 100,
  "cache_hit": false,
  "rate_limit_remaining": 87
}
```

**Monitoring Metrics:**

```
API Gateway tracks:
1. Request rate (requests/sec)
2. Error rate (4xx, 5xx %)
3. Response time (p50, p95, p99)
4. Cache hit rate
5. Rate limit violations
6. Upstream service health

Alerts:
- Response time > 1 second
- Error rate > 5%
- Rate limit violations > 100/min
- Service down (no healthy instances)
```

---

### 6.3 Popular API Gateways

| Gateway | Pros | Cons | Best For |
|---------|------|------|----------|
| **Kong** | Open-source, powerful plugins | Complex setup | Production, on-prem |
| **Nginx** | Lightweight, fast | Limited features | Simple routing, cache |
| **AWS API Gateway** | Serverless, managed | Limited control, expensive | AWS ecosystem |
| **Traefik** | Kubernetes-native, dynamic | Kubernetes-only | Kubernetes clusters |
| **Istio** | Service mesh + gateway, powerful | Heavy, complex | Complex microservices |
| **Apigee** | Enterprise features, analytics | Expensive | Large enterprises |

---

## 7. Service Discovery {#service-discovery}

### 7.1 The Problem Service Discovery Solves

**Without Service Discovery (Monolithic Era):**

```
Hardcoded Service Locations:

Order Service:
const PAYMENT_SERVICE_URL = "payment-service.prod.internal:3000"
const INVENTORY_SERVICE_URL = "inventory-service.prod.internal:3001"

Issues:
1. If Payment Service restarts → New IP → Hardcoded URL fails
2. Auto-scaling: New instances added, Order Service doesn't know
3. Maintenance: Must coordinate updates across services
4. Manual: DevOps must update hardcoded URLs
```

**With Service Discovery (Microservices):**

```
Dynamic Service Locations:

Order Service queries Service Registry:
"Where is payment-service?"
    ↓
Registry responds with 3 instances:
  - 192.168.1.10:3000
  - 192.168.1.11:3000
  - 192.168.1.12:3000

Order Service picks one (load balanced)
Makes request to that instance

If instance dies:
  - Stops sending heartbeat
  - Registry removes it
  - Order Service won't route to it anymore
  - Auto-replacement spins up new instance
  - Order Service finds it automatically
```

---

### 7.2 How Service Discovery Works

```
Architecture:

┌────────────────────────────────┐
│   Service Registry             │
│   (Eureka/Consul/etcd)         │
│                                │
│ Services registered:           │
│ {                              │
│   "payment": [                 │
│     {ip: 192.168.1.10, port: 3000},
│     {ip: 192.168.1.11, port: 3000},
│     {ip: 192.168.1.12, port: 3000}
│   ],                           │
│   "inventory": [               │
│     {ip: 192.168.1.20, port: 3001},
│     {ip: 192.168.1.21, port: 3001}
│   ]                            │
│ }                              │
└────────────────────────────────┘
        ▲   ▲   ▲   ▲   ▲
        │   │   │   │   │
    Heartbeat & queries


┌──────────────────────────────────┐
│ Service Instances                │
├──────────────────────────────────┤
│ Payment Service #1: 192.168.1.10 │
│ Payment Service #2: 192.168.1.11 │
│ Payment Service #3: 192.168.1.12 │
│ Inventory Service #1: 192.168.1.20│
│ Inventory Service #2: 192.168.1.21│
└──────────────────────────────────┘
```

### 7.3 Lifecycle: Service Registration & Discovery

**Phase 1: Service Startup & Registration**

```
Payment Service Instance 1 starts:

1. Read config:
   SERVICE_NAME=payment
   PORT=3000
   REGISTRY_URL=http://eureka:8761

2. Get own IP: 192.168.1.10

3. Register with Eureka:
   POST http://eureka:8761/eureka/apps/payment
   {
     "instance": {
       "instanceId": "payment-instance-1",
       "hostName": "payment-1.internal",
       "app": "PAYMENT",
       "ipAddr": "192.168.1.10",
       "status": "UP",
       "port": {"@enabled": true, "$": 3000},
       "healthCheckUrl": "http://192.168.1.10:3000/health",
       "leaseInfo": {
         "renewalIntervalInSecs": 30,
         "durationInSecs": 90
       }
     }
   }

4. Eureka response: 204 No Content (accepted)

5. Service ready to receive requests ✓
```

**Phase 2: Heartbeat (Keep Alive)**

```
Every 30 seconds, Payment Service sends heartbeat:

PUT http://eureka:8761/eureka/apps/payment/payment-instance-1

Eureka receives heartbeat:
  ├─ Check if instance exists
  ├─ If exists: Update "last heartbeat" timestamp
  ├─ If not exists: Register it (recovery)
  └─ Return 200 OK

Timeline:
  10:00:00 - Payment Service registers
  10:00:30 - Sends heartbeat #1
  10:01:00 - Sends heartbeat #2
  10:01:30 - Sends heartbeat #3
  ...

If heartbeat stops:
  10:02:00 - Last heartbeat was 60 seconds ago
  10:02:30 - Eureka marks as DOWN (missed 2 heartbeats)
  10:03:00 - Eureka removes from registry
  → Order Service won't route to it anymore
```

**Phase 3: Client Discovery (Order Service)**

```
Order Service wants to call Payment Service:

1. Query Eureka:
   GET http://eureka:8761/eureka/apps/payment

2. Eureka returns healthy instances:
   {
     "application": {
       "name": "PAYMENT",
       "instance": [
         {
           "instanceId": "payment-instance-1",
           "hostName": "payment-1.internal",
           "ipAddr": "192.168.1.10",
           "port": 3000,
           "status": "UP"
         },
         {
           "instanceId": "payment-instance-2",
           "hostName": "payment-2.internal",
           "ipAddr": "192.168.1.11",
           "port": 3000,
           "status": "UP"
         },
         {
           "instanceId": "payment-instance-3",
           "hostName": "payment-3.internal",
           "ipAddr": "192.168.1.12",
           "port": 3000,
           "status": "UP"
         }
       ]
     }
   }

3. Order Service caches this list (for 30 seconds)

4. Load balancer picks one: 192.168.1.10

5. Make request:
   POST http://192.168.1.10:3000/process-payment

6. If request fails:
   ├─ Retry to another instance
   └─ Or query Eureka for fresh list
```

---

### 7.4 Client-Side vs Server-Side Discovery

#### Client-Side Discovery (Eureka, Consul)

```
Order Service is responsible for discovery

┌─────────────────────────────────────┐
│ Order Service                       │
├─────────────────────────────────────┤
│ 1. Query Service Registry           │
│ 2. Get list of instances            │
│ 3. Implement load balancing logic   │
│ 4. Choose one instance              │
│ 5. Call it directly                 │
└─────────────────────────────────────┘
    ▲
    │ Query/get list
    │
┌───▼────────────────────────────────┐
│ Service Registry                    │
│ payment: [inst1, inst2, inst3]      │
└─────────────────────────────────────┘

Pros:
  - Simple
  - No extra hop
  - Client controls load balancing

Cons:
  - Client must implement logic
  - Language-specific libraries
  - Each client must have same logic
```

**Implementation (Spring Cloud Eureka):**

```java
// Order Service calls Payment Service
@Service
public class OrderService {
    
    @Autowired
    private DiscoveryClient discoveryClient;
    
    @Autowired
    private RestTemplate restTemplate;
    
    public void processOrder(Order order) {
        // Get all Payment Service instances
        List<ServiceInstance> instances = 
            discoveryClient.getInstances("payment-service");
        
        if (instances.isEmpty()) {
            throw new ServiceNotAvailableException("Payment service not found");
        }
        
        // Load balance: pick random instance
        ServiceInstance instance = instances.get(
            new Random().nextInt(instances.size())
        );
        
        // Build URL
        String url = String.format(
            "http://%s:%d/process-payment",
            instance.getHost(),
            instance.getPort()
        );
        
        // Call Payment Service directly
        PaymentResponse response = restTemplate.postForObject(
            url,
            order,
            PaymentResponse.class
        );
    }
}
```

---

#### Server-Side Discovery (Kubernetes, AWS ELB)

```
Load Balancer is responsible for discovery

┌──────────────────────────────┐
│ Order Service                │
│ Calls: payment-service.svc   │
└──────┬───────────────────────┘
       │
   ┌───▼──────────────────────┐
   │ Load Balancer / DNS      │
   │ payment-service.svc      │
   │ → 192.168.1.5 (LB IP)    │
   └───┬──────────────────────┘
       │
   ┌───┴─────┬──────────┬────────┐
   │         │          │        │
┌──▼──┐ ┌──▼──┐ ┌──▼───┐ ┌──▼──┐
│ inst1│ │inst2│ │ inst3│ │inst4│
└─────┘ └─────┘ └──────┘ └─────┘

Pros:
  - Language-agnostic
  - LB handles everything
  - Simple client code

Cons:
  - Extra network hop
  - LB is potential bottleneck
  - LB is single point of failure
```

**Implementation (Kubernetes):**

```yaml
# Service resource in Kubernetes
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
  type: ClusterIP

---

# Order Service deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order
  template:
    metadata:
      labels:
        app: order
    spec:
      containers:
      - name: order
        image: order-service:latest
        env:
          - name: PAYMENT_SERVICE_URL
            value: "http://payment-service:3000"
        # No hardcoded IPs!
```

**Order Service Code (Simple):**

```java
// Very simple - no discovery logic needed
@Service
public class OrderService {
    
    @Value("${payment.service.url}")
    private String paymentServiceUrl;
    
    @Autowired
    private RestTemplate restTemplate;
    
    public void processOrder(Order order) {
        // Just call the URL
        // Kubernetes handles load balancing internally
        PaymentResponse response = restTemplate.postForObject(
            paymentServiceUrl + "/process-payment",
            order,
            PaymentResponse.class
        );
    }
}
```

---

### 7.5 Popular Service Discovery Tools

#### Eureka (Netflix/Spring Cloud)

```
For: On-premises, Java/Spring Boot
Register: Active (services register themselves)
Discovery: Client-side

Architecture:
  Eureka Server (Central Registry)
    ↑ Register/Heartbeat
  All Service Instances

Workflow:
1. Service registers on startup
2. Service sends heartbeat every 30s
3. Eureka marks as DOWN if no heartbeat for 90s
4. Clients query Eureka periodically
5. Clients use Ribbon for load balancing

Example in Spring Boot:
@SpringBootApplication
@EnableEurekaClient
public class PaymentServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
    }
}
```

#### Consul (HashiCorp)

```
For: Multi-datacenter, cross-platform
Register: Both (servers register services)
Discovery: Server-side or client-side

Architecture:
  Consul Server Cluster
    ↑ Register/Health Checks
  All Service Instances

Features:
- Multi-datacenter support (DC1, DC2, DC3)
- Built-in health checks (HTTP, TCP, Exec)
- DNS interface (services.consul)
- Key-value store
- ACLs for security

Example:
Service registers:
  curl -X PUT http://localhost:8500/v1/agent/service/register -d '{
    "ID": "payment-1",
    "Name": "payment",
    "Address": "192.168.1.10",
    "Port": 3000,
    "Check": {
      "HTTP": "http://192.168.1.10:3000/health",
      "Interval": "10s"
    }
  }'

Query:
  DNS: dig payment.service.consul
  → Returns: 192.168.1.10, 192.168.1.11, 192.168.1.12
```

#### Kubernetes Service Discovery

```
For: Kubernetes-only
Register: Automatic (kubelet registers)
Discovery: DNS-based

Architecture:
  Kubernetes API Server
    ↑ Pod lifecycle events
  All Pod Instances

How it works:
1. Pod starts
2. kubelet reports pod status to API Server
3. Kubernetes updates Service endpoints
4. CoreDNS watches Service endpoints
5. Clients query DNS: payment-service.default.svc.cluster.local
6. CoreDNS responds with current pod IPs

Benefits:
- Zero configuration
- Automatic scaling
- Self-healing
- Built into Kubernetes

Example:
payment-service.default.svc.cluster.local
├─ default = namespace
├─ svc = Kubernetes Service
├─ cluster.local = cluster DNS domain
└─ Resolves to service ClusterIP (192.168.1.5)
   Which load balances to pod IPs
```

---

## 8. Logging & Monitoring {#logging-monitoring}

### 8.1 Why Externalize Logs?

**Problem: Logs in Containers**

```
Traditional Monolith:
┌────────────────────────────────┐
│ Single Server                  │
│ /var/log/application.log       │
│ 100GB of logs                  │
│                                │
│ Easy to SSH and debug:         │
│ $ tail -f /var/log/app.log     │
└────────────────────────────────┘

Microservices with Containers:
┌──────────────────────────────────────────────────┐
│ Kubernetes Cluster                               │
├──────────────────────────────────────────────────┤
│ Pod 1 (Order Service)                            │
│ /var/log/app.log - 10GB                          │
│ But where is it physically?                      │
│ It's inside a container!                         │
│                                                  │
│ Pod 2 (Payment Service)                          │
│ /var/log/app.log - 5GB                           │
│                                                  │
│ Pod 3 (Inventory Service)                        │
│ /var/log/app.log - 8GB                           │
│                                                  │
│ Pod 4 (Order Service) - CRASHED                  │
│ /var/log/app.log - DELETED!                      │
│ (Container is gone, logs gone!)                  │
│                                                  │
│ To debug Pod 4's crash:                          │
│ Logs are lost forever                            │
└──────────────────────────────────────────────────┘

Problems:
1. Logs scattered across pods
2. Container dies → logs disappear
3. Scaling: 50 instances → 50 separate log files
4. How to debug cross-service issues?
5. Need to SSH into specific pod
6. By the time you SSH, pod might be restarted
```

**Solution: Externalize Logs**

```
All logs go to centralized system

┌──────────────────────────────────┐
│ Kubernetes Cluster               │
├──────────────────────────────────┤
│ Pod 1: Order Service             │
│ Pod 2: Payment Service           │
│ Pod 3: Inventory Service         │
│ Pod 4: Order Service (crashed)   │
│                                  │
│ All pods log to:                 │
│ STDOUT (container logging)       │
└──────────────┬───────────────────┘
               │
        ┌──────▼──────┐
        │ Log Shipper  │
        │ (Fluentd)    │
        └──────┬───────┘
               │
      ┌────────▼─────────┐
      │ Log Storage       │
      │ (Elasticsearch)   │
      │ 1 TB of logs      │
      │ Searchable        │
      └────────┬──────────┘
               │
      ┌────────▼─────────┐
      │ Log Viewer        │
      │ (Kibana)          │
      │ Web UI            │
      └──────────────────┘

Benefits:
✓ All logs in one place
✓ Logs persist after pod dies
✓ Full-text searchable
✓ Correlate logs across services
✓ Analytics on logs
```

---

### 8.2 ELK Stack (Elasticsearch, Logstash, Kibana)

**Architecture:**

```
Services → Logs → Logstash → Elasticsearch → Kibana

┌────────────────────────────────────────┐
│ Services (log to STDOUT)               │
├────────────────────────────────────────┤
│ Order Service logs to console          │
│ Payment Service logs to console        │
│ Inventory Service logs to console      │
│ Notification Service logs to console   │
└────────────┬────────────────────────────┘
             │ docker logs / kubectl logs
             ▼
┌────────────────────────────────────────┐
│ Log Collector (Fluentd/Logstash)       │
├────────────────────────────────────────┤
│ Collect from all containers            │
│ Parse unstructured text                │
│ Enrich with metadata:                  │
│  - container_name                      │
│  - pod_id                              │
│  - service_name                        │
│  - timestamp                           │
│  - host                                │
│ Convert to JSON                        │
└────────────┬────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ Elasticsearch (Index & Store)          │
├────────────────────────────────────────┤
│ Receives JSON logs                     │
│ Indexes them (full-text search)        │
│ Stores in time-series indices:         │
│  logs-2024.10.30                       │
│  logs-2024.10.31                       │
│ Retention: Keep 30 days                │
│ Delete older logs automatically        │
└────────────┬────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ Kibana (Query & Visualize)             │
├────────────────────────────────────────┤
│ Web interface to search logs           │
│ Query: order_id = 5001                 │
│ Results: All logs mentioning 5001      │
│ Timeline: See flow of events           │
│ Analytics: Count, aggregate, chart     │
└────────────────────────────────────────┘
```

**Real Log Flow:**

```
Time: 10:30:00

Order Service logs:
  2024-10-30T10:30:00.123Z [INFO] Order created: order_id=5001, user_id=123

Console output (pod stdout):
  Order created: order_id=5001, user_id=123

Fluentd reads stdout:
  Parses timestamp, level, message
  Adds metadata: container=order-service-1, pod=order-pod-1

JSON created:
  {
    "timestamp": "2024-10-30T10:30:00.123Z",
    "level": "INFO",
    "message": "Order created",
    "order_id": "5001",
    "user_id": "123",
    "container_name": "order-service",
    "pod_name": "order-pod-1",
    "host": "worker-1"
  }

Elasticsearch indexes:
  Index: logs-2024.10.30
  Doc ID: unique_id_123
  Searchable by: timestamp, level, order_id, container_name, etc

User searches Kibana:
  Query: order_id:5001
  Results:
    10:30:00 - Order Service: Order created
    10:30:01 - Payment Service: Payment processing
    10:30:02 - Payment Service: Payment failed
    10:30:02 - Order Service: Rollback order
    10:30:03 - Notification Service: Sent email
  
  Complete trace! All 50 logs for that order!
```

---

### 8.3 Log Levels & Structured Logging

**Log Levels:**

```
1. DEBUG
   Most verbose, for development
   Example: "Query executed: SELECT * FROM users WHERE id=123"

2. INFO
   Normal operations
   Example: "Order processed: order_id=5001"

3. WARN
   Something unexpected, but handled
   Example: "Payment retry: attempt 2/3"

4. ERROR
   Something went wrong
   Example: "Database connection failed"

5. FATAL
   Critical error, service might crash
   Example: "Out of memory"

Production config:
  ├─ Dev environment: DEBUG
  ├─ Staging environment: INFO
  └─ Production: WARN or ERROR (spam reduction)
```

**Unstructured Logging (Bad):**

```
User logs in and something goes wrong

Log output:
  User john@example.com login attempt
  Verification failed
  User account locked
  
Problems:
- Hard to parse programmatically
- Can't search for specific fields
- Requires human interpretation
- Can't aggregate statistics
```

**Structured Logging (Good):**

```
Same scenario, structured:

Log JSON:
{
  "timestamp": "2024-10-30T10:30:00.123Z",
  "level": "WARN",
  "event": "login_failed",
  "user_id": 123,
  "email": "john@example.com",
  "ip_address": "203.45.67.89",
  "failure_reason": "invalid_password",
  "attempt": 3,
  "service": "auth-service",
  "version": "1.2.3"
}

Benefits:
✓ Easily parsed by machine
✓ Searchable fields:
  - Find all logins from IP 203.45.67.89
  - Find all failed attempts for john@example.com
  - Count failed logins per hour
✓ Aggregatable:
  - Average failed attempts
  - Alert if > 10 failed logins in 1 minute
✓ Alertable: Set up thresholds
```

**Structured Logging Example (Code):**

```javascript
// ❌ Bad - Unstructured
console.log("User login: " + email + " from IP " + ip);

// ✓ Good - Structured
console.log(JSON.stringify({
  timestamp: new Date().toISOString(),
  level: "INFO",
  event: "user_login",
  user_id: 123,
  email: email,
  ip_address: ip,
  session_id: session_id,
  duration_ms: loginTime
}));

// In Elasticsearch, this becomes:
{
  "user_id": 123,
  "email": "john@example.com",
  "ip_address": "203.45.67.89",
  "event": "user_login"
}

// Can query:
// "Find all logins from IP 203.45.67.89"
// "Count logins per user per day"
// "Alert if 100+ logins in 1 minute from same IP"
```

---

### 8.4 Distributed Tracing (Jaeger, Zipkin)

**Problem: Slow Request, Where's the Bottleneck?**

```
User clicks "Buy Now"
App waits 5 seconds
Response comes back

Question: Where was the bottleneck?

Without tracing:
  - Was it the database?
  - Was it the API?
  - Was it the payment service?
  - Was it the network?
  - ???

With distributed tracing:
  10:30:00.000 - API Gateway: receives request (1ms)
  10:30:00.001 - Auth Service call (15ms)
     10:30:00.001 - Auth DB query (10ms)
  10:30:00.016 - Order Service call (100ms)
     10:30:00.016 - Validation (2ms)
     10:30:00.018 - Order DB insert (5ms)
     10:30:00.023 - Inventory Service call (50ms)
        10:30:00.023 - Inventory DB query (30ms)
     10:30:00.073 - Payment Service call (1000ms) ← SLOW!
        10:30:00.073 - External payment API call (950ms) ← BOTTLENECK!
  10:30:01.073 - Response

Total: 1074ms
Bottleneck: External payment API is slow!
```

**Distributed Tracing Architecture:**

```
┌──────────────┐
│ Client       │
│ request_id:  │
│ abc-123      │
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│ API Gateway          │
│ span: gateway (1ms)  │
│ request_id: abc-123  │
├──────────────────────┤
│ Passes header:       │
│ X-Request-ID: abc123 │
└─────┬────────────────┘
      │
      ├─────────────┬──────────────┬─────────────┐
      │             │              │             │
      ▼             ▼              ▼             ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│Auth Service  │ │Order Service │ │Payment Svc   │
│span:         │ │span:         │ │span:         │
│auth(15ms)    │ │order(100ms)  │ │payment(1000)│
│request_id:   │ │request_id:   │ │request_id:   │
│abc-123       │ │abc-123       │ │abc-123       │
└──────────────┘ └──────────────┘ └──────────────┘

All traces have same request_id: abc-123
Tracer collects all spans
Builds timeline
Identifies bottleneck
```

**Jaeger UI (Visualization):**

```
Request Timeline:
[████████|██████████████|████████████████████]
  Auth   Order            Payment
  15ms   100ms            1000ms

Waterfall View:
  ├─ Gateway (1ms)
  │  ├─ Auth Service (15ms)
  │  │  └─ Auth DB (10ms)
  │  ├─ Order Service (100ms)
  │  │  ├─ Validation (2ms)
  │  │  ├─ Order DB (5ms)
  │  │  └─ Inventory Service (50ms)
  │  │     └─ Inventory DB (30ms)
  │  └─ Payment Service (1000ms)
  │     └─ External API (950ms) ← RED (slow)
  └─ Return response

Color coding:
  ✓ Green: < 100ms (fast)
  ⚠ Yellow: 100-500ms (ok)
  ✗ Red: > 500ms (slow)
```

**Implementation (OpenTelemetry):**

```python
from opentelemetry import trace, propagate
from opentelemetry.exporters.jaeger import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Set up Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

tracer = trace.get_tracer(__name__)

# In your service
def process_order(order_id, request_id):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order_id", order_id)
        span.set_attribute("request_id", request_id)
        
        # Database call
        with tracer.start_as_current_span("db_query") as db_span:
            db_span.set_attribute("table", "orders")
            order = db.query(f"SELECT * FROM orders WHERE id={order_id}")
        
        # Payment call
        with tracer.start_as_current_span("payment_service") as pay_span:
            pay_span.set_attribute("amount", order.total)
            result = call_payment_service(order)
        
        return result
```

---

### 8.5 Monitoring & Alerting

**Metrics to Monitor:**

```
1. Request Metrics
   - Requests per second (throughput)
   - Response time (p50, p95, p99)
   - Error rate (4xx, 5xx %)
   - Success rate

2. System Metrics
   - CPU usage
   - Memory usage
   - Disk I/O
   - Network I/O

3. Application Metrics
   - Database query time
   - Cache hit rate
   - Queue depth
   - Active connections

4. Business Metrics
   - Orders processed per minute
   - Transactions per hour
   - Revenue per hour
   - Customer conversion rate
```

**Alerting Thresholds:**

```
Alert if:
  - Response time > 1 second
  - Error rate > 5%
  - CPU > 80%
  - Memory > 85%
  - Disk space < 10%
  - Service is down (no healthy instances)
  - Failed deployments
  - High latency on Payment Service (> 500ms)

Example Alert (Prometheus):
alert: HighErrorRate
expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
for: 5m
annotations:
  summary: "High error rate detected"
  description: "Error rate is {{ $value | humanizePercentage }}"
  severity: critical
```

**Real-World Monitoring Stack:**

```
Prometheus (Metrics Collection)
    ↓ (Scrapes every 15s)
Services expose /metrics endpoint
    ↓
Prometheus stores time-series data
    ↓
Grafana (Visualization)
    ├─ Dashboard 1: Overall health
    ├─ Dashboard 2: Per-service metrics
    ├─ Dashboard 3: Business metrics
    └─ Dashboard 4: Infrastructure
    ↓
Alertmanager (Alert Routing)
    ├─ If error rate high → Slack notification
    ├─ If CPU high → PagerDuty alert
    └─ If database down → Phone call
```

---

## 9. 12-Factor Apps {#12-factor-apps}

**Reference:** https://12factor.net/

The 12-Factor App is a methodology for building cloud-native applications.

### Factor 1: Codebase

**Principle:** One codebase tracked in version control, many deploys

```
git repo: github.com/company/order-service

Deploy to:
  dev environment:     git checkout dev branch
  staging environment: git checkout staging branch
  production:          git checkout main branch

Same code, different config = different environments

❌ WRONG:
  dev branch has: ENVIRONMENT=development
  main branch has: ENVIRONMENT=production
  (Code differences!)

✓ RIGHT:
  All branches: const ENV = process.env.ENVIRONMENT
  dev deploy: ENVIRONMENT=development (env var)
  prod deploy: ENVIRONMENT=production (env var)
  (Config differences only!)
```

### Factor 2: Dependencies

**Principle:** Explicitly declare and isolate dependencies

```
✓ RIGHT:
  package.json lists: express@4.18.2, postgres@14.5
  docker build creates image with all deps
  Same image runs everywhere
  
❌ WRONG:
  apt-get install nodejs (version varies)
  npm install (might get different versions)
  Works on dev machine, fails on prod
  "It works on my machine" syndrome
```

### Factor 3: Config

**Principle:** Store config in environment variables

```
✓ RIGHT:
  const DB_HOST = process.env.DB_HOST;
  
  dev: docker run -e DB_HOST=localhost
  prod: docker run -e DB_HOST=prod-db.internal
  
  Same code, different config!

❌ WRONG:
  const DB_HOST = "localhost";  // Hardcoded!
  Must recompile for prod with "prod-db.internal"
  Risk of deploying wrong version
```

### Factor 4: Backing Services

**Principle:** Treat databases, caches, queues as attached resources

```
✓ RIGHT:
  DATABASE_URL=postgresql://user:pass@host:5432/db
  REDIS_URL=redis://host:6379
  AMQP_URL=amqp://host:5672
  
  Can swap databases without code change
  Dev: local database
  Prod: managed RDS

❌ WRONG:
  Hardcoded connection string in code
  Can't easily switch databases
  Must change code and redeploy
```

### Factor 5: Build, Release, Run

**Principle:** Strictly separate build, release, and run stages

```
CORRECT CI/CD Pipeline:

1. BUILD STAGE
   Input: Source code
   ├─ Compile/test
   ├─ Create artifact (Docker image)
   └─ Output: order-service:1.2.3
   
2. RELEASE STAGE
   Input: Docker image + config
   ├─ Tag version: 1.2.3
   ├─ Create release bundle
   └─ Output: Release 1.2.3 (ready to deploy)
   
3. RUN STAGE
   Input: Release 1.2.3
   ├─ Start containers
   ├─ Set environment variables
   └─ Output: Running service

Never mix stages!
```

### Factor 6: Processes

**Principle:** Execute app as one or more stateless processes

```
❌ WRONG: Stateful process
  let sessions = {}  // Stored in memory
  
  Instance 1 restarts → Memory cleared → All sessions lost
  Can't scale: Data in instance 1 not accessible in instance 2

✓ RIGHT: Stateless process
  Store session in Redis (external)
  
  Instance 1: GET session from Redis
  Instance 2: GET session from Redis
  Instance 1 crashes → Session still in Redis
  Easy to scale: Add instance 3, 4, 5 ...
```

### Factor 7: Port Binding

**Principle:** Export HTTP service via port binding

```
✓ RIGHT:
  App listens on port 3000
  No web server needed
  Container port 3000 → host port 3000
  Works anywhere
  
  docker run -p 3000:3000 order-service
  curl http://localhost:3000/health
  ✓ Works!

❌ WRONG:
  App depends on Nginx being installed
  Nginx forwards to app
  Tightly coupled to server
  Doesn't work in containers without installing Nginx
```

### Factor 8: Concurrency

**Principle:** Scale out via process model

```
❌ WRONG: Single process with threads
  java -Xmx4G app.jar
  
  One big server: 4 CPU, 16GB RAM
  Fails → Entire app down
  Can't scale

✓ RIGHT: Multiple lightweight processes
  docker run order-service (Process 1)
  docker run order-service (Process 2)
  docker run order-service (Process 3)
  docker run order-service (Process 4)
  ...
  
  Each process handles one request
  One crashes → Others still running
  Add more processes → Scale horizontally
```

### Factor 9: Disposability

**Principle:** Maximize robustness with fast startup and graceful shutdown

```
❌ WRONG: Slow startup, harsh shutdown
  Startup: Load entire database cache (5 minutes)
  Shutdown: Kill process immediately
  Result: Lost requests, corrupted data

✓ RIGHT: Fast startup, graceful shutdown
  Startup: < 5 seconds
  Shutdown: SIGTERM → Stop accepting requests → Wait for in-flight requests → Close connections → Exit
  
  Implementation:
  process.on('SIGTERM', async () => {
    server.close();  // Stop accepting new requests
    await sleep(10000);  // Wait 10 seconds for existing requests
    process.exit(0);
  });
```

### Factor 10: Dev/Prod Parity

**Principle:** Keep dev and prod as similar as possible

```
❌ WRONG: Different stacks
  Dev: SQLite in-memory
  Prod: PostgreSQL
  
  Dev: No authentication
  Prod: Full OAuth
  
  Dev: Sync processing
  Prod: Async with Kafka
  
  Works in dev, breaks in prod!

✓ RIGHT: Same stack
  Dev: PostgreSQL (local docker-compose)
  Prod: PostgreSQL (managed RDS)
  
  Dev: OAuth enabled (test credentials)
  Prod: OAuth enabled (real credentials)
  
  Dev: Kafka (docker-compose)
  Prod: Kafka (managed MSK)
  
  Same tech, different configs only
```

### Factor 11: Logs

**Principle:** Treat logs as event streams

```
❌ WRONG:
  app.log written to /var/log/
  Requires SSH to view
  Lost when container dies
  Hard to search

✓ RIGHT:
  App writes to stdout
  Container runtime captures it
  Forwarded to centralized logging (ELK)
  Easily searchable
  Persisted forever
```

### Factor 12: Admin Processes

**Principle:** Run admin/maintenance tasks as one-off processes

```
❌ WRONG:
  SSH to server
  Run migration manually
  Run cleanup scripts on prod
  Risky, hard to track

✓ RIGHT:
  Same container image as production
  docker exec order-service npm run migrate
  kubectl create job migrate --image=order-service npm run migrate
  
  Tracked in git
  Logged
  Can replay anytime
  Same environment as production
```

---

## 10. Case Studies {#case-studies}

### Case Study 1: Netflix - From Monolith to Microservices

**Timeline:**

```
2007-2008: Monolithic Era
├─ Single Java app
├─ Single Oracle database
├─ Monthly deployments
├─ Scaling: Add bigger servers ($$$)
└─ Problems: Every bug affected entire system

2009-2010: Transitional Phase
├─ Started using Cassandra (NoSQL)
├─ First service extracted: Recommendation Engine
├─ Streaming infrastructure overhauled
└─ Realized microservices needed

2011-2012: Major Migration
├─ Eureka (service discovery)
├─ Hystrix (resilience/circuit breakers)
├─ Ribbon (client-side load balancing)
├─ Deployed 100+ microservices
└─ Zero downtime deployments achieved

Present Day (Netflix)
├─ 1000+ microservices
├─ 100+ deployments per day
├─ Zero-downtime deployment standard
├─ Global scale (200+ countries)
└─ Self-service platform for developers
```

**Architecture Evolution:**

```
BEFORE (Monolith):
┌─────────────────────────┐
│ Netflix API             │
│ ├─ Streaming           │
│ ├─ Billing             │
│ ├─ Recommendations     │
│ ├─ Search              │
│ ├─ User Management     │
│ └─ Analytics           │
└──────────┬──────────────┘
           │
      ┌────▼──────┐
      │ Oracle DB │
      │ (Single)  │
      └───────────┘

Issues:
- Bug in billing → Entire service down
- Scaling search affects all services
- Can't deploy recommendations without affecting streaming
- Database single point of failure
```

**AFTER (Microservices):**

```
┌────────────────────────────────────────────┐
│ API Gateway (OAuth, Rate Limiting)         │
└────────┬───┬──────┬──────┬─────┬───────────┘
         │   │      │      │     │
    ┌────▼┐ ┌▼──┐ ┌─▼──┐ ┌▼──┐ ┌▼──┐
    │Reco │ │Sear│ │Bill│ │User│ │Ana│
    │mmnd │ │ch  │ │ing │ │Mgmt│ │lyt│
    └────┘ └────┘ └────┘ └────┘ └────┘
     │      │      │      │      │
     ▼      ▼      ▼      ▼      ▼
   Cass  Solr   Psql   Psql   Hbase

Benefits:
✓ Recommendation service down → Others work
✓ Search service scales independently
✓ Billing uses PostgreSQL, recommendations uses Cassandra
✓ Easy to update recommendation algorithm
✓ Deploy 100+ times per day safely
```

**Key Learnings:**

`## 10. Case Studies {#case-studies}

### Case Study 1: Netflix - From Monolith to Microservices

**Key Learnings:**

```
1. Infrastructure Must Come First
   ├─ Service discovery (Eureka)
   ├─ Circuit breakers (Hystrix)
   ├─ Monitoring (Atlas)
   ├─ Distributed tracing
   └─ Without these, microservices fail

2. Organizational Structure (Conway's Law)
   ├─ Before: One monolithic team
   ├─ After: Small independent teams
   ├─ Each team owns 1-2 services
   ├─ Each team has own deployment schedule
   └─ "The structure of your org determines your architecture"

3. Gradual Migration (Strangler Pattern)
   ├─ Don't rewrite everything at once
   ├─ Extract one service at a time
   ├─ Monolith still handles most traffic
   ├─ New service takes over gradually
   └─ Eventually monolith is completely replaced

4. Resilience is Critical
   ├─ One service down → Others must survive
   ├─ Timeouts on all external calls
   ├─ Fallback logic for failures
   ├─ Circuit breakers to fail fast
   └─ "Assume every service will fail"

5. Monitoring from Day 1
   ├─ Every service has /metrics endpoint
   ├─ Metrics streamed to centralized system
   ├─ Alerts configured before issues happen
   └─ "You can't fix what you can't see"
```

**Results (Netflix Today):**

```
Deployment: Every 15 minutes (per team)
  - 200+ deployments per day
  - Zero-downtime rolling updates
  - Instant rollback if issues
  
Availability: 99.99%+
  - Multiple data centers
  - Auto-failover
  - Self-healing infrastructure
  
Scalability:
  - Handles 200+ million concurrent users
  - Global CDN for content
  - Per-service scaling
  
Cost Efficiency:
  - Runs on commodity hardware
  - Auto-scaling reduces waste
  - Saved $100M+ annually
```

---

### Case Study 2: Amazon E-Commerce

**Scale:**

```
Peak Traffic: 1 million requests per second (Prime Day)
Normal Traffic: 100,000 requests per second
Global: 150+ countries
Users: 300+ million
```

**Service Architecture:**

```
Core Services:
├─ Product Service
│  ├─ Database: DynamoDB (NoSQL, fast reads)
│  ├─ Cache: CloudFront CDN + ElastiCache
│  └─ Instances: 50+ (scales during sales)
│
├─ Catalog Service
│  ├─ Database: Elasticsearch (full-text search)
│  ├─ Reindex: Real-time as products update
│  └─ Instances: 20+
│
├─ Shopping Cart Service
│  ├─ Database: ElastiCache (Redis)
│  ├─ TTL: 30 days (cart expires)
│  └─ Instances: 100+ (high traffic)
│
├─ Order Service
│  ├─ Database: RDS PostgreSQL (ACID critical)
│  ├─ State Machine: pending→processing→shipped
│  └─ Instances: 30+
│
├─ Payment Service
│  ├─ Database: PostgreSQL (PCI DSS compliance)
│  ├─ Encryption: All payment details encrypted
│  └─ Instances: 20+ (synchronized across regions)
│
├─ Inventory Service
│  ├─ Database: DynamoDB (millions of writes/sec)
│  ├─ Real-time: Updates instantly on purchase
│  └─ Instances: 40+
│
└─ Notification Service
   ├─ Async: SQS queue
   ├─ Sends: Email, SMS, notifications
   └─ Instances: 15+
```

**Black Friday Architecture:**

```
Planning Phase (Weeks before):
├─ Predict traffic surge (typically 10x normal)
├─ Pre-warm caches
├─ Start new EC2 instances
├─ Load test infrastructure
└─ Have rollback plan ready

During Event:
├─ Auto-scaling active
│  ├─ Cart Service: 100 → 500 instances
│  ├─ Order Service: 30 → 100 instances
│  └─ Product Service: 50 → 150 instances
│
├─ Rate limiting per user
│  ├─ Prevent single user from overwhelming
│  └─ Fair distribution
│
├─ Queue management
│  ├─ Orders queued if backend overloaded
│  ├─ Customers see: "High demand, your order #: 12345"
│  └─ Process queue as capacity available
│
├─ Graceful degradation
│  ├─ Recommendations disabled (non-essential)
│  ├─ Analytics paused (non-real-time)
│  └─ Core services prioritized
│
└─ Monitoring 24/7
   ├─ Team on-call watching dashboards
   ├─ Instant alerts if issues
   └─ Rollback capability

Result: 99.99% availability even at 10x load
```

**Key Insights:**

```
1. Database per Service Works
   ├─ Product: DynamoDB (fast, flexible)
   ├─ Order: PostgreSQL (transactions critical)
   ├─ Search: Elasticsearch (full-text)
   └─ Cart: Redis (in-memory, fast)

2. Caching is Essential
   ├─ CloudFront: CDN for product images
   ├─ ElastiCache: Session data, popular items
   ├─ Memcached: Database query results
   └─ 99% of requests hit cache

3. Async Processing
   ├─ Don't wait for notifications
   ├─ Queue them, process later
   ├─ User gets order confirmation in < 1 second
   └─ Notification email arrives in minutes

4. Graceful Degradation
   ├─ During surge: Disable nice-to-have features
   ├─ Recommendations off (but site still works)
   ├─ Core functionality always available
   └─ Better than complete outage

5. Operational Excellence
   ├─ Automated deployments
   ├─ Instant rollback
   ├─ Chaos engineering (test failures)
   └─ "Assume everything will fail"
```

---

### Case Study 3: Uber - Real-Time at Scale

**Problem:**

```
Need to match drivers and riders in < 50ms
Millions of concurrent users
Global scale (60+ countries)
High availability critical (no trips = no revenue)
```

**Microservices:**

```
├─ Request Service
│  ├─ Accept ride requests
│  ├─ Validate pickup/dropoff
│  └─ Create request record
│
├─ Matching Service
│  ├─ Find best available driver
│  ├─ ML algorithm to predict acceptance
│  ├─ 10,000+ calculations/second
│  └─ < 50ms response time (critical!)
│
├─ Trip Service
│  ├─ Track active trips
│  ├─ Real-time location updates
│  ├─ ETA calculation
│  └─ State: accepted→in_progress→completed
│
├─ Location Service
│  ├─ Real-time GPS tracking
│  ├─ Update every 2-3 seconds
│  ├─ Google Maps API integration
│  └─ Billions of location points/day
│
├─ Payment Service
│  ├─ Calculate fare
│  ├─ Process payment
│  ├─ Handle refunds
│  └─ PCI compliance
│
└─ Rating Service
   ├─ Post-trip reviews
   ├─ Driver/Rider ratings
   └─ Fraud detection
```

**Matching Engine Deep Dive:**

```
Scenario: Rider hails in busy downtown area
         50 available drivers within 2km

Algorithm:
1. Get rider location: (lat: 40.7128, lon: -74.0060)
2. Get available drivers: 50 drivers nearby
3. For each driver:
   ├─ Calculate distance (Haversine formula)
   ├─ Estimate ETA
   ├─ Check acceptance probability (ML model)
   ├─ Factor in driver rating
   ├─ Factor in rider rating
   └─ Calculate match score

4. Sort by match score
5. Offer to top 3 drivers (in parallel)
6. First to accept → Trip created

All in < 50ms!

Tech Stack:
├─ Language: C++ (speed critical)
├─ Database: Redis (cache nearby drivers)
├─ Queues: Kafka (event streaming)
├─ Caching: Distributed cache layer
└─ ML: TensorFlow models loaded in memory
```

**Handling Scale:**

```
Peak Hours (Friday night):
  ├─ Requests/second: 50,000+
  ├─ Active trips: 2+ million
  ├─ Location updates: 100,000+ per second
  └─ Matching operations: 10,000+ per second

Infrastructure:
  ├─ Multiple data centers (active-active)
  ├─ Auto-scaling enabled
  ├─ Request Service: 100 instances
  ├─ Matching Service: 200 instances (heavy computing)
  ├─ Payment Service: 50 instances
  └─ Location Service: 300 instances (most traffic)

Result: Sub-second matching even at peak
```

**Resilience Features:**

```
If Matching Service is slow:
  ├─ Request still accepted
  ├─ Matching happens in background
  ├─ User sees: "Finding drivers..."
  ├─ Match sent when ready
  └─ Acceptable if < 5 seconds

If Payment Service down:
  ├─ Trip still completes
  ├─ Payment queued for later
  ├─ Charge happens within hours
  └─ Revenue not lost

If Location Service fails:
  ├─ Use last known location
  ├─ Graceful degradation
  ├─ Recovery in background
  └─ User experience minimally impacted
```

---

## 11. Comparison Matrix: When to Use Microservices

### Start with Monolith If:

```
✓ Team size: < 10 people
✓ Early stage (MVP/proof of concept)
✓ Single product line
✓ Not yet determined scalability needs
✓ Need to ship quickly
✓ Limited DevOps resources

Why:
- Faster to develop initially
- Easier to test (no inter-service calls)
- Simpler deployment
- Less operational overhead
- Easier to debug issues

Examples: 
- Startups (first 6-12 months)
- Internal tools
- Simple CRUD applications
```

### Migrate to Microservices When:

```
⚠ Team growing beyond 20 people
⚠ Different scaling needs per feature
⚠ Want to use different technologies
⚠ Deploy frequency needs are high (daily+)
⚠ Team organization naturally aligns with services
⚠ Can invest in infrastructure

Why:
- Team bottlenecks
- Deployment risk (want independent deploys)
- Scaling one feature affects others
- Technology mismatch (need different stacks)

Red Flags (Do NOT migrate yet):
- No monitoring infrastructure
- No CI/CD pipeline
- Team unfamiliar with distributed systems
- Can't afford DevOps investment
- Still in discovery phase
```

### Microservices Benefits vs Costs

```
BENEFITS:

✓ Independent Scaling
  - Scale only what needs it
  - Save money (no over-provisioning)
  - Better resource utilization

✓ Technology Flexibility
  - Use best tool for each job
  - Python for ML, Java for payments, Node for APIs
  - Upgrade tech per service

✓ Team Autonomy
  - Teams own full lifecycle
  - Deploy on their schedule
  - Faster decision making

✓ Resilience
  - One service down ≠ entire app down
  - Better fault isolation
  - Can implement circuit breakers

✓ Fast Innovation
  - Deploy changes without coordinating
  - Easier to experiment
  - Rollback individual services

COSTS:

✗ Operational Complexity
  - Many more moving parts
  - Monitoring 10+ services instead of 1
  - Need to manage distributed logs
  - Service discovery overhead

✗ Testing Complexity
  - Integration testing harder (cross-service)
  - End-to-end tests slower
  - Need to mock external services

✗ Data Consistency
  - ACID transactions don't span services
  - Eventual consistency is hard to debug
  - Saga pattern adds complexity

✗ Network Latency
  - Every inter-service call adds 5-50ms
  - Can add up (10 services = 50-500ms)
  - Chatty APIs become bottleneck

✗ Infrastructure Investment
  - Need service mesh (Istio)
  - Need observability (Prometheus, Jaeger)
  - Need CI/CD (Jenkins, GitLab CI)
  - Need load balancers, API gateway
  - ~$100K+ per year in tools
```

---

## 12. Decision Framework: Monolith vs Microservices

```
┌─ Question: Team size > 15?
│
├─ NO → Stay monolith (for now)
│
└─ YES → Continue
   │
   ├─ Question: Different scaling needs per feature?
   │
   ├─ NO → Monolith still fine
   │
   └─ YES → Continue
      │
      ├─ Question: Can you afford DevOps investment?
      │
      ├─ NO → Modular monolith (compromise)
      │
      └─ YES → Continue
         │
         ├─ Question: Team infrastructure maturity?
         │
         ├─ IMMATURE → Build infrastructure first
         │
         └─ MATURE → Migrate to microservices

Final Decision Tree:

Small Team + Low complexity → MONOLITH
                           ↓
                    [SHIP FAST]

Medium Team + Growing complexity → MODULAR MONOLITH
                                ↓
                    [BALANCE SPEED & SCALE]

Large Team + Complex product → MICROSERVICES
                            ↓
                    [TEAM AUTONOMY & SCALE]
```

---

## Summary & Key Takeaways

### What You Should Remember

```
1. Monolithic vs Microservices
   ├─ Not binary choice
   ├─ Monolith good for small teams
   ├─ Microservices for large teams
   └─ Can transition gradually

2. Database Strategy
   ├─ One DB per service
   ├─ Polyglot persistence
   ├─ Accept eventual consistency
   └─ Use right tool per service

3. Communication
   ├─ Sync for critical paths (payments)
   ├─ Async for non-critical (notifications)
   ├─ Always assume failures
   └─ Implement circuit breakers

4. Scalability
   ├─ X-axis: Add instances (horizontal)
   ├─ Y-axis: Partition data (split)
   ├─ Z-axis: Increase resources (vertical)
   └─ Usually need X + Y together

5. Infrastructure First
   ├─ Service discovery
   ├─ Monitoring & logging
   ├─ CI/CD pipeline
   ├─ Load balancing
   └─ Before writing first microservice

6. Operational Excellence
   ├─ Automate deployments
   ├─ Implement health checks
   ├─ Graceful shutdown/startup
   ├─ Extensive logging
   └─ Always monitor
```

### Common Mistakes

```
❌ Mistake 1: Microservices from day one
   Problem: Complexity kills velocity
   Solution: Start monolith, extract services

❌ Mistake 2: No monitoring
   Problem: Can't see what's wrong
   Solution: Implement observability first

❌ Mistake 3: Synchronous everywhere
   Problem: Cascading failures
   Solution: Use async for non-critical paths

❌ Mistake 4: Shared database
   Problem: Services tightly coupled
   Solution: Database per service

❌ Mistake 5: Manual deployments
   Problem: Error-prone, slow
   Solution: Full automation with CI/CD

❌ Mistake 6: No API gateway
   Problem: Clients need to know all service URLs
   Solution: Centralize at API gateway

❌ Mistake 7: Ignoring network failures
   Problem: Silent data loss
   Solution: Assume network always fails
```

### Next Steps to Learn

```
1. Hands-On Practice
   ├─ Build a microservices demo
   ├─ Set up Docker + Kubernetes
   ├─ Implement basic service mesh
   └─ Try Prometheus + Grafana

2. Infrastructure
   ├─ Learn Kubernetes
   ├─ Docker deep dive
   ├─ Service mesh (Istio)
   └─ Observability tools

3. Patterns & Practices
   ├─ Circuit breaker pattern
   ├─ Saga for distributed transactions
   ├─ Event sourcing
   └─ CQRS (Command Query Responsibility Segregation)

4. Case Studies
   ├─ Read Netflix tech blog
   ├─ Amazon architecture papers
   ├─ GitHub engineering blog
   └─ Twitter systems blog

5. Tools to Explore
   ├─ Kong API Gateway
   ├─ Consul service discovery
   ├─ Prometheus monitoring
   ├─ Jaeger distributed tracing
   └─ ELK stack for logging
```

---

## References & Resources

**Books:**
- "Building Microservices" by Sam Newman
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "The Phoenix Project" by Gene Kim
- "Site Reliability Engineering" by Google

**Online Resources:**
- https://12factor.net/ (12-Factor App methodology)
- https://microservices.io/ (Chris Richardson's microservices guide)
- https://www.nginx.com/ (Load balancing & reverse proxy)
- https://netflix.tech/ (Netflix tech blog)
- https://aws.amazon.com/microservices/ (AWS microservices guide)

**Tools & Platforms:**
- Kubernetes: Container orchestration
- Docker: Containerization
- Istio: Service mesh
- Prometheus: Metrics
- Grafana: Visualization
- ELK Stack: Logging
- Jaeger: Distributed tracing
- Kong: API Gateway

---

## Conclusion

Microservices architecture is **not a silver bullet**. It solves organizational and scalability problems, but introduces operational complexity.

**The Right Approach:**

```
┌─────────────────────────────────┐
│ Start small                     │
│ ├─ Monolithic architecture     │
│ ├─ Focus on product            │
│ └─ Validate market             │
└──────────────┬──────────────────┘
               │
      ┌────────▼─────────┐
      │ Build           │
      │ infrastructure  │
      │ (monitoring,    │
      │  CI/CD, logging)│
      └────────┬────────┘
               │
      ┌────────▼────────────────────┐
      │ Extract services            │
      │ ├─ Strangler pattern        │
      │ ├─ Gradual migration        │
      │ └─ One service at a time    │
      └────────┬────────────────────┘
               │
      ┌────────▼──────────────────┐
      │ Scale with confidence     │
      │ ├─ Independent scaling    │
      │ ├─ Team autonomy          │
      │ └─ High availability      │
      └──────────────────────────┘
```

**Remember:**
> "Microservices don't solve problems. They create opportunities for problems to be solved in isolation."

Start simple. Add complexity only when needed. Measure, monitor, and adapt.
