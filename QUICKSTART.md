# Quick Start Guide - Fidentity WAF with Traefik + Coraza

## Prerequisites

- Docker and Docker Compose installed
- Basic knowledge of YAML and WAF concepts

## Setup in 5 Minutes

### Step 1: Configure Your Backend

Edit `docker-compose.yml` and replace the placeholder backend service:

```yaml
backend:
  image: your-backend-image:latest
  # OR build from source:
  # build: ../path-to-backend
  environment:
    - YOUR_ENV_VARS=value
```

### Step 2: Start the Services

```bash
docker compose up -d
```

### Step 3: Verify It's Working

```bash
# Check if services are running
docker compose ps

# Test a simple request
curl -I http://localhost:8080/api/v1/health

# View logs
docker compose logs -f traefik
```

### Step 4: Test the WAF

```bash
# This should work (if your backend has this endpoint)
curl http://localhost:8080/api/v1/app

# Check the Traefik dashboard
# Open http://localhost:8080/dashboard/ in your browser
```

## Common Tasks

### View WAF Logs

```bash
docker compose logs -f traefik | grep -i "coraza\|waf"
```

### Restart Traefik (to reload config)

```bash
docker compose restart traefik
```

### Stop All Services

```bash
docker compose down
```

### Update WAF Rules

1. Edit `config-dynamic.yaml`
2. Traefik will automatically reload (file watcher enabled)
3. Check logs to confirm: `docker compose logs traefik`

## Configuration Files Quick Reference

| File | Purpose |
|------|---------|
| `config-static.yaml` | Traefik settings, plugin config |
| `config-dynamic.yaml` | Routes, WAF rules, middleware |
| `docker-compose.yml` | Service definitions |
| `conf/*.conf` | Reference files (not actively used) |

## Testing Specific Rules

### Test API Endpoint Protection

```bash
# Should pass through (with proper rule exclusions)
curl -X POST http://localhost:8080/api/v1/app/logger \
  -H "Content-Type: application/json" \
  -d '{"message": "test"}'
```

### Test Static Files

```bash
# Should work
curl http://localhost:8080/assets/style.css
curl http://localhost:8080/static/logo.png
```

### Test Dashboard Endpoints

```bash
# Should work with rule exclusions
curl http://localhost:8080/api/v1/dashboard
```

## Troubleshooting

### "Connection Refused" Error

**Cause**: Backend service not running

**Fix**:
```bash
docker compose ps                    # Check status
docker compose logs backend          # Check backend logs
docker compose restart backend       # Restart backend
```

### "502 Bad Gateway" Error

**Cause**: Backend URL mismatch

**Fix**: Check `config-dynamic.yaml` → `services.backend.loadBalancer.servers[0].url` matches your backend

### Rules Not Blocking/Allowing as Expected

**Cause**: Configuration issue

**Fix**:
```bash
# Check for YAML errors
docker compose config

# Increase logging
# Edit config-dynamic.yaml: SecDebugLogLevel 9
docker compose restart traefik
docker compose logs -f traefik
```

## Production Checklist

Before deploying to production:

- [ ] Replace placeholder backend with actual service
- [ ] Set `SecDebugLogLevel` to 0 or 1
- [ ] Configure TLS/HTTPS in `config-static.yaml`
- [ ] Set up proper logging/monitoring
- [ ] Review all rule exclusions for your use case
- [ ] Test with production-like traffic
- [ ] Configure API security in `config-static.yaml`
- [ ] Set up health checks for backend
- [ ] Document custom rules for your team
- [ ] Plan rollback strategy

## Next Steps

1. **Read the full README**: `README.md` for detailed documentation
2. **Review Migration Guide**: `MIGRATION_GUIDE.md` if coming from nginx setup
3. **Customize Rules**: Edit `config-dynamic.yaml` for your specific needs
4. **Monitor**: Set up alerts for blocked requests
5. **Optimize**: Fine-tune rules based on real traffic

## Getting Help

- **Coraza Docs**: https://coraza.io/docs
- **Traefik Docs**: https://doc.traefik.io/traefik/
- **Plugin Repo**: https://github.com/jcchavezs/coraza-http-wasm-traefik
- **Issue Tracker**: Create issues in this repository

## Quick Commands Reference

```bash
# Start services
docker compose up -d

# Stop services
docker compose down

# View logs
docker compose logs -f traefik

# Restart Traefik
docker compose restart traefik

# Check configuration
docker compose config

# View service status
docker compose ps

# Access Traefik dashboard
open http://localhost:8080/dashboard/
```

---

**That's it!** You now have a working WAF protecting your Fidentity application. 🎉
