# Fidentity WAF Migration Guide

## From nginx + ModSecurity to Traefik + Coraza

This document describes the migration process from the fidentity-waf (nginx-based) to the coraza-http-wasm-traefik setup.

## What Was Migrated

### 1. Configuration Files

| Source (fidentity-waf) | Destination (coraza-traefik) | Status |
|------------------------|------------------------------|--------|
| `conf/modsecurity-fity-before.conf` | `conf/modsecurity-fity-before.conf` + `config-dynamic.yaml` | ✅ Migrated |
| `conf/modsecurity-fity-after.conf` | `conf/modsecurity-fity-after.conf` + `config-dynamic.yaml` | ✅ Migrated |
| `conf/crs/crs-fidentity.conf` | `conf/crs/crs-fidentity.conf` + `config-dynamic.yaml` | ✅ Migrated |
| `conf/setup.conf` | Not needed (included in Coraza) | ℹ️ Reference |
| `nginx/local/nginx.conf` | `config-static.yaml` + `config-dynamic.yaml` | ✅ Replaced |
| TypeScript files (`src/*.ts`) | Not migrated | ⏸️ Manual process |

### 2. Docker Setup

**Old (fidentity-waf)**:
- nginx with ModSecurity v3
- Custom Dockerfile with multi-stage build
- Node.js for TS → conf generation

**New (coraza-traefik)**:
- Traefik v3.2.0
- Coraza WASM plugin
- Direct YAML configuration (no build step needed)

### 3. Rule Configuration

All 52 custom Fidentity rule exclusions have been migrated:
- Cookie-based exclusions (Optanon, etc.)
- Content-Type based rules
- API endpoint-specific exclusions for:
  - `/api/v1/app/*` endpoints
  - `/api/v1/dashboard/*` endpoints
  - Authentication endpoints
  - Asset and static file endpoints
  - Special processing endpoints (qessignature, webauthn, etc.)

## What Was NOT Migrated

### TypeScript Configuration Files

The following files were **not migrated** as they are part of your custom rule generation workflow:

- `src/generate-modsecurity.ts`
- `src/models.ts`
- `src/regexer.ts`
- `src/sorter.ts`
- `src/utils.ts`
- `src/validation.ts`
- `conf/modsecurity-fity-before-conf.ts`
- `conf/experimental_conf.ts`

**Reason**: These files are TypeScript wrappers for generating ModSecurity configurations. With Coraza in Traefik, you can edit rules directly in YAML format without a build step.

### If You Need to Continue Using TypeScript:

1. **Option A**: Keep using the fidentity-waf project for rule generation, then manually copy rules to `config-dynamic.yaml`

2. **Option B**: Create a new TypeScript tool that outputs YAML instead of .conf files

3. **Option C**: Edit rules directly in `config-dynamic.yaml` (recommended for simplicity)

## Key Architecture Changes

### Request Flow

**Before (nginx)**:
```
Client → nginx → ModSecurity → Backend
```

**After (Traefik)**:
```
Client → Traefik → Coraza WASM Middleware → Backend
```

### Configuration Approach

**Before**:
1. Write rules in TypeScript
2. Run `npm run generate-modsecurity`
3. Generate .conf files
4. Build Docker image with configs
5. Deploy

**After**:
1. Edit `config-dynamic.yaml`
2. Traefik automatically reloads
3. No build step required

### Rule Syntax

Both use ModSecurity-compatible syntax:

**Before (.conf file)**:
```apache
SecRule REQUEST_URI "@beginsWith /api/v1/app" \
    "id:20008,phase:1,nolog,pass,\
    ctl:ruleRemoveById=920300,\
    ctl:ruleRemoveById=942100,\
    ctl:ruleRemoveById=942440"
```

**After (YAML)**:
```yaml
- |
  SecRule REQUEST_URI "@beginsWith /api/v1/app" "id:20008,phase:1,nolog,pass,ctl:ruleRemoveById=920300,ctl:ruleRemoveById=942100,ctl:ruleRemoveById=942440"
```

## Deployment Steps

### 1. Update Backend Configuration

Edit `docker-compose.yml` to point to your actual backend:

```yaml
backend:
  image: your-fidentity-backend:latest
  environment:
    - ENVIRONMENT=production
  # Add your backend configuration
```

### 2. Configure Environment Variables

If needed, add environment variables to `docker-compose.yml`:

```yaml
traefik:
  environment:
    - BACKEND_URL=http://backend:8080
    - ENVIRONMENT=production
```

### 3. Adjust Routes

Update `config-dynamic.yaml` if your backend uses different paths:

```yaml
http:
  routers:
    backend:
      rule: PathPrefix(`/`)  # or Host(`example.com`) && PathPrefix(`/api`)
```

### 4. Test Before Production

```bash
# Start services
docker compose up -d

# Test health endpoint
curl -I http://localhost:8080/api/v1/health

# Test protected endpoints
curl -I http://localhost:8080/api/v1/app

# Check logs
docker compose logs -f traefik
```

### 5. Monitor False Positives

After deployment, monitor logs for blocked requests that should be allowed:

```bash
docker compose logs traefik | grep -i deny
```

If you find false positives, add rule exclusions in `config-dynamic.yaml`.

## Advantages of New Setup

1. **Simpler deployment**: No build step required
2. **Faster updates**: Edit YAML and Traefik auto-reloads
3. **Better observability**: Traefik dashboard at http://localhost:8080/dashboard/
4. **Modern stack**: Traefik v3 + WASM-based WAF
5. **Lower complexity**: No TypeScript tooling needed

## Potential Issues & Solutions

### Issue 1: Rules Not Loading

**Symptom**: WAF not blocking expected requests

**Solution**: 
- Check YAML syntax in `config-dynamic.yaml`
- View logs: `docker compose logs traefik`
- Verify SecRuleEngine is "On"

### Issue 2: Backend Connection Errors

**Symptom**: 502 Bad Gateway or connection refused

**Solution**:
- Verify backend service is running: `docker compose ps`
- Check backend URL in `config-dynamic.yaml`
- Ensure backend service name matches in docker-compose.yml

### Issue 3: False Positives

**Symptom**: Legitimate requests being blocked

**Solution**:
- Review logs to identify rule ID
- Add exclusion in `config-dynamic.yaml`:
  ```yaml
  - |
    SecRule REQUEST_URI "@beginsWith /your-path" "id:60001,phase:1,nolog,pass,ctl:ruleRemoveById=RULE_ID"
  ```

### Issue 4: Performance Concerns

**Symptom**: Slow response times

**Solution**:
- Reduce `SecDebugLogLevel` to 0 or 1
- Disable `SecResponseBodyAccess` if not needed
- Consider adjusting `SecRequestBodyLimit`

## Rollback Plan

If you need to rollback to the nginx setup:

1. Stop Traefik: `docker compose down`
2. Switch to fidentity-waf directory
3. Start nginx: `docker compose up -d`

Both setups can coexist on different ports if needed.

## Next Steps

After successful migration:

1. ✅ Update your CI/CD pipelines to deploy the new setup
2. ✅ Update documentation for your team
3. ✅ Set up monitoring and alerting
4. ✅ Configure log aggregation (if not already done)
5. ✅ Fine-tune rules based on production traffic
6. ✅ Consider implementing the CRS rule set fully

## Getting Help

- **Coraza**: https://coraza.io/docs
- **Traefik**: https://doc.traefik.io/traefik/
- **Plugin**: https://github.com/jcchavezs/coraza-http-wasm-traefik

## Conclusion

This migration simplifies your WAF setup while maintaining all the security features from the nginx-based configuration. The new Traefik + Coraza setup is more maintainable and follows modern cloud-native practices.
