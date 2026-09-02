# Architecture

This project documents the architecture of my self-hosted home server and the surrounding client infrastructure.

The system is based on a Debian Linux server running containerized applications with Docker. Network access is provided through the home network or an authenticated WireGuard VPN connection. Internal DNS is handled by Pi-hole, while Caddy acts as a reverse proxy for the hosted applications.

## High-Level Architecture

![Network Architecture](../diagrams/network-architecture.svg)

The architecture can be divided into four main areas:

1. **Client devices** — computers and mobile devices accessing the infrastructure.
2. **Network access** — LAN, home Wi-Fi, guest Wi-Fi and remote VPN access.
3. **Core network services** — WireGuard, Pi-hole and Caddy.
4. **Application services** — containerized applications such as Nextcloud, Immich and Vaultwarden.

## Server

The server runs Debian Linux and uses Docker to host the applications.

The applications are organized into individual Docker Compose stacks where appropriate. Multi-container applications such as Immich, Nextcloud and Firefly III use separate containers for their application, database and supporting components.

A shared Docker network named `homelab` is used to allow the relevant containers to communicate with each other.

## Network Access

Devices connected to the home LAN or home Wi-Fi are part of the home network and can access the hosted services.

Remote devices can obtain equivalent network access through WireGuard. The VPN is the only intended public entry point to the infrastructure.

The guest Wi-Fi is isolated from the home network. Guest devices can access the Internet but cannot access the internal services.

## DNS

Pi-hole provides DNS for devices in the home network and for authenticated VPN clients.

It provides two main functions:

* network-wide DNS blocking
* internal DNS resolution for hosted services

For example, a request for `immich.example.net` is resolved internally by Pi-hole and directed towards the reverse proxy.

## Reverse Proxy

Caddy provides the internal HTTP(S) routing layer.

After a service hostname has been resolved by Pi-hole, the request reaches Caddy. Caddy then forwards the request to the corresponding Docker service.

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
     Immich
```

This provides a consistent access method for the different applications without requiring each application to expose its own network entry point.

## Remote Access

A remote device first resolves the public VPN hostname through public DNS.

The resulting connection reaches the Fritz!Box and is forwarded to WireGuard. After successful VPN authentication, the device becomes part of the home network and can use the same internal DNS and service access mechanisms as devices physically connected at home.

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

## Client Ecosystem

The server is not used exclusively through a web browser. Several applications on personal devices integrate with the hosted services.

Examples include:

* Immich mobile application for photo backup and access
* Mealie as a Progressive Web App
* DAVx⁵ for Nextcloud calendar and contact synchronization
* ICSx⁵ for calendar subscription/synchronization use cases
* Gadgetbridge for wearable data
* Health Connect as an intermediary for fitness data
* SparkyFitness mobile application
* Waterfly III as a mobile frontend for Firefly III

The client-side setup is documented separately in [Clients](clients.md).

## Storage and Backups

Application data is stored on dedicated storage attached to the server.

A second local drive is used for automated Restic backups. The backup process runs periodically, with the backup drive normally remaining idle outside the backup window.

The current setup is primarily designed to protect against accidental deletion, corruption and individual service failures. Protection against physical loss of both the server and local backup storage requires an off-site backup and is planned as a future improvement.

See [Backup](backup.md) for details.

## Design Goals

The infrastructure was designed around the following goals:

* keep application services inaccessible from the public Internet
* provide remote access through a single authenticated VPN entry point
* centralize DNS resolution and blocking
* provide consistent service routing through a reverse proxy
* keep services isolated in containers
* maintain automated backups
* provide convenient native or mobile access where available
* document the architecture in a reproducible and understandable way

## Security Boundary

The primary network security boundary is the separation between the Internet, guest network and home network.

Application services are intended to be reachable only from the home network or through an authenticated VPN connection.

The guest network intentionally does not have access to the home network.

The public-facing component is therefore limited to the VPN access path rather than exposing each individual application.
