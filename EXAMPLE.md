# Configuration Examples

This document provides configuration examples for various deployment scenarios.

## Table of Contents

- [Basic Usage](#basic-usage)
- [SSL/TLS Configuration](#ssltls-configuration)
- [Cloudflare Integration](#cloudflare-integration)
- [Mixed Environment](#mixed-environment)
- [Reverse Proxy](#reverse-proxy)
- [Certificate Distribution via Vault](#certificate-distribution-via-vault)
- [Performance Optimization](#performance-optimization)
- [Complete Production Example](#complete-production-example)

---

## Basic Usage

### Simple HTTP Website

```yaml
nginx_vhosts:
  - server_name: "example.com www.example.com"
    root: "/var/www/example.com"
    index: "index.html index.htm"
```

### Multiple HTTP Websites

```yaml
nginx_vhosts:
  - server_name: "site1.example.com"
    root: "/var/www/site1"

  - server_name: "site2.example.com"
    root: "/var/www/site2"

  - server_name: "site3.example.com"
    root: "/var/www/site3"
```

### HTTP to HTTPS Redirect Only

```yaml
nginx_vhosts:
  - server_name: "example.com www.example.com"
    return: "301 https://example.com$request_uri"
```

---

## SSL/TLS Configuration

### Single HTTPS Website with Custom Certificate Path

```yaml
nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_certificate: "/etc/letsencrypt/live/example.com/fullchain.pem"
    ssl_certificate_key: "/etc/letsencrypt/live/example.com/privkey.pem"
    root: "/var/www/example.com"
```

### HTTPS Website Using Vault-Distributed Certificate

```yaml
# Certificate distribution
nginx_vault_certificates:
  - domain: "example.com"
    cert: "{{ vault_example_com_fullchain }}"
    key: "{{ vault_example_com_key }}"

# Vhost configuration
nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    root: "/var/www/example.com"
```

### Multiple HTTPS Websites

```yaml
nginx_vault_certificates:
  - domain: "site1.example.com"
    cert: "{{ vault_site1_fullchain }}"
    key: "{{ vault_site1_key }}"

  - domain: "site2.example.com"
    cert: "{{ vault_site2_fullchain }}"
    key: "{{ vault_site2_key }}"

nginx_vhosts:
  - server_name: "site1.example.com"
    ssl: true
    ssl_domain: "site1.example.com"
    root: "/var/www/site1"

  - server_name: "site2.example.com"
    ssl: true
    ssl_domain: "site2.example.com"
    root: "/var/www/site2"
```

### Disable HTTP/2

```yaml
nginx_vhosts:
  - server_name: "legacy.example.com"
    ssl: true
    http2: false
    ssl_domain: "legacy.example.com"
    root: "/var/www/legacy"
```

### Use Intermediate SSL Policy (TLSv1.2 + TLSv1.3)

```yaml
# For compatibility with older clients
nginx_ssl_policy: "intermediate"
```

### Disable HSTS

```yaml
nginx_hsts_enabled: false
```

---

## Cloudflare Integration

### All Websites Behind Cloudflare

```yaml
# Enable Cloudflare integration
nginx_cloudflare_enabled: true
nginx_cloudflare_real_ip_enabled: true
nginx_cloudflare_origin_pull_enabled: true

# Certificates
nginx_vault_certificates:
  - domain: "site1.example.com"
    cert: "{{ vault_site1_fullchain }}"
    key: "{{ vault_site1_key }}"

  - domain: "site2.example.com"
    cert: "{{ vault_site2_fullchain }}"
    key: "{{ vault_site2_key }}"

  - domain: "site3.example.com"
    cert: "{{ vault_site3_fullchain }}"
    key: "{{ vault_site3_key }}"

# All vhosts use Cloudflare Authenticated Origin Pulls
nginx_vhosts:
  - server_name: "site1.example.com"
    ssl: true
    ssl_domain: "site1.example.com"
    ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
    root: "/var/www/site1"

  - server_name: "site2.example.com"
    ssl: true
    ssl_domain: "site2.example.com"
    ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
    root: "/var/www/site2"

  - server_name: "site3.example.com"
    ssl: true
    ssl_domain: "site3.example.com"
    ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
    root: "/var/www/site3"
```

### Cloudflare Without Authenticated Origin Pulls

If you only need real IP restoration without client certificate verification:

```yaml
nginx_cloudflare_enabled: true
nginx_cloudflare_real_ip_enabled: true
nginx_cloudflare_origin_pull_enabled: false

nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    root: "/var/www/example.com"
```

### Custom Cloudflare IP Update Schedule

```yaml
nginx_cloudflare_enabled: true
nginx_cloudflare_ip_update_calendar: "weekly"  # Options: daily, weekly, monthly, or systemd calendar syntax
```

---

## Mixed Environment

### Some Sites Behind Cloudflare, Some Direct

```yaml
# Enable Cloudflare for real IP (applies to all sites)
nginx_cloudflare_enabled: true
nginx_cloudflare_real_ip_enabled: true
nginx_cloudflare_origin_pull_enabled: true

nginx_vault_certificates:
  - domain: "public.example.com"
    cert: "{{ vault_public_fullchain }}"
    key: "{{ vault_public_key }}"

  - domain: "admin.example.com"
    cert: "{{ vault_admin_fullchain }}"
    key: "{{ vault_admin_key }}"

  - domain: "api.example.com"
    cert: "{{ vault_api_fullchain }}"
    key: "{{ vault_api_key }}"

nginx_vhosts:
  # Public site - behind Cloudflare with Origin Pulls
  - server_name: "public.example.com"
    ssl: true
    ssl_domain: "public.example.com"
    ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
    root: "/var/www/public"

  # Admin panel - NOT behind Cloudflare (direct access)
  - server_name: "admin.example.com"
    ssl: true
    ssl_domain: "admin.example.com"
    # No ssl_client_certificate - direct HTTPS access
    root: "/var/www/admin"
    extra_parameters: |
      # Restrict to specific IPs
      allow 10.0.0.0/8;
      allow 192.168.1.0/24;
      deny all;

  # API - behind Cloudflare with Origin Pulls
  - server_name: "api.example.com"
    ssl: true
    ssl_domain: "api.example.com"
    ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
    root: "/var/www/api"
```

### HTTP and HTTPS Mixed

```yaml
nginx_vhosts:
  # HTTPS site
  - server_name: "secure.example.com"
    ssl: true
    ssl_domain: "secure.example.com"
    root: "/var/www/secure"

  # HTTP only (internal service)
  - server_name: "internal.example.local"
    listen: "80"
    root: "/var/www/internal"

  # HTTP redirect to HTTPS
  - server_name: "example.com www.example.com"
    listen: "80"
    return: "301 https://example.com$request_uri"

  # HTTPS main site
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    root: "/var/www/example.com"
```

---

## Reverse Proxy

### Single Backend Application

```yaml
nginx_upstreams:
  - name: app_backend
    servers:
      - "127.0.0.1:3000"

nginx_vhosts:
  - server_name: "app.example.com"
    ssl: true
    ssl_domain: "app.example.com"
    extra_parameters: |
      location / {
          proxy_pass http://app_backend;
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
      - "10.0.1.12:8080 weight=2"
      - "10.0.1.13:8080 backup"

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

          # Timeouts
          proxy_connect_timeout 60s;
          proxy_send_timeout 60s;
          proxy_read_timeout 60s;
      }
```

### Multiple Applications with Different Backends

```yaml
nginx_upstreams:
  - name: frontend
    servers:
      - "127.0.0.1:3000"

  - name: api
    strategy: "ip_hash"
    servers:
      - "127.0.0.1:8080"
      - "127.0.0.1:8081"

  - name: websocket
    servers:
      - "127.0.0.1:9000"

nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    extra_parameters: |
      # Frontend
      location / {
          proxy_pass http://frontend;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
      }

      # API
      location /api/ {
          proxy_pass http://api;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
      }

      # WebSocket
      location /ws/ {
          proxy_pass http://websocket;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_set_header Host $host;
      }
```

---

## Certificate Distribution via Vault

### Define Certificates in Vault

In your Ansible Vault file (e.g., `vault.yml`):

```yaml
# Fullchain certificate (server cert + intermediate CA chain)
# This is the recommended format for ssl_certificate
vault_example_com_fullchain: |
  -----BEGIN CERTIFICATE-----
  MIIFazCCBFOgAwIBAgISA... (your server certificate)
  -----END CERTIFICATE-----
  -----BEGIN CERTIFICATE-----
  MIIFFjCCAv6gAwIBAgIRAJ... (intermediate CA certificate)
  -----END CERTIFICATE-----

# Private key
vault_example_com_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvgIBADANBgkqhkiG9w...
  -----END PRIVATE KEY-----
```

### Use in Playbook

```yaml
nginx_vault_certificates:
  - domain: "example.com"
    cert: "{{ vault_example_com_fullchain }}"
    key: "{{ vault_example_com_key }}"

nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"
    root: "/var/www/example.com"
```

---

## Performance Optimization

### Enable Kernel Optimization

```yaml
# Enable Linux kernel optimization for web server workloads
nginx_kernel_optimization_enabled: true
```

### Custom Worker Configuration

```yaml
nginx_worker_processes: "auto"
nginx_worker_connections: "4096"
nginx_multi_accept: "on"
```

### Enable Proxy Cache

```yaml
nginx_proxy_cache_path: "levels=1:2 keys_zone=cache_zone:10m max_size=1g inactive=60m use_temp_path=off"

nginx_vhosts:
  - server_name: "cached.example.com"
    ssl: true
    ssl_domain: "cached.example.com"
    extra_parameters: |
      location / {
          proxy_pass http://backend;
          proxy_cache cache_zone;
          proxy_cache_valid 200 60m;
          proxy_cache_valid 404 1m;
          add_header X-Cache-Status $upstream_cache_status;
      }
```

### Custom Logrotate Settings

```yaml
nginx_logrotate_enabled: true
nginx_logrotate_days: 30  # Keep logs for 30 days
```

---

## Complete Production Example

A comprehensive example combining multiple features:

```yaml
---
# Production nginx configuration

# Kernel optimization (Linux only)
nginx_kernel_optimization_enabled: true

# SSL/TLS settings
nginx_ssl_policy: "modern"  # TLSv1.3 only
nginx_hsts_enabled: true

# Security
nginx_server_tokens: "off"
nginx_default_server_enabled: true
nginx_default_server_return: "444"

# Performance
nginx_worker_processes: "auto"
nginx_worker_connections: "4096"
nginx_keepalive_timeout: "65"
nginx_keepalive_requests: "1000"

# Cloudflare integration
nginx_cloudflare_enabled: true
nginx_cloudflare_real_ip_enabled: true
nginx_cloudflare_origin_pull_enabled: true
nginx_cloudflare_ip_update_calendar: "daily"

# Logrotate
nginx_logrotate_enabled: true
nginx_logrotate_days: 30

# Certificates from vault (use fullchain for cert)
nginx_vault_certificates:
  - domain: "www.example.com"
    cert: "{{ vault_www_fullchain }}"
    key: "{{ vault_www_key }}"

  - domain: "api.example.com"
    cert: "{{ vault_api_fullchain }}"
    key: "{{ vault_api_key }}"

  - domain: "admin.example.com"
    cert: "{{ vault_admin_fullchain }}"
    key: "{{ vault_admin_key }}"

# Upstreams
nginx_upstreams:
  - name: web_backend
    strategy: "least_conn"
    keepalive: 32
    servers:
      - "127.0.0.1:3000"
      - "127.0.0.1:3001"

  - name: api_backend
    strategy: "ip_hash"
    servers:
      - "127.0.0.1:8080"
      - "127.0.0.1:8081"

# Virtual hosts
nginx_vhosts:
  # HTTP to HTTPS redirect
  - server_name: "example.com www.example.com"
    listen: "80"
    return: "301 https://www.example.com$request_uri"
    filename: "example.com-redirect.conf"

  # Main website (behind Cloudflare)
  - server_name: "www.example.com"
    ssl: true
    ssl_domain: "www.example.com"
    ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
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

  # API (behind Cloudflare)
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

          # CORS headers
          add_header Access-Control-Allow-Origin "*" always;
          add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
      }

  # Admin panel (direct access, not behind Cloudflare)
  - server_name: "admin.example.com"
    ssl: true
    ssl_domain: "admin.example.com"
    root: "/var/www/admin"
    extra_parameters: |
      # IP whitelist
      allow 10.0.0.0/8;
      allow 192.168.0.0/16;
      deny all;

      location / {
          try_files $uri $uri/ /index.html;
      }

# Remove default vhost from package installation
nginx_remove_default_vhost: true
```

---

## Disabling Features

### Disable Default Server Block

```yaml
nginx_default_server_enabled: false
```

### Disable DH Parameters

```yaml
nginx_ssl_dhparam_enabled: false
```

### Disable Self-signed Default Certificate

```yaml
nginx_ssl_default_cert_enabled: false
```

### Disable Logrotate Management

```yaml
nginx_logrotate_enabled: false
```

### Disable OCSP Stapling

```yaml
nginx_ssl_stapling_enabled: false
```
