# Technitium DNS Security Configuration

This repository contains configuration files, installation guides, and security documentation for implementing **Technitium DNS Server** in a home network environment.

The goal is to provide centralized DNS filtering, DNS redundancy, and network-level protection against unauthorized DNS services.

The environment uses two Technitium DNS servers configured in a cluster, along with firewall rules to enforce DNS security across multiple network VLANs.

---

## DNS Infrastructure

| Component | Configuration |
|---|---|
| DNS Software | Technitium DNS Server |
| Primary DNS | 192.168.XXX.XX |
| Secondary DNS | 192.168.XXX.XX |
| Primary Operating System | Ubuntu Server (Proxmox VM) |
| Secondary Operating System | Ubuntu Server (Raspberry Pi 4) |
| DNS Configuration | Technitium DNS Cluster |
| DNS Filtering | Advanced Blocking App |
| Network Firewall | UniFi Cloud Gateway Ultra |

Both Technitium DNS servers operate as part of a cluster to synchronize supported DNS zones and configuration.

**Important:** Technitium clustering does not automatically redirect DNS requests when a server becomes unavailable. DNS redundancy is achieved by configuring clients with both DNS server addresses through DHCP or another supported method.

---

## Repository Contents

### DNS Filtering Lists

| File | Description |
|---|---|
| [blocklist.txt](blocklist.txt) | Custom DNS domain blocklist used by Technitium Advanced Blocking. |
| [allowlist.txt](allowlist.txt) | Custom DNS allowlist for approved domains. |
| [firefox-doh-blocklist.txt](firefox-doh-blocklist.txt) | DNS-over-HTTPS resolver domains used to help reduce browser DNS bypass. |

### Installation and Configuration Guides

| File | Description |
|---|---|
| [HowToInstallTechnitium.md](HowToInstallTechnitium.md) | Step-by-step instructions for installing Technitium DNS Server on Ubuntu Server (Proxmox VM) and Raspberry Pi 4. |
| [Technitium-DNS-Cluster-Setup.md](Technitium-DNS-Cluster-Setup.md) | Instructions for configuring primary and secondary Technitium DNS servers in a cluster, including DNS records, TLSA/DANE validation, synchronization, and failover testing. |
| [firewall-rules.md](firewall-rules.md) | Firewall configuration for enforcing approved DNS servers and restricting DNS bypass across network VLANs. |

---

## Security Goals

The primary security objectives of this project include:

1. **Centralized DNS Filtering**
   - Block unwanted or potentially malicious domains.
   - Maintain custom blocklists and allowlists through GitHub.
   - Apply consistent DNS filtering policies across network devices.

2. **DNS Redundancy and Availability**
   - Maintain primary and secondary Technitium DNS servers.
   - Synchronize supported DNS zones and configuration through clustering.
   - Allow clients to continue resolving DNS queries if one server becomes unavailable.

3. **Prevent Unauthorized DNS Access**
   - Restrict direct DNS requests to unauthorized external resolvers.
   - Allow approved DNS traffic through designated Technitium servers.
   - Apply DNS enforcement policies across multiple VLANs.

4. **Restrict Encrypted DNS Bypass**
   - Restrict unauthorized DNS-over-TLS (DoT) traffic.
   - Reduce DNS-over-HTTPS (DoH) bypass through firewall rules and domain filtering.
   - Prevent unauthorized DNS traffic where technically feasible.

5. **Maintain Device Compatibility**
   - Preserve connectivity for smart TVs, gaming consoles, and computers.
   - Investigate DNS filtering issues before creating exceptions.
   - Apply narrowly scoped firewall or DNS exceptions when necessary.

---

## Deployment

The following steps outline the general deployment process.

### Step 1 — Install Technitium DNS Servers

Install Technitium DNS Server on two systems:

- Primary: Ubuntu Server running on Proxmox.
- Secondary: Ubuntu Server running on Raspberry Pi 4.

Follow the [Technitium Installation Guide](HowToInstallTechnitium.md).

### Step 2 — Configure DNS Clustering

Configure the two Technitium DNS servers as a cluster.

This includes:

- Initializing the cluster on the primary server.
- Configuring the cluster domain and DNS records.
- Configuring TLSA/DANE certificate validation.
- Joining the secondary server.
- Verifying cluster connectivity and synchronization.

Follow the [Technitium DNS Cluster Setup Guide](Technitium-DNS-Cluster-Setup.md).

### Step 3 — Configure DHCP DNS Servers

Configure the network's DHCP settings to distribute both Technitium DNS server addresses to clients.

This provides DNS redundancy if either server becomes unavailable.

### Step 4 — Configure DNS Filtering

Install and configure the Technitium Advanced Blocking App.

Configure the following:

- Custom domain blocklists.
- Custom domain allowlists.
- External DNS security blocklists.
- DNS filtering groups and policies.
- Automatic blocklist updates.

### Step 5 — Configure Firewall Rules

Configure firewall rules to:

- Allow DNS traffic to approved Technitium servers.
- Restrict unauthorized external DNS requests.
- Restrict DNS-over-TLS traffic.
- Reduce DNS-over-HTTPS bypass.
- Apply DNS security policies across network VLANs.

Refer to [Firewall Rules](firewall-rules.md).

### Step 6 — Configure Device Exceptions

Some devices and applications may require additional DNS or firewall considerations.

Examples include:

- Smart TVs
- Gaming consoles
- Gaming PCs
- Communication applications

Investigate DNS query logs and firewall logs before adding exceptions.

### Step 7 — Verify DNS Resolution and Failover

Verify that:

- Both Technitium DNS servers respond to DNS queries.
- Cluster synchronization is functioning.
- DNS filtering works on both servers.
- Unauthorized external DNS queries are restricted.
- Clients continue resolving DNS when either server is unavailable.
- TVs, gaming systems, and other applications function correctly.

---

## Important Notes

### Firewall Configuration

Firewall configurations documented in this repository are intended for manual implementation.

**Firewall rules are not automatically installed or applied by Technitium DNS Server.**

Review and adapt all firewall rules to match your network environment before deployment.

### DNS Clustering

Technitium clustering provides synchronization of supported configuration and DNS zones between cluster members.

Clustering alone does not provide automatic client DNS failover.

Clients should be configured with both DNS server addresses, or an appropriate highly available DNS endpoint.

### DNS Security

DNS filtering and firewall restrictions provide additional layers of network security but cannot guarantee that all DNS bypass methods are prevented.

VPNs, proxies, encrypted DNS services, and other techniques may require additional network security controls.

---

## Project Documentation

This repository documents the installation, configuration, troubleshooting, and ongoing maintenance of a home network DNS security environment using Technitium DNS Server.

Configuration examples should be reviewed and adjusted for individual network environments.
