
# DNS Bypass Prevention — Technitium DNS

## Approved DNS Servers

Primary: 192.168.XXX.XX
Secondary: 192.168.XXX.XXX

### Firewall Aliases

####DoH_Servers — Host/IP alias
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

####DoH_Blocklist — URL Table (IPs)
Create a Firewall Allias for URLs
    Name: DoH_BlockList
    Description: Automatically updated public DoH server IPv4 blocklist
    Type: URL Table (IPs)
    URL Table IPs: https://raw.githubusercontent.com/dibdot/DoH-IP-blocklists/master/doh-ipv4.txt / 1
            Note: /1 is the number of days this will reach out to update.


## Firewall Rule Order — Each Client VLAN

1. PASS — Approved device exceptions, where required
2. PASS — VLAN net to 192.168.XXX.XX, TCP/UDP 53
3. PASS — VLAN net to 192.168.XXX.XX, TCP/UDP 53
4. PASS — 192.168.XXX.XX to VLAN net, TCP/UDP 53
5. PASS — 192.168.XXX.XX to VLAN net, TCP/UDP 53
6. REJECT — VLAN net to any other destination, TCP/UDP 53
7. REJECT — VLAN net to any destination, TCP/UDP 853
   
Number 9 below may interfere with PC's that game as they use som of the DoH_servers listed below REJECT — VLAN net to DoH_Servers, TCP 443

9. REJECT — VLAN net to DoH_Blocklist, TCP 443  (Note: this is optional, this may result in blocking TV's, gaming, and other IoT items)
10. PASS — Normal permitted traffic


### Important Exceptions

Do not apply client DNS restrictions to Technitium's own upstream traffic. Technitium DNS servers must be permitted to
reach their configured upstream DNS services. This is added in the firewall rule(s) above (4 and 5)



Optional: Allow specific device exceptions only where needed. as rule 9 may block certain systems as noted, but you can put them on a different vLAN without rule 9. This includes different WIFI that is on a different vLAN

### VLAN Configuration

Apply the DNS enforcement rules to every
client VLAN that requires DNS restrictions.

Configure DHCP to advertise both Technitium servers.

## Notes

Rule order matters.
Exceptions must appear before applicable blocks.

This setup reduces DNS bypass but does not
guarantee prevention of every DoH, VPN,
proxy, or encrypted DNS method.
