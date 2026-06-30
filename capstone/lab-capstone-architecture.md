# Capstone — Mini ZTNA Architecture

## Overview

This is a working, simplified Zero Trust Network Access architecture built using WireGuard, Nginx, and Keycloak — the same building blocks (conceptually) as the InstaSafe product stack.

## Architecture Diagram

```
+-------------------+
|     My Laptop      |
|  WireGuard Client  |
|     10.8.0.2        |
+----------+----------+
           |
           | Encrypted WireGuard Tunnel
           | (UDP 51820)
           v
+-----------------------------+
|   VM2 - vpn-client            |
|   140.245.253.205              |
|                                |
|   +----------------------+    |
|   |  WireGuard Server     |    |
|   |     10.8.0.1            |    |
|   +-----------+----------+    |
|               |                |
|   +-----------v----------+    |
|   |  Nginx Reverse Proxy  |    |
|   |  Port 80 -> /app        |    |
|   +-----------+----------+    |
+---------------+----------------+
                |
                | Private network (10.0.0.0/24)
                v
+-----------------------------+
|   VM1 - vpn-server             |
|   140.245.218.252               |
|   10.0.0.40                     |
|                                 |
|   +----------------------+     |
|   |  Keycloak IdP +        |     |
|   |  App Server             |     |
|   |  Port 8080               |     |
|   +----------------------+     |
+-----------------------------+

           Monitored by:
+-----------------------------+
|  Uptime Kuma (VM1:3001)       |
|  -> Capstone WireGuard GW       |
|  -> Capstone Nginx Proxy        |
|  -> Capstone Keycloak IdP       |
+-----------------------------+
```

## Component Mapping to InstaSafe

| Component | This Lab | InstaSafe Equivalent |
|---|---|---|
| WireGuard tunnel | Laptop <-> VM2 encrypted tunnel | InstaSafe Agent + tunnel encryption |
| Nginx reverse proxy | VM2 routes traffic to backend | InstaSafe Gateway application routing |
| Keycloak | Identity provider on VM1 | Azure AD / Okta integration |
| Uptime Kuma | Health monitoring of all 3 components | Site24x7 / internal monitoring |

## Build Steps Summary

1. Generated WireGuard keypairs for laptop (client) and VM2 (server)
2. Configured WireGuard server on VM2 with 10.8.0.1/24, listening on UDP 51820
3. Configured WireGuard client on laptop with 10.8.0.2/24, peer pointing to VM2's public IP
4. Verified tunnel connectivity: ping 10.8.0.1 from laptop - 0% packet loss
5. Installed and configured Nginx on VM2 as a reverse proxy, forwarding /app to VM1's Keycloak on port 8080
6. Started Keycloak container on VM1 (already provisioned in Lab 2.2)
7. Verified end-to-end: curl http://10.8.0.1/app from laptop returned the Keycloak welcome page
8. Added 3 Uptime Kuma monitors covering the WireGuard gateway, Nginx proxy, and Keycloak IdP

## End-to-End Verification

```bash
# From laptop, connected via WireGuard:
curl http://10.8.0.1/app
```

This request travels: Laptop -> WireGuard tunnel (encrypted) -> VM2 Nginx (gateway) -> VM1 Keycloak (identity-aware app)

Response: Keycloak welcome page HTML returned successfully - confirming the full chain works.

## Issues I Ran Into

**WireGuard interface kept dropping.** During testing, wg0 went down on both the laptop and VM2 multiple times, breaking the tunnel silently - wg show would return empty with no error. Fixed each time with sudo wg-quick up wg0. In production this is solved by enabling the systemd unit (systemctl enable wg-quick@wg0) so it survives reboots and restarts automatically.

**iptables REJECT rule blocked Nginx traffic.** VM2 had a catch-all REJECT rule on the INPUT chain that silently dropped TCP port 80 traffic even though Nginx was correctly listening. Diagnosed using iptables -L INPUT -n -v --line-numbers which showed 0 packets hitting the port 80 chain entirely - meaning traffic never reached Nginx. Fixed by inserting an explicit ACCEPT rule for port 80 before the REJECT rule.

**Keycloak container had crashed silently.** docker ps showed no Keycloak container running even though it had been working in earlier labs. The container had exited with status 255 due to an internal hostname resolution error. Restarted with docker start keycloak and it came back healthy.

**Curl from laptop showed wrong source IP.** At one point curl -v showed the request originating from my laptop's real IP instead of the tunnel IP 10.8.0.2 - meaning traffic was leaking outside the tunnel. This was because the WireGuard interface had silently gone down. Once wg0 was confirmed up with wg show and ip route, the source IP corrected itself.

## Monitoring Setup

Added 3 dedicated Uptime Kuma monitors for the capstone stack, separate from the Module 1-3 lab monitors:

- **Capstone Keycloak IdP** - HTTP check on 140.245.218.252:8080
- **Capstone Nginx Proxy** - HTTP check on 140.245.253.205/app
- **Capstone WireGuard Gateway** - Ping check on 140.245.253.205 (TCP port checks don't work reliably for WireGuard's UDP-based handshake, so ping is the more accurate health signal)

All 3 show green after setup.
