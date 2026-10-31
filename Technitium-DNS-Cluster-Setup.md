# Technitium DNS — Primary and Secondary Cluster Setup

> Companion guide to `HowToInstallTechnitium.md`. This guide documents the clustering work completed in September–October 2026 and distinguishes verified results from steps that must be checked against the live configuration. It is not an export of the servers' settings.

## 1. Overview

This setup uses two Technitium DNS Server 15.4 instances. The primary runs on an Ubuntu Server VM in Proxmox; the secondary runs on a Raspberry Pi 4 with Ubuntu Server. Both can answer DNS queries. **Clustering synchronizes supported cluster data; it does not itself provide a floating DNS address or automatically reroute clients to the secondary.** Clients should be configured with both DNS server addresses.

| Setting | Primary | Secondary |
| --- | --- | --- |
| Role | Primary cluster node | Secondary cluster node |
| Host | Ubuntu Server on Proxmox | Ubuntu Server on Raspberry Pi 4 |
| Example LAN IP from this installation | `192.168.10.10` | `192.168.10.11` |
| Technitium version at time of setup | 15.4 | 15.4 |
| DNS service | TCP/UDP 53 | TCP/UDP 53 |
| HTTP administration | TCP 5380 | TCP 5380 |
| HTTPS administration / cluster communication | TCP 53443 | TCP 53443 |

**Replace the example addresses and names before reusing these instructions.** The original notes used `dns.MYDOMAIN.com` as a cluster-domain placeholder, with `primary.dns.MYDOMAIN.com` and `secondary.dns.MYDOMAIN.com` as node names. Use your actual cluster names consistently.

## 2. Prerequisites

1. Complete the installation and initial configuration of Technitium on both machines using `HowToInstallTechnitium.md`.
2. Give both servers stable LAN IP addresses and verify they can reach one another.
3. Verify accurate time on both systems. This is especially important for DNSSEC and certificate validation. For the Raspberry Pi 4 Chrony setup and troubleshooting, see `HowToInstallTechnitium.md`.
4. Ensure DNS resolution works independently on each server before clustering.
5. Allow required management/cluster traffic **only between trusted management hosts and cluster nodes**. In the recorded setup, TCP 53443 initially worked locally but not remotely until the firewall was adjusted. Do not expose TCP 5380 or 53443 to the public internet.
6. Have a current configuration backup or snapshot before changing cluster settings.

Useful preflight commands on both Ubuntu systems:

```bash
timedatectl
sudo ss -lntup | grep -E ':(53|5380|53443)\b'
sudo ufw status verbose
```

From a trusted management machine, check that each node answers DNS:

```bash
dig @192.168.xxx.xx example.com A
dig @192.168.xxx.xx example.com A
```

## 3. Initialize the cluster on the primary

1. Sign in to the **primary** Technitium administration interface.
2. Open the **Clustering** controls in the Technitium web interface. The exact menu labels can vary with Technitium version; verify them in your installed version.
   - a. Administration > Cluster
4. Choose **Initialize New Cluster**.
5. Enter the intended cluster domain (the original notes used `dns.MYDOMAIN.com`).
6. Supply the primary node name, using the naming scheme configured for your network.
7. Review the certificate-validation options. **Use trusted certificate validation where possible.** Record the cluster domain, primary hostname, and any DNS records or certificate requirements shown by the interface.
8. Finish initialization and confirm the primary shows as a connected primary cluster node.

**Recorded result:** The primary cluster was initialized successfully. Cluster-related zones observed included `dns.MYDOMAIN.com` and `cluster-catalog.dns.MYDOMAIN.com` (placeholder names here).

## 4. Prepare cluster DNS and certificate validation

The original setup encountered a failed secondary join involving a DANE/TLSA lookup or certificate-validation error for the primary hostname on HTTPS port 53443.

1. Verify that the **primary cluster hostname** resolves to the primary's correct IP address. In the example naming scheme, this is `primary.dns.MYDOMAIN.com` → `192.168.xxx.xx`.
2. If using DANE validation, ensure the required TLSA record exists for the **actual cluster service name and port**, and that it matches the certificate/key material used by the HTTPS endpoint. A TLSA record for TCP port 53443 uses the owner-name pattern `_53443._tcp.primary.dns.MYDOMAIN.com`.
   - a. Check > Zones > Type: TLSA : and ensure that record in in the zones file
      - * Should have Name: _53443._tcp (Note: if there is a .ns1 after TCP in the name, remove)
4. Check that the relevant DNS zone and DNSSEC validation chain are correct for your chosen DANE configuration. A DNS A record alone does **not** satisfy TLSA validation.
5. Verify the primary's HTTPS service is reachable from the secondary, and that both clocks are accurate.

Example checks (substitute your real domain):

```bash
dig @192.168.xxx.xx primary.dns.MYDOMAIN.com A
dig @192.168.xxx.xx _53443._tcp.ns1.dnscluster.home TLSA
```

**Historical workaround:** The supplied notes state the secondary was eventually joined with **Ignore Certificate Validation Errors** after DANE validation failed. This reduced authentication assurance and should **not** be treated as the recommended permanent setting. Revisit the live cluster configuration and establish validated TLS/PKI or DANE before considering this security issue resolved. Earlier troubleshooting also involved creating DNS A/TLSA records; the exact final validated certificate state was not independently confirmed in the notes.

## 5. Join the secondary node

1. Sign in to the **secondary** Technitium administration interface.
2. Open the **Clustering** controls and choose **Join Cluster**.
3. Enter the primary cluster hostname and required connection details from the primary's configuration.
4. Choose the intended certificate-validation method. Prefer successful validation rather than bypassing errors.
5. Complete the join and inspect the cluster status on **both** nodes.
6. Confirm the primary shows connected and the secondary identifies itself as the secondary node.
7. Confirm cluster-related zones and records are visible on the secondary.

**Recorded result:** Both nodes joined the cluster and the secondary served replicated cluster records and recursive DNS responses. The exact UI labels and prompts were not preserved in the original notes, so confirm them while following this section.

## 6. Validate cluster operation

### 6.1 Confirm cluster status

In the Technitium web interface, inspect the cluster status on each node. Verify both nodes are listed and connected, and inspect the cluster-related zones/records on both servers.

### 6.2 Test DNS independently on each node

```bash
dig @192.168.xxx.xx example.com A
dig @192.168.xxx.xx example.com A
```

For a record that is actually configured in your cluster zone, query **that exact name** against each server:

```bash
dig @192.168.xxx.xx primary.dns.MYDOMAIN.com A
dig @192.168.xxx.xx primary.dns.MYDOMAIN.com A
```

Replace `MYDOMAIN.com` with your real domain. Do not use the old notes' `dns.dns.MYDOMAIN.com` test name unless that record truly exists.

### 6.3 Verify firewall and listener state

```bash
sudo ss -lntup | grep -E ':(53|5380|53443)\b'
sudo ufw status verbose
```

If the secondary cannot join, verify that TCP 53443 is reachable **between cluster members**, then review name resolution, certificates, TLSA records, DNSSEC status, and system time.

## 7. Configure clients and test failover

Configure your DHCP/network settings to advertise **both** DNS addresses:

```text
Primary DNS:   192.168.xxx.xx
Secondary DNS: 192.168.xxx.xx
```

A client using only the primary will not automatically discover the secondary simply because the servers are clustered. Clients and operating systems may use multiple configured resolvers differently; some may use either server even while both are healthy.

**Controlled failover test:**

1. Verify both servers resolve a test domain individually.
2. Confirm a test client has both DNS addresses assigned (`resolvectl status` on Ubuntu, or `ipconfig /all` on Windows).
3. During a maintenance window, stop the primary DNS service or otherwise make the primary unavailable to the test client.
4. Confirm the primary no longer answers and the secondary still does.
5. From the **client**, test ordinary name resolution and browsing without explicitly targeting the secondary in the command. This is what verifies client failover behavior.
6. Restore the primary and confirm it rejoins and answers queries.

```bash
# Run from a trusted client; explicit queries verify server availability.
dig @192.168.xxx.xx example.com A
dig @192.168.xxx.xx example.com A

# This tests the client's normal resolver selection.
dig example.com A
```

**Recorded test:** The original notes say the primary was disabled, the secondary still answered direct `dig` queries, and web browsing continued. This supports successful operation in that test, but does not guarantee identical failover behavior for every client or VLAN.

## 8. Troubleshooting reference

| Symptom | Check |
| --- | --- |
| Secondary cannot join | Primary hostname resolution, TCP 53443 connectivity, cluster settings, TLS certificate validation |
| DANE/TLSA validation error | TLSA owner name and contents, certificate/key match, DNSSEC chain, accurate clocks |
| `Signature Not Yet Valid` or DNSSEC `SERVFAIL` on Pi | See the Raspberry Pi 4 time synchronization section in `HowToInstallTechnitium.md`; check DNSSEC status |
| Primary works, secondary does not answer | Secondary service and firewall, DNS listening addresses, cluster status |
| Secondary answers direct `dig` but clients fail when primary is down | DHCP-advertised DNS addresses and each client's resolver selection |
| Cluster records not visible on secondary | Cluster status and synchronization; inspect logs before changing records |

## 9. Security and remaining checks

- [ ] Confirm the live cluster uses **validated** certificates rather than permanently ignoring certificate errors.
- [ ] Confirm the exact production cluster domain, node hostnames, and TLSA records.
- [ ] Limit admin/cluster ports to trusted IP addresses; do not expose them publicly.
- [ ] Verify both nodes have accurate time; see `HowToInstallTechnitium.md` for the Raspberry Pi 4 fix.
- [ ] Confirm cluster synchronization and recursive DNS operation on both nodes.
- [ ] Confirm DHCP advertises both DNS servers on every intended VLAN.
- [ ] Test client failover from representative Windows, Linux, and other devices.
- [ ] Document Advanced Blocking separately; an app or its settings should not be assumed synchronized without checking both nodes.
- [ ] Document firewall-based DNS bypass prevention separately; do not assume that cluster membership enforces client DNS routing.

---

*Based on historical installation notes last documented October 12, 2026. Review against the live Technitium 15.4 interfaces before publishing as a universally reproducible procedure.*
