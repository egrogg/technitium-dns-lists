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

## Step 1 — Decide on your Cluster Domain  
You need a domain name for the cluster. Since this is a private/home/internal DNS setup, you can use something like:
|||  
|-|-|  
| Item | Configuration |  
|Cluster domain |	dns.MYDOMAIN.com|  
|Primary cluster hostname |	primary.dns.MYDOMAIN.com|  
|Secondary cluster hostname |	secondary.dns.MYDOMAIN.com|  

You do not need to create these DNS records yourself. Technitium creates/manages them as part of the cluster.  

## Step 2 — Check the Primary first  

a. Log in to the primary DNS first  
b. Goto: Settings - General    
c. Look for: DNS Server Domain  
d. you can have it say: dns.MYDOMAIN.com  / then save
e. Goto: Administration - Cluster  
f. Then click: Initialize  
g. Select "New Cluster" (Join is for the secondary, we are setting up the primary)  
    1. Should show three fields:  
        a. Cluster Domain: dns.groggbible.com  
        b. Primary Node Address: 192.168.xxx.xx  
        c. Quick Add: Leave Blank  
    2. One important thing: The message at the bottom says that initializing the cluster will change the DNS Server Domain Name so the current hostname becomes a subdomain of the cluster domain. Technitium will determine the appropriate node name from the current hostname and update it as part of initialization. This is expected behavior.  
    3. Click "Initialize"  
        a. Once you click Initialize, wait for it to finish.  
        b. One other important point: Technitium says initialization automatically enables HTTPS with a self-signed certificate if HTTPS wasn't already configured. That's normal and part of how the cluster authenticates node-to-node communication.  
    4. In "Administration" / Cluster this should show the below:
        a. Node name: "dns.dns.MYDOMAIN.com"  
        b. IP Address: "192.168.xxx.xx" (This is the IP of your primary Domain)  
        c. URL: "https://dns.dns.MYDOMAIN.com: 53443/"  
        d. Type: "Primary"  
        e: State: "Self"  
        f: Up Since: shows a date/time group  
        g. Last seen: blank  
        h: Last synced: blank  
    5. Note: you can goto: Settings / General in the primary domain and change it to: "primary.dns.groggbible.com" if you wish but not mandatory, just makes it easier later on to determine which is which.  
    prim

















