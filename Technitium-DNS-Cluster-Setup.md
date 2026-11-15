# Technitium DNS - Primary and Secondary Cluster Setup

Companion guide to `HowToInstallTechnitium.md`.

This guide builds a two-node Technitium cluster: the primary runs on an Ubuntu Server VM in Proxmox and the secondary on a Raspberry Pi 4 with Ubuntu Server. It is based on the clustering work done in September-October 2026 and on Technitium's own clustering documentation. Menu labels can differ between Technitium versions, so confirm them in your installed version.

---

## 1. What Clustering Does (and Does Not Do)

- **It synchronizes configuration.** One node is the primary; secondaries pull configuration from it. Configuration changes are made **on the primary only**, and the primary has to be online for changes to happen.
- **Each node still answers DNS on its own.** If the primary is offline, the secondary keeps serving DNS.
- **It does not move clients between servers.** There is no floating IP. Clients must be given both DNS addresses by DHCP (see `pfSense-DHCP-and-DNS.md`).

### What gets synchronized

According to Technitium's documentation for joining a cluster, joining **overwrites** the secondary's local **Allowed, Blocked, Apps, Settings, and Administration** configuration with the primary's, and the secondary then keeps synchronizing from the primary.

Practical consequences:

- **Back up the secondary before joining.** Its existing settings are replaced.
- **Make changes on the primary only.** Edits made on a secondary are overwritten.
- **Apps and their configs (including Advanced Blocking) come from the primary.** See `AdvancedBlocking.md`.
- **Query logs stay local.** Each node keeps its own SQLite query log.

Confirm each of these in your own cluster (Section 6).

---

## 2. Example Settings

| Setting | Primary | Secondary |
| --- | --- | --- |
| Host | Ubuntu Server VM on Proxmox | Ubuntu Server on Raspberry Pi 4 |
| Example LAN IP | `192.168.XXX.XX` | `192.168.XXX.XX` |
| Technitium version at setup | 15.4 | 15.4 |
| DNS | TCP/UDP 53 | TCP/UDP 53 |
| HTTP admin | TCP 5380 | TCP 5380 |
| HTTPS admin and cluster traffic | TCP 53443 | TCP 53443 |

| Name | Example |
| --- | --- |
| Cluster domain | `dns.MYDOMAIN.com` |
| Primary node name | `primary.dns.MYDOMAIN.com` |
| Secondary node name | `secondary.dns.MYDOMAIN.com` |

Use one set of names consistently. Each node's DNS server domain must be a subdomain of the cluster domain. Choose a name that is **not** already used as an authoritative zone elsewhere on your network; reusing an existing namespace is a common cause of cluster name-resolution and DANE errors.

---

## 3. Prerequisites

1. Both nodes installed and working independently (`HowToInstallTechnitium.md`).
2. Stable IP addresses (DHCP reservations) and working connectivity between the nodes.
3. Accurate time on both nodes (see the Raspberry Pi clock section in `HowToInstallTechnitium.md`).
4. Both nodes on the same Technitium version.
5. TCP 53443 allowed **between the two nodes** and from trusted admin hosts only. If the nodes are on different VLANs, add a pfSense rule for it. Do not expose 5380 or 53443 to the internet. See `Maintenance-and-Hardening.md` for UFW examples.
6. A current backup of both nodes (`BackupAndRecovery.md`).
7. Recommended: configure the web service with a valid HTTPS certificate on the primary **before** initializing the cluster (Settings > Web Service). Technitium recommends this so the secondary can authenticate the primary when it joins.

Preflight on both nodes:

```
timedatectl
sudo ss -lntup | grep -E ':(53|5380|53443)\b'
sudo ufw status verbose
```

From an admin machine:

```
dig @192.168.XXX.XX example.com A
dig @192.168.XXX.XX example.com A
```

---

## 4. Initialize the Cluster on the Primary

1. Sign in to the **primary** web interface.
2. Go to **Administration > Cluster**.
3. Choose **Initialize Cluster**.
4. Enter the cluster domain (for example `dns.MYDOMAIN.com`).
5. Enter the primary node's static IP address(es), the addresses other nodes use to reach it.
6. Finish initialization and confirm the primary shows as the primary cluster node.

Technitium creates the cluster domain as a primary zone and manages the **A, AAAA, and TLSA records for each node** in it. You normally do not create these records by hand.

**Recorded result:** the primary initialized successfully, and the cluster zones appeared (for example `dns.MYDOMAIN.com` and a `cluster-catalog` zone).

---

## 5. Join the Secondary

1. Sign in to the **secondary** web interface.
2. Go to **Administration > Cluster** and choose to initialize by **joining a cluster**.
3. Enter the secondary node's IP address and the primary node's URL (shown on the primary's cluster screen).
4. Enter the primary's administrator credentials (and two-factor code if enabled).
5. Complete the join and check the cluster status on **both** nodes.

### Certificate validation

Technitium authenticates cluster nodes to each other with DANE-EE: the cluster creates a TLSA record for each node, and node-to-node TLS is validated against it. The **join** step is different, because the secondary has to trust the primary before any TLSA record is available to it.

- Best: give the primary a valid HTTPS certificate before initializing the cluster, so the primary can be authenticated at join time.
- Technitium documents the **Ignore Certificate Errors** option for joining when the primary uses a self-signed certificate on a private network.

**Recorded history:** the first join failed with a DANE/TLSA validation or name-resolution error. The secondary was eventually joined with **Ignore Certificate Errors** enabled. The final certificate state was not independently confirmed in the original notes.

What to verify now (instead of assuming the setup is fully authenticated):

- [ ] Both nodes show **Connected** (not Unreachable) in the cluster status, and stay that way after a refresh and after a reboot of each node.
- [ ] No DANE/TLSA errors appear in the cluster status or in the logs.
- [ ] Each node can resolve the other's cluster name (Section 7, troubleshooting).
- [ ] The primary's web service uses a certificate you chose on purpose (valid certificate, or a self-signed one you accepted knowingly). If you want to remove the self-signed trust assumption, install a valid certificate and review Technitium's cluster documentation for your version.

If you created A or TLSA records by hand while troubleshooting, review them against what the cluster manages, since duplicates or mismatches can cause validation failures.

---

## 6. Validate the Cluster

### 6.1 Status

Open **Administration > Cluster** on each node. Both should be listed and connected.

### 6.2 DNS on each node

```
dig @192.168.XXX.XX example.com A
dig @192.168.XXX.XX example.com A
dig @192.168.XXX.XX primary.dns.MYDOMAIN.com A
dig @192.168.XXX.XX primary.dns.MYDOMAIN.com A
```

Use real names for your cluster. Query each server by its own address.

### 6.3 Configuration sync test

1. On the **primary**, add a harmless test entry (for example a test domain in the Blocked section).
2. Wait for synchronization.
3. On the **secondary**, confirm the entry appears.
4. Remove the test entry on the primary and confirm it disappears from the secondary.
5. Open **Apps > Advanced Blocking > Config** on both nodes and confirm they match.

### 6.4 Ports

```
sudo ss -lntup | grep -E ':(53|5380|53443)\b'
sudo ufw status verbose
```

| Verification | Primary | Secondary |
| --- | --- | --- |
| Cluster status Connected | NOT TESTED | NOT TESTED |
| DNS answers over UDP and TCP | NOT TESTED | NOT TESTED |
| Config change syncs from primary | n/a | NOT TESTED |
| Advanced Blocking config matches | NOT TESTED | NOT TESTED |
| Survives a reboot of each node | NOT TESTED | NOT TESTED |

---

## 7. Troubleshooting

| Symptom | Check |
| --- | --- |
| Secondary cannot join | Primary URL and port 53443 reachability, admin credentials, time on both nodes, certificate options |
| DANE/TLSA validation error | Each node can resolve the cluster names, TLSA records for the nodes exist and match, clocks are correct |
| "Cannot resolve cluster name" | The node's DNS server domain is a subdomain of the cluster domain, and the cluster domain is not shadowed by another zone or forwarder |
| Node shows Unreachable | Connectivity on 53443, firewall rules between nodes, name resolution, time |
| `Signature Not Yet Valid` or DNSSEC SERVFAIL on the Pi | Raspberry Pi time synchronization section in `HowToInstallTechnitium.md` |
| Secondary answers but clients fail when primary is down | The DHCP-advertised DNS addresses and each client's resolver behavior |
| Config changes disappear on a secondary | They were made on the secondary; make changes on the primary |

Name-resolution checks:

```
dig @192.168.XXX.XX primary.dns.MYDOMAIN.com A
dig @192.168.XXX.XX secondary.dns.MYDOMAIN.com A
```

Run each against both nodes.

---

## 8. Clients and Failover Test

Hand out **both** addresses through DHCP (`pfSense-DHCP-and-DNS.md`). A client using only the primary does not discover the secondary just because the servers are clustered, and clients differ in how they use multiple resolvers.

Controlled test:

1. Confirm both servers resolve a test domain.
2. Confirm a test client has both DNS addresses (`resolvectl status` on Linux, `ipconfig /all` on Windows).
3. In a maintenance window, stop the DNS service on the primary.
4. From the client, run `dig example.com A` **without** naming a server. This tests the client's own failover.
5. Confirm browsing still works.
6. Restore the primary and confirm it reconnects.

**Recorded test:** the primary was disabled, the secondary still answered direct `dig` queries, and browsing continued. This does not guarantee identical failover on every client or VLAN, so repeat the test from a representative device on each VLAN.

---

## 9. If the Primary Is Lost

- The secondary keeps answering DNS, but configuration cannot be changed while the primary is down.
- Technitium provides an option to promote a secondary to primary when the primary is offline and unrecoverable. Use it only in that case and read Technitium's current documentation first.
- After a rebuild, a replacement node joins as a secondary and receives its configuration from the cluster.

See `BackupAndRecovery.md` for the full recovery scenarios.

---

## 10. Remaining Checks

- [ ] Both nodes stay Connected after reboot and after a refresh.
- [ ] The primary web service uses a certificate you chose deliberately.
- [ ] Config and Advanced Blocking changes on the primary appear on the secondary.
- [ ] TCP 53443 is limited to the cluster peers and trusted admin hosts.
- [ ] Both nodes have accurate time.
- [ ] DHCP advertises both servers on every client VLAN.
- [ ] Failover tested from a representative client on each VLAN.
- [ ] Backups of both nodes exist and a restore has been tested.

*Based on installation notes from September-October 2026 and Technitium's clustering documentation. Review against the live Technitium 15.4 interface before treating this as a universal procedure.*
