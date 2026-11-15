# DNS Bypass Prevention - pfSense and Technitium

This guide restricts DNS bypass on **pfSense** so that client VLANs can only use the approved Technitium servers. It is applied manually; Technitium does not manage firewall rules.

Rules reduce bypass but cannot guarantee that every DoH, VPN, proxy, or encrypted-DNS method is blocked.

---

## 1. Approved DNS Servers

| Role | Address |
| --- | --- |
| Primary | `192.168.XXX.XX` |
| Secondary | `192.168.XXX.XX` |

Both servers must be handed out by DHCP. See [pfSense-DHCP-and-DNS.md](pfSense-DHCP-and-DNS.md).

---

## 2. Firewall Aliases

Create these under **Firewall > Aliases**.

| Alias | Type | Contents | Purpose |
| --- | --- | --- | --- |
| `Technitium_DNS` | Host(s) | Primary and secondary IPs | The only allowed resolvers |
| `AdGuard_DoT` | Host(s) | `94.140.14.14`, `94.140.15.15` | Technitium's upstream DNS-over-TLS forwarders |
| `DoH_Servers` | Host(s) | `1.1.1.1`, `1.0.0.1`, `8.8.8.8`, `8.8.4.4`, `9.9.9.9`, `149.112.112.112`, `162.159.61.4`, `172.64.41.4` | Well-known public resolvers (optional but useful) |
| `DoH_Blocklist_v4` | URL Table (IPs) | `https://raw.githubusercontent.com/dibdot/DoH-IP-blocklists/master/doh-ipv4.txt`, update frequency 1 day | Auto-updated public DoH server IPs (optional) |
| `DoH_Blocklist_v6` | URL Table (IPs) | The matching IPv6 list from the same project (confirm the file name before using it) | IPv6 equivalent (optional) |

---

## 3. Where the Technitium Servers Live

**Option A (recommended):** put the Technitium servers on a VLAN that has **no** DNS bypass rules. Their upstream traffic (DNS-over-TLS to AdGuard, NTP/NTS time sync) then needs no special handling.

**Option B:** if the servers are on a restricted VLAN, add these rules at the top of that VLAN's rule list:

| # | Action | Protocol | Source | Destination | Port |
| --- | --- | --- | --- | --- | --- |
| B1 | PASS | TCP | `Technitium_DNS` | `AdGuard_DoT` | 853 |
| B2 | PASS | UDP | `Technitium_DNS` | any | 123 (NTP) |
| B3 | PASS | TCP | `Technitium_DNS` | any | 4460 (NTS key exchange) |
| B4 | PASS | TCP/UDP | `Technitium_DNS` | any | 53 (only if you resolve recursively instead of forwarding) |

pfSense is stateful, so replies to allowed queries need no extra rules.

---

## 4. Rule Order for Each Client VLAN

Add these under **Firewall > Rules > [VLAN interface]**. Rules are evaluated top to bottom and the first match wins, so exceptions must come first.

| # | Action | Protocol | Source | Destination | Port | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | PASS | as needed | specific device or alias | the specific destination it needs | as needed | Approved device exceptions, only where required. One rule per exception with a description. |
| 2 | PASS | TCP/UDP | VLAN net | `Technitium_DNS` | 53 | Allow the approved resolvers |
| 3 | REJECT | TCP/UDP | VLAN net | any | 53 | Block every other DNS server |
| 4 | REJECT | TCP/UDP | VLAN net | any | 853 | DNS-over-TLS (TCP) and DNS-over-QUIC (UDP) |
| 5 | REJECT | TCP/UDP | VLAN net | `DoH_Servers` | 443 | DoH to well-known resolvers (UDP covers HTTP/3) |
| 6 | REJECT | TCP/UDP | VLAN net | `DoH_Blocklist_v4` | 443 | Optional. Can break TVs, consoles, and IoT devices. |
| 7 | PASS | any | VLAN net | any | any | Normal traffic |

Use **Reject** instead of **Block** for DNS rules so clients get an immediate failure instead of waiting for a timeout.

### Rules 5 and 6 can break devices

Some gaming PCs and consoles use addresses in `DoH_Servers`, and rule 6 can hit TVs and IoT devices. If a device breaks:

1. Confirm the cause in **Status > System Logs > Firewall**.
2. Add a narrow rule 1 exception for that device.
3. Or move the device to a separate VLAN or Wi-Fi network that does not have rules 5 and 6.

### IPv6

Either disable IPv6 on the client VLANs, or repeat rules 2-6 for IPv6 (using the Technitium servers' IPv6 addresses and `DoH_Blocklist_v6`). IPv6 left open is an easy way around IPv4-only rules.

---

## 5. Applying the Rules to Many VLANs

You can repeat the rules on each VLAN, or use a floating rule or an interface group so new VLANs are not missed.

> **Ordering caveat:** floating rules with **Quick** enabled are evaluated *before* the rules on an interface tab. If you move the block rules to a floating rule, put the approved exceptions in the floating list **above** them, or the interface-level exceptions will never match. Per-VLAN rules are simpler to reason about; use floating or group rules only if you are comfortable with this behavior.

---

## 6. Optional: Redirect Port 53 Instead of Rejecting

A NAT port forward can redirect stray port-53 queries to Technitium so devices with hardcoded DNS still work, but through your filter.

- **Firewall > NAT > Port Forward**: interface = client VLAN, protocol TCP/UDP, destination = invert match `Technitium_DNS`, destination port 53, redirect target = primary DNS, port 53.
- **Limits:** a redirect has one target, so redirected queries do not get secondary-server redundancy. If clients and the DNS server share a subnet, the redirect can cause return-path problems that need extra NAT settings. Test with one device first.
- Keep the REJECT rules as a safety net.

---

## 7. pfSense's Own Resolver

If pfSense's DNS Resolver or Forwarder is enabled and handed out by DHCP, clients can use it instead of Technitium. See [pfSense-DHCP-and-DNS.md](pfSense-DHCP-and-DNS.md).

---

## 8. Verify

From a client on a restricted VLAN:

```
dig @8.8.8.8 example.com A                      # expect refused / no answer
dig @192.168.XXX.XX example.com A               # expect NOERROR (primary)
dig @192.168.XXX.XX example.com A               # expect NOERROR (secondary)
openssl s_client -connect 1.1.1.1:853 -brief </dev/null   # expect failure (DoT blocked)
```

In Firefox, **Settings > Privacy & Security > DNS over HTTPS** should show that it is off when the Firefox canary domain is blocked (see [firefox-doh-blocklist.txt](firefox-doh-blocklist.txt)).

Check **Status > System Logs > Firewall** to confirm the reject rules are matching, and look for unexpected hits after a rule change.

| Verification | Result |
| --- | --- |
| Direct DNS to an outside resolver is blocked | NOT TESTED |
| Both Technitium servers answer from the client VLAN | NOT TESTED |
| DNS-over-TLS to an outside resolver is blocked | NOT TESTED |
| Browser DoH is off or blocked | NOT TESTED |
| TVs, consoles, and gaming PCs work | NOT TESTED |
| IPv6 handled (disabled or mirrored rules) | NOT TESTED |

---

## 9. Device Exceptions

Record every exception here so it can be reviewed later.

| Device | VLAN | Rule | Reason | Date added |
| --- | --- | --- | --- | --- |
| (example) Gaming PC | Gaming | PASS to `DoH_Servers` TCP 443 | Game launcher needs 1.1.1.1 | YYYY-MM-DD |

---

## 10. What These Rules Do Not Stop

- VPNs, proxies, and iCloud Private Relay
- Tor (DNS filtering and these rules do not block Tor traffic; a pfSense package such as pfBlockerNG with a Tor feed is the usual approach)
- DoH to servers that are in neither alias
- Devices on networks without these rules

Rule order matters. Exceptions must appear before the blocks they override.
