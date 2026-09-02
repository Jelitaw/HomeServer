# Clients and Device Integration

The self-hosted services are accessed not only through web browsers, but also through dedicated mobile applications, Progressive Web Apps and synchronization tools.

The client setup connects the server with personal computers, smartphones and wearable devices while keeping the underlying services self-hosted.

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

## Mobile Access

Mobile applications connect to the self-hosted services through the same network access model as other clients.

When a device is connected to the home Wi-Fi, it can directly access the services through the home network.

When outside the home, the device first establishes a WireGuard VPN connection. Once connected, the device can resolve and access the internal service hostnames in the same way as a device physically connected to the home network.

```text id="l3gq7h"
At home:

Phone
  │
  │ Home Wi-Fi
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
Self-hosted service
```

```text id="5w7whx"
Away from home:

Phone
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
Self-hosted service
```

## Immich

Immich is accessed using its dedicated mobile application.

The application provides access to the self-hosted photo library and can automatically upload photos and videos from the mobile device.

```text id="m24f3a"
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

This allows the phone to function as a client for the self-hosted photo management system without relying on a third-party cloud photo service.

## Mealie

Mealie is used through its web interface as a Progressive Web App (PWA).

The PWA provides an app-like interface on the phone while the application itself remains hosted on the server.

```text id="t7wq8j"
Phone
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

## Nextcloud

Nextcloud provides both general file access and synchronization functionality.

The Nextcloud mobile application is used for general access to the server, while dedicated Android synchronization applications are used for calendar and contact integration.

### Calendar and Contacts

DAVx⁵ provides CalDAV and CardDAV synchronization between Nextcloud and the Android calendar and contact providers.

The resulting data can therefore be used by the standard applications on the Android device.

```text id="d7s5qf"
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

The separation between the server and the Android applications means that the calendar and contact data remains synchronized with the self-hosted Nextcloud instance while still being available through the normal Android user interfaces.

## Firefly III

Firefly III is accessed from mobile devices using Waterfly III.

Waterfly III acts as a mobile frontend for the self-hosted Firefly III instance.

```text id="4ojx1r"
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

This provides a dedicated mobile interface while keeping the underlying financial data on the self-hosted server.

## Fitness and Wearable Integration

Fitness data uses a multi-stage integration between the wearable device, Android applications and the self-hosted SparkyFitness installation.

The main data flow is:

```text id="6d5i5k"
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

The application avoids requiring a vendor cloud service for the basic data flow.

### Health Connect

Android Health Connect acts as an intermediary data layer between applications.

Gadgetbridge can make supported health and activity data available through Health Connect, allowing other applications to consume the data without requiring a direct integration between the applications.

### SparkyFitness

The SparkyFitness mobile application can then use the available health data and synchronize it with the self-hosted SparkyFitness server.

This creates a complete path from the wearable device to the self-hosted application:

```text id="g7h9bf"
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
┌─────────────┐
│Health Connect│
└──────┬──────┘
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

```text id="4o8v7m"
                           Self-hosted Server
                                  │
             ┌────────────────────┼────────────────────┐
             │                    │                    │
         Nextcloud              Immich             Firefly III
             │                    │                    │
        ┌────┴────┐               │               Waterfly III
        │         │               │
      DAVx⁵     ICSx⁵        Immich App
        │
        ▼
 Android Calendar
 Android Contacts


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

## Client Installation

The following applications form the main client-side software stack:

### Android

* **Immich** — dedicated photo management and backup application
* **DAVx⁵** — CalDAV/CardDAV synchronization with Nextcloud
* **ICSx⁵** — calendar subscription/synchronization
* **Gadgetbridge** — wearable device integration
* **Health Connect** — health-data integration layer
* **SparkyFitness** — fitness tracking client
* **Waterfly III** — Firefly III mobile frontend
* **Nextcloud** — general file and service access

### Mobile / Desktop

* **Mealie PWA** — app-like access to Mealie through the browser's PWA functionality

The exact applications installed on a particular device may vary depending on which services and integrations are required.

## Design Principle

A central goal of the client architecture is to keep the data flow under personal control wherever practical.

Instead of replacing every service with a proprietary cloud platform, the server provides the central data storage and application layer, while dedicated client applications provide convenient access from personal devices.

The result is an integrated ecosystem in which computers, smartphones and wearable devices can interact with the self-hosted services while using the same underlying network and authentication model.
