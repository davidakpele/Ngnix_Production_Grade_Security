# NGINX Banking Microservices Platform - Complete Documentation

## Overview
This NGINX configuration serves as a reverse proxy and API gateway for a comprehensive banking microservices platform, providing load balancing, security, rate limiting, caching, and request routing across 12+ backend services.

---

## 1. Core System Configuration

### Worker Process Settings
- **Worker Processes**: `auto` - Automatically matches CPU core count
- **Worker Connections**: 4,096 connections per worker
- **File Descriptors**: 65,535 (worker_rlimit_nofile)
- **Connection Model**: `epoll` - Efficient Linux event notification
- **Multi-Accept**: Enabled - Accepts multiple connections simultaneously

---

## 2. Global HTTP Settings

### Timeout Configuration
- **Client Body Timeout**: 10 seconds
- **Client Header Timeout**: 10 seconds
- **Send Timeout**: 60 seconds
- **Keepalive Timeout**: 65 seconds (100 requests max)
- **Proxy Connect Timeout**: 60 seconds
- **Proxy Read/Send Timeout**: 60 seconds each
- **Reset Timed-out Connections**: Enabled

### Buffer Settings
- **Client Max Body Size**: 50MB
- **Client Body Buffer**: 16KB
- **Client Header Buffer**: 1KB
- **Large Client Header Buffers**: 2 buffers × 4KB
- **Proxy Buffer Size**: 16KB
- **Proxy Buffers**: 4 × 32KB
- **Proxy Busy Buffers**: 64KB

### Performance Optimizations
- **Sendfile**: Enabled
- **TCP No Push**: Enabled
- **TCP No Delay**: Enabled

---

## 3. Security Headers (12 Total)

1. **X-Frame-Options**: `SAMEORIGIN` - Prevents clickjacking
2. **X-Content-Type-Options**: `nosniff` - Blocks MIME-type sniffing
3. **X-XSS-Protection**: `1; mode=block` - Enables XSS filtering
4. **Referrer-Policy**: `strict-origin-when-cross-origin`
5. **Strict-Transport-Security**: 1-year HSTS with subdomains
6. **Content-Security-Policy**: Comprehensive CSP with restricted sources
7. **Permissions-Policy**: Blocks geolocation, microphone, camera
8. **Cross-Origin-Embedder-Policy**: `require-corp`
9. **Cross-Origin-Opener-Policy**: `same-origin`
10. **Cross-Origin-Resource-Policy**: `same-origin`
11. **X-Banking-API-Version**: `2.0` (Custom header)
12. **X-Request-ID**: Unique request identifier

---

## 4. Rate Limiting Strategy (8 Zones)

### Rate Limit Zones
1. **auth_limit**: 10 requests/second - Authentication endpoints
2. **api_limit**: 20 requests/second - General API calls
3. **strict_limit**: 5 requests/second - Sensitive operations
4. **global_limit**: 100 requests/second - Global traffic control
5. **bot_limit**: 2 requests/second - Bot traffic management
6. **txn_limit**: 5 requests/minute - Transaction operations
7. **ddos_protect**: 50 requests/second - DDoS protection
8. **uri_limit**: 10 requests/second - Per-URI limiting

### User-Based Rate Limiting
- **per_user zone**: 100 requests/minute per JWT user ID
- Extracts user ID from JWT Bearer token in Authorization header

### Connection Limiting
- **conn_limit**: Per-IP connection limiting (50 connections max)
- **serv_limit**: Per-server connection limiting

### Dynamic Body Size Limits
- Default: 1MB
- Deposits/Withdrawals: 5MB
- File Uploads: 10MB

---

## 5. Security Detection & Blocking

### Bot Detection
- **Bad Bot Detection**: Blocks scrapers, crawlers, scanners
- **Legitimate Bots**: Allows Google, Bing, DuckDuckGo
- **Suspicious User Agents**: Flags Python, Java, Ruby, PHP clients
- **Browser Fingerprinting**: Validates text/html acceptance

### Attack Pattern Detection
- **SQL Injection**: Detects UNION, SELECT, INSERT, DELETE, DROP patterns
- **XSS Attempts**: Blocks script tags, JavaScript execution
- **Command Injection**: Blocks wget, curl, bash, shell commands
- **Path Traversal**: Blocks ../, etc/passwd, Windows system paths

### JWT Token Validation
- Validates Bearer token format
- Maps token to "valid" or "invalid" status
- Used for access control decisions

### Referer Validation
- Required for deposit and withdrawal POST requests
- Prevents CSRF attacks on financial operations

### Content-Type Validation
- Accepts: application/json, application/x-www-form-urlencoded, multipart/form-data
- Rejects invalid content types

---

## 6. IP Access Control

### Admin Access (geo-based)
- Localhost: 127.0.0.1
- Private Networks: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16

### Maintenance Mode
- Can be toggled via geo configuration
- Bypassed for admin IPs

### Real IP Configuration
- Extracts real IP from X-Forwarded-For header
- Trusts private network ranges
- Recursive IP resolution enabled

---

## 7. Caching Strategy

### Cache Paths
1. **API Cache**: 10MB zone, 60-minute inactive timeout, 1GB max
2. **Static Cache**: 10MB zone, 6-hour inactive timeout, 2GB max
3. **Temp Path**: /var/cache/nginx/tmp

### Cache Key
- `$scheme$request_method$host$request_uri$http_authorization`
- Includes auth header for user-specific caching

### Cache Policies
- **History Service**: 10-minute cache for GET/HEAD requests
- **Static Assets**: 7-day cache with immutable flag
- **Cache Bypass**: Honors Cache-Control headers

---

## 8. Compression (Gzip)

### Settings
- **Enabled**: Yes
- **Compression Level**: 6 (balanced)
- **Minimum Length**: 1024 bytes
- **Vary Header**: Enabled

### Compressed Types
- text/plain, text/css, text/xml, text/javascript
- application/javascript, application/json, application/xml
- application/x-font-ttf, font/opentype, image/svg+xml

### Exclusions
- IE6 browsers (gzip_disable)

---

## 9. Logging Configuration

### Log Formats (4 Types)

1. **security_log** (JSON)
   - Timestamp, IP addresses, request details
   - User agent, referer, timing metrics
   - JWT status, suspicious activity flags
   - Request ID for correlation

2. **json_combined** (JSON)
   - Standard access log in JSON format
   - Request/response metrics
   - Upstream response times

3. **auth_failures**
   - Failed authentication attempts
   - IP, timestamp, user agent

4. **sensitive_log**
   - Financial operations logging
   - Enhanced tracking for compliance

### Log Files
- `/var/log/nginx/access.log` - General access (JSON)
- `/var/log/nginx/security.log` - Security events
- `/var/log/nginx/error.log` - Error events (warn level)
- `/var/log/nginx/auth.log` - Authentication events
- `/var/log/nginx/user.log` - User service events
- `/var/log/nginx/wallet.log` - Wallet operations
- `/var/log/nginx/transactions.log` - Financial transactions
- `/var/log/nginx/bank.log` - Bank operations
- `/var/log/nginx/sensitive.log` - Sensitive operations
- `/var/log/nginx/attacks.log` - Attack attempts
- `/var/log/nginx/security_blocked.log` - Blocked requests

### Conditional Logging
- Success responses (2xx, 3xx) not logged in failure logs
- Attack attempts separately logged

---

## 10. Upstream Services (13 Services)

### Service Configuration
All upstreams use:
- **Load Balancing**: least_conn algorithm
- **Health Checks**: 3 max failures, 30s timeout
- **Keepalive**: 32 connections (1000 requests, 60s timeout)

### Services List
1. **authentication-service**: Port 8187 - Auth & user management
2. **wallet-service**: Port 8035 - Wallet operations & WebSocket
3. **deposit-service**: Port 8020 - Deposit processing
4. **withdraw-service**: Port 8068 - Withdrawal processing
5. **history-service**: Port 8390 - Transaction history
6. **revenue-service**: Port 8085 - Revenue tracking
7. **bank-collection-service**: Port 8040 - Bank integrations
8. **beneficiary-service**: Port 8084 - Beneficiary management
9. **blacklist-service**: Port 8013 - Blacklist management
10. **escrow-service**: Port 8755 - Escrow operations
11. **notification-service**: Port 8079 - Notifications
12. **maintenance-service**: Port 8086 - System maintenance
13. **banking-frontend**: Port 4173 - React/Vue frontend

### Circuit Breaker
- **Next Upstream Events**: error, timeout, invalid_header, 5xx errors
- **Max Tries**: 2 attempts
- **Timeout**: 5 seconds

---

## 11. Main Server Configuration

### Server Block
- **Listen Port**: 8000
- **Server Name**: localhost
- **Global Connection Limit**: 50 per IP

---

## 12. Endpoint Routing & Configuration

### Health & Monitoring (3 Endpoints)

1. **`/health`**
   - Returns: JSON health status
   - Access logging: Disabled
   - No authentication required

2. **`/nginx_status`**
   - Returns: NGINX stub_status metrics
   - Access: Admin IPs only
   - Logging: Disabled

3. **`/container-health`**
   - Returns: Container health check
   - Access logging: Disabled

---

### Static Assets

**`/assets/`**
- Proxy to frontend service
- Cache: 7 days (static_cache)
- Headers: public, max-age=604800, immutable
- X-Cache-Status header included

---

### Frontend SPA

**`/` (Root)**
- **Security Checks**:
  - Maintenance mode check
  - Bot blocking
  - Global rate limiting (100 req/s, burst 200)
- **Proxy Settings**:
  - HTTP/1.1 with Connection keep-alive
  - Buffering disabled for SPA
  - Redirect handling disabled
  - WebSocket upgrade support
- **Error Handling**: Falls back to @frontend_fallback

---

### WebSocket Connection

**`/ws`**
- Routes to wallet-service
- HTTP/1.1 upgrade to WebSocket
- Rate limited: 20 req/s (burst 10)
- Extended timeouts: 3600s read/send
- Connection upgrade headers configured

---

### API Endpoints (12 Services)

#### 1. Authentication Service

**`/api/auth/`**
- **Rate Limit**: 10 req/s (burst 20)
- **Methods**: GET, POST, PUT, DELETE, OPTIONS
- **Features**:
  - Request body validation for POST/PUT
  - Dedicated auth.log logging
  - Custom fallback for service unavailability

**`/api/user/`**
- **Rate Limit**: 10 req/s (burst 20)
- **Methods**: GET, POST, PUT, DELETE, PATCH, OPTIONS
- **Logging**: security_log format

**`/api/receipt/`**
- **Rate Limit**: 10 req/s (burst 20)
- **Methods**: GET, POST, PUT, DELETE, PATCH, OPTIONS
- **Logging**: security_log format

---

#### 2. Wallet Service

**`/api/wallet/`**
- **Rate Limits**: 
  - API: 20 req/s (burst 30)
  - Per-user: 100 req/min (burst 10)
- **Methods**: GET, POST, PUT, PATCH, OPTIONS
- **Security**: SQL injection pattern detection
- **Logging**: Dedicated wallet.log

---

#### 3. Deposit Service

**`/api/deposit/`**
- **Rate Limits**:
  - Strict: 5 req/s (burst 10)
  - Transaction: 5 req/min (burst 3)
  - Per-user: 100 req/min (burst 5)
- **Methods**: POST, GET, OPTIONS only
- **Security**:
  - Referer header required for POST
  - Attack pattern detection
  - SQL/XSS blocking
- **Logging**: transactions.log

---

#### 4. Withdrawal Service

**`/api/withdrawals/`**
- **Rate Limits**: Same as deposit (strict)
- **Methods**: POST, GET, OPTIONS only
- **Security**: Identical to deposit service
- **Logging**: transactions.log

---

#### 5. History Service

**`/api/history/`**
- **Rate Limit**: 20 req/s (burst 20)
- **Caching**:
  - Cache zone: api_cache
  - Valid: 10 minutes (200/302), 1 minute (404)
  - Methods: GET, HEAD
  - Bypass: Cache-Control header
- **Headers**: X-Cache-Status included

---

#### 6. Bank Collection Service

**`/api/bank/`**
- **Rate Limit**: 5 req/s (burst 10)
- **Logging**: Dedicated bank.log

---

#### 7. Beneficiary Service

**`/api/beneficiary/`**
- **Rate Limit**: 5 req/s (burst 10)
- **Methods**: GET, POST, PUT, DELETE, OPTIONS

---

#### 8. Blacklist Service

**`/api/blacklist/`**
- **Rate Limit**: 5 req/s (burst 10)
- **Methods**: GET, POST, PUT, DELETE, OPTIONS

---

#### 9. Notification Service

**`/api/notification/`**
- **Rate Limit**: 20 req/s (burst 20)
- **Methods**: GET, POST, PUT, DELETE, OPTIONS

---

#### 10. Revenue Service

**`/api/revenue/`**
- **Rate Limit**: 5 req/s (burst 10)
- **Methods**: GET, POST, OPTIONS only

---

#### 11. Escrow Service

**`/api/escrow/`**
- **Rate Limits**:
  - Strict: 5 req/s (burst 10)
  - Transaction: 5 req/min (burst 3)
- **Methods**: POST, GET, OPTIONS only
- **Logging**: sensitive_log for compliance

---

#### 12. Maintenance Service

**`/api/maintenance/`**
- **Rate Limit**: 20 req/s (burst 20)
- **Methods**: GET, POST, PUT, DELETE, OPTIONS

**`/api/maintenance/health`**
- Health check endpoint
- No logging
- Direct proxy to service

---

### Monitoring Endpoints

**`/zipkin/`**
- Distributed tracing interface
- Access: Admin IPs only (private networks)
- Proxy to Zipkin service on port 9411

---

## 13. Error Handling

### Custom Error Pages (9 Types)

1. **400 Bad Request**
   - JSON response with error details

2. **401 Unauthorized**
   - Authentication required message

3. **403 Forbidden**
   - Access denied message

4. **404 Not Found**
   - Resource not found message

5. **405 Method Not Allowed**
   - Invalid HTTP method message

6. **429 Too Many Requests**
   - Rate limit exceeded message

7. **500 Internal Server Error**
   - Server error message

8. **502/503/504 Server Errors**
   - Upstream service issues

9. **503 Maintenance**
   - System maintenance message

### Error Response Format
All errors return JSON with:
- `error`: Error description
- `code`: HTTP status code
- Additional context where applicable

---

## 14. Advanced Security Blocks

### File & Directory Protection
- Hidden files (. prefixed): Blocked
- Version control (.git, .svn): Blocked
- Environment files (.env): Blocked
- Config files (composer.json, package.json): Blocked
- Docker files: Blocked
- Backup files (.bak, .old, .tmp): Blocked

### Attack Pattern Blocking (5 Categories)

1. **SQL Injection**
   - Patterns: UNION SELECT, INSERT INTO, DELETE FROM, DROP TABLE
   - Response: 403 with logged attempt

2. **XSS Attempts**
   - Patterns: <script>, javascript:, onerror=, eval()
   - Response: 403 with logged attempt

3. **Path Traversal**
   - Patterns: ../, etc/passwd, windows/system
   - Response: 403 with logged attempt

4. **Command Injection**
   - Patterns: wget, curl, /bin/bash, cmd.exe, powershell
   - Response: 403 with logged attempt

5. **Code Execution**
   - Patterns: base64_decode, phpinfo, shell_exec
   - Response: 403 with logged attempt

All blocked requests logged to `/var/log/nginx/security_blocked.log`

---

## 15. Request Flow & Headers

### Standard Proxy Headers (Set for all requests)
- `Host`: Original host header
- `X-Real-IP`: Client's real IP address
- `X-Forwarded-For`: Proxy chain
- `X-Forwarded-Proto`: Protocol (http/https)
- `X-Forwarded-Host`: Original host
- `X-Forwarded-Port`: Original port
- `X-Request-ID`: Unique request identifier
- `Connection`: "" (keepalive)

### Custom Response Headers
- `X-Request-ID`: Request correlation ID
- `X-API-Version`: API version (1.0 or 2.0)
- `X-Cache-Status`: Cache hit/miss status
- `Retry-After`: Seconds to retry (on errors)
- `X-Blocked-Reason`: Reason for blocking

---

## 16. Performance Features

### Connection Management
- HTTP/1.1 with keepalive
- Connection pooling (32 connections per upstream)
- 1000 requests per keepalive connection
- 60-second keepalive timeout

### Load Balancing
- **Algorithm**: least_conn (least connections)
- **Failover**: Automatic to healthy backends
- **Health Checks**: Passive monitoring

### Request Buffering
- Client request buffering: Disabled
- Proxy buffering: Disabled (streaming)
- Optimal for real-time applications

---

## 17. Compliance & Audit Features

### Transaction Logging
- All financial operations logged separately
- Request correlation via X-Request-ID
- Timestamp, IP, user agent tracking
- JWT user ID extraction for audit trails

### Security Event Tracking
- Failed authentication attempts
- Blocked attack patterns
- Suspicious user agent activity
- Rate limit violations

### Data Retention
- Multiple specialized log files
- JSON format for easy parsing
- Suitable for SIEM integration

---

## 18. Operational Features

### Maintenance Mode
- Geo-based toggle
- Graceful degradation
- Admin bypass capability
- JSON response with retry_after

### Health Monitoring
- NGINX status endpoint
- Service health checks
- Container health verification
- Distributed tracing (Zipkin)

### Request Tracking
- Unique request IDs
- End-to-end correlation
- Cross-service tracing capability

---

## Summary Statistics

- **Total Upstreams**: 13 microservices
- **Total Endpoints**: 20+ API routes
- **Rate Limit Zones**: 8 zones
- **Security Headers**: 12 headers
- **Log Formats**: 4 specialized formats
- **Log Files**: 11 separate logs
- **Error Pages**: 9 custom handlers
- **Security Blocks**: 5 attack categories
- **Cache Zones**: 2 zones (API + Static)
- **Connection Limits**: 4,096 per worker
- **Max File Descriptors**: 65,535

---

This configuration provides enterprise-grade security, performance, and observability for a production banking platform with comprehensive protection against common web attacks, DDoS attempts, and malicious traffic while maintaining high performance through caching, connection pooling, and optimized buffering strategies.
