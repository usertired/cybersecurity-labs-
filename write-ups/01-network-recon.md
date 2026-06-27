# 01 — Network Reconnaissance & Traffic Analysis

**Date:** June 21, 2026  
**Environment:** Kali Linux 2026.1 → Home network (192.168.100.0/24)  
**Tools:** Nmap 7.98, Wireshark 4.6.4, curl  

---

## Objective

Map all active devices on a local network, identify exposed services, and analyze network traffic to understand the difference between encrypted and unencrypted communication.

---

## Methodology

### 1. Host Discovery

```bash
nmap -sn 192.168.100.0/24
```

Scanned all 256 possible addresses in the subnet. Result: **5 active hosts**.

| IP | MAC | Vendor | Device Type |
|----|-----|--------|-------------|
| 192.168.100.1 | BC:46:32:24:6D:80 | Fiberhome | Router/Gateway |
| 192.168.100.2 | 9C:24:72:FB:14:5A | Sagemcom | Set-top box / ONT |
| 192.168.100.5 | FE:EB:1D:9E:26:03 | Unknown | Mobile device (randomized MAC) |
| 192.168.100.9 | 20:0B:74:5B:D2:37 | AzureWave | Smart TV / IoT device |
| 192.168.100.10 | — | — | Kali Linux (attacker machine) |

**Key concept:** MAC address vendor prefixes allow passive device identification without authentication.

---

### 2. Port & Service Scanning

```bash
nmap -sV 192.168.100.1
nmap -sV 192.168.100.2 192.168.100.5 192.168.100.9
```

| Host | Port | Service | Version |
|------|------|---------|---------|
| .1 (Router) | 80/tcp | HTTP | nginx (reverse proxy) |
| .2 (Sagemcom) | 80/tcp | HTTP | Unknown |
| .2 (Sagemcom) | 9080/tcp | HTTP | NRDP/2020.1.7.0 |
| .5 | 6100/tcp | Unknown | Unrecognized |
| .9 | — | — | All ports filtered (firewall active) |

---

### 3. Service Enumeration

Investigated the open endpoint on port 9080:

```bash
curl -v http://192.168.100.2:9080/
```

**Response headers:**
HTTP/1.1 200 OK

Server: NRDP/2020.1.7.0

Cache-Control: no-cache

Content-Length: 9
status=ok
**Finding:** Unauthenticated endpoint exposing Netflix Ready Device Platform (NRDP) version 2020.1.7.0. No credentials required to reach the API.

---

### 4. Traffic Analysis with Wireshark

Captured live traffic on eth0 and applied display filters.

**HTTP filter — unencrypted traffic:**

Browsed to `http://neverssl.com` and captured the full request in plaintext:
GET / HTTP/1.1

Host: 192.168.100.2:9080

User-Agent: Mozilla/5.0 (X11; Linux x86_64)

Accept-Language: en-US, en;q=0.5

Connection: keep-alive
Everything readable. No encryption. A threat actor on the same network could capture credentials, session tokens, or any data transmitted over HTTP.

**TLS filter — encrypted traffic:**

Browsed to any HTTPS site and applied `tls` filter. Result: only `Application Data` visible — content fully encrypted. Only the SNI (Server Name Indication) field leaked the destination hostname, which is unavoidable by design.

---

## Findings

| # | Finding | Severity | Notes |
|---|---------|----------|-------|
| 1 | NRDP/2020.1.7.0 endpoint exposed without authentication on port 9080 | Low | No public CVEs found. Unnecessary attack surface. |
| 2 | HTTP traffic fully readable in Wireshark | Informational | Affects any HTTP (non-HTTPS) communication on the network. |
| 3 | Device fingerprinting possible via MAC vendor lookup | Informational | All devices identifiable by manufacturer without authentication. |

---

## Key Concepts Learned

- **Network reconnaissance** — systematic discovery of hosts and services before any exploitation attempt
- **Attack surface** — every open port and exposed service increases the potential entry points for an attacker
- **MAC address vendor identification** — passive device fingerprinting using OUI prefixes
- **HTTP vs TLS** — unencrypted traffic is fully readable by any device on the same network segment
- **SYN scan (-sS)** — stealthier than default scan, requires root privileges (`sudo`)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap -sn` | Host discovery (ping sweep) |
| `nmap -sV` | Service and version detection |
| `nmap -sS` | SYN stealth scan |
| `curl -v` | Manual HTTP request with verbose headers |
| Wireshark | Live packet capture and traffic analysis |
