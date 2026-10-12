# Setting up DNS Clustering with Technitium

Technitium's clustering is designed around one Primary node and one or more Secondary nodes.  
---
The primary holds the cluster configuration, and the secondary synchronizes from it.  
---
Both servers continue functioning independently as DNS servers, so this is not a "single DNS server with a backup"—both can answer DNS queries.
---

The setup we'll build  
1. LAN (Note this can be any vLAN you want but in home use, normally there is only one network--a flat one, but this can apply to multiple vLANS, just populate this across each vLAN to hand out the DNS server(s) to each vLAN.
2. Primary DNS
3. Secondary DNS
4. DNS Clients  

The important part is that your DHCP server/router should ultimately hand out both DNS addresses:  
```bash
DNS 1 = 192.168.xxx.xx
DNS 2 = 192.168.xxx.xx
```  
My install consists of a Virtual Machine on Proxmox installed as:  
1. Ubuntu server  
2. Raspberry Pi 4

One important distinction: clustering synchronizes the Technitium configuration, while putting both DNS IPs into your DHCP/client configuration provides the actual client-side redundancy. The two servers still operate independently as DNS servers.  

How Technitium clustering works  
Technitium's cluster has:  
  - Primary node — where you make cluster-wide configuration changes.  
  - Secondary node(s) — synchronize those changes from the primary.  
  - Cluster Catalog — lets you specify zones that should exist on all cluster nodes.  
  - Common configuration such as allowed/blocked lists and settings can be synchronized.  
  - Cache and logging remain node-specific.  
  - DHCP scopes are not currently part of clustering, so those need separate consideration.

Since I am writing this now, hopefully you are downloading and installing version than 15.4 or newer. Technitium clustering is available in v15.4, and the nodes remain independently functioning DNS servers while the cluster synchronizes configuration.  

Step 1 — Decide on your Cluster Domain
