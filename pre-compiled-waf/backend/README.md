# Fidentity Backend Placeholder

This directory contains a placeholder HTML file for testing purposes.

## Replace with Your Backend

To use your actual Fidentity backend:

1. Update `docker-compose.yml` to use your backend service:
   ```yaml
   backend:
     image: your-backend-image:tag
     # or use build:
     build: ../path-to-your-backend
     environment:
       - DATABASE_URL=...
       - API_KEY=...
     ports:
       - 8000:8080
   ```

2. Update the service URL in `config-dynamic.yaml`:
   ```yaml
   services:
     backend:
       loadBalancer:
         servers:
           - url: http://backend:8080  # Match your backend port
   ```

3. Remove this placeholder directory if not needed.
