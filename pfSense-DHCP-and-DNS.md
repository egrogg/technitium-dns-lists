# pfSense - DHCP and DNS Settings

How to give every client both Technitium servers, keep the servers on stable addresses, and make sure pfSense's own resolver does not undercut the filtering.

Menu names are from recent pfSense versions and may differ slightly in yours.

---

## 1. DHCP Reservations for the DNS Servers

Each Technitium server needs a fixed address.

1. Find each server's MAC address:
   - Proxmox VM: **VM > Hardware > Network Device**, or `ip link show` inside the VM.
   - Raspberry Pi: `ip link show` (use the `eth0` MAC).
2. In pfSense open **Services > DHCP Server > [interface of the server VLAN]**.
3. Scroll to **DHCP Static Mappings** and click **Add**.
4. Enter the MAC address, the IP address, and a hostname/description (`technitium-primary`, `technitium-secondary`).
5. Save, apply changes, then renew the lease on the server (or reboot it).
6. Confirm with `ip -br addr` that the server has the reserved address.

Keep the reserved addresses **outside** the dynamic DHCP pool.

---

## 2. Hand Out Both DNS Servers

Do this for **every** client VLAN.

1. **Services > DHCP Server > [client VLAN]**.
2. In **DNS Servers**, enter the primary in the first field and the secondary in the second.
3. Save and apply.

If these fields are left blank, pfSense hands out its own address as the DNS server, which sends clients to the pfSense resolver instead of Technitium.

Existing clients keep their old DNS settings until their lease renews. Renew a lease to test:

```
# Linux
sudo dhclient -r && sudo dhclient     # or reconnect the network
resolvectl status

# Windows
ipconfig /renew
ipconfig /all
```

Confirm each client lists **both** Technitium addresses.

---

## 3. Clients Mapped in Advanced Blocking

Advanced Blocking maps groups by client IP address. Any device that is mapped by address (see [AdvancedBlocking.md](AdvancedBlocking.md)) needs a DHCP static mapping, or its address can change and it will silently fall into the default group.

---

## 4. pfSense's Own DNS Resolver

pfSense runs a DNS Resolver (Unbound) by default. Make sure clients do not use it.

1. **Services > DNS Resolver**: either disable it, or limit **Network Interfaces** to Localhost so it does not answer clients.
2. **System > General Setup > DNS Servers**: point pfSense itself at the Technitium servers so its own lookups use the same filtering.
3. Confirm each VLAN's DHCP DNS fields list Technitium (Section 2).

> **Caution:** if you disable the resolver and both Technitium servers are down, pfSense and every client lose DNS. See the emergency steps in [BackupAndRecovery.md](BackupAndRecovery.md).

---

## 5. IPv6

If a VLAN uses IPv6, clients can learn DNS servers from router advertisements or DHCPv6 and bypass the IPv4 settings above.

- Either disable IPv6 on client VLANs, or
- Set the DNS servers in **Services > DHCPv6 Server & RA** to the Technitium servers' IPv6 addresses and mirror the firewall rules for IPv6 (see [firewall-rules.md](firewall-rules.md)).

---

## 6. Verification

| Verification | Result |
| --- | --- |
| Both servers have DHCP reservations | NOT TESTED |
| Each client VLAN hands out both DNS servers | NOT TESTED |
| A renewed client lists both servers | NOT TESTED |
| pfSense resolver does not answer clients | NOT TESTED |
| Mapped devices have static mappings | NOT TESTED |
| IPv6 handled (disabled or advertised correctly) | NOT TESTED |

Per-VLAN checklist:

| VLAN | DNS 1 | DNS 2 | Rules applied | Tested |
| --- | --- | --- | --- | --- |
| (example) Trusted | Primary | Secondary | Yes | NOT TESTED |
