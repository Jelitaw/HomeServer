# Security

Security is a central consideration of the server architecture. The infrastructure is designed to minimize public exposure, separate trusted and untrusted networks, and keep sensitive configuration outside the public repository.

The security model is based primarily on **network isolation, VPN-based remote access, controlled service exposure, and separation of secrets from configuration**.

## Security Objectives

The main security objectives are:

* avoid directly exposing application services to the public Internet
* restrict remote access to authenticated VPN clients
* isolate guest devices from the trusted home network
* centralize web access through a reverse proxy
* keep credentials and secrets outside the public repository
* separate persistent application data from public configuration
* maintain regular backups of important application data

## Network Exposure

Application services are intended to remain inaccessible directly from the public Internet.

Remote access is provided through WireGuard instead:

```text
                        Internet
                           │
                           │
                    ┌──────▼──────┐
                    │  WireGuard  │
                    │ VPN Endpoint│
                    └──────┬──────┘
                           │
                    Authenticated
                       VPN client
                           │
                           ▼
                     Home Network
                           │
              ┌────────────┴────────────┐
              │                         │
           Pi-hole                   Caddy
              │                         │
              └────────────┬────────────┘
                           │
                           ▼
                    Docker Services
```

The intended public entry point is the WireGuard VPN. Application services such as Nextcloud, Immich and Vaultwarden are accessed through the home network or an authenticated VPN connection.

Actual Internet exposure is additionally controlled by the home router's firewall and port-forwarding configuration.

## VPN Access

WireGuard provides remote access to the home network.

A remote device must first establish an authenticated WireGuard connection before it can access the internal service infrastructure.

```text
Remote Device
      │
      │ WireGuard
      ▼
VPN Endpoint
      │
      │ Authenticated tunnel
      ▼
Home Network
      │
      ▼
Internal Services
```

This creates a trust boundary between devices on the public Internet and devices that have been granted VPN access.

## Guest Network Isolation

The guest Wi-Fi is separated from the trusted home network.

Guest devices receive Internet access but cannot access the internal services.

```text
                 ┌─────────────────┐
                 │   Home Network  │
                 │                 │
                 │  Server         │
                 │  Pi-hole        │
                 │  Docker         │
                 └─────────────────┘
                         ▲
                         │
                    No access
                         │
                 ┌───────┴─────────┐
                 │   Guest Network │
                 │                 │
                 │ Guest Devices   │
                 └─────────────────┘
                         │
                         ▼
                      Internet
```

This prevents guest devices from directly reaching the server and other devices on the trusted network.

## DNS and Service Discovery

Pi-hole is used as the DNS resolver for the home network and authenticated VPN clients.

Besides DNS blocking, it provides internal resolution for service hostnames. Caddy then routes requests to the appropriate Docker service.

```text
Client
  │
  │ DNS request
  ▼
Pi-hole
  │
  │ Internal resolution
  ▼
Caddy
  │
  ▼
Application
```

Pi-hole therefore provides centralized DNS and filtering, while network isolation and VPN access provide the actual access control.

## Reverse Proxy

Caddy acts as the central reverse proxy and TLS termination point for the hosted web applications.

Instead of exposing individual application containers directly, web traffic is routed through Caddy.

```text
Client
  │
  │ HTTPS
  ▼
Caddy
  │
  ├──► Immich
  ├──► Nextcloud
  ├──► Vaultwarden
  ├──► Mealie
  ├──► Firefly III
  └──► SparkyFitness
```

Most application containers therefore do not need their own host-level HTTP/HTTPS ports.

## Docker Networking

The services communicate through Docker networking.

The application containers are connected to the shared external `homelab` network. This allows Caddy and application components to communicate using Docker service/container names without requiring every service port to be published on the host.

Several applications use additional internal components such as databases, Redis instances or background workers. These components communicate internally within the Docker environment.

## Credential Management

Vaultwarden is used for personal credential storage.

Infrastructure and application secrets themselves are supplied separately through environment variables or other local configuration and are not committed to the public repository.

Sensitive values such as:

* passwords
* API keys
* tokens
* database credentials
* VPN credentials
* other secrets

are intentionally excluded from the repository.

Configuration examples use placeholders or environment variables, for example:

```yaml
environment:
  DB_PASSWORD: ${DB_PASSWORD}
```

## Public Repository Security

The infrastructure documentation repository is deliberately separate from the live server configuration.

It documents the architecture and provides sanitized examples rather than acting as the source of truth for the running system.

The repository does **not** contain:

* real passwords
* private keys
* VPN configuration files
* database credentials
* backup passwords
* personal access tokens
* private certificates
* sensitive live configuration
* application data

Public examples use placeholders where configuration values would otherwise reveal sensitive information.

## Backup Security

Application data is backed up regularly using Restic.

```text
Application Data
      │
      │ Nightly backup
      ▼
    Restic
      │
      ▼
 Dedicated HDD
```

The backup disk is used primarily as a backup target and is normally inactive outside the backup process.

The current backup remains physically located at home. If both the server and local backup storage were destroyed or stolen, the local backup would not protect against that physical loss.

An **off-site backup is therefore a planned improvement**.

## Security Boundaries

The architecture can be divided into several trust zones:

| Zone                            | Trust Level | Access                                |
| ------------------------------- | ----------- | ------------------------------------- |
| Public Internet                 | Untrusted   | No intended direct application access |
| Guest Network                   | Untrusted   | Internet only                         |
| Home Network                    | Trusted     | Internal services                     |
| Authenticated WireGuard Clients | Trusted     | Home network and internal services    |
| Docker Network                  | Internal    | Service-to-service communication      |

The most important boundary is between untrusted networks and the trusted home network.

```text
                 UNTRUSTED

        ┌────────────┴────────────┐
        │                         │
    Internet                 Guest Network
        │                         │
        └────────────┬────────────┘
                     │
                  VPN only
                     │
                     ▼
              TRUSTED NETWORK
                     │
        ┌────────────┼────────────┐
        │            │            │
     Pi-hole       Caddy       Docker
                                  │
                           Hosted Services
```

## Current Security Measures

Currently implemented measures include:

* WireGuard for authenticated remote access
* guest Wi-Fi isolation
* centralized DNS and blocking through Pi-hole
* centralized HTTPS routing through Caddy
* Docker networking for internal service communication
* separation of secrets from public configuration
* regular local backups using Restic
* separation of public documentation from live configuration

## Known Limitations and Future Improvements

Security is treated as an ongoing process.

Known limitations and planned improvements include:

### Off-site Backups

The current backup system does not protect against simultaneous physical loss of the server and local backup drive.

An off-site backup is planned to improve resilience against:

* fire
* theft
* major hardware loss
* other physical disasters affecting the home

### Further Hardening

Additional hardening can be evaluated over time, including:

* more restrictive network policies
* stronger service-specific isolation
* systematic update management
* monitoring and alerting
* stronger host-level access controls
* periodic review of exposed network services

These are treated as areas for continued improvement rather than being claimed as already implemented.

## Security Philosophy

The overall approach is to reduce the attack surface rather than relying on a single security mechanism.

The architecture follows a simple principle:

> **Do not expose a service when network-level access control can provide the required access instead.**

WireGuard provides the remote access boundary, Pi-hole provides centralized DNS and blocking, Caddy provides centralized web routing, Docker provides service-level isolation, and the public repository deliberately excludes sensitive configuration.

Together, these components form a layered security architecture for the personal self-hosted infrastructure.
