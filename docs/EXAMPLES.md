# Configuration Examples

This document provides configuration examples for various use cases with the eyebrowkang.nginx role.

## Table of Contents

- [Basic Usage](#basic-usage)
- [SSL/TLS Configuration](#ssltls-configuration)
- [Cloudflare Integration](#cloudflare-integration)
- [Mixed HTTP/HTTPS Environment](#mixed-httphttps-environment)
- [Reverse Proxy](#reverse-proxy)
- [Vault-based Certificate Distribution](#vault-based-certificate-distribution)
- [Performance Optimization](#performance-optimization)

## Differences from geerlingguy.nginx

This role is forked from [geerlingguy.nginx](https://github.com/geerlingguy/ansible-role-nginx) with the following enhancements:

| Feature | geerlingguy.nginx | eyebrowkang.nginx |
|---------|-------------------|-------------------|
| SSL/TLS security | Manual configuration | Built-in modern/intermediate policies |
| DH parameters | Not included | Auto-generated (2048-bit) |
| Default server block | Not included | Catches unmatched requests (returns 444) |
| HSTS | Manual | Built-in with configurable toggle |
| OCSP Stapling | Manual | Built-in with configurable toggle |
| Cloudflare integration | Not included | Real IP + Authenticated Origin Pulls |
| HTTP/2 | Manual listen directive | Auto-enabled with SSL, version-aware |
| Kernel optimization | Not included | Optional sysctl tuning |
| Logrotate | Not included | Built-in configuration |
| Certificate management | Manual | Vault-based distribution |
| Configuration cleanup | Not included | Backup + clean deploy option |

---

## Basic Usage

Minimal configuration with sensible defaults:

```yaml
- hosts: webservers
  roles:
    - role: eyebrowkang.nginx
      vars:
        nginx_vhosts:
          - server_name: "example.com"
            root: "/var/www/example.com"
```

## SSL/TLS Configuration

### Modern Policy (TLSv1.3 Only)

Recommended for maximum security when all clients support TLSv1.3:

```yaml
- hosts: webservers
  roles:
    - role: eyebrowkang.nginx
      vars:
        nginx_ssl_policy: "modern"  # Default, TLSv1.3 only

        nginx_vault_certificates:
          - domain: "example.com"
            cert: "{{ vault_example_com_fullchain }}"
            key: "{{ vault_example_com_key }}"

        nginx_vhosts:
          - server_name: "example.com"
            root: "/var/www/example.com"
            ssl: true
            ssl_domain: "example.com"
            server_name_redirect: "www.example.com"
```

### Intermediate Policy (TLSv1.2 + TLSv1.3)

For compatibility with older clients:

```yaml
- hosts: webservers
  roles:
    - role: eyebrowkang.nginx
      vars:
        nginx_ssl_policy: "intermediate"
        nginx_ssl_dhparam_enabled: true  # Required for TLSv1.2

        nginx_vhosts:
          - server_name: "example.com"
            ssl: true
            ssl_certificate: "/etc/letsencrypt/live/example.com/fullchain.pem"
            ssl_certificate_key: "/etc/letsencrypt/live/example.com/privkey.pem"
```

### Disabling HSTS

If you need to disable HSTS (not recommended):

```yaml
nginx_hsts_enabled: false
```

## Cloudflare Integration

### Real IP Only

Restore real visitor IPs when behind Cloudflare:

```yaml
- hosts: webservers
  roles:
    - role: eyebrowkang.nginx
      vars:
        nginx_cloudflare_enabled: true
        nginx_cloudflare_real_ip_enabled: true
        nginx_cloudflare_real_ip_header: "CF-Connecting-IP"
```

### Full Cloudflare Setup with Authenticated Origin Pulls

Maximum security with client certificate verification:

```yaml
- hosts: webservers
  roles:
    - role: eyebrowkang.nginx
      vars:
        nginx_cloudflare_enabled: true
        nginx_cloudflare_real_ip_enabled: true
        nginx_cloudflare_origin_pull_enabled: true

        nginx_vhosts:
          - server_name: "example.com"
            ssl: true
            ssl_domain: "example.com"
            ssl_client_certificate: "{{ nginx_cloudflare_origin_pull_cert_path }}"
```

### Cleaning Up Cloudflare Integration

To completely remove Cloudflare integration files:

```bash
ansible-playbook playbook.yml -e nginx_cloudflare_cleanup=true
```

## Mixed HTTP/HTTPS Environment

Serve some sites over HTTP, others over HTTPS:

```yaml
nginx_vhosts:
  # HTTPS site with automatic www redirect
  - server_name: "secure.example.com"
    root: "/var/www/secure"
    ssl: true
    ssl_domain: "secure.example.com"
    server_name_redirect: "www.secure.example.com"

  # HTTP-only site (internal use)
  - server_name: "internal.example.com"
    root: "/var/www/internal"
    listen: "80"
```

## Reverse Proxy

Load balancing with upstream servers:

```yaml
- hosts: webservers
  roles:
    - role: eyebrowkang.nginx
      vars:
        nginx_upstreams:
          - name: backend
            strategy: "least_conn"
            keepalive: 32
            servers:
              - "10.0.0.10:8080"
              - "10.0.0.11:8080"
              - "10.0.0.12:8080 backup"

        nginx_vhosts:
          - server_name: "api.example.com"
            ssl: true
            ssl_domain: "api.example.com"
            extra_parameters: |
              location / {
                  proxy_pass http://backend;
                  proxy_http_version 1.1;
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                  proxy_set_header X-Forwarded-Proto $scheme;
                  proxy_set_header Connection "";
              }
```

## Vault-based Certificate Distribution

Store certificates in Ansible Vault for secure deployment:

1. Create encrypted vault file:

```bash
ansible-vault create group_vars/webservers/vault.yml
```

2. Add certificate content:

```yaml
# group_vars/webservers/vault.yml (encrypted)
vault_example_com_fullchain: |
  -----BEGIN CERTIFICATE-----
  MIIFazCCBFOgAwIBAgISA...
  -----END CERTIFICATE-----
  -----BEGIN CERTIFICATE-----
  MIIFFjCCAv6gAwIBAgIRA...
  -----END CERTIFICATE-----

vault_example_com_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvgIBADANBgkqhki...
  -----END PRIVATE KEY-----
```

3. Reference in playbook:

```yaml
nginx_vault_certificates:
  - domain: "example.com"
    cert: "{{ vault_example_com_fullchain }}"
    key: "{{ vault_example_com_key }}"

nginx_vhosts:
  - server_name: "example.com"
    ssl: true
    ssl_domain: "example.com"  # References the vault certificate
```

## Performance Optimization

### High-traffic Static Site

Optimized for serving static files:

```yaml
- hosts: webservers
  roles:
    - role: eyebrowkang.nginx
      vars:
        # Worker settings
        nginx_worker_connections: "4096"
        nginx_multi_accept: "on"
        nginx_worker_rlimit_nofile: "8192"

        # Enable open file cache
        nginx_open_file_cache_enabled: true
        nginx_open_file_cache_max: "10000"
        nginx_open_file_cache_inactive: "60s"

        # Timeouts
        nginx_keepalive_timeout: "30"
        nginx_keepalive_requests: "1000"

        # Gzip (already enabled by default)
        nginx_gzip_enabled: true
        nginx_gzip_comp_level: "5"
```

### Linux Kernel Optimization

Enable kernel tuning for high-performance scenarios:

```yaml
nginx_kernel_optimization_enabled: true
```

This configures sysctl parameters for:
- Increased connection backlog
- TCP Fast Open
- Optimized buffer sizes
- Reduced TIME_WAIT duration

### Clean Deployment

Reset nginx configuration to a clean state (creates backup first):

```bash
ansible-playbook playbook.yml -e nginx_cleanup_existing_config=true
```

---

## Variable Quick Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `nginx_ssl_policy` | `"modern"` | `modern` (TLSv1.3) or `intermediate` (TLSv1.2+1.3) |
| `nginx_hsts_enabled` | `true` | Enable HSTS header |
| `nginx_ssl_stapling_enabled` | `true` | Enable OCSP stapling |
| `nginx_default_server_enabled` | `true` | Enable default server block |
| `nginx_cloudflare_enabled` | `false` | Enable Cloudflare integration |
| `nginx_kernel_optimization_enabled` | `false` | Enable sysctl tuning |
| `nginx_open_file_cache_enabled` | `false` | Enable file descriptor caching |
| `nginx_cleanup_existing_config` | `false` | Clean deploy mode |
| `nginx_cloudflare_cleanup` | `false` | Remove all Cloudflare files |

For a complete list of variables, see `defaults/main.yml`.
