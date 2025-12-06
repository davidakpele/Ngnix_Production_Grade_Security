# Security Implementation Quick Start

## Step 1: Generate SSL Certificates

### Development (Self-Signed - 1 day setup)
```bash
# Create certificate directory
mkdir -p /etc/nginx/certs

# Generate self-signed certificate valid for 365 days
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout /etc/nginx/certs/server.key \
  -out /etc/nginx/certs/server.crt \
  -subj "/C=US/ST=State/L=City/O=Bank/OU=IT/CN=localhost"

# Create chain.crt (same as .crt for self-signed)
cp /etc/nginx/certs/server.crt /etc/nginx/certs/chain.crt

# Set permissions
chmod 600 /etc/nginx/certs/server.key
chmod 644 /etc/nginx/certs/server.crt
```

### Production (Let's Encrypt - Free)
```bash
# Install certbot
apt-get install certbot python3-certbot-nginx

# Generate certificate (requires domain)
certbot certonly --standalone -d your-domain.com -d www.your-domain.com

# Auto-renewal
certbot renew --dry-run  # Test
certbot renew             # Actual renewal

# Certificates stored in: /etc/letsencrypt/live/your-domain.com/
# Update nginx.conf paths accordingly
```

---

## Step 2: Update Docker Compose

### Add Certificate Volumes
```yaml
services:
  nginx:
    volumes:
      - ./certs:/etc/nginx/certs:ro
      - ./logs:/var/log/nginx
    ports:
      - "80:80"
      - "443:443"
```

### Create certs directory
```bash
cd project-001
mkdir -p certs logs deployment/nginx
```

---

## Step 3: Validate Configuration

```bash
# Check syntax
docker-compose exec nginx nginx -t

# Reload nginx
docker-compose exec nginx nginx -s reload

# Monitor logs
tail -f logs/security.log
tail -f logs/attacks.log
```

---

## Step 4: Test Security

### Check HTTPS
```bash
# Test SSL
curl -I https://localhost:8000

# Show headers
curl -I --insecure https://localhost:8000
```

### Test Rate Limiting
```bash
# Quick burst test
for i in {1..30}; do 
  curl https://localhost:8000/api/auth/login & 
done
```

### Test Attack Blocking
```bash
# SQL Injection (should block)
curl "https://localhost:8000/api/wallet/?id=1' UNION SELECT"

# XSS (should block)
curl "https://localhost:8000/api/wallet/?data=<script>alert(1)</script>"

# Path Traversal (should block)
curl "https://localhost:8000/api/wallet/?file=../../../etc/passwd"
```

---

## Step 5: Log Monitoring

### Real-time Security Events
```bash
# Watch security log
tail -f logs/security.log | jq .

# Count blocked attacks
grep "403" logs/security_blocked.log | wc -l

# List suspicious IPs
grep "SUSPICIOUS_AGENT" logs/suspicious.log | jq '.client_ip' | sort | uniq -c
```

### Set up Log Rotation
Create `/etc/logrotate.d/nginx-security`:
```
/var/log/nginx/*.log {
    daily
    rotate 30
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    prerotate {
        if [ -d /etc/logrotate.d/httpd-prerotate.d ]; then \
            run-parts /etc/logrotate.d/httpd-prerotate.d; \
        fi
    }
    postrotate {
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    }
}
```

---

## Step 6: Alerts & Monitoring

### Alert Rules (example for ELK/Splunk)

#### High Attack Rate
```
status: 403 AND (request_uri: /api/* OR request_uri: /admin/*)
| stats count by client_ip
| where count > 50 in 5 minutes
```

#### Suspicious User Agents
```
is_suspicious: 1
| stats count by http_user_agent
| where count > 10
```

#### Multiple Failed Auths
```
request_uri: /api/auth/* AND status: 401
| stats count by client_ip
| where count > 5 in 1 minute
```

#### CSP Violations
```
source: csp-violations.log
| stats count by violated_directive
```

---

## Step 7: Continuous Monitoring

### OWASP ZAP Integration
```bash
# Docker image for OWASP ZAP
docker pull owasp/zap2docker-stable

# Scan your API
docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t https://your-domain.com/api/wallet/ \
  -r report.html
```

### Scheduled Security Scanning
```bash
# Add to crontab for daily scans
0 2 * * * /usr/bin/owasp-zap-baseline.sh > /var/log/zap-scan.log 2>&1
```

---

## Step 8: Performance Optimization

### Monitor SSL Performance
```bash
# Check TLS handshake time
time curl -w "Time: %{time_total}s\n" -o /dev/null -s https://localhost:8000

# Expected: <200ms for local, <1s for remote
```

### Cache Configuration (if needed)
```nginx
# Add to static assets location
proxy_cache_valid 200 7d;
proxy_cache_lock on;
proxy_cache_key "$scheme$request_method$host$request_uri";
```

---

## Step 9: Compliance Verification

### HTTPS Verification
- [ ] HTTPS on all endpoints
- [ ] HTTP redirects to HTTPS
- [ ] TLS 1.2+ only
- [ ] Strong ciphers configured

### Header Verification
- [ ] HSTS header present
- [ ] CSP header set correctly
- [ ] X-Frame-Options: DENY
- [ ] X-Content-Type-Options: nosniff
- [ ] Permissions-Policy configured

### Rate Limiting Verification
- [ ] Auth endpoints limited
- [ ] Transaction endpoints limited
- [ ] Per-user limits working

### Logging Verification
- [ ] Security log created
- [ ] Attack patterns logged
- [ ] JSON format correct
- [ ] Log rotation configured

---

## Step 10: Incident Response

### If Under Attack
```bash
# Immediate actions
1. Check attack log
   tail -f logs/attacks.log

2. Identify source IP
   grep "403" logs/security_blocked.log | jq '.client_ip' | sort | uniq -c

3. Add to blocklist
   # Add IP to nginx geo block
   # Restart nginx

4. Check service logs
   docker-compose logs auth-service
   docker-compose logs wallet-service

5. Notify security team
   # Send alert with details
```

### Rate Limit Tuning
```bash
# If legitimate users blocked, adjust limits
# Edit nginx.conf rate limit zones
# Example: increase from 10r/s to 20r/s

# Reload config
docker-compose exec nginx nginx -s reload
```

---

## Summary of Changes

| Area | Before | After |
|------|--------|-------|
| HTTPS | None | ✅ TLS 1.2+ Required |
| CSP | Permissive | ✅ Strict, no unsafe-* |
| CSRF | Missing | ✅ Referer validation |
| Rate Limiting | Basic | ✅ Enhanced per endpoint |
| Attack Detection | Limited | ✅ SQL/XSS/Command Injection |
| Logging | Basic | ✅ JSON Security logs |
| Security Headers | Partial | ✅ Complete set |
| HSTS | 1 year | ✅ 2 years + preload |
| User-Agent Check | Basic | ✅ Suspicious detection |

---

**Status**: Ready for deployment ✅
**Next**: Generate certificates and test in dev environment
