# Technitium Advanced Blocking Configuration

Companion guide to `HowToInstallTechnitium.md` and `Technitium-DNS-Cluster-Setup.md`.

This guide documents DNS filtering with the **Advanced Blocking** app: how to install it, what the configuration does, which block and allow lists it loads, and how to verify it.

> **Note:** The full configuration lives in one place: [advanced-blocking-config.json](advanced-blocking-config.json). This document explains it and does not repeat it, so the two cannot drift apart. The JSON file is based on the live primary config with a few cleanups (Section 9), and private IP addresses are replaced with placeholders. After any change, export the config from Technitium and replace that file.

---

## 1. Overview

| Item | Value |
| --- | --- |
| App | Advanced Blocking (Technitium App Store) |
| Configured on | The **primary** node; the cluster synchronizes it to the secondary |
| Blocking response | NXDOMAIN (`blockAsNxDomain: true`) |
| List refresh | Every 24 hours |
| Client groups | `everyone else` (default), `netgear-restricted` (one device) |
| Custom lists | `blocklist.txt`, `allowlist.txt`, `firefox-doh-blocklist.txt` in this repository |

Filtering only works for clients that actually use the Technitium servers. See `firewall-rules.md` for how DNS bypass is restricted.

---

## 2. Install the App

### On a clustered setup

Technitium's cluster synchronizes **Apps** (and their configuration) from the primary to secondary nodes. So:

1. Install and configure Advanced Blocking on the **primary** only.
2. Confirm the secondary receives it (Section 7).
3. Never edit the config on the secondary; changes there are overwritten.

### Steps

1. Log in to the primary's Technitium web interface.
2. Open **Apps**, then **App Store**.
3. Locate **Advanced Blocking** and click **Install**.
4. Open **Apps > Advanced Blocking > Config**.
5. Paste the configuration from Section 3, replace the placeholder IP, and click **Save**.

If a node is **not** in a cluster, repeat these steps on that node.

---

## 3. Configuration

The complete configuration is in [advanced-blocking-config.json](advanced-blocking-config.json).

To apply it:

1. Open the file and replace `192.168.XXX.XX` in `networkGroupMap` with the IP address of the client that belongs in `netgear-restricted`. That client needs a DHCP reservation so its address does not change (`pfSense-DHCP-and-DNS.md`).
2. In the Technitium web interface open **Apps > Advanced Blocking > Config** on the **primary**.
3. Paste the edited JSON and click **Save**.

Do not commit the version containing your real IP address to the repository; keep the placeholder there.

### Structure at a glance

The file has two parts: a map from clients to groups, and the groups themselves.

```json
{
  "enableBlocking": true,
  "blockAsNxDomain": true,
  "blockListUrlUpdateIntervalHours": 24,
  "networkGroupMap": {
    "192.168.XXX.XX": "netgear-restricted",
    "0.0.0.0/0": "everyone else",
    "[::]/0": "everyone else"
  },
  "groups": [
    { "name": "everyone else",      "blocked": [],              "allowListUrls": [...], "blockListUrls": [...] },
    { "name": "netgear-restricted", "blocked": ["netgear.com"], "allowListUrls": [...], "blockListUrls": [...] }
  ]
}
```

This excerpt is abbreviated and not valid to paste. Use the full file. Sections 4 to 6 explain each part.

---

## 4. Client Groups

Clients are matched to a group by source IP address through `networkGroupMap`. The most specific entry should apply, so the single-host entry takes precedence over the catch-all ranges. Confirm this in your Technitium version (Section 7).

| Network | Group | Purpose |
| --- | --- | --- |
| `192.168.XXX.XX` | `netgear-restricted` | One specific device; also blocks `netgear.com` |
| `0.0.0.0/0` | `everyone else` | All other IPv4 clients |
| `[::]/0` | `everyone else` | All other IPv6 clients |

To give other devices or VLANs stricter filtering, add a group with extra blocks and map the device address or VLAN range (CIDR) to it in `networkGroupMap`.

---

## 5. Block and Allow Lists

| List | Type | Used by | Purpose |
| --- | --- | --- | --- |
| [Hagezi Pro++](https://github.com/hagezi/dns-blocklists) (`pro.plus-onlydomains.txt`) | Block (domain list) | Both groups | Broad ads, trackers, malware, and related domains |
| [StevenBlack hosts, fakenews-gambling-porn](https://github.com/StevenBlack/hosts) | Block (hosts format) | Both groups | Adult content, gambling, and fake-news domains |
| `blocklist.txt` (this repo) | Block (domain list) | Both groups | Custom category blocks: dating, adult, social media, anonymity tools |
| `firefox-doh-blocklist.txt` (this repo) | Block (domain list) | Both groups | DoH resolver hostnames and the Firefox DoH canary domain |
| `allowlist.txt` (this repo) | Allow | Both groups | Domains that must stay reachable |

Allow-list entries take precedence over block-list entries in the same group, so keep allow-list entries as narrow as practical.

The `fakenews-gambling-porn` list is what provides broad adult-content coverage; `blocklist.txt` only adds a few specific sites on top.

---

## 6. Settings Explained

| Setting | Value | Meaning |
| --- | --- | --- |
| `enableBlocking` | `true` | Blocking is active |
| `blockAsNxDomain` | `true` | Blocked names return NXDOMAIN; `blockingAddresses` is not used while this is true |
| `blockingAnswerTtl` | `30` | Blocked answers are cached for 30 seconds, so allow-list changes take effect quickly |
| `blockListUrlUpdateIntervalHours` | `24` | Lists are re-downloaded daily |
| `allowTxtBlockingReport` | `true` | Blocked responses include an Extended DNS Error naming the group, which helps troubleshooting |
| `localEndPointGroupMap` | `{}` | Not used; groups are chosen by client IP |

The custom lists are served from the `main` branch of this repository. A commit becomes active at the next list refresh. Anyone who can write to the repository can change what is blocked, so protect the GitHub account with two-factor authentication.

---

## 7. Verify

### A blocked name is blocked

```
dig @192.168.XXX.XX pof.com A
```

Expected: `status: NXDOMAIN` and an Extended DNS Error similar to:

```
EDE: 15 (Blocked): (source=advanced-blocking-app; group=everyone else; domain=pof.com)
```

### An allowed name resolves

```
dig @192.168.XXX.XX discord.com A
```

Expected: `status: NOERROR` with an answer.

### The restricted group applies

From the mapped device, query `netgear.com` and confirm the EDE text shows `group=netgear-restricted`. From any other device it should resolve normally.

### Both nodes match

1. Open **Apps > Advanced Blocking > Config** on the primary and secondary.
2. Compare them (for example paste each into a file and run `diff`).
3. Run the `dig` tests against **each** server.

| Verification | Primary | Secondary |
| --- | --- | --- |
| App installed | NOT TESTED | NOT TESTED |
| Config matches this document | NOT TESTED | NOT TESTED |
| Blocked domain returns NXDOMAIN with EDE | NOT TESTED | NOT TESTED |
| Allowed domain resolves | NOT TESTED | NOT TESTED |
| Restricted-group device matched correctly | NOT TESTED | NOT TESTED |

Replace NOT TESTED with PASS or FAIL after running each test.

---

## 8. Day-to-Day Changes

- **Add an exception:** add the domain to `allowlist.txt` with a comment explaining why, commit it, and wait for the next refresh (or re-save the config to pick it up sooner; confirm this in your version).
- **Block something for everyone:** add it to `blocklist.txt` under the right heading.
- **Block something for one device or group:** use that group's `blocked` array in the config, on the primary.
- **Find out why a site is blocked:** see Troubleshooting - Identify Blocked DNS Queries in `HowToInstallTechnitium.md`.
- **After any config change:** export the config from Technitium, restore the placeholder IP, and replace `advanced-blocking-config.json` in the repository.

---

## 9. Changes From the Original Config

- `netgear.com` moved out of `blocklist.txt` and into the `netgear-restricted` group's `blocked` array, so only that device is affected instead of every client.
- The redundant manual `pof.com` entry in `everyone else` removed (it is already in `blocklist.txt`).

## 10. Open Items

- [ ] Confirm the primary's config matches this document and the secondary matches the primary.
- [ ] Give the device in `netgear-restricted` a DHCP reservation.
- [ ] Consider stricter groups for particular devices or VLANs.
- [ ] Review the optional entries in `blocklist.txt` and `firefox-doh-blocklist.txt` against your query logs.
- [ ] Narrow `netgear.com` to the specific hostnames seen in the query logs if the whole domain is too broad.
