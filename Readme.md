# Personal Server Infrastructure

A self-hosted infrastructure project running on a Debian-based home server.

The system hosts several containerized applications and provides
centralized DNS, reverse proxying, VPN access, and automated backups.

## Architecture

![Network Architecture](diagrams/network-architecture.svg)

The complete architecture is described in
[`docs/architecture.md`](docs/architecture.md).

## Services

| Service | Purpose |
|---|---|
| Caddy | Reverse proxy |
| WireGuard | VPN access |
| Pi-hole | DNS and ad blocking |
| Nextcloud | Cloud storage |
| Immich | Photo management |
| Vaultwarden | Password management |
| Mealie | Recipe management |
| Firefly III | Financial management |
| SparkyFitness | Fitness tracking |

## Technical Focus

- Debian Linux
- Docker / Docker Compose
- Reverse proxying
- VPN networking
- Internal DNS
- Containerized services
- Automated backups
- Infrastructure documentation

## Documentation

- [Architecture](docs/architecture.md)
- [Networking](docs/networking.md)
- [Services](docs/services.md)
- [Security](docs/security.md)
- [Backup](docs/backup.md)