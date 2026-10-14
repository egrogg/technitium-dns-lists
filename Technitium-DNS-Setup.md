**Technitium DNS — Home Network Configuration**  
*Configuration notes based on the Technitium work completed in September–October 2026. \* **Verified changes** \* and \* **planned/recommended changes** \* are identified separately. This is a reconstruction, not an export of the live configuration.*  
**1. Environment**  
| | |  
|-|-|
| **Item**| **Configuration**|  
|DNS software|Technitium DNS Server 15.4|  
|Primary | 192.168.xxx.xx (Ubuntu Server)|  
|Secondary | 192.168.xxx.xx (Ubuntu Server)|  
|Cluster domain|dns.MYDOMAIN.com|  
|Primary cluster hostname |primary.dns.MYDOMAIN.com |  
| Secondary cluster hostname | secondary.dns.MYDOMAIN.com |  
| HTTP admin port | 5380 |  
| HTTPS admin port | 53443 |  
| Custom lists repository | [https://github.com/MYUSERNAME/technitium-dns-lists](https://github.com/egrogg/technitium-dns-lists) |  
 
**Verified:** The two Technitium servers were joined into a cluster. The primary showed connected and the secondary showed its secondary/self role.   
Cluster-related zones included dns.MYDOMAIN.com and cluster-catalog.dns.MYDOMAIN.com.  

**2. Cluster setup and troubleshooting**

1. Set up Technitium DNS Server 15.4 on both Ubuntu servers.

2. Created the cluster on the primary (192.168.xxx.xx) using the dns.MYDOMAIN.com cluster domain.

3. Joined the secondary (192.168.xxx.xx) to the primary.

4. Investigated a failed join caused by a DANE TLSA lookup/certificate validation error for primary.dns.MYDOMAIN.com:53443.

5. Confirmed the cluster hostname's DNS A record pointed to 192.168.xxx.xx.

6. Found HTTPS management on TCP 53443 reachable locally but initially not from another host. Added an appropriate inbound firewall allowance and verified remote connectivity.

7. The join was completed with **Ignore Certificate Validation Errors** after DANE validation failed. This was a workaround; proper certificate validation should be revisited.

8. Verified the secondary served replicated cluster records and recursive DNS responses.  
**Useful checks**


# Setup of Technitium
## 1. Setup two DNS servers and make them a cluster.
See Technitium to get the cluster working.  Listed below is some issues I experienced such as the time fix because I am using Raspberry PI.  


## 2. Node-Specific Resolution Check

### Check name resolution against each node


Use these commands to verify name resolution against each individual node:

```bash
# Check name resolution against each node
dig @192.168.xxx.xx dns.dns.MYDOMAIN.com A  
dig @192.168.xxx.xx dns.dns.MYDOMAIN.com A  
```

---

## 2a. Recursive Resolution Testing

Run these commands to verify that external recursion is working properly across both nodes:

```bash
# Test recursive resolution on both nodes
dig @192.168.xxx.xx example.com
dig @192.168.xxx.xx example.com
```

On a DNS server, inspect listening ports  
```bash
 sudo ss -lntup | grep -E ':(53|5380|53443)\\b'  
```

 ---
Inspect local firewall configuration  
```bash
 sudo ufw status verbose  
 sudo iptables -S
```

---
   
***Security note:*** \* Do not expose the admin ports \* *5380* \* or \* *53443* \* to untrusted networks. Prefer fixing trusted certificate validation over permanently ignoring validation errors.\*  

## 2b. Secondary DNSSEC / Chrony time fix**  
**Verified:** The secondary initially returned SERVFAIL for signed domains with an Extended DNS Error of  **Signature Not Yet Valid**. Its clock was not synchronized. NTS/TLS certificate checks also failed because the clock was far behind, creating a circular dependency.  
The working fix on the secondary included these Chrony settings:  
```bash
**/etc/chrony/chrony.conf**  
authselectmode ignore  
 makestep 1 3  
```

---
```bash
**/etc/chrony/sources.d/bootstrap-ip.sources**  
server 91.189.91.111 iburst trust  
server 185.125.190.121 iburst trust  
```

---
   
These IP-based bootstrap sources allowed the system to correct its clock without first requiring working DNS or a valid TLS time check. The clock was corrected, and the regular Ubuntu time sources could then operate.  
Verification commands:  
```bash
timedatectl  
chronyc tracking  
chronyc sources -v  
dig @192.168.xxx.xx google.com  
```

---   
**Maintenance:** The bootstrap IP addresses are hard-coded and should be reviewed if they stop responding. Unauthenticated bootstrap time has security tradeoffs; protect access to these servers and prefer authenticated synchronization after bootstrapping.  

## 2c. Advanced Blocking**  
Install Advanced Blocking under the apps tab. Note: This may install to the secondary since this is in a cluster, but if it doesn't install also on the secondary DNS

The built-in Technitium blocking feature was disabled on the primary and secondary to avoid overlapping filtering methods. Both DNS servers were tested and returned blocked responses showing NXDOMAIN, with Extended DNS Error identifying advanced-blocking-app and the everyone else group.  

**GitHub-hosted lists**  
Main blocklist:  
[https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/blocklist.txt](https://raw.githubusercontent.com/egrogg/technitium-dns-lists/main/blocklist.txt)  
   
Allowlist:  
[https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/allowlist.txt](https://raw.githubusercontent.com/egrogg/technitium-dns-lists/main/allowlist.txt)  
   
The configuration used a **24-hour refresh** interval for lists and an everyone else default group covering IPv4/IPv6 clients.  
**Other external lists discussed**  
 [https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/wildcard/pro.plus-onlydomains.txt]  
 [https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/firefox-doh-blocklist.txt](https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/firefox-doh-blocklist.txt)  
 [https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/porn/hosts](https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/porn/hosts)  
 [https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/fakenews-gambling-porn/hosts](https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/fakenews-gambling-porn/hosts)  
   
**Important:** These URLs were discussed as possible sources. Verify the current live Advanced Blocking configuration before treating every URL as enabled. The Hagezi wildcard list and StevenBlack hosts files use different formats; ensure the selected Advanced Blocking fields/parser support each format. Overlapping sources can cause redundant entries.  

**Single policy for everyone**  
After discussing separate restrictions for children and other clients, the preference was to use **one shared list for everyone** for now. The custom list discussions included dating apps and social platforms, such as Tinder, Plenty of Fish, Wizz, Grindr, OnlyFans, Snapchat, Reddit, Discord, Instagram, mail.ru, and ok.ru.  **Check \*\* blocklist.txt \*\* to see exactly which domains were ultimately committed; this document does not assert that every discussed domain is currently blocked.**  

**Netgear Wi-Fi device**  
The Wi-Fi device at 192.168.xxx.xx repeatedly queried netgear.com, prompting a request to block that domain. A netgear-restricted group was discussed, but the later stated preference was one list for everyone. Verify whether the separate group is still present before documenting it as active. Blocking the device's netgear.com lookups is distinct from exempting that device from filtering.  

### In the code below, please note that the netgear-restricted group contains 1 IP which is restricted to this group.  This is because I wanted my wifi to not reach out to netgear.com
```bash
{
  "enableBlocking": true,
  "blockingAnswerTtl": 30,
  "blockListUrlUpdateIntervalHours": 24,
  "blockListUrlUpdateIntervalMinutes": 0,
  "localEndPointGroupMap": {},
  "networkGroupMap": {
    "192.168.xxx.xx": "netgear-restricted",
    "0.0.0.0/0": "everyone else",
    "[::]/0": "everyone else"
  },
  "groups": [
    {
      "name": "everyone else",
      "enableBlocking": true,
      "allowTxtBlockingReport": true,
      "blockAsNxDomain": true,
      "blockingAddresses": [
        "0.0.0.0",
        "::"
      ],
      "allowed": [],
      "blocked": [
        "pof.com"
      ],
      "allowListUrls": [
        "https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/allowlist.txt"
      ],
      "blockListUrls": [
        "https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/wildcard/pro.plus-onlydomains.txt",
        "https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/fakenews-gambling-porn/hosts",
        "https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/firefox-doh-blocklist.txt",
        "https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/blocklist.txt"
      ],
      "allowedRegex": [],
      "blockedRegex": [],
      "regexAllowListUrls": [],
      "regexBlockListUrls": [],
      "adblockListUrls": []
    },
    {
      "name": "netgear-restricted",
      "enableBlocking": true,
      "allowTxtBlockingReport": true,
      "blockAsNxDomain": true,
      "blockingAddresses": [
        "0.0.0.0",
        "::"
      ],
      "allowed": [],
      "blocked": [
        "www.netgear.com",
        "pof.com"
      ],
      "allowListUrls": [
        "https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/allowlist.txt"
      ],
      "blockListUrls": [
        "https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/wildcard/pro.plus-onlydomains.txt",
        "https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/fakenews-gambling-porn/hosts",
        "https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/firefox-doh-blocklist.txt",
        "https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/blocklist.txt"
      ],
      "allowedRegex": [],
      "blockedRegex": [],
      "regexAllowListUrls": [],
      "regexBlockListUrls": [],
      "adblockListUrls": []
    }
  ]
}
```

---

## 3. DNS client configuration and failover**  
I intended to use both servers, so I am advertising the following DNS servers through DHCP:  
192.168.xxx.xx  
192.168.xxx.xx  
   
**Important:** A client configured with only the primary DNS does  **not** automatically switch to the secondary DNS simply because the Technitium servers are clustered. Clients need both DNS server addresses (or a separate highly available virtual DNS address). Different operating systems choose between advertised resolvers differently, so testing failover found that Linux and Windows will continue to work.  
What I did to test was turn off the primary and cleared my browser cookies, etc and went to a site.  I also tried using dig to see what came back.  If I disabled the primary, and nothing returned on the primary, but I got a response in the secondary, assumed the secondary took over.  Using a browser still returned the test site(s).  

```bash
# Test recursive resolution on both nodes
dig @192.168.xxx.xx example.com
dig @192.168.xxx.xx example.com
```

## 4. DNS bypass prevention — network layer**  

**Earlier verified setup:** DNS enforcement was tested with AdGuard Home at 192.168.xxx.xx, including allowing that server upstream DNS traffic while blocking direct client DNS to outside resolvers.  

**The rules below are the updated Technitium design, of my current firewall.**  

Recommended per-client-VLAN logic:  

1. Allow clients from LAN subnets to 192.168.xxx.xx on TCP/UDP port 53. --Primary DNS

2. Allow clients from LAN subnets to 192.168.xxx.xx on TCP/UDP port 853. --Primary DNS

3. Allow clients from LAN subnets to 192.168.xxx.xx on TCP/UDP port 53. -- Secondary DNS

4. Allow clients from LAN subnets to 192.168.xxx.xx on TCP/UDP port 853. -- Secondary DNS

5. Allow clients from 192.168.xxx.xx to Any on TCP/UDP port 53 -- Primary DNS

6. Allow clients from 192.168.xxx.xx to Any on TCP/UDP port 853 -- Primary DNS

7. Allow clients from 192.168.xxx.xx to Any on TCP/UDP port 53 -- Secondary DNS

8. Allow clients from 192.168.xxx.xx to Any on TCP/UDP port 853 -- Secondary DNS

9. Reject clients from LAN subnets to ANY on TCP/UDP port 53 

10. Reject clients from LAN subnets to ANY on TCP/UDP port 853 

Note: The below Blocked my TV's, games  

11. Block clients from LAN subnets to DoH Servers port 443 (HTTPS) -- This has an URL IP table alias that contains known DoH servers (IP addresses). This pulls from: https://raw.githubusercontent.com/dibdot/DoH-IP-blocklists/master/doh-ipv4.txt and updates every day "/1".  Since this is breaking my TV connection, Gaming PC's to connect, I have disabled it.  

12. Allow clients from LAN subnets to ANY on Any  


Rules 1-10 Permit Technitium servers to reach **their actual configured upstream resolvers**, whether using DNS port 53, DoT port 853. When implementing, place the relevant allowances before broad blocks, and account for rule direction, interface, and stateful firewall behavior.  

Previously discussed aliases included DoH Servers URL IP Table of approximately 1,968 IPv4 addresses. **Do not copy a historical count or alias as proof of the current list contents.** A GitHub DNS domain list cannot replace firewall IP-based enforcement.  

**Firefox and DoH**  
A DNS domain list can block some known DoH resolver names, but cannot guarantee all encrypted DNS bypass is prevented. Firefox may use other providers, and VPNs/proxies or direct IP connections can bypass DNS-domain filtering. On managed Firefox clients, a browser policy to disable DoH is more deterministic.  




## 5. Validation checklist**  
Run from a **client VLAN** with DNS restrictions in place:

# Both approved DNS servers should answer
```bash
 dig @192.168.xxx.xx example.com  
 dig @192.168.xxx.xx example.com  
```

---   
 \# Direct external DNS should be blocked if firewall enforcement is active  
 \# A timeout is expected in a DROP configuration; REJECT may fail immediately.  
```bash
dig @8.8.8.8 example.com  
```

---   
 \# Check the currently assigned DNS resolvers on Ubuntu  
 ```bash
 resolvectl status  
```

---
Also check the Technitium query log for blocked domains and the blocking-app identifier. Test IPv4 and IPv6 separately **if IPv6 is enabled**; this environment was intended to operate without IPv6 on affected clients. Test both UDP and TCP DNS separately. Verify the secondary still resolves queries when the primary is intentionally unavailable.  

## 7. Repository structure**  
technitium-dns-lists/  
 ├── README.md  
 ├── blocklist.txt  
 ├── allowlist.txt  
 ├── TECHNITIUM-SETUP.md  
 ├── firewall-rules.md  
 ├── dns-exceptions.md  
 └── firefox-doh-blocklist.txt  \# only if created/committed  
   
\*The tree above is the intended organization, \* ***not a claim that every file is currently committed*** *.*  

## 6. Outstanding checks**

- Confirm Advanced Blocking group configuration on both nodes and whether netgear-restricted still exists.

- Confirm exact external list URLs enabled in the live configuration.

- Confirm the contents of the committed GitHub blocklist and allowlist.

- Verify current firewall rules reference .xx and .xx, not the older AdGuard .90 address.

- Test DHCP distribution and DNS failover on each client VLAN.

- Revisit certificate validation for the Technitium cluster connection.

- Confirm TV, gaming, and Discord connectivity after the Technitium cutover.  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAACXBIWXMAAA7EAAAPBgFtJdHgAAAAOElEQVRYhe3OwQmAMAwAwCgi3cJls4bLOUcgXaG/ItxNcEdmPlX1BgAAf/Fd3X1GxNg9AQBg2T0BsHMJMebSjScAAAAASUVORK5CYII=)  
*Last documented: October 12, 2026.*

