# URL Shortener System Architecture

## Overview

A URL shortener service that converts long URLs into short, unique, and shareable links. This document outlines the complete system architecture, design patterns, and implementation details.

---

## System Requirements

### Functional Requirements
- Users can submit a long URL and receive a shortened URL
- Users can redirect from a shortened URL to the original long URL
- Shortened URLs should be unique and non-guessable
- Support custom short codes (optional)
- URL expiration support
- Analytics tracking (click counts, timestamps)

### Non-Functional Requirements
- **High Availability**: 99.9% uptime
- **Low Latency**: Redirect within <100ms
- **Scalability**: Handle millions of shortened URLs
- **Read-Heavy**: ~100:1 read-to-write ratio
- **Data Durability**: All URLs must be persistently stored

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                             │
│                   (Web/Mobile/API Client)                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                      API Gateway                                 │
│           (Load Balancing & Request Routing)                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
┌───────▼─────┐  ┌──────▼────────┐  ┌───▼──────────────┐
│  Write API  │  │  Read API     │  │  Analytics API   │
│  (POST)     │  │  (GET)        │  │  (Async Events)  │
└───────┬─────┘  └──────┬────────┘  └───┬──────────────┘
        │                │               │
        └────────────────┼───────────────┘
                         │
        ┌────────────────▼───────────┐
        │   Cache Layer (Redis)      │
        │  - Short Code → Long URL   │
        │  - TTL Management          │
        └───────────┬────────────────┘
                    │
        ┌───────────▼──────────┐
        │   Application Layer  │
        │  - URL Hashing       │
        │  - Encoding/Decoding │
        │  - Validation        │
        └───────────┬──────────┘
                    │
        ┌───────────▼──────────────┐
        │   Database Layer         │
        │  - Primary DB (SQL)      │
        │  - Replica (Read-Only)   │
        └──────────┬───────────────┘
                   │
        ┌──────────▼─────────────┐
        │  Analytics Datastore   │
        │  (NoSQL/Time Series)   │
        └────────────────────────┘
```

---

## Component Details

### 1. **API Gateway**
- **Responsibility**: Route requests, load balancing, rate limiting
- **Technology**: Nginx/HAProxy or cloud-native (ALB/API Gateway)
- **Features**:
  - SSL/TLS termination
  - Request validation
  - Rate limiting (prevent abuse)
  - CORS handling

### 2. **Write API Service**
- **Endpoint**: `POST /api/shorten`
- **Request**: `{ "original_url": "https://example.com/very/long/path", "custom_code": "abc123" (optional), "expiry_days": 365 }`
- **Response**: `{ "short_url": "https://short.url/abc123", "short_code": "abc123", "created_at": "2026-02-14T10:00:00Z" }`
- **Operations**:
  1. Validate URL format
  2. Check for existing URL (deduplication)
  3. Generate unique short code (if not custom)
  4. Store in database
  5. Cache in Redis
  6. Return short URL

### 3. **Read API Service**
- **Endpoint**: `GET /api/redirect/:short_code`
- **Response**: `{ "original_url": "https://example.com/..." }`
- **Operations**:
  1. Check Redis cache (fast path)
  2. If miss, query database
  3. Update cache
  4. Emit analytics event (asynchronously)
  5. Return redirect

### 4. **Cache Layer (Redis)**
- **Structure**:
  ```
  Key: short_code (e.g., "abc123")
  Value: {
    original_url: "https://example.com/...",
    created_at: timestamp,
    expiry: timestamp,
    hits: count,
    last_accessed: timestamp
  }
  TTL: 24 hours to 30 days (configurable)
  ```
- **Strategy**: Lazy loading + Write-through caching
- **Benefits**: 
  - Sub-millisecond response times
  - Reduces database load

### 5. **Database Schema**

#### URLs Table
```sql
CREATE TABLE urls (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code VARCHAR(20) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);
```

#### Analytics Table
```sql
CREATE TABLE url_analytics (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code VARCHAR(20) NOT NULL,
    user_ip VARCHAR(45),
    user_agent TEXT,
    referrer VARCHAR(2048),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    country VARCHAR(2),
    device_type VARCHAR(20),
    INDEX idx_short_code (short_code),
    INDEX idx_timestamp (timestamp),
    FOREIGN KEY (short_code) REFERENCES urls(short_code)
);
```

#### Users Table (For Authentication)
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Short Code Generation

### Algorithm: Base62 Encoding
- **Character Set**: 0-9, a-z, A-Z (62 characters)
- **Collision Handling**: Increment counter and rehash if collision detected
- **Length**: 6 characters generates ~56.8 trillion unique codes

```
Implementation Steps:
1. Generate unique ID (e.g., using Twitter Snowflake or database auto-increment)
2. Convert ID to Base62 string
3. Check if code already exists
4. If duplicate, increment counter and retry
5. Store mapping in database
```

### Alternative: Random String Generation
```
1. Generate random alphanumeric string (6-8 chars)
2. Check uniqueness in database
3. If duplicate, retry
4. Store mapping
```

---

## API Endpoints

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/shorten` | Create shortened URL | Optional |
| GET | `/api/redirect/:code` | Redirect to original URL | Public |
| GET | `/api/analytics/:code` | Get click statistics | Required |
| DELETE | `/api/shorten/:code` | Delete shortened URL | Required |
| GET | `/api/health` | Health check | Public |

---

## Data Flow Diagrams

### URL Creation Flow
```
Client
  │
  ├─ POST /api/shorten
  │
API Gateway
  │
  ├─ Validate URL format
  ├─ Check if URL already shortened (DB query)
  │
  ├─ If exists: Return existing short code
  │
  ├─ If new:
  │   ├─ Generate unique short code
  │   ├─ Store in Database (primary)
  │   ├─ Store in Redis Cache (with TTL)
  │   └─ Return short code
  │
Response to Client
  └─ { short_url, created_at }
```

### URL Redirect Flow
```
Client (click short link)
  │
  ├─ GET /api/redirect/:code
  │
API Gateway
  │
  ├─ Try Redis Cache (fast path)
  │
  ├─ If Cache Hit:
  │   ├─ Increment hit counter (async)
  │   ├─ Emit analytics event
  │   └─ Return original URL
  │
  ├─ If Cache Miss:
  │   ├─ Query Database
  │   ├─ Check if URL expired
  │   ├─ Cache result in Redis
  │   ├─ Emit analytics event
  │   └─ Return original URL
  │
  ├─ If Not Found:
  │   └─ Return 404 Error
  │
Response to Client
  └─ HTTP 301/302 Redirect to original URL
```

---

## Scalability Considerations

### 1. **Database Sharding**
- Shard by user ID or hash of short code
- Reduces load on primary database
- Enables horizontal scaling

### 2. **Read Replicas**
- Deploy read-only replicas in multiple regions
- Route read queries to nearest replica
- Increases geographic redundancy

### 3. **Caching Strategy**
- Multi-tier caching: Edge Cache (CDN) → Redis → Database
- Cache invalidation: TTL-based + event-driven

### 4. **Message Queue**
- Use Kafka/RabbitMQ for analytics events
- Decouple analytics processing from API response
- Prevents slow analytics from impacting user experience

### 5. **Content Delivery Network (CDN)**
- Cache short URL redirects globally
- Reduce latency for geographically distributed users

### 6. **Rate Limiting**
- Token bucket algorithm
- Limit per IP / per user
- Prevent abuse and DoS attacks

---

## Technology Stack

| Component | Technology Options |
|-----------|-------------------|
| API Server | NodeJS (Express), Python (FastAPI), Go (Gin), Java (Spring Boot) |
| Cache | Redis, Memcached |
| Primary DB | PostgreSQL, MySQL, MySQL |
| Analytics DB | MongoDB, Cassandra, ClickHouse |
| Queue | Kafka, RabbitMQ |
| API Gateway | Nginx, HAProxy, AWS ALB |
| Monitoring | Prometheus, Grafana, ELK Stack |
| Containerization | Docker, Kubernetes |

---

## Performance Metrics

| Metric | Target | Notes |
|--------|--------|-------|
| P99 Latency (Redirect) | <50ms | With cache hits |
| P99 Latency (Shorten) | <200ms | Database write operations |
| Availability | 99.9% | 3 nines SLA |
| Cache Hit Rate | >95% | Hot URLs cached efficiently |
| Throughput | 10K requests/sec per instance | Scales horizontally |

---

## Security Considerations

1. **URL Validation**
   - Whitelist protocols (http, https)
   - Prevent malicious redirects
   - Scan URLs with malware detection service

2. **Authentication & Authorization**
   - OAuth 2.0 / JWT for user APIs
   - API keys for service-to-service
   - RBAC for admin operations

3. **Rate Limiting**
   - Prevent spam and abuse
   - Sliding window algorithm
   - Different limits for authenticated vs anonymous users

4. **Data Protection**
   - HTTPS only (TLS 1.2+)
   - Encrypt sensitive data at rest
   - Hash passwords with bcrypt/Argon2

5. **Logging & Monitoring**
   - Audit logs for all operations
   - Monitor for suspicious patterns
   - Alert on anomalies

---

## Deployment Architecture

### Development Environment
- Single instance with embedded cache
- SQLite or PostgreSQL

### Staging Environment
- 2-3 API instances with load balancer
- Redis instance
- PostgreSQL with replication

### Production Environment
- Auto-scaling group (3-10 instances based on load)
- Multi-region deployment
- Redis cluster with sentinel
- Database with read replicas + read-only followers
- CDN for static assets
- Monitoring and alerting

---

## Monitoring & Alerting

### Key Metrics to Monitor
- API response time (P50, P95, P99)
- Cache hit ratio
- Database query latency
- Error rates (4xx, 5xx)
- Active connections
- Memory usage
- CPU utilization

### Alerts
- API latency > 500ms (P99)
- Cache hit ratio < 80%
- Error rate > 1%
- Replica lag > 30 seconds
- Disk usage > 80%

---

## Future Enhancements

1. **QR Code Generation**: Generate QR codes for shortened URLs
2. **Link Expiration & Renewal**: Automatic renewal notifications
3. **Analytics Dashboard**: Detailed statistics and visitor insights
4. **Custom Domain**: Allow users to use custom domains
5. **Bulk URL Shortening**: API for batch operations
6. **Geolocation Tracking**: Track clicks by location
7. **URL Groups**: Organize URLs by categories
8. **Link Password Protection**: Require password to access link

---

## References

- [System Design Interview - URL Shortener](https://github.com/donnemartin/system-design-primer)
- [Base62 Encoding](https://en.wikipedia.org/wiki/Base62)
- [Building Scalable Systems](https://www.youtube.com/watch?v=jbjUbibn1lg)
