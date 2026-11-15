# Technitium DNS Security Configuration

Configuration files, installation guides, and security documentation for running **Technitium DNS Server** as a filtered, redundant DNS service on a home network behind a pfSense firewall.

Goals:

1. **Centralized DNS filtering** with the Advanced Blocking app and lists maintained in this repository.
2. **DNS redundancy** with two Technitium nodes in a cluster and clients configured with both addresses.
3. **Restrict DNS bypass** (direct DNS, DNS-over-TLS, and common DNS-over-HTTPS resolvers) with pfSense firewall rules.
4. **Keep devices working**: smart TVs, consoles, and gaming PCs are tested and given narrow exceptions only when needed.

> **Placeholders:** IP addresses appear as `192.168.XXX.XX` and cluster names as `dns.MYDOMAIN.com`. Replace them with your own values. Never commit real credentials, certificates, or an unredacted pfSense `config.xml`.

---

## Infrastructure

| Component | Configuration |
| --- | --- |
| DNS software | Technitium DNS Server (clustered, primary + secondary) |
| Primary node | Ubuntu Server VM on Proxmox |
| Secondary node | Ubuntu Server on Raspberry Pi 4 |
| DNS filtering | Advanced Blocking app (configured on the primary, synchronized by the cluster) |
| Upstream DNS | AdGuard DNS over TLS (concurrent forwarding) |
| Firewall / router / DHCP | pfSense on Netgate hardware |

Clustering synchronizes configuration. **It does not redirect clients if a server fails.** Redundancy comes from handing out both DNS addresses through DHCP.

---

## Documentation

| File | What it covers |
| --- | --- |
| [HowToInstallTechnitium.md](HowToInstallTechnitium.md) | Installing both nodes, optional Raspberry Pi time-sync repair, initial configuration, DNS verification tests, troubleshooting |
| [Technitium-DNS-Cluster-Setup.md](Technitium-DNS-Cluster-Setup.md) | Creating the cluster, certificate/DANE notes, sync checks, failover test, primary recovery |
| [AdvancedBlocking.md](AdvancedBlocking.md) | Advanced Blocking app, client groups, list sources, verification |
| [advanced-blocking-config.json](advanced-blocking-config.json) | The Advanced Blocking config (source of truth; IP redacted) |
| [pfSense-DHCP-and-DNS.md](pfSense-DHCP-and-DNS.md) | DHCP reservations, handing out both DNS servers, pfSense's own resolver |
| [firewall-rules.md](firewall-rules.md) | pfSense aliases and rules that restrict DNS bypass |
| [BackupAndRecovery.md](BackupAndRecovery.md) | What to back up, how to restore, recovery scenarios |
| [Maintenance-and-Hardening.md](Maintenance-and-Hardening.md) | Updates, server firewall (UFW), HTTPS admin, SSH, routine checks |

### Lists

| File | Purpose |
| --- | --- |
| [blocklist.txt](blocklist.txt) | Custom domain blocklist (dating, adult, social media, anonymity tools) |
| [allowlist.txt](allowlist.txt) | Domains that must stay reachable |
| [firefox-doh-blocklist.txt](firefox-doh-blocklist.txt) | DNS-over-HTTPS resolver hostnames and the Firefox DoH canary domain |

The lists are loaded by the Advanced Blocking app straight from the raw GitHub URLs, so a commit to `main` takes effect at the next list refresh (24 hours). Protect the GitHub account with two-factor authentication.

---

## Recommended Order

1. Install both nodes and configure the **primary** fully (forwarders, logging, Query Logs): [HowToInstallTechnitium.md](HowToInstallTechnitium.md). The secondary needs only the basics before joining, because joining overwrites its settings.
2. Create DHCP reservations and hand out both DNS servers: [pfSense-DHCP-and-DNS.md](pfSense-DHCP-and-DNS.md).
3. Build the cluster: [Technitium-DNS-Cluster-Setup.md](Technitium-DNS-Cluster-Setup.md).
4. Configure filtering on the primary: [AdvancedBlocking.md](AdvancedBlocking.md).
5. Add the pfSense bypass rules: [firewall-rules.md](firewall-rules.md).
6. Set up backups: [BackupAndRecovery.md](BackupAndRecovery.md).
7. Harden and schedule maintenance: [Maintenance-and-Hardening.md](Maintenance-and-Hardening.md).
8. Test devices (TVs, consoles, gaming PCs) and record any exceptions.

---

## Important Notes

- **Firewall rules are applied by hand.** Technitium does not install or manage them. Review every rule before applying it to your network.
- **Cluster changes happen on the primary.** Secondary nodes synchronize from it, and joining a cluster overwrites the secondary's local configuration.
- **Bypass prevention is best effort.** VPNs, proxies, iCloud Private Relay, Tor, and unknown DoH servers may still get around DNS filtering and need additional controls.
- **Query logs are sensitive.** They record which sites each device looks up. Keep retention short, restrict admin access, and do not publish logs.

## Testing Status

Record real results in each guide's verification tables. A test that has not been run stays **NOT TESTED**. The DNS tests are defined once, in Part 4 of [HowToInstallTechnitium.md](HowToInstallTechnitium.md); the other guides refer back to them.
