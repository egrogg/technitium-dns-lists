
# Technitium DNS Security Configuration

Home network DNS filtering and DNS bypass prevention.

## DNS Infrastructure

Primary Technitium DNS: 192.168.XXX.XX
Secondary Technitium DNS: 192.168.XXX.XX

## Repository Contents

- blocklist.txt — Main DNS filtering list
- firefox-doh-blocklist.txt — DoH resolver domains
- firewall-rules.md — Firewall DNS enforcement
- vlan-dns-enforcement.md — VLAN configuration
- dns-exceptions.md — Device exceptions

## Security Goals

- Centralized DNS filtering
- Primary and secondary DNS availability
- Block unauthorized external DNS
- Restrict DNS-over-TLS
- Reduce browser DNS-over-HTTPS bypass
- Preserve compatibility with gaming and TVs

## Deployment

1. Configure DHCP with both Technitium DNS servers.
2. Add the DNS domain blocklists to Technitium.
3. Configure firewall aliases.
4. Apply firewall rules to client VLANs.
5. Add tested device exceptions as needed.
6. Verify DNS resolution and application access.

## Notes

Firewall configurations are documented here.
They are not automatically installed by Technitium.
