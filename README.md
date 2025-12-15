# Portainer Apps

Self-hosted applications deployed on TrueNAS storage and managed via Portainer. This repository contains Docker Compose configurations for various services running in my private infrastructure.

## Overview

This repository contains production-ready Docker Compose configurations with hardened security settings for self-hosted services. Each service is isolated in its own directory with dedicated configuration files.

**Deployment Platform**: TrueNAS Storage
**Management**: Portainer
**Reverse Proxy**: nginx-proxy-manager
**Automated Updates**: Renovate Bot

## Services

### Infrastructure & Management

- **[uptime-kuma](uptime-kuma/)** - Service monitoring and status dashboard
- **[zoraxy](zoraxy/)** - Alternative reverse proxy solution

### Productivity & Collaboration

- **[nextcloud](nextcloud/)** - Personal cloud storage and collaboration platform with PostgreSQL, Valkey (Redis), Imaginary, notify_push, and Whiteboard integration
- **[collabora](collabora/)** - Online office suite integration for Nextcloud
- **[n8n](n8n/)** - Workflow automation platform with PostgreSQL backend
- **[paperless-ngx](paperless-ngx/)** - Document management system with OCR, PostgreSQL, Valkey, Gotenberg, and Tika

### Media & Entertainment

- **[jellyfin](jellyfin/)** - Media server for movies, TV shows, and music
- **[immich](immich/)** - Photo and video backup solution

### Development & Code Hosting

- **[gitea](gitea/)** - Self-hosted Git service

### Storage & Backup

- **[minio](minio/)** - S3-compatible object storage server

### Security & Authentication

- **[vaultwarden](vaultwarden/)** - Self-hosted Bitwarden password manager with PostgreSQL backend

### Communication

- **[teamspeak3](teamspeak3/)** - Voice communication server
- **[smtp-relay](smtp-relay/)** - SMTP relay service for outbound email

### Personal

- **[hello](hello/)** - Personal portfolio static website showcasing skills and professional career

## Architecture

### Security-First Design

All services implement consistent security hardening:

- **Capability Management**: Drop all capabilities by default (`cap_drop: [ALL]`), selectively re-add only required ones (e.g., `CHOWN`, `SETUID`, `SETGID`)
- **Privilege Escalation Prevention**: `security_opt: [no-new-privileges]` on all containers
- **Non-Root Execution**: Explicit user IDs (typically `568:568` or `999:999`)
- **Read-Only Mounts**: Applied where applicable to prevent unauthorized modifications
- **Secrets Management**: Docker secrets sourced from environment variables

### Network Architecture

- **Internal Networks**: Each service stack uses isolated internal networks for inter-service communication
- **Reverse Proxy Network**: External `reverse-proxy` network connects web-accessible services to nginx-proxy-manager
- **Isolation**: Services not requiring external access remain on internal networks only

### Service Patterns

Each service directory contains:

```text
service-name/
├── docker-compose.yml  # Service definition with security configurations
├── .env                # Environment variables and secrets (gitignored)
└── README.md           # Service-specific setup commands (optional)
```

**Health Checks**: All services define comprehensive health checks with configurable start periods, intervals, retries, and timeouts.

**Volume Management**: Consistent naming using `VOLUME_<NAME>_SRC` and `VOLUME_<NAME>_DST` patterns in environment variables.

## Quick Start

### Prerequisites

- TrueNAS storage system with Docker/Portainer configured
- Portainer installed and accessible
- Docker and Docker Compose available

### Deploying a Service

1. Navigate to service directory:

   ```bash
   cd <service-name>
   ```

2. Create `.env` file with required variables (see service's docker-compose.yml for required variables)

3. Validate configuration:

   ```bash
   docker compose config
   ```

4. Deploy via Portainer or command line:

   ```bash
   docker compose up -d
   ```

5. View logs:

   ```bash
   docker compose logs -f
   ```

### Common Commands

```bash
# Pull latest images
docker compose pull

# Restart service
docker compose restart

# Stop and remove containers
docker compose down

# View container status
docker compose ps
```

## Service-Specific Commands

### Nextcloud

Nextcloud uses the `occ` command-line interface for configuration and maintenance:

```bash
# Execute occ commands
docker exec --user www-data nextcloud-nextcloud-1 php occ <command>

# Examples
docker exec --user www-data nextcloud-nextcloud-1 php occ app:install <app-name>
docker exec --user www-data nextcloud-nextcloud-1 php occ config:system:set <key> --value <value>
docker exec --user www-data nextcloud-nextcloud-1 php occ notify_push:setup https://${OVERWRITEHOST}/push
```

See [nextcloud/README.md](nextcloud/README.md) for extensive setup commands.

### Database Access

PostgreSQL services (nextcloud, paperless-ngx, n8n, vaultwarden, gitea):

```bash
# Generic command
docker exec -it <service>-db-1 psql -U <username> -d <database>

# Example for nextcloud
docker exec -it nextcloud-db-1 psql -U $(grep POSTGRES_USER nextcloud/.env | cut -d= -f2) -d $(grep POSTGRES_DB nextcloud/.env | cut -d= -f2)
```

## Maintenance

### Automated Updates

This repository uses [Renovate Bot](https://github.com/renovatebot/renovate) for automated dependency updates. Configuration is stored in [renovate.json](renovate.json).

- Docker images are automatically monitored for updates
- Pull requests are created for new versions
- Special versioning rules for services like MinIO (date-based releases)

### Manual Updates

1. Review Renovate PRs for available updates
2. Test updates in development environment if needed
3. Merge PR to update service version
4. Pull changes and restart service:

   ```bash
   git pull
   cd <service-name>
   docker compose pull
   docker compose up -d
   ```

### Backup Considerations

- **Volumes**: All persistent data is stored in TrueNAS datasets mapped via environment variables
- **Databases**: PostgreSQL databases should be backed up regularly using `pg_dump`
- **Environment Files**: `.env` files contain secrets and are gitignored - ensure these are backed up separately
- **Configurations**: Service-specific configs may be stored in volumes or Docker configs

## Security Notes

- All `.env` files are gitignored and contain sensitive credentials
- Never commit secrets or passwords to the repository
- Services run as non-root users (568:568 or 999:999)
- Container capabilities are minimized following principle of least privilege
- Regular updates via Renovate Bot help maintain security patches
- SSL/TLS termination handled by nginx-proxy-manager

## Network Configuration

- **Timezone**: Most services use `Europe/Berlin`
- **IPv6**: May be disabled on some services (e.g., nginx-proxy-manager)
- **Port Mapping**: Only exposed through nginx-proxy-manager or specific service requirements
- **DNS**: Services use internal Docker DNS for service discovery

## Technology Stack

- **Container Runtime**: Docker
- **Orchestration**: Docker Compose
- **Management**: Portainer
- **Storage**: TrueNAS
- **Databases**: PostgreSQL (Alpine Linux based)
- **Cache/Queue**: Valkey (Redis fork)
- **Reverse Proxy**: Zoraxy
- **Restart Policy**: `unless-stopped` or `always`

## Contributing

This is a personal infrastructure repository. Changes should:

1. Follow existing security patterns
2. Include comprehensive health checks
3. Use environment variables for configuration
4. Never commit `.env` files or secrets
5. Document service-specific commands in README.md
6. Validate with `docker compose config` before committing

## Troubleshooting

### Service won't start

1. Check logs: `docker compose logs <service-name>`
2. Verify `.env` file exists with required variables
3. Validate compose file: `docker compose config`
4. Check volume permissions match service user ID

### Database connection issues

1. Ensure database container is healthy: `docker compose ps`
2. Verify credentials in `.env` match database configuration
3. Check network connectivity between containers
4. Review database logs for authentication errors

### Permission denied errors

1. Verify user ID in docker-compose.yml matches volume ownership
2. Check capabilities are sufficient for service requirements
3. Ensure `no-new-privileges` isn't blocking necessary operations

## License

Private repository for personal use.

## Additional Resources

- [Portainer Documentation](https://docs.portainer.io/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [TrueNAS Documentation](https://www.truenas.com/docs/)
- Service-specific documentation linked in respective README.md files
