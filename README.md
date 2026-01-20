# Ansible Headscale + Authelia Stack

[![Ansible](https://img.shields.io/badge/Ansible-2.14+-blue.svg)](https://www.ansible.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Deploy a self-hosted VPN solution using [Headscale](https://github.com/juanfont/headscale) (Tailscale coordination server) with [Authelia](https://www.authelia.com/) for OIDC authentication and 2FA.

## Features

- **Self-hosted Tailscale**: Run your own Tailscale coordination server
- **OIDC Authentication**: Secure login via Authelia with username/password + TOTP
- **Two-Factor Authentication**: Built-in 2FA support via TOTP (Google Authenticator, etc.)
- **Subnet Routing**: Access internal networks through the VPN
- **Flexible Traefik Integration**: Deploy Traefik with the stack OR use an existing instance
- **Automatic HTTPS**: Let's Encrypt certificates via Traefik
- **Docker Compose**: All services containerized for easy management
- **ACL Support**: Fine-grained access control between nodes and networks

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Internet                                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │      Traefik      │
                    │  (Reverse Proxy)  │
                    └────────┬──────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼───────┐    ┌───────▼───────┐    ┌───────▼───────┐
│   Authelia    │◄───│   Headscale   │    │   Tailscale   │
│  (OIDC/2FA)   │    │ (Coordination)│    │   Router      │
└───────────────┘    └───────────────┘    └───────┬───────┘
                                                  │
                                          ┌───────▼───────┐
                                          │   Internal    │
                                          │   Networks    │
                                          └───────────────┘
```

## Requirements

### Control Node
- Ansible 2.14+
- Python 3.8+

### Target Host
- Debian 11/12 or Ubuntu 20.04/22.04/24.04
- Docker Engine with Docker Compose v2
- Python 3.8+
- Root or sudo access
- **Either**: Existing Traefik deployment with Docker provider enabled, **OR** set `deploy_traefik: true` to deploy Traefik with the stack

### Ansible Collections

```bash
ansible-galaxy install -r requirements.yml
```

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/tegge/ansible-headscale-authelia.git
cd ansible-headscale-authelia
```

### 2. Install Dependencies

```bash
ansible-galaxy install -r requirements.yml
```

### 3. Configure Inventory

```bash
cp inventories/example/inventory.yml inventories/production/inventory.yml
# Edit with your server details
```

### 4. Configure Variables

```bash
cp group_vars/all/vars.example.yml group_vars/all/vars.yml
# Edit with your settings (domains, networks, users)
```

### 5. Create Vault for Secrets

```bash
ansible-vault create group_vars/all/vault.yml
```

Add the required secrets (see [Secrets Configuration](#secrets-configuration)).

### 6. Deploy

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

## Configuration

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `headscale_hostname` | Public hostname for Headscale | `headscale.example.com` |
| `authelia_hostname` | Public hostname for Authelia | `auth.example.com` |
| `traefik_network` | Existing Traefik Docker network | `traefik_default` |
| `authelia_users` | List of users with hashed passwords | See example |

### Network Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `tailscale_routes` | `["192.168.1.0/24"]` | Subnets to advertise via VPN |
| `tailscale_router_ip` | `192.168.1.222` | Dedicated IP for router container |
| `tailscale_router_subnet` | `192.168.1.0/24` | Subnet for macvlan network |
| `tailscale_router_gateway` | `192.168.1.1` | Gateway for subnet routing |
| `docker_host_interface` | `eth0` | Physical interface for macvlan |
| `opnsense_ip` | `192.168.1.1` | DNS server for split DNS |

### Traefik Configuration

This stack supports two Traefik modes:

1. **External Traefik** (`deploy_traefik: false`) - Use an existing Traefik instance (default)
2. **Deployed Traefik** (`deploy_traefik: true`) - Deploy Traefik with the stack

#### Common Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `deploy_traefik` | `false` | Deploy Traefik with stack (`true`) or use external (`false`) |
| `traefik_network` | `traefik_default` | Traefik network name |
| `traefik_certresolver` | `letsencrypt` | Certificate resolver name |

#### Deployed Traefik Settings (when `deploy_traefik: true`)

| Variable | Default | Description |
|----------|---------|-------------|
| `traefik_image` | `traefik:v3.2` | Traefik Docker image |
| `traefik_container_name` | `traefik` | Container name |
| `traefik_acme_email` | `admin@example.com` | Email for Let's Encrypt |
| `traefik_acme_staging` | `false` | Use LE staging server |
| `traefik_http_port` | `80` | HTTP port |
| `traefik_https_port` | `443` | HTTPS port |
| `traefik_dashboard_enabled` | `false` | Enable Traefik dashboard |
| `traefik_log_level` | `INFO` | Log level |

### Secrets Configuration

Create `group_vars/all/vault.yml` with:

```yaml
# Generate with: openssl rand -hex 32
vault_jwt_secret: "<32-char-hex>"
vault_session_secret: "<32-char-hex>"
vault_oidc_hmac_secret: "<32-char-hex>"

# Must be exactly 32 characters
vault_storage_encryption_key: "<exactly-32-characters>"

# OIDC client secret
vault_oidc_client_secret: "<random-string>"

# Generate hash with:
# docker run --rm authelia/authelia:latest authelia crypto hash generate pbkdf2 --password "YOUR_SECRET"
vault_oidc_client_secret_hash: "$pbkdf2-sha512$310000$..."

# User password hashes
# Generate with:
# docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password "PASSWORD"
vault_hashed_pass_admin: "$argon2id$v=19$m=65536,t=3,p=4$..."
```

### User Configuration

Define users in `vars.yml`:

```yaml
authelia_users:
  - username: "admin"
    displayname: "Administrator"
    password: "{{ vault_hashed_pass_admin }}"
    email: "admin@example.com"
    groups:
      - "users"
      - "admins"
```

## Post-Deployment

### DNS Configuration

Create A records pointing to your server:
- `headscale.example.com` → `YOUR_SERVER_IP`
- `auth.example.com` → `YOUR_SERVER_IP`

**Important**: If using Cloudflare, set Proxy to OFF (DNS only) for Let's Encrypt validation.

### Port Forwarding (if behind NAT)

| External Port | Internal | Protocol | Purpose |
|---------------|----------|----------|---------|
| 443 | SERVER:443 | TCP | HTTPS |
| 3478 | SERVER:3478 | UDP | DERP/STUN |

### Enroll 2FA

1. Visit `https://auth.example.com`
2. Log in with your username/password
3. Follow prompts to set up TOTP

### Connect VPN Clients

```bash
# Install Tailscale client
curl -fsSL https://tailscale.com/install.sh | sh

# Connect to your Headscale server
tailscale login --login-server=https://headscale.example.com

# Accept advertised routes
tailscale up --accept-routes
```

## Directory Structure

```
.
├── README.md
├── LICENSE
├── ansible.cfg
├── requirements.yml
├── playbooks/
│   └── site.yml                    # Main playbook entry point
├── roles/
│   └── headscale_authelia/
│       ├── defaults/main.yml       # Default variables
│       ├── tasks/
│       │   ├── main.yml            # Task orchestration
│       │   ├── prerequisites.yml   # User/system setup
│       │   ├── traefik_verify.yml  # Traefik checks
│       │   ├── volumes.yml         # Volume management
│       │   ├── authelia_config.yml # Authelia setup
│       │   ├── headscale_config.yml # Headscale setup
│       │   ├── deploy.yml          # Docker deployment
│       │   └── post_deploy.yml     # Final messages
│       ├── templates/              # Jinja2 templates
│       ├── handlers/main.yml       # Service handlers
│       └── meta/main.yml           # Role metadata
├── inventories/
│   └── example/
│       └── inventory.yml           # Example inventory
└── group_vars/
    └── all/
        ├── vars.example.yml        # Configuration template
        └── vault.example.yml       # Secrets template
```

## Troubleshooting

### Traefik Network Not Found

Ensure your Traefik stack is running and the network exists:
```bash
docker network ls | grep traefik
```

### Certificate Issues

1. Verify DNS is correctly configured (not proxied if using Cloudflare)
2. Check Traefik logs: `docker logs traefik`
3. Use HTTP-01 challenge (`httpchallenge`) instead of TLS-ALPN-01

### Headscale Can't Reach Authelia

The containers resolve hostnames via Traefik. Verify:
```bash
docker exec headscale_headscale-authelia wget -q -O- https://auth.example.com/.well-known/openid-configuration
```

### VPN Client Can't Connect

1. Check Headscale logs: `docker logs headscale_headscale-authelia`
2. Verify OIDC configuration matches between Authelia and Headscale
3. Ensure PKCE is enabled on both sides

### Subnet Routes Not Working

1. Verify IP forwarding is enabled: `sysctl net.ipv4.ip_forward`
2. Check router container logs: `docker logs tailscale-router_headscale-authelia`
3. Verify routes are approved: `docker exec headscale_headscale-authelia headscale routes list`

## Security Considerations

- **Vault Encryption**: Always encrypt sensitive variables with `ansible-vault`
- **2FA Required**: All OIDC logins require TOTP by default
- **PKCE Enabled**: Enhanced OAuth security with S256 challenge
- **Network Isolation**: Services communicate via internal Docker network
- **Principle of Least Privilege**: ACLs restrict access between nodes

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Run linting: `ansible-lint playbooks/ roles/`
5. Commit your changes (`git commit -m 'Add amazing feature'`)
6. Push to the branch (`git push origin feature/amazing-feature`)
7. Open a Pull Request

### Development

```bash
# Install dev dependencies
pip install ansible-lint yamllint

# Run linting
ansible-lint playbooks/ roles/
yamllint .
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- [Headscale](https://github.com/juanfont/headscale) - Open source Tailscale control server
- [Authelia](https://www.authelia.com/) - Authentication and authorization server
- [Tailscale](https://tailscale.com/) - Modern VPN built on WireGuard
- [Traefik](https://traefik.io/) - Cloud native reverse proxy

## Related Documentation

- [Headscale Documentation](https://headscale.net/)
- [Authelia Documentation](https://www.authelia.com/docs/)
- [Tailscale Documentation](https://tailscale.com/kb/)
