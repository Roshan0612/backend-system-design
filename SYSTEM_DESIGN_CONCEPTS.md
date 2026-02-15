# System Design Concepts - Complete Guide with Examples

## Table of Contents
1. [Scalability](#scalability)
2. [Load Balancing](#load-balancing)
3. [Caching](#caching)
4. [Database Design](#database-design)
5. [Message Queues](#message-queues)
6. [Microservices](#microservices)
7. [API Design](#api-design)
8. [Monitoring & Logging](#monitoring--logging)
9. [Security](#security)
10. [Case Studies](#case-studies)

---

## Scalability

### Definition
Scalability is the ability of a system to handle increased load by adding resources and remain responsive.

### Types of Scaling

#### 1. **Horizontal Scaling (Scale Out)**
Adding more servers to distribute load.

**Example: E-commerce Platform**
```
Initial: 1 Server handling 1000 requests/sec → Bottleneck
Solution: Add 5 servers with load balancer
Each server handles: 200 requests/sec ✓
```

**Advantages**:
- No hardware limits
- High availability
- Easier to implement in cloud

**Disadvantages**:
- Data consistency complexity
- Increased network latency

#### 2. **Vertical Scaling (Scale Up)**
Upgrading hardware of existing servers.

**Example: Database Server**
```
Initial: 8GB RAM, 4 CPU Cores → Memory full
Upgrade: 64GB RAM, 32 CPU Cores
Handles 10x more data ✓
```

**Advantages**:
- Simple implementation
- No data consistency issues

**Disadvantages**:
- Hardware limits
- Single point of failure
- Vendor lock-in

### Scalability Metrics

| Metric | Good Range | How to Measure |
|--------|-----------|-----------------|
| Response Time | <100ms (P99) | Monitor API latency |
| Throughput | Linear with resources | Requests/sec |
| Resource Utilization | 60-80% | CPU, Memory, Disk |

---

## Load Balancing

### Definition
Distributes incoming requests across multiple servers to optimize resource utilization.

### Load Balancing Algorithms

#### 1. **Round Robin**
Requests distributed sequentially to each server.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (cycle)
```

**Use Case**: Equal capacity servers  
**Drawback**: Ignores server health/capacity

#### 2. **Least Connections**
Routes to server with fewest active connections.

```
Server A: 10 connections
Server B: 25 connections
Server C: 5 connections ← New request goes here
```

**Use Case**: Long-lived connections (WebSockets)

#### 3. **IP Hash**
Routes based on client IP address hash.

```
client_IP = "192.168.1.100"
hash(IP) % num_servers = server to route
Same client always hits same server = session affinity
```

**Use Case**: Sticky sessions needed  
**Benefit**: Consistent routing

#### 4. **Weighted Round Robin**
Assigns different weights to servers.

```
Server A (4 CPU): weight = 4
Server B (2 CPU): weight = 2
Ratio: A receives 4 requests per 2 for B
```

**Use Case**: Heterogeneous infrastructure

#### 5. **Least Load (CPU/Memory Based)**
Routes to server with lowest resource usage.

```
Monitor real-time metrics:
Server A: CPU 20%, Memory 30% ← Choose
Server B: CPU 70%, Memory 85%
```

### Load Balancer Types

#### Layer 4 (Transport Layer) - L4
- Works with TCP/UDP protocols
- Fast, low CPU overhead
- Example: HAProxy, Nginx (TCP mode)

#### Layer 7 (Application Layer) - L7
- Understands HTTP/HTTPS
- Can route based on URL, hostname, headers
- Example: Nginx, HAProxy (HTTP mode), AWS ALB

**L7 Routing Example**:
```
GET /api/users → API Server Pool
GET /images/* → Image CDN
POST /upload → Upload Server
```

---

## Caching

### Definition
Store frequently accessed data in fast-access storage to reduce latency and database load.

### Caching Strategies

#### 1. **Cache-Aside (Lazy Loading)**
Application checks cache first, fetches from DB if miss.

```
Algorithm:
1. Client requests data
2. Check Cache
   ├─ Hit → Return data (fast)
   └─ Miss → Query DB → Store in Cache → Return data
   
get_user(user_id):
    cache_key = f"user:{user_id}"
    cached_user = cache.get(cache_key)
    
    if cached_user:
        return cached_user
    else:
        user = db.query("SELECT * FROM users WHERE id = ?", user_id)
        cache.set(cache_key, user, ttl=3600)
        return user
```

**Pros**: Simple, easy to implement  
**Cons**: Cache miss penalty, stale data possible

#### 2. **Write-Through**
Data written to cache and DB simultaneously.

```
Algorithm:
1. Write to Cache
2. Write to DB (in parallel)
3. Return success after both complete

set_user(user_id, data):
    cache.set(f"user:{user_id}", data)
    db.update("UPDATE users SET ... WHERE id = ?", user_id)
    return success
```

**Pros**: Consistent data, no stale data  
**Cons**: Slower writes, redundant operations

#### 3. **Write-Behind (Write-Back)**
Data written to cache immediately, DB updated asynchronously.

```
Algorithm:
1. Write to Cache
2. Return success immediately
3. Async process writes to DB later

set_user(user_id, data):
    cache.set(f"user:{user_id}", data)
    queue.enqueue(write_to_db, user_id, data)
    return success
```

**Pros**: Fast writes  
**Cons**: Data loss risk if cache fails before DB write

#### 4. **Refresh-Ahead**
Proactively refresh cache before expiration.

```
Algorithm:
1. Detect cache key about to expire
2. Refresh from DB in background
3. Update TTL

monitor_cache():
    for key in cache.keys():
        ttl = cache.ttl(key)
        if ttl < REFRESH_THRESHOLD:
            async_refresh(key)
```

**Use Case**: Critical data, expensive queries

### Cache Invalidation

#### Time-Based (TTL)
```
cache.set("user:123", user_data, ttl=3600)
After 1 hour, cache automatically expires
```

#### Event-Based
```
def update_user(user_id, new_data):
    db.update(user_id, new_data)
    cache.delete(f"user:{user_id}")  # Invalidate immediately
```

#### Pattern-Based
```
def bulk_update_users(department_id):
    db.update_where(department=department_id)
    cache.delete_pattern(f"user:dept:{department_id}:*")
```

### Caching Layers

```
┌─────────────────┐
│  Client Layer   │ Browser Cache
│  (1-30 mins)    │
└────────┬────────┘
         │
┌────────▼────────────┐
│  CDN Cache          │ Cloudflare, CloudFront
│  (1-24 hours)       │
└────────┬────────────┘
         │
┌────────▼────────────┐
│  Application Cache  │ Redis, Memcached
│  (minutes-hours)    │ In-memory
└────────┬────────────┘
         │
┌────────▼────────────┐
│  Database Cache     │ Query cache, Buffer pool
│  (seconds-minutes)  │
└─────────────────────┘
```

### Example: Social Media Feed Cache

```python
def get_user_feed(user_id):
    cache_key = f"feed:{user_id}"
    
    # Check cache
    cached_feed = redis.get(cache_key)
    if cached_feed:
        return json.loads(cached_feed)
    
    # Cache miss - fetch from DB
    feed = db.query("""
        SELECT posts.* FROM posts
        JOIN followers ON posts.author_id = followers.following_id
        WHERE followers.follower_id = ?
        ORDER BY posts.created_at DESC
        LIMIT 50
    """, user_id)
    
    # Store in cache with 1 hour TTL
    redis.setex(cache_key, 3600, json.dumps(feed))
    return feed

def post_created(post_id, author_id):
    # Invalidate cache for all followers
    followers = db.query("SELECT follower_id FROM followers WHERE following_id = ?", author_id)
    for follower_id in followers:
        redis.delete(f"feed:{follower_id}")
```

---

## Database Design

### Relational vs Non-Relational

#### Relational Database (SQL)
```sql
-- ACID compliance
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255) UNIQUE,
    created_at TIMESTAMP
);

-- Structured schema
-- Strong consistency
-- ACID properties
-- Joins possible
```

**Use Cases**: Financial systems, e-commerce, relational data  
**Examples**: PostgreSQL, MySQL, Oracle

#### Non-Relational Database (NoSQL)
```javascript
// Document-based (MongoDB)
{
    _id: ObjectId("..."),
    name: "John",
    email: "john@example.com",
    metadata: {
        age: 25,
        location: "NYC"
    },
    tags: ["vip", "active"]
}

// Flexible schema
// Eventual consistency
// Horizontal scalability
```

**Use Cases**: Content storage, real-time analytics, IoT  
**Examples**: MongoDB, Cassandra, DynamoDB, Firebase

### Database Indexing

#### Index Types

##### 1. **Primary Key Index**
Uniquely identifies row.

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(255)
);
-- Automatically indexed, O(1) lookup
```

##### 2. **Secondary Index**
Speed up searches on non-primary columns.

```sql
CREATE INDEX idx_email ON users(email);
SELECT * FROM users WHERE email = 'john@example.com';
-- With index: O(log n)
-- Without index: O(n)
```

##### 3. **Compound Index**
Index on multiple columns.

```sql
CREATE INDEX idx_user_date ON orders(user_id, created_at);

-- Optimal for queries like:
SELECT * FROM orders WHERE user_id = 5 AND created_at > '2026-01-01';
```

##### 4. **Full-Text Index**
Search text efficiently.

```sql
CREATE FULLTEXT INDEX idx_content ON articles(title, body);
SELECT * FROM articles WHERE MATCH(title, body) AGAINST('database' IN BOOLEAN MODE);
```

### Sharding

Distribute data across multiple databases by a shard key.

```
User ID Range Sharding:

Shard 1: User IDs 1-1000000      → Database 1
Shard 2: User IDs 1000001-2000000 → Database 2
Shard 3: User IDs 2000001-3000000 → Database 3

get_user_shard(user_id):
    shard_id = user_id // 1000000
    db = databases[shard_id]
    return db.query(f"SELECT * FROM users WHERE id = {user_id}")
```

**Types of Sharding**:
- **Range-based**: ID ranges
- **Hash-based**: hash(key) % num_shards
- **Directory-based**: Lookup table for shard location
- **Geographic**: Data locality

### Replication

Maintain copies of data across multiple servers.

```
Master-Slave Replication:

Write Operations → Master Database ─────→ Apply logs
                                          ↓
                                    Slave Database ✓ Read-Only

Benefits:
- Read scalability (distribute reads to slaves)
- High availability (failover to slave)
- Backup protection
```

---

## Message Queues

### Definition
Asynchronous communication pattern where producers send messages to consumers.

### Key Benefits
1. **Decoupling**: Producer doesn't wait for consumer
2. **Scalability**: Handle traffic spikes
3. **Reliability**: Persistent storage
4. **Resilience**: Retry logic

### Message Queue Patterns

#### 1. **Point-to-Point**
Single producer → Single consumer

```
Producer: Order Service
Message: {"order_id": 123, "items": [...]}
↓
Queue
↓
Consumer: Payment Service (processes one message)
```

#### 2. **Publish-Subscribe**
Single producer → Multiple consumers

```
Producer: User Service (user created event)
          ↓
        Event: user.created
          ↓
    ┌─────┴─────┬─────────┐
    ↓           ↓         ↓
Email Service Email sent
Notification Service → Push notification
Analytics Service → Track event
```

### Example: E-commerce Order Processing

```python
# Producer: Order API
def create_order(order_data):
    # Save order to database
    order = db.create_order(order_data)
    
    # Publish event to message queue
    message_queue.publish('orders.created', {
        'order_id': order.id,
        'user_id': order.user_id,
        'items': order.items,
        'total': order.total,
        'timestamp': datetime.now().isoformat()
    })
    
    return {'order_id': order.id, 'status': 'pending'}

# Consumer 1: Payment Service
def process_payment():
    while True:
        message = message_queue.consume('orders.created')
        result = payment_gateway.charge(message['user_id'], message['total'])
        message_queue.publish('payment.processed', {
            'order_id': message['order_id'],
            'status': 'success' if result else 'failed'
        })

# Consumer 2: Notification Service
def send_notifications():
    while True:
        message = message_queue.consume('orders.created')
        email_service.send(message['user_id'], f"Order {message['order_id']} received")

# Consumer 3: Analytics Service
def track_analytics():
    while True:
        message = message_queue.consume('orders.created')
        analytics.track_event('order_created', {
            'amount': message['total'],
            'items_count': len(message['items'])
        })
```

### Popular Message Queues

| Queue | Throughput | Latency | Durability | Use Case |
|-------|-----------|---------|-----------|----------|
| Kafka | Very High (1M+/sec) | Medium | Highly Durable | Event Streaming |
| RabbitMQ | Medium (50K/sec) | Low | Configurable | Traditional MQ |
| Redis Queue | High (100K/sec) | Very Low | Non-persistent | Job Queue |
| AWS SQS | High (Elastic) | Medium | Durable | Cloud Services |

---

## Microservices

### Definition
Architectural style breaking application into small, independent services that communicate via APIs.

### Microservices Architecture

```
┌────────────────────────────────────────────────┐
│              API Gateway / Load Balancer       │
└──────┬──────────┬──────────┬──────────┬────────┘
       │          │          │          │
    ┌──▼──┐   ┌──▼──┐   ┌──▼──┐   ┌──▼──┐
    │User │   │Order│   │Payment  │Notification
    │Svc  │   │Svc  │   │Svc      │Svc
    └─────┘   └─────┘   └─────┘   └─────┘
       │          │          │          │
    ┌──▼──┐   ┌──▼──┐   ┌──▼──┐   ┌──▼──┐
    │User │   │Order│   │Payment  │Notification
    │DB   │   │DB   │   │DB       │DB
    └─────┘   └─────┘   └─────┘   └─────┘
    
    Connected via: REST API, gRPC, Message Queues
```

### Service Communication

#### Synchronous (RPC)
Direct request-response.

```python
# Order Service calling Payment Service
response = requests.post(
    'http://payment-service/api/charge',
    json={'amount': 100, 'user_id': 5}
)
if response.status_code == 200:
    print("Payment successful")
```

**Pros**: Simple, immediate feedback  
**Cons**: Tight coupling, cascading failures

#### Asynchronous (Event-Driven)
Using message queues.

```python
# Order Service publishes event
message_queue.publish('payment.charge', {
    'order_id': 123,
    'amount': 100,
    'user_id': 5
})

# Payment Service subscribes independently
def process_payment_charge(event):
    result = charge_user(event['user_id'], event['amount'])
    message_queue.publish('payment.charged', result)
```

**Pros**: Loose coupling, resilience  
**Cons**: Eventual consistency, debugging complexity

### Microservices Challenges

| Challenge | Solution |
|-----------|----------|
| Network latency | Caching, async communication |
| Data consistency | Compensating transactions, event sourcing |
| Deployment complexity | Containerization (Docker), Orchestration (Kubernetes) |
| Monitoring difficulty | Distributed tracing (Jaeger, Zipkin) |
| Service discovery | Service registry (Consul, Eureka) |

---

## API Design

### REST API Principles

#### 1. **Resource-Based URLs**
Use nouns, not verbs.

```
✓ Good:
GET    /api/users              → List users
GET    /api/users/123          → Get user 123
POST   /api/users              → Create user
PUT    /api/users/123          → Update user 123
DELETE /api/users/123          → Delete user 123

✗ Bad:
GET /api/getUsers
GET /api/getUserById?id=123
POST /api/createUser
```

#### 2. **HTTP Status Codes**

| Code | Meaning | Example |
|------|---------|---------|
| 200 | OK | Request successful |
| 201 | Created | Resource created |
| 204 | No Content | Successful, no response body |
| 400 | Bad Request | Invalid parameters |
| 401 | Unauthorized | Authentication required |
| 403 | Forbidden | Access denied |
| 404 | Not Found | Resource doesn't exist |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Server Error | Internal error |

#### 3. **Request/Response Format**

```json
// Request
POST /api/users
Content-Type: application/json

{
    "name": "John Doe",
    "email": "john@example.com",
    "age": 30
}

// Response (201 Created)
{
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com",
    "age": 30,
    "created_at": "2026-02-15T10:30:00Z"
}
```

#### 4. **Pagination**

```
GET /api/posts?page=2&limit=20&sort=created_at:desc

Response:
{
    "data": [...],
    "pagination": {
        "page": 2,
        "limit": 20,
        "total": 500,
        "has_next": true,
        "has_prev": true
    }
}
```

#### 5. **Versioning**

```
// URL-based versioning
GET /api/v1/users
GET /api/v2/users

// Header-based versioning
GET /api/users
Accept: application/vnd.myapi.v2+json

// Query parameter versioning
GET /api/users?version=2
```

### gRPC API

Binary protocol, more efficient than REST.

```protobuf
// user_service.proto
syntax = "proto3";

service UserService {
    rpc GetUser (GetUserRequest) returns (User);
    rpc ListUsers (Empty) returns (stream User);
    rpc CreateUser (User) returns (User);
}

message GetUserRequest {
    int32 user_id = 1;
}

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
}
```

**REST vs gRPC**:

| Aspect | REST | gRPC |
|--------|------|------|
| Protocol | HTTP/1.1 | HTTP/2 |
| Format | JSON | Protocol Buffers |
| Performance | Good | Excellent |
| Browser Support | Yes | Limited |
| Debugging | Easy | Harder |
| Use Case | Public APIs | Internal Services |

---

## Monitoring & Logging

### Types of Monitoring

#### 1. **Application Metrics**
Count, gauge, histogram observations.

```python
from prometheus_client import Counter, Gauge, Histogram

# Counter: Cumulative count
request_count = Counter('requests_total', 'Total requests', ['method', 'endpoint'])
request_count.labels(method='GET', endpoint='/users').inc()

# Gauge: Current value
active_connections = Gauge('active_connections', 'Active connections')
active_connections.set(150)

# Histogram: Distribution
request_duration = Histogram('request_duration_seconds', 'Request duration')
with request_duration.time():
    process_request()
```

#### 2. **Infrastructure Metrics**
CPU, Memory, Disk, Network.

```
Tools: Prometheus, Grafana, CloudWatch

Alerts:
- CPU > 80% for 5 minutes
- Memory > 90%
- Disk free < 10%
- Network latency > 100ms
```

#### 3. **Business Metrics**
Revenue, conversion rate, user growth.

```
Daily Revenue: $50,000
Conversion Rate: 3.5%
Active Users: 100,000
Customer Churn: 2%
```

### Logging Strategy

#### Log Levels

```python
import logging

logging.debug("Detailed diagnostic information")
logging.info("General informational messages")
logging.warning("Warning messages (unusual situations)")
logging.error("Error messages (failures)")
logging.critical("Critical errors (system failures)")
```

#### Structured Logging

```json
{
    "timestamp": "2026-02-15T10:30:00Z",
    "level": "ERROR",
    "service": "order-service",
    "trace_id": "abc123def456",
    "user_id": 789,
    "message": "Payment processing failed",
    "error": "Connection timeout",
    "duration_ms": 5000,
    "retry_count": 3
}
```

#### Log Aggregation

```
┌──────────────┐
│ Service Logs │
└──────┬───────┘
       │
    ┌──▼──────────┐
    │ Log Shipper  │ (Filebeat, Logstash)
    └──┬──────────┘
       │
    ┌──▼────────────────┐
    │ Central Log Store  │ (Elasticsearch)
    └──┬────────────────┘
       │
    ┌──▼──────────┐
    │ Visualization│ (Kibana, Grafana)
    └─────────────┘
```

### Distributed Tracing

Track request flow across services.

```
User Request → API Gateway
    ↓ trace_id: abc123
    └─→ User Service ──→ User DB
    └─→ Order Service ──→ Order DB
    └─→ Payment Service ──→ Payment Gateway
    
Each step records:
- Service name
- Operation duration
- Success/failure
- Any errors
```

---

## Security

### Authentication Methods

#### 1. **API Keys**
Simple, suitable for service-to-service.

```
GET /api/data
Authorization: ApiKey sk_live_51234...

Server validates key against store.
```

#### 2. **OAuth 2.0**
Delegated authorization.

```
User → "Login with Google"
    → Google Authorization Server
    → Gets authorization code
    → Exchanges for access token
    → Accesses user data
```

#### 3. **JWT (JSON Web Token)**
Self-contained token.

```
Header.Payload.Signature

Payload contains:
{
    "sub": "user_id_123",
    "email": "user@example.com",
    "iat": 1626000000,
    "exp": 1626003600
}
```

### Authorization

#### Role-Based Access Control (RBAC)

```python
# Define roles and permissions
ROLES = {
    'admin': ['read', 'write', 'delete'],
    'editor': ['read', 'write'],
    'viewer': ['read']
}

# Check authorization
def require_permission(permission):
    def decorator(func):
        def wrapper(request, *args, **kwargs):
            user_role = request.user.role
            if permission not in ROLES[user_role]:
                raise ForbiddenError()
            return func(request, *args, **kwargs)
        return wrapper
    return decorator

@require_permission('delete')
def delete_user(user_id):
    # Only admins can delete
    pass
```

#### Attribute-Based Access Control (ABAC)

```
Grant access based on:
- Resource attributes (data classification: public, private)
- User attributes (department, level)
- Environment (time, location, device)

Example:
Allow: user from Finance dept OR
       user is Admin AND
       accessing file from trusted IP AND
       during business hours
```

### Data Protection

#### Encryption at Rest
```python
from cryptography.fernet import Fernet

key = Fernet.generate_key()
cipher = Fernet(key)

# Encrypt sensitive data before storing
encrypted = cipher.encrypt(b"sensitive_data")
db.store(encrypted)

# Decrypt when needed
decrypted = cipher.decrypt(encrypted)
```

#### Encryption in Transit
```
Always use HTTPS/TLS
- Secure connection
- Certificate validation
- Prevents man-in-middle attacks
```

### Input Validation

```python
def validate_and_sanitize(user_input):
    # Check type
    if not isinstance(user_input, str):
        raise ValueError("Must be string")
    
    # Check length
    if len(user_input) > 255:
        raise ValueError("Too long")
    
    # Prevent SQL injection
    escaped = db.escape(user_input)
    
    # Prevent XSS
    sanitized = html.escape(escaped)
    
    return sanitized
```

---

## Case Studies

### Case Study 1: Twitter/X - High-Frequency Social Network

**Scale**: 500M daily active users, 2M tweets/minute

**Key Challenges**:
- Real-time data distribution
- Handling traffic spikes
- Timeline feed generation

**Architecture Components**:
1. **Geographically distributed data centers**
2. **Cache layers**: Redis for timelines, hot tweets
3. **Message queue**: Kafka for event streaming
4. **Search**: Elasticsearch for tweets
5. **Graph DB**: Neo4j for follow relationships

**Key Insights**:
- Heavy caching for frequently accessed data
- Asynchronous processing for non-critical tasks
- Replication across regions for availability

### Case Study 2: Uber - Real-Time Matching System

**Scale**: 100M+ rides/month, Sub-second matching

**Key Challenges**:
- Real-time driver-rider matching
- Location tracking at scale
- Surge pricing calculations

**Architecture Components**:
1. **Real-time location service**: Redis geospatial indexes
2. **Message queue**: Kafka for ride events
3. **Search spatial DB**: Geohashing for proximity
4. **Consistent hashing**: Route users to correct shard
5. **Microservices**: Separate matching, payment, routing services

**Key Insights**:
- Geospatial indexing for efficient location queries
- Event-driven architecture for responsiveness
- Data sharding by geographic region

### Case Study 3: Netflix - Content Delivery at Scale

**Scale**: 230M+ subscribers, 1B+ requests/day

**Key Challenges**:
- Massive content delivery
- Regional availability
- Personalized recommendations

**Architecture Components**:
1. **CDN**: Massive CDN investment for video delivery
2. **Caching**: Multi-tier caching
3. **Database**: Cassandra for massive scale
4. **Real-time streaming**: Kafka for analytics
5. **Recommendation engine**: ML models on cached data

**Key Insights**:
- Own CDN infrastructure for control
- Eventual consistency acceptable for streaming
- Batch processing during off-peak hours

---

## Best Practices Summary

| Category | Best Practice |
|----------|---------------|
| **Scalability** | Plan for 10x growth, test with load testing |
| **Caching** | Cache strategically, monitor invalidation |
| **Database** | Index appropriately, shard when needed |
| **API** | Version early, document thoroughly |
| **Monitoring** | Monitor everything, alert on anomalies |
| **Security** | Encrypt data, validate inputs, authenticate users |
| **Communication** | Async when possible, sync only when necessary |
| **Failure** | Design for failure, implement circuit breakers |

---

## References & Further Reading

- [Designing Data-Intensive Applications](https://dataintensive.net/)
- [System Design Primer - GitHub](https://github.com/donnemartin/system-design-primer)
- [AWS Architecture Patterns](https://aws.amazon.com/architecture/patterns/)
- [Google Site Reliability Engineering](https://sre.google/books/)
- [Distributed Systems Paper Reading](https://dsrg.pdos.csail.mit.edu/papers/)

---

**Last Updated**: February 15, 2026
