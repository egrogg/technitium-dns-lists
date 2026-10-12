
# DNS Bypass Prevention — Technitium DNS

## Approved DNS Servers

Primary: 192.168.XXX.XX
Secondary: 192.168.XXX.XXX

## Firewall Aliases

#DoH_Servers — Host/IP alias
  This is optional but gets some of the DoH servers that are normally used.
  Create a Firewall Alias for IP list
  Name: DoH_Servers
  Description:  Known public DNS / DoH servers
  Type: Host(s)
  Host List to add:
      1.1.1.1
      1.0.0.1
      8.8.8.8
      8.8.4.4
      9.9.9.9
      149.112.112.112
      162.159.61.4
      172.64.41.4

#DoH_Blocklist — URL Table (IPs)
Create a Firewall Allias for URLs
    Name: DoH_BlockList
    Description: Automatically updated public DoH server IPv4 blocklist
    Type: URL Table (IPs)
    URL Table IPs: https://raw.githubusercontent.com/dibdot/DoH-IP-blocklists/master/doh-ipv4.txt / 1
            Note: /1 is the nubmer of days this will reach out to update.


## Firewall Rule Order — Each Client VLAN

1. PASS — Approved device exceptions, where required
2. PASS — VLAN net to 192.168.XXX.XX, TCP/UDP 53
3. PASS — VLAN net to 192.168.XXX.XX, TCP/UDP 53
4. PASS — 192.168.XXX.XX to VLAN net, TCP/UDP 53
5. PASS — 192.168.XXX.XX to VLAN net, TCP/UDP 53
6. REJECT — VLAN net to any other destination, TCP/UDP 53
7. REJECT — VLAN net to any destination, TCP/UDP 853
8. ## this one may interfere with PC's that game as they use som of the DoH_servers listed below REJECT — VLAN net to DoH_Servers, TCP 443
9. REJECT — VLAN net to DoH_Blocklist, TCP 443
10. PASS — Normal permitted traffic


## Important Exceptions

Technitium DNS servers must be permitted to
reach their configured upstream DNS services.

Do not apply client DNS restrictions to
Technitium's own upstream traffic.

Allow specific device exceptions only where needed.

## VLAN Configuration

Apply the DNS enforcement rules to every
client VLAN that requires DNS restrictions.

Configure DHCP to advertise both Technitium servers.

## Notes

Rule order matters.
Exceptions must appear before applicable blocks.

DoH IP blocking can affect other HTTPS services
sharing those IP addresses.

This setup reduces DNS bypass but does not
guarantee prevention of every DoH, VPN,
proxy, or encrypted DNS method.
