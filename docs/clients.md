# Clients and Device Integration

The self-hosted services are accessed not only through web browsers, but also through dedicated mobile applications, Progressive Web Apps, and synchronization tools.

The client setup connects the server with personal computers, smartphones, and wearable devices while keeping the underlying services self-hosted.

## Overview

The main client integrations are:

| Service       | Client / Integration | Platform         | Purpose                              |
| ------------- | -------------------- | ---------------- | ------------------------------------ |
| Immich        | Immich App           | Android / iOS    | Photo and video backup               |
| Mealie        | Progressive Web App  | Mobile / Desktop | Recipe management                    |
| Nextcloud     | Nextcloud App        | Android / iOS    | Files and general access             |
| Nextcloud     | DAVx⁵                | Android          | Calendar and contact synchronization |
| Nextcloud     | ICSx⁵                | Android          | Calendar subscriptions               |
| Firefly III   | Waterfly III         | Android          | Mobile finance frontend              |
| SparkyFitness | SparkyFitness App    | Android          | Fitness tracking                     |
| Smart Watch   | Gadgetbridge         | Android          | Wearable data collection             |
| Fitness data  | Health Connect       | Android          | Data exchange between applications   |

## Network Access Model

Client applications use the same underlying network access model as other devices.

When a device is connected to the home Wi-Fi or LAN, it can access the internal services through the home network. DNS requests are handled by Pi-hole, which provides internal service resolution and DNS blocking. Requests to the service hostnames are then handled by Caddy, which forwards them to the appropriate application container.

When outside the home, the device first establishes a WireGuard VPN connection. After successful VPN authentication, the device has access to the home network and can use the same internal DNS and service hostnames as a device connected locally.

The VPN therefore provides network-level remote access rather than exposing each application directly to the public Internet.

### At home

```text
Phone / Laptop / PC
        │
        │ LAN / Home Wi-Fi
        ▼
  Home Network
        │
        ▼
     Pi-hole
        │
        ▼
      Caddy
        │
        ▼
Self-hosted Service
```

### Away from home

```text
Phone / Laptop
        │
        │ Mobile Data / External Wi-Fi
        ▼
     Internet
        │
        ▼
   WireGuard VPN
        │
        ▼
  Home Network
        │
        ▼
     Pi-hole
        │
        ▼
      Caddy
        │
        ▼
Self-hosted Service
```

Guest devices use a separate guest network and do not have access to the home network or its internal services.

## Immich

Immich is accessed using its dedicated mobile application.

The application provides access to the self-hosted photo library and can automatically upload photos and videos from the mobile device.

```text
Phone Camera
     │
     ▼
 Immich App
     │
     │ Upload
     ▼
Immich Server
     │
     ▼
Self-hosted Storage
```

The Immich deployment itself consists of several cooperating containers:

* Immich server
* machine-learning service
* Redis
* PostgreSQL

The application data and database are stored persistently outside the containers.

This allows the phone to function as a client for the self-hosted photo management system without relying on a third-party cloud photo service.

## Mealie

Mealie is used through its web interface as a Progressive Web App (PWA).

The PWA provides an app-like interface on mobile and desktop devices while the application itself remains hosted on the server.

```text
Phone / Desktop
       │
       ▼
    Mealie PWA
       │
       ▼
      Caddy
       │
       ▼
     Mealie
```

Mealie is deployed as a container with persistent application data.

## Nextcloud

Nextcloud provides both general file access and synchronization functionality.

The Nextcloud mobile application is used for general access to files and the service, while dedicated Android synchronization applications are used for calendar and contact integration.

The Nextcloud deployment consists of:

* Nextcloud application
* MariaDB
* Redis

The application and supporting services communicate internally through the Docker network.

### Calendar and Contacts

DAVx⁵ provides CalDAV and CardDAV synchronization between Nextcloud and the Android calendar and contact providers.

The resulting data can therefore be used by the standard applications on the Android device.

```text
                 Nextcloud
                     │
              CalDAV / CardDAV
                     │
                     ▼
                   DAVx⁵
                 /       \
                ▼         ▼
          Android       Android
          Calendar      Contacts
```

ICSx⁵ is additionally used for calendar subscription and synchronization use cases.

This setup keeps calendar and contact data synchronized with the self-hosted Nextcloud instance while still making it available through the normal Android user interfaces.

## Firefly III

Firefly III is accessed from mobile devices using Waterfly III.

Waterfly III acts as a mobile frontend for the self-hosted Firefly III instance.

```text
Phone
  │
  ▼
Waterfly III
  │
  ▼
 Caddy
  │
  ▼
Firefly III
```

The Firefly III deployment consists of:

* Firefly III application
* PostgreSQL database
* dedicated Alpine-based cron container for scheduled tasks

The financial data remains stored on the self-hosted infrastructure.

## Fitness and Wearable Integration

Fitness data uses a multi-stage integration between the wearable device, Android applications, and the self-hosted SparkyFitness installation.

The main data flow is:

```text
Smart Watch
     │
     ▼
Gadgetbridge
     │
     ▼
Health Connect
     │
     ▼
SparkyFitness App
     │
     ▼
SparkyFitness Server
```

### Gadgetbridge

Gadgetbridge is used to communicate with the supported wearable device and collect its data locally on the Android device.

The application can provide the wearable data locally without requiring the vendor's cloud service for the basic data flow.

### Health Connect

Android Health Connect acts as an intermediary data layer between applications.

Gadgetbridge can make supported health and activity data available through Health Connect, allowing other applications to consume the data without requiring a direct integration between the applications.

### SparkyFitness

The SparkyFitness client can use the available health data and synchronize it with the self-hosted SparkyFitness server.

The server-side deployment consists of:

* SparkyFitness frontend
* SparkyFitness backend
* PostgreSQL database

The frontend and backend communicate internally through the Docker network, while the database is kept internal to the application stack.

The resulting data flow is:

```text
┌─────────────┐
│ Smart Watch │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Gadgetbridge│
└──────┬──────┘
       │
       ▼
┌───────────────┐
│ Health Connect│
└──────┬────────┘
       │
       ▼
┌───────────────┐
│ SparkyFitness │
│      App      │
└──────┬────────┘
       │
       ▼
┌───────────────┐
│ SparkyFitness │
│    Server     │
└───────────────┘
```

## Client Architecture

The complete client ecosystem can be summarized as follows:

```text
                         Self-hosted Server
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
          Nextcloud           Immich          Firefly III
              │                 │                 │
        ┌─────┴─────┐           │           Waterfly III
        │           │           │
      DAVx⁵       ICSx⁵    Immich App
        │
   ┌────┴────┐
   ▼         ▼
Android   Android
Calendar  Contacts


                       SparkyFitness
                             ▲
                             │
                      SparkyFitness App
                             ▲
                             │
                      Health Connect
                             ▲
                             │
                       Gadgetbridge
                             ▲
                             │
                        Smart Watch


                           Mealie
                             ▲
                             │
                           PWA
```

The network path to the server is shared across these integrations:

```text
Client
  │
  ├── Home network ──────────────┐
  │                              │
  └── WireGuard when remote ─────┤
                                 ▼
                             Pi-hole
                                 │
                                 ▼
                               Caddy
                                 │
                                 ▼
                          Docker Services
```

## Client Installation

The following applications form the main client-side software stack:

### Android

* **Immich** — dedicated photo management and backup application
* **DAVx⁵** — CalDAV/CardDAV synchronization with Nextcloud
* **ICSx⁵** — calendar subscription and synchronization
* **Gadgetbridge** — wearable device integration
* **Health Connect** — health-data integration layer
* **SparkyFitness** — fitness tracking client
* **Waterfly III** — Firefly III mobile frontend
* **Nextcloud** — general file and service access

### Mobile / Desktop

* **Mealie PWA** — app-like access to Mealie through browser PWA functionality

The exact applications installed on a particular device may vary depending on which services and integrations are required.

## Design Principles

A central goal of the client architecture is to keep the data flow under personal control wherever practical.

Instead of replacing every service with a proprietary cloud platform, the server provides the central data storage and application layer, while dedicated client applications provide convenient access from personal devices.

The network architecture also avoids exposing each application independently to the public Internet. Remote clients first establish an authenticated WireGuard connection and then use the same internal DNS and reverse-proxy infrastructure as local clients.

The result is an integrated ecosystem in which computers, smartphones, and wearable devices can interact with the self-hosted services while using a consistent underlying network access model.
