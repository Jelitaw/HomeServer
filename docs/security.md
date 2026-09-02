# Security

Security is a central consideration of the server architecture. The infrastructure is designed to minimize public exposure, separate trusted and untrusted networks, and keep credentials under personal control.

The security model is based primarily on **network isolation, VPN-based remote access, controlled service exposure, and centralized credential management**.

## Security Objectives

The main security objectives are:

* avoid directly exposing application services to the public Internet
* restrict remote access to authenticated VPN clients
* isolate guest devices from the trusted home network
* keep service credentials under personal control
* centralize access to web applications through a reverse proxy
* separate application data from the public configuration of the infrastructure
* maintain backups of important application data

## Network Exposure

The applications hosted on the server are intended to remain inaccessible directly from the public Internet.

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

This approach means that services such as Nextcloud, Immich and Vaultwarden do not need to be individually exposed to the public Internet.

The intended public entry point is the WireGuard VPN.

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

This creates a clear trust boundary between devices on the public Internet and devices that have been granted access to the home network.

## Guest Network Isolation

The guest Wi-Fi is separated from the trusted home network.

Guest devices receive Internet access but cannot access the internal services.

```text
                 ┌─────────────────┐
                 │   Home Network  │
                 │                 │
                 │  Server         │
                 │  Pi-hole       │
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

This prevents visitors from directly reaching the server and other devices on the trusted network.

## DNS and Service Discovery

Pi-hole is used as the DNS resolver for the home network and authenticated VPN clients.

Besides DNS blocking, it provides internal resolution for the service hostnames.

This allows the infrastructure to use stable hostnames for applications without requiring those applications to have individual public DNS records pointing to publicly accessible services.

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

The DNS layer therefore also contributes to the network access model by keeping service discovery inside the trusted network.

## Reverse Proxy

Caddy acts as the central reverse proxy for the hosted web applications.

Instead of exposing individual application containers directly, web traffic is handled through Caddy.

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

This creates a single controlled application entry point inside the trusted network.

It also separates the externally visible web interface from the internal application containers.

## Docker Network Isolation

The hosted applications communicate through Docker networking.

The services that need to interact with one another are connected to the shared external `homelab` network.

This allows, for example, Caddy to communicate directly with application containers without requiring each application to expose its service port on the host.

The application stacks themselves may contain additional internal components such as databases, Redis instances or background workers.

## Credential Management

Service credentials are managed using the self-hosted Vaultwarden installation.

The goal is to avoid storing credentials directly in application configuration files or in the public infrastructure repository.

Sensitive values such as:

* passwords
* API keys
* tokens
* database credentials
* VPN credentials
* other secrets

are intentionally excluded from the public repository.

Where configuration examples are useful, sensitive values are represented using placeholders or environment variables.

For example:

```yaml
environment:
  DB_PASSWORD: ${DB_PASSWORD}
```

The actual value is supplied separately and is not part of the publicly documented configuration.

## Public Repository Security

The infrastructure documentation repository is deliberately separate from the live server configuration.

It documents the architecture and provides sanitized examples rather than acting as the source of truth for the running system.

The repository therefore does **not** contain:

* real passwords
* private keys
* VPN configuration files
* database credentials
* backup passwords
* personal access tokens
* private certificates
* complete live configuration containing sensitive values

Public examples use placeholders where configuration values would otherwise reveal sensitive information.

This separation also prevents accidental deployment changes from being made directly to the running infrastructure through the documentation repository.

## Backup Security

Application data is backed up regularly using Restic.

The current backup architecture consists of:

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

The backup disk is used primarily as a backup target and is normally kept inactive outside the backup process.

This reduces unnecessary operating time for the backup drive.

However, the current setup has an important limitation: the backup is still physically located at home.

If both the server and the local backup storage were destroyed or stolen at the same time, the local backup would not provide protection against that physical loss.

An **off-site backup is therefore a planned improvement**.

## Security Boundaries

The architecture can be divided into several trust zones:

| Zone                            | Trust Level | Access                             |
| ------------------------------- | ----------- | ---------------------------------- |
| Public Internet                 | Untrusted   | No direct application access       |
| Guest Network                   | Untrusted   | Internet only                      |
| Home Network                    | Trusted     | Internal services                  |
| Authenticated WireGuard Clients | Trusted     | Home network and internal services |
| Docker Service Network          | Internal    | Service-to-service communication   |

The most important boundary is between the public/guest networks and the trusted home network.

```text
                 UNTRUSTED
                     │
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

The currently implemented security-related measures include:

* VPN-only remote access to application services
* guest Wi-Fi isolation
* internal DNS resolution through Pi-hole
* centralized reverse proxy through Caddy
* credentials managed through Vaultwarden
* Docker network separation for service communication
* regular local backups using Restic
* separation of public documentation from live configuration

## Known Limitations and Future Improvements

The current infrastructure is functional, but security is an ongoing process.

Known limitations and planned improvements include:

### Off-site Backups

The current backup system protects against certain forms of data loss, but does not protect against simultaneous physical loss of the server and local backup drive.

An off-site backup is planned to improve resilience against:

* fire
* theft
* major hardware loss
* other physical disasters affecting the home

### Further Hardening

Additional hardening can be evaluated over time, including:

* more restrictive network policies
* service-specific isolation
* systematic update management
* monitoring and alerting
* stronger host-level access controls
* periodic review of exposed network services

These are treated as areas for continued improvement rather than being claimed as already implemented.

## Security Philosophy

The overall approach is to reduce the attack surface rather than relying on a single security mechanism.

The architecture follows a simple principle:

> **Do not expose a service when network-level access control can provide the required access instead.**

WireGuard provides the remote access boundary, Pi-hole provides internal DNS and blocking, Caddy provides centralized web routing, Docker provides service-level isolation, and Vaultwarden provides centralized credential management.

Together, these components form a layered security architecture appropriate for a personal self-hosted infrastructure.
