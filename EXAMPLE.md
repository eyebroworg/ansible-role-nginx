# Configuration Examples

This document provides practical configuration examples for common deployment scenarios.

## Table of Contents

- [Quick Start](#quick-start)
- [Domain Redirects](#domain-redirects)
- [Cloudflare Integration](#cloudflare-integration)
- [Reverse Proxy](#reverse-proxy)
- [Advanced Configuration](#advanced-configuration)
- [Complete Production Example](#complete-production-example)

---

## Quick Start

### Pure HTTP Website

The simplest configuration - a static website served over HTTP:

```yaml
nginx_vhosts:
  - server_name: "example.com"
    root: "/var/www/example.com"
```

### Pure HTTPS Website

A secure website with SSL/TLS. Certificate can be from Let's Encrypt or any CA:

```yaml
nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_certificate: "/etc/letsencrypt/live/example.com/fullchain.pem"
    ssl_certificate_key: "/etc/letsencrypt/live/example.com/privkey.pem"
    root: "/var/www/example.com"
```

### HTTPS with Certificate from Vault

Distribute certificates securely via Ansible Vault:

```yaml
# In your vault file (vault.yml):
# vault_example_com_cert: |
#   -----BEGIN CERTIFICATE-----
#   ...
#   -----END CERTIFICATE-----
# vault_example_com_key: |
#   -----BEGIN PRIVATE KEY-----
#   ...
#   -----END PRIVATE KEY-----

nginx_vault_certificates:
  - domain: "example.com"
    cert: "{{ vault_example_com_cert }}"
    key: "{{ vault_example_com_key }}"

nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"  # Uses cert from nginx_vault_certificates
    root: "/var/www/example.com"
```

---

## Domain Redirects

### WWW to Non-WWW Redirect (HTTPS)

Redirect `www.example.com` to `example.com`. The `server_name_redirect` option handles both HTTP and HTTPS redirects automatically:

```yaml
nginx_vault_certificates:
  - domain: "example.com"
    cert: "{{ vault_example_com_cert }}"
    key: "{{ vault_example_com_key }}"

nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    server_name_redirect: "www.example.com"  # www -> non-www
    root: "/var/www/example.com"
```

**What this generates:**
- HTTP `www.example.com` → 301 → `https://example.com`
- HTTPS `www.example.com` → 301 → `https://example.com`
- Main site served at `https://example.com`

### Non-WWW to WWW Redirect (HTTPS)

Redirect `example.com` to `www.example.com`:

```yaml
nginx_vault_certificates:
  - domain: "www.example.com"
    cert: "{{ vault_www_example_com_cert }}"
    key: "{{ vault_www_example_com_key }}"

nginx_vhosts:
  - server_name: "www.example.com"
    ssl: true
    ssl_domain: "www.example.com"
    server_name_redirect: "example.com"  # non-www -> www
    root: "/var/www/example.com"
```

### HTTP to HTTPS Redirect Only

If you only need to redirect HTTP to HTTPS (no domain change):

```yaml
nginx_vhosts:
  # HTTP redirect
  - server_name: "example.com"
    return: "301 https://example.com$request_uri"

  # HTTPS main site
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    root: "/var/www/example.com"
```

### Multiple Domains to Single Site

Redirect multiple domains to a canonical domain:

```yaml
nginx_vhosts:
  # Main site
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    server_name_redirect: "www.example.com example.net www.example.net"
    root: "/var/www/example.com"
```

---

## Cloudflare Integration

### Basic Cloudflare Setup (Real IP Only)

Restore real visitor IPs when behind Cloudflare:

```yaml
nginx_cloudflare_enabled: true
nginx_cloudflare_real_ip_enabled: true

nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    root: "/var/www/example.com"
```

### Cloudflare with Authenticated Origin Pulls

Maximum security - only accept connections from Cloudflare:

```yaml
nginx_cloudflare_enabled: true
nginx_cloudflare_real_ip_enabled: true
nginx_cloudflare_origin_pull_enabled: true

nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
    root: "/var/www/example.com"
```

### HTTPS + WWW Redirect + Cloudflare (Complete Example)

A common production setup:

```yaml
nginx_cloudflare_enabled: true
nginx_cloudflare_real_ip_enabled: true
nginx_cloudflare_origin_pull_enabled: true

nginx_vault_certificates:
  - domain: "example.com"
    cert: "{{ vault_example_com_cert }}"
    key: "{{ vault_example_com_key }}"

nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
    server_name_redirect: "www.example.com"
    root: "/var/www/example.com"
```

### Mixed Environment (Some Sites Behind Cloudflare)

```yaml
nginx_cloudflare_enabled: true
nginx_cloudflare_origin_pull_enabled: true

nginx_vhosts:
  # Public site - behind Cloudflare
  - server_name: "www.example.com"
    ssl: true
    ssl_domain: "www.example.com"
    ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
    root: "/var/www/public"

  # Admin panel - direct access (NOT behind Cloudflare)
  - server_name: "admin.example.com"
    ssl: true
    ssl_domain: "admin.example.com"
    # No ssl_client_certificate = direct HTTPS access
    root: "/var/www/admin"
```

---

## Reverse Proxy

### Single Backend Application

```yaml
nginx_upstreams:
  - name: app
    servers:
      - "127.0.0.1:3000"

nginx_vhosts:
  - server_name: "app.example.com"
    ssl: true
    ssl_domain: "app.example.com"
    extra_parameters: |
      location / {
          proxy_pass http://app;
          proxy_http_version 1.1;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }
```

### Load Balanced Application

```yaml
nginx_upstreams:
  - name: app_cluster
    strategy: "least_conn"
    keepalive: 32
    servers:
      - "10.0.1.10:8080"
      - "10.0.1.11:8080"
      - "10.0.1.12:8080 backup"

nginx_vhosts:
  - server_name: "app.example.com"
    ssl: true
    ssl_domain: "app.example.com"
    extra_parameters: |
      location / {
          proxy_pass http://app_cluster;
          proxy_http_version 1.1;
          proxy_set_header Connection "";
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }
```

### WebSocket Support

```yaml
nginx_upstreams:
  - name: websocket
    servers:
      - "127.0.0.1:9000"

nginx_vhosts:
  - server_name: "ws.example.com"
    ssl: true
    ssl_domain: "ws.example.com"
    extra_parameters: |
      location / {
          proxy_pass http://websocket;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_set_header Host $host;
          proxy_read_timeout 86400;
      }
```

---

## Advanced Configuration

### Cleanup Existing Configuration (Backup + Clean Deploy)

If you want a clean slate and full state convergence, enable cleanup. This backs up current configs and removes them before deploying. Recommended to run it once via CLI:

```bash
ansible-playbook site.yml -e nginx_cleanup_existing_config=true
```

### Gzip Compression

Gzip is enabled by default. To customize:

```yaml
# Disable gzip
nginx_gzip_enabled: false

# Or customize settings
nginx_gzip_enabled: true
nginx_gzip_comp_level: "6"
nginx_gzip_min_length: "1024"
```

### SSL/TLS Policy

```yaml
# Modern (TLSv1.3 only) - recommended for new deployments
nginx_ssl_policy: "modern"

# Intermediate (TLSv1.2 + TLSv1.3) - for legacy client compatibility
nginx_ssl_policy: "intermediate"
```

### Performance Tuning

```yaml
# Worker processes (default: auto-detected CPU count)
nginx_worker_processes: "auto"
nginx_worker_connections: "4096"
nginx_multi_accept: "on"

# Keepalive settings
nginx_keepalive_timeout: "65"
nginx_keepalive_requests: "1000"

# Linux kernel optimization
nginx_kernel_optimization_enabled: true
```

### Logrotate Settings

```yaml
nginx_logrotate_enabled: true
nginx_logrotate_days: 30  # Keep logs for 30 days
```

### Disable IPv6

```yaml
nginx_listen_ipv6: false
```

---

## Complete Production Example

A comprehensive example combining multiple features for a production deployment:

```yaml
---
# Production nginx configuration

# =============================================================================
# Cleanup (backup + clean deploy)
# =============================================================================
# Recommended to run once via CLI:
# ansible-playbook site.yml -e nginx_cleanup_existing_config=true

# =============================================================================
# Performance
# =============================================================================
nginx_kernel_optimization_enabled: true
nginx_worker_processes: "auto"
nginx_worker_connections: "4096"
nginx_multi_accept: "on"

# =============================================================================
# Security
# =============================================================================
nginx_ssl_policy: "modern"
nginx_hsts_enabled: true
nginx_server_tokens: "off"
nginx_default_server_enabled: true
nginx_default_server_return: "444"

# =============================================================================
# Gzip (enabled by default)
# =============================================================================
nginx_gzip_enabled: true
nginx_gzip_comp_level: "5"

# =============================================================================
# Cloudflare
# =============================================================================
nginx_cloudflare_enabled: true
nginx_cloudflare_real_ip_enabled: true
nginx_cloudflare_origin_pull_enabled: true

# =============================================================================
# Certificates
# =============================================================================
nginx_vault_certificates:
  - domain: "example.com"
    cert: "{{ vault_example_com_cert }}"
    key: "{{ vault_example_com_key }}"

  - domain: "api.example.com"
    cert: "{{ vault_api_cert }}"
    key: "{{ vault_api_key }}"

# =============================================================================
# Upstreams
# =============================================================================
nginx_upstreams:
  - name: web_backend
    strategy: "least_conn"
    keepalive: 32
    servers:
      - "127.0.0.1:3000"
      - "127.0.0.1:3001"

  - name: api_backend
    servers:
      - "127.0.0.1:8080"

# =============================================================================
# Virtual Hosts
# =============================================================================
nginx_vhosts:
  # Main website with www redirect
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
    server_name_redirect: "www.example.com"
    extra_parameters: |
      location / {
          proxy_pass http://web_backend;
          proxy_http_version 1.1;
          proxy_set_header Connection "";
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }

      location /static/ {
          alias /var/www/static/;
          expires 30d;
          add_header Cache-Control "public, immutable";
      }

  # API server
  - server_name: "api.example.com"
    ssl: true
    ssl_domain: "api.example.com"
    ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
    extra_parameters: |
      location / {
          proxy_pass http://api_backend;
          proxy_http_version 1.1;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }

# =============================================================================
# Logrotate
# =============================================================================
nginx_logrotate_enabled: true
nginx_logrotate_days: 30
```

---

## Disabling Features

```yaml
# Disable default server block (catches unmatched requests)
nginx_default_server_enabled: false

# Disable DH parameters generation
nginx_ssl_dhparam_enabled: false

# Disable self-signed default certificate
nginx_ssl_default_cert_enabled: false

# Disable HSTS header
nginx_hsts_enabled: false

# Disable OCSP stapling
nginx_ssl_stapling_enabled: false

# Disable gzip compression
nginx_gzip_enabled: false

# Disable logrotate management
nginx_logrotate_enabled: false

# Disable kernel optimization
nginx_kernel_optimization_enabled: false
```
