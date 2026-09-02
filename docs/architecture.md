# Architecture

This project documents the architecture of my self-hosted home server and the surrounding client infrastructure.

The system is based on a Debian Linux server running containerized applications with Docker. Network access is provided through the home network or an authenticated WireGuard VPN connection. Internal DNS is handled by Pi-hole, while Caddy acts as a reverse proxy for the hosted applications.

The repository contains documentation and sanitized configuration examples of the infrastructure. It is not the deployment source of truth for the live server.

## High-Level Architecture

The architecture can be divided into four main areas:

1. **Client devices** — computers and mobile devices accessing the infrastructure.

2. **Network access** — LAN, home Wi-Fi, guest Wi-Fi and remote VPN access.

3. **Core network services** — WireGuard, Pi-hole and Caddy.

4. **Application services** — containerized applications such as Nextcloud, Immich, Vaultwarden, Mealie, Firefly III and SparkyFitness.

The overall network architecture is illustrated in [NetworkArchitecture.puml](NetworkArchitecture.puml).

## Server

The server runs Debian Linux and uses Docker to host the applications.

The applications are organized into individual Docker Compose stacks. Some applications consist of multiple cooperating containers for the application itself, databases and supporting services.

Examples include:

* **Immich** — application server, machine-learning service, Redis and PostgreSQL
* **Nextcloud** — application, MariaDB and Redis
* **Firefly III** — application, PostgreSQL and a dedicated cron container
* **SparkyFitness** — frontend, backend and PostgreSQL

Single-container services such as Mealie and Vaultwarden are also deployed using Docker Compose.

A shared Docker network named `homelab` is used for communication between the reverse proxy and the relevant application containers. Databases and supporting services can therefore communicate internally without requiring their service ports to be published on the host.

Persistent application data is stored separately from the container filesystem.

## Network Access

Devices connected to the home LAN or home Wi-Fi are part of the home network and can access the hosted services.

Remote devices can obtain equivalent network access through WireGuard. The WireGuard endpoint is the only intended public entry point to the infrastructure.

The guest Wi-Fi is isolated from the home network. Guest devices can access the Internet but cannot access the internal services or the home-network DNS infrastructure.

The actual exposure of services to the public Internet is controlled by the Fritz!Box firewall and port-forwarding configuration. Docker port mappings on the server alone do not imply that a service is reachable from the Internet.

## DNS

Pi-hole provides DNS for devices in the home network and for authenticated VPN clients.

It provides two main functions:

* network-wide DNS blocking
* internal DNS resolution for hosted services

For example, a request for `immich.example.net` is resolved internally by Pi-hole and directed towards the reverse proxy.

This allows the same service hostnames to be used by devices connected locally and by devices connected through the VPN.

## Reverse Proxy

Caddy provides the central HTTP(S) routing layer.

After a service hostname has been resolved by Pi-hole, the request reaches Caddy. Caddy then forwards the request to the corresponding Docker service over the internal `homelab` network.

For example:

```text
immich.example.net
        │
        ▼
    Pi-hole
        │
        ▼
      Caddy
        │
        ▼
 Immich server
```

Caddy provides a consistent HTTPS-based access method for the hosted applications. The applications therefore do not need to expose their service ports directly to the host for normal client access.

Caddy also handles TLS configuration. DNS-based certificate validation is used for the configured domains.

## Remote Access

A remote device first resolves the public VPN hostname through public DNS.

The resulting connection reaches the home network through the Fritz!Box and is forwarded to the WireGuard endpoint. After successful VPN authentication, the device can access the internal network and use the same DNS and service routing mechanisms as devices physically connected at home.

The remote access path is therefore:

```text
Remote device
      │
      ▼
   Internet
      │
      ▼
 Public DNS
      │
      ▼
  Fritz!Box
      │
      ▼
 WireGuard
      │
      ▼
Home network
      │
      ▼
  Pi-hole
      │
      ▼
    Caddy
      │
      ▼
  Service
```

The WireGuard tunnel itself uses a dedicated UDP endpoint. WG-Easy additionally provides a web-based management interface for managing the VPN configuration.

## Client Ecosystem

The server is not used exclusively through a web browser. Several applications on personal devices integrate with the hosted services.

Examples include:

* Immich mobile application for photo backup and access
* Mealie as a Progressive Web App
* DAVx⁵ for Nextcloud calendar and contact synchronization
* ICSx⁵ for calendar subscription and synchronization use cases
* Gadgetbridge for wearable data
* Health Connect as an intermediary for fitness data
* SparkyFitness mobile application
* Waterfly III as a mobile frontend for Firefly III

The client-side setup is documented separately in [Clients](clients.md).

## Storage and Backups

Application data is stored on dedicated active storage attached to the server.

A second local drive is used as the backup target for automated Restic backups. The backup drive is normally idle outside the backup window in order to reduce unnecessary runtime.

The backup process creates regular Restic snapshots and applies a retention policy to older snapshots. The repository contains a sanitized example of the backup script and its configuration.

The current setup is primarily designed to protect against accidental deletion, corruption and individual storage or service failures. Because the backup storage is located at the same physical site as the server, it does not provide protection against events such as fire or theft affecting both devices.

An off-site backup strategy is planned as a future improvement.

See [Backup](backup.md) for details.

## Design Goals

The infrastructure was designed around the following goals:

* keep application services inaccessible from the public Internet where possible
* provide remote access through a single authenticated VPN entry point
* centralize DNS resolution and network-wide blocking
* provide consistent service routing through a reverse proxy
* keep application components isolated in containers
* separate application data from container filesystems
* maintain automated backups
* provide convenient native or mobile access where available
* document the architecture in a reproducible and understandable way

## Security Boundary

The primary network security boundaries are the separation between the Internet, guest network and home network.

Application services are intended to be reachable from the home network or through an authenticated VPN connection rather than being individually exposed to the public Internet.

The guest network is intentionally isolated from the home network and therefore cannot access internal services or the internal DNS infrastructure.

The public-facing access path is consequently centered around the VPN rather than exposing each individual application directly.

Within the server, Docker provides an additional layer of service isolation. Application containers communicate through the internal `homelab` network, while persistent data and backup storage are managed separately from the container lifecycle.
