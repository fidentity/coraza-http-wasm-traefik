# Coraza HTTP WASM Traefik Plugin - Fidentity WAF

This repository contains the Coraza WAF configuration for Fidentity, migrated from the nginx-based fidentity-waf setup to use Traefik with the Coraza HTTP WASM middleware.

## Migration Overview

This setup replaces the previous nginx + ModSecurity configuration with:
- **Traefik v3.2.0** as the reverse proxy
- **Coraza HTTP WASM** middleware for WAF functionality
- All fidentity-specific ModSecurity rules converted to Coraza directives

The wasm executable is built in the [coraza-http-wasm](https://github.com/jcchavezs/coraza-http-wasm/tree/main) repository.

## Architecture

```
Client Request → Traefik (Port 8080) → Coraza WAF Middleware → Backend Service
```

## Configuration Files

- `config-static.yaml` - Traefik static configuration (entrypoints, plugins)
- `config-dynamic.yaml` - Traefik dynamic configuration (routes, middlewares, WAF rules)
- `conf/modsecurity-fity-before.conf` - Fidentity-specific rule exclusions (reference)
- `conf/modsecurity-fity-after.conf` - Additional configuration overrides (reference)
- `conf/crs/crs-fidentity.conf` - CRS configuration for Fidentity (reference)

## Getting Started

### Prerequisites

- Docker and Docker Compose
- Your backend application (replace the placeholder nginx service in docker-compose.yml)

### Running the WAF

1. **Configure your backend service** in `docker-compose.yml`:
   ```yaml
   backend:
     image: your-backend-image:tag
     # Add your backend configuration
   ```

2. **Start the services**:
   ```bash
   docker compose up traefik
   ```

3. **Test the WAF**:
   - Health check: `curl -I 'http://localhost:8080/api/v1/health'`
   - Protected endpoints: `curl -I 'http://localhost:8080/api/v1/app'`

### Configuration

The WAF rules are defined in `config-dynamic.yaml` under the `middlewares.waf.plugin.coraza.directives` section. Key features:

- **SecRuleEngine On** - WAF is active
- **Audit logging** to stdout in JSON format
- **Request body limit** set to 52428800 bytes (50MB)
- **CRS configuration** for allowed content types and restricted extensions
- **Fidentity-specific rule exclusions** for all API endpoints

#### Customizing Rules

To add or modify rules, edit `config-dynamic.yaml`:

```yaml
middlewares:
  waf:
    plugin:
      coraza:
        directives:
          - SecRuleEngine On
          - SecRule REQUEST_URI "@streq /custom-path" "id:60001,phase:1,log,deny,status:403"
```

#### Debug Logging

To increase logging verbosity, change the debug level in `config-dynamic.yaml`:

```yaml
- SecDebugLogLevel 9  # Maximum verbosity (1-9)
```

## Migrated Features

All features from the fidentity-waf nginx setup have been migrated:

✅ All 52+ custom rule exclusions for Fidentity API endpoints
✅ Content-Type based exclusions
✅ Cookie and header-based rule tuning
✅ Request body access controls
✅ CRS configuration for allowed content types
✅ File extension restrictions
✅ JSON audit logging

## Differences from nginx Setup

| Feature | nginx + ModSecurity | Traefik + Coraza |
|---------|-------------------|------------------|
| Reverse Proxy | nginx | Traefik |
| WAF Engine | ModSecurity v3 | Coraza (ModSecurity compatible) |
| Configuration | Multiple .conf files | YAML directives |
| Rule Format | Native ModSecurity | ModSecurity syntax in YAML |
| TypeScript Wrapper | Required | Not needed (direct YAML config) |

## File Reference

The `conf/` directory contains the original ModSecurity configuration files for reference. These are not directly used by Coraza but are preserved for:
- Understanding the rule logic
- Future maintenance
- Comparison with the YAML configuration

## Troubleshooting

### View WAF Logs

```bash
docker compose logs -f traefik
```

### Test Specific Rules

```bash
# Test blocked request
curl -I 'http://localhost:8080/admin'

# Test allowed request
curl -I 'http://localhost:8080/api/v1/health'
```

### Common Issues

1. **Rules not loading**: Check YAML syntax in `config-dynamic.yaml`
2. **False positives**: Review and add rule exclusions in the directives section
3. **Performance issues**: Reduce `SecDebugLogLevel` in production

## Production Deployment

For production:

1. **Reduce logging**:
   ```yaml
   - SecDebugLogLevel 0
   ```

2. **Enable API security**:
   ```yaml
   api:
     insecure: false
     # Configure TLS and authentication
   ```

3. **Configure proper backend URLs** in `config-dynamic.yaml`:
   ```yaml
   services:
     backend:
       loadBalancer:
         servers:
           - url: http://your-production-backend:port
   ```

4. **Set up health checks** for your backend service

5. **Monitor logs** for attacks and false positives

## Additional Resources

- [Coraza Documentation](https://coraza.io/docs)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [coraza-http-wasm-traefik Plugin](https://github.com/jcchavezs/coraza-http-wasm-traefik)
- [OWASP Core Rule Set](https://coreruleset.org/)

## Support

For issues related to:
- **Fidentity-specific rules**: Review the conf/ directory and config-dynamic.yaml
- **Coraza plugin**: See [coraza-http-wasm-traefik](https://github.com/jcchavezs/coraza-http-wasm-traefik)
- **Traefik configuration**: See [Traefik docs](https://doc.traefik.io/traefik/)

