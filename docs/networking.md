# Networking

The server is integrated into the home network and is designed so that hosted applications are accessible from the home network and from authenticated remote VPN clients, while remaining inaccessible to guests and not directly exposed to the public Internet.

## Network Overview

There are three relevant access scenarios:

1. **Home network** — devices connected through LAN or home Wi-Fi.
2. **Remote access** — devices outside the home connecting through WireGuard.
3. **Guest network** — devices connected through the isolated guest Wi-Fi.

The home network and authenticated VPN clients use Pi-hole for DNS resolution and can access the hosted services through Caddy.

Guest devices only receive Internet access and are isolated from the home network.

## Home Network

Devices connected to the home LAN or home Wi-Fi are part of the home network.

```text
PC / Laptop / Phone
        │
        │ LAN / Home Wi-Fi
        ▼
   Home Network
        │
        ├── Pi-hole
        │      │
        │      └── Internal DNS
        │
        └── Caddy
               │
               └── Docker Services
```

The home network uses Pi-hole as its DNS resolver. This provides both network-wide DNS blocking and internal resolution of the hosted services.

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

The actual service hostname therefore does not need to resolve to a publicly accessible application server.

## Remote Access

Remote devices such as laptops and phones can access the infrastructure from mobile networks or external Wi-Fi.

The remote connection consists of two distinct DNS stages.

### 1. Finding the VPN endpoint

Before establishing the VPN connection, the remote device needs to resolve the public VPN hostname.

```text
Laptop / Phone
      │
      │ Mobile Data / External Wi-Fi
      ▼
   Internet
      │
      │ DNS lookup
      │ vpn.example.net
      ▼
 Public DNS
   (Porkbun)
      │
      │ Current public address
      ▼
  Fritz!Box
      │
      │ WireGuard
      ▼
 WireGuard
```

The public DNS record points the VPN hostname to the current public address of the home connection. The Fritz!Box provides the dynamic DNS endpoint and forwards the WireGuard connection to the VPN service.

WireGuard is the only intended public entry point to the infrastructure.

### 2. Accessing internal services

After the WireGuard tunnel has been successfully authenticated, the remote device has network access to the home network.

It can then use Pi-hole for internal DNS resolution just like a device physically connected to the home network.

For example:

```text
Remote Laptop / Phone
        │
        │ WireGuard
        ▼
   Home Network
        │
        │ DNS
        ▼
     Pi-hole
        │
        │ immich.example.net
        ▼
      Caddy
        │
        ▼
     Immich
```

The VPN therefore provides **network-level access** rather than exposing individual applications directly to the Internet.

## DNS

Pi-hole is used as the DNS resolver for devices in the home network and for authenticated WireGuard clients.

It provides two main functions:

### DNS blocking

DNS requests are filtered using Pi-hole's blocking functionality.

This provides network-wide blocking for supported devices without requiring each application or device to implement its own filtering.

### Internal service resolution

Pi-hole also resolves the hostnames used to access the self-hosted applications.

For example:

```text
immich.example.net
nextcloud.example.net
vaultwarden.example.net
```

These hostnames are resolved internally and direct the client towards the Caddy reverse proxy.

This means that the applications themselves do not need to be individually exposed to the public Internet.

## Reverse Proxy

Caddy acts as the reverse proxy for the hosted web applications.

Once Pi-hole has resolved a service hostname, the client connects to Caddy. Caddy then determines the requested service and forwards the request to the corresponding Docker container.

Conceptually:

```text
Client
  │
  │ HTTPS request
  │
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

This provides a consistent access mechanism for the different applications while keeping the individual application containers behind the reverse proxy.

## Guest Network

The guest Wi-Fi is intentionally separated from the home network.

Guest devices can access the Internet, but cannot access the internal infrastructure.

```text
Guest Phone
     │
     │ Guest Wi-Fi
     ▼
Guest Network
     │
     └──────────► Internet

     X
     │
     ▼
Home Network
```

As guest devices are not part of the home network, they do not use the internal Pi-hole DNS configuration and cannot resolve or access the internal service endpoints.

This separation allows visitors to use the Internet without granting them access to the self-hosted infrastructure.

## Network Access Model

The resulting access model is:

| Connection              | Internet | Home Network | Pi-hole | Hosted Services |
| ----------------------- | -------: | -----------: | ------: | --------------: |
| Home LAN                |      Yes |          Yes |     Yes |             Yes |
| Home Wi-Fi              |      Yes |          Yes |     Yes |             Yes |
| Authenticated WireGuard |      Yes |          Yes |     Yes |             Yes |
| Guest Wi-Fi             |      Yes |           No |      No |              No |

The central principle is that **application access follows network access**. Services are available to trusted devices inside the home network and to authenticated VPN clients, while the guest network and unauthenticated Internet users remain outside the trusted network boundary.
