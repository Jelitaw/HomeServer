# Services

The server hosts a collection of self-hosted applications using Docker. Services are organized into separate application stacks where appropriate, with supporting databases and infrastructure components running in dedicated containers.

## Infrastructure Services

### Caddy

**Purpose:** Reverse proxy

Caddy provides the central HTTP(S) entry point for the hosted web applications. It receives requests for the configured service hostnames and forwards them to the corresponding application containers.

Caddy is connected to the shared `homelab` Docker network so that it can communicate with the hosted services without exposing their application ports directly.

### WireGuard

**Purpose:** Remote network access

WireGuard provides authenticated VPN access to the home network.

Remote devices first connect to the public VPN endpoint and, after successful authentication, can access the home network and its services using the same network and DNS infrastructure as devices physically connected at home.

### Pi-hole

**Purpose:** DNS resolution and network-wide blocking

Pi-hole provides DNS services for the home network and authenticated VPN clients.

It is responsible for:

* DNS blocking
* internal DNS resolution
* resolving hostnames for the self-hosted services

The internal DNS configuration allows service hostnames to resolve to the reverse proxy rather than requiring individual services to be publicly exposed.

---

## Application Services

### Nextcloud

**Purpose:** Personal cloud and synchronization

Nextcloud provides self-hosted file storage as well as calendar and contact synchronization.

The deployment consists of multiple containers:

* Nextcloud application
* MariaDB database
* Redis

The service is accessed through Caddy and is available to devices inside the home network or connected through WireGuard.

On mobile devices, Nextcloud is integrated with the Android ecosystem using dedicated client applications and synchronization tools. The client-side setup is documented in [Clients](clients.md).

### Immich

**Purpose:** Photo and video management

Immich provides self-hosted photo and video management, including automatic mobile photo backup.

The deployment consists of multiple containers:

* Immich server
* Immich machine-learning service
* PostgreSQL database
* Redis

The Immich mobile application is used as the primary mobile client.

### Vaultwarden

**Purpose:** Password management

Vaultwarden provides a self-hosted password-management service compatible with Bitwarden clients.

The deployment uses a dedicated Vaultwarden container with persistent application data.

The service is accessed through Caddy and is intended to be available only from the trusted home network or authenticated VPN clients.

### Mealie

**Purpose:** Recipe management

Mealie provides self-hosted recipe management and organization.

It runs as a containerized web application and is accessed through Caddy.

On mobile devices, Mealie is used as a Progressive Web App (PWA), providing an app-like interface without requiring a separate native mobile application.

### Firefly III

**Purpose:** Personal finance management

Firefly III provides self-hosted personal finance management.

The deployment consists of multiple containers:

* Firefly III application
* PostgreSQL database
* scheduled task container

The service is accessed through Caddy.

A mobile client, Waterfly III, is used as an alternative frontend for accessing the Firefly III instance from Android devices.

### SparkyFitness

**Purpose:** Fitness and health tracking

SparkyFitness provides self-hosted fitness and health tracking.

The deployment consists of multiple containers:

* SparkyFitness frontend
* SparkyFitness backend
* PostgreSQL database

The service is integrated with the mobile device and wearable ecosystem.

Wearable data can be collected through Gadgetbridge and made available through Android Health Connect before being synchronized with SparkyFitness.

---

## Container Architecture

The following table summarizes the current deployment:

| Service       | Containers | Main Function                     |
| ------------- | ---------: | --------------------------------- |
| Caddy         |          1 | Reverse proxy                     |
| WireGuard     |          1 | VPN                               |
| Pi-hole       |          1 | DNS and blocking                  |
| Vaultwarden   |          1 | Password management               |
| Mealie        |          1 | Recipe management                 |
| Immich        |          4 | Photo management                  |
| Nextcloud     |          3 | Cloud storage and synchronization |
| Firefly III   |          3 | Personal finance                  |
| SparkyFitness |          3 | Fitness tracking                  |

The exact container images and deployment configuration are intentionally not reproduced in this public documentation. The repository describes the architecture and design of the infrastructure rather than serving as a copy of the live deployment configuration.

## Shared Docker Network

The relevant containers use a shared external Docker network named `homelab`.

This allows services such as Caddy to communicate with application containers using Docker networking rather than relying on host-level port exposure.

Conceptually:

```text
                    Docker Host
                         │
                  ┌──────┴──────┐
                  │  homelab    │
                  │   network   │
                  └──────┬──────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
      Caddy           Immich          Nextcloud
        │                │                │
        │          ┌─────┴─────┐    ┌────┴────┐
        │          │ Redis     │    │ MariaDB │
        │          │ PostgreSQL│    │ Redis   │
        │          └───────────┘    └─────────┘
        │
        └──► Other hosted services
```

This network provides the internal connectivity required by the application stacks while keeping the services behind the reverse proxy.

## Service Access

All application services follow the same general access model:

```text
Trusted client
      │
      ▼
Home Network
      │
      ▼
Pi-hole
      │
      │ service hostname
      ▼
Caddy
      │
      ▼
Docker Service
```

Remote clients follow the same path after establishing the WireGuard VPN connection.

The applications therefore share a consistent access model regardless of whether the client is physically at home or connected remotely.
