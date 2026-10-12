##Technitium DNS — Home Network Configuration**  
#Configuration notes based on the Technitium work completed in September–October 2026. \* **Verified changes** \* and \* **planned/recommended changes** \* are identified separately. This is a reconstruction, not an export of the live configuration.*  
**1. Environment**  
| | |  
|-|-|
| **Item**| **Configuration**|  
|DNS software|Technitium DNS Server 15.4|  
|Primary | 192.168.xxx.xx (Ubuntu Server)|  
|Secondary | 192.168.xxx.xx (Ubuntu Server)|  
|Cluster domain|dns.MYDOMAIN.com|  
|Primary cluster hostname |dns.dns.MYDOMAIN.com |  
| Secondary cluster hostname | ubuntu.dns.MYDOMAIN.com |  
| HTTP admin port | 5380 |  
| HTTPS admin port | 53443 |  
| Custom lists repository | [https://github.com/MYUSERNAME/technitium-dns-lists](https://github.com/egrogg/technitium-dns-lists) |  
 
##Verified:## The two Technitium servers were joined into a cluster. The primary showed connected and the secondary showed its secondary/self role.   
Cluster-related zones included dns.MYDOMAIN.com and cluster-catalog.dns.MYDOMAIN.com.  

**2. Cluster setup and troubleshooting**

1. Set up Technitium DNS Server 15.4 on both Ubuntu servers.

2. Created the cluster on the primary (192.168.xxx.xx) using the dns.MYDOMAIN.com cluster domain.

3. Joined the secondary (192.168.xxx.xx) to the primary.

4. Investigated a failed join caused by a DANE TLSA lookup/certificate validation error for dns.dns.MYDOMAIN.com:53443.

5. Confirmed the cluster hostname's DNS A record pointed to 192.168.xxx.xx.

6. Found HTTPS management on TCP 53443 reachable locally but initially not from another host. Added an appropriate inbound firewall allowance and verified remote connectivity.

7. The join was completed with **Ignore Certificate Validation Errors** after DANE validation failed. This was a workaround; proper certificate validation should be revisited.

8. Verified the secondary served replicated cluster records and recursive DNS responses.  
**Useful checks**

# Check name resolution against each node

 dig @192.168.xxx.xx dns.dns.MYDOMAIN.com A  
 dig @192.168.xxx.xx dns.dns.MYDOMAIN.com A  
   
 \# Test recursive resolution on both nodes  
 dig @192.168.xxx.xx example.com  
 dig @192.168.xxx.xx example.com  
   
 \# On a DNS server, inspect listening ports  
 sudo ss -lntup | grep -E ':(53|5380|53443)\\b'  
   
 \# Inspect local firewall configuration  
 sudo ufw status verbose  
 sudo iptables -S  
   
***Security note:*** \* Do not expose the admin ports \* *5380* \* or \* *53443* \* to untrusted networks. Prefer fixing trusted certificate validation over permanently ignoring validation errors.\*  
**3. Secondary DNSSEC / Chrony time fix**  
**Verified:** The secondary initially returned SERVFAIL for signed domains with an Extended DNS Error of  **Signature Not Yet Valid**. Its clock was not synchronized. NTS/TLS certificate checks also failed because the clock was far behind, creating a circular dependency.  
The working fix on the secondary included these Chrony settings:  
**/etc/chrony/chrony.conf**  
authselectmode ignore  
 makestep 1 3  
   
**/etc/chrony/sources.d/bootstrap-ip.sources**  
server 91.189.91.111 iburst trust  
 server 185.125.190.121 iburst trust  
   
These IP-based bootstrap sources allowed the system to correct its clock without first requiring working DNS or a valid TLS time check. The clock was corrected, and the regular Ubuntu time sources could then operate.  
Verification commands:  
timedatectl  
 chronyc tracking  
 chronyc sources -v  
 dig @192.168.xxx.xx google.com  
   
**Maintenance:** The bootstrap IP addresses are hard-coded and should be reviewed if they stop responding. Unauthenticated bootstrap time has security tradeoffs; protect access to these servers and prefer authenticated synchronization after bootstrapping.  
**4. Advanced Blocking**  
**Verified in October 2026:** Advanced Blocking was installed and active. The built-in Technitium blocking feature was disabled on the primary to avoid overlapping filtering methods. Both DNS servers were tested and returned blocked responses showing NXDOMAIN, with Extended DNS Error identifying advanced-blocking-app and the everyone else group.  
**GitHub-hosted lists**  
Main blocklist:  
[https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/blocklist.txt](https://raw.githubusercontent.com/egrogg/technitium-dns-lists/main/blocklist.txt)  
   
Allowlist:  
[https://raw.githubusercontent.com/MYUSERNAME/technitium-dns-lists/main/allowlist.txt](https://raw.githubusercontent.com/egrogg/technitium-dns-lists/main/allowlist.txt)  
   
The configuration used a **24-hour refresh** interval for lists and an everyone else default group covering IPv4/IPv6 clients.  
**Other external lists discussed**  
https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/wildcard/pro.plus-onlydomains.txt  
 [https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/porn/hosts](https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/porn/hosts)  
 [https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/fakenews-gambling-porn/hosts](https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/fakenews-gambling-porn/hosts)  
   
**Important:** These URLs were discussed as possible sources. Verify the current live Advanced Blocking configuration before treating every URL as enabled. The Hagezi wildcard list and StevenBlack hosts files use different formats; ensure the selected Advanced Blocking fields/parser support each format. Overlapping sources can cause redundant entries.  
**Single policy for everyone**  
After discussing separate restrictions for children and other clients, the preference was to use **one shared list for everyone** for now. The custom list discussions included dating apps and social platforms, such as Tinder, Plenty of Fish, Wizz, Grindr, OnlyFans, Snapchat, Reddit, Discord, Instagram, mail.ru, and ok.ru.  **Check \*\* blocklist.txt \*\* to see exactly which domains were ultimately committed; this document does not assert that every discussed domain is currently blocked.**  
**Netgear Wi-Fi device**  
The Wi-Fi device at 192.168.101.30 repeatedly queried netgear.com, prompting a request to block that domain. A netgear-restricted group was discussed, but the later stated preference was one list for everyone. Verify whether the separate group is still present before documenting it as active. Blocking the device's netgear.com lookups is distinct from exempting that device from filtering.  
**5. DNS client configuration and failover**  
For client networks intended to use both servers, advertise the following DNS servers through DHCP:  
192.168.xxx.xx  
 192.168.xxx.xx  
   
**Important:** A client configured with only .72 does  **not** automatically switch to .73 simply because the Technitium servers are clustered. Clients need both DNS server addresses (or a separate highly available virtual DNS address). Different operating systems choose between advertised resolvers differently, so test failover rather than assuming strict primary/secondary ordering.  
**6. DNS bypass prevention — network layer**  
**Earlier verified setup:** DNS enforcement was tested with AdGuard Home at 192.168.101.90, including allowing that server upstream DNS traffic while blocking direct client DNS to outside resolvers.  **The rules below are the updated Technitium design, not a verified export of your current firewall.**  
Recommended per-client-VLAN logic:

1. Allow clients to 192.168.xxx.xx on TCP/UDP port 53.

2. Allow clients to 192.168.xxx.xx on TCP/UDP port 53.

3. Reject client DNS to other destinations on TCP/UDP port 53.

4. Reject client DNS-over-TLS on TCP port 853, unless a specific exception is required.

5. Optionally block **verified** DoH resolver IP addresses on TCP port 443, taking care with shared hosting and application breakage.

6. Allow other normal permitted traffic.  
Permit Technitium servers to reach **their actual configured upstream resolvers**, whether using DNS port 53, DoT port 853, or DoH port 443. Place the relevant allowances before broad blocks, and account for rule direction, interface, and stateful firewall behavior.  
Previously discussed aliases included DoH\_Servers and a DoH\_Blocklist URL Table of approximately 1,968 IPv4 addresses. **Do not copy a historical count or alias as proof of the current list contents.** A GitHub DNS domain list cannot replace firewall IP-based enforcement.  
**Firefox and DoH**  
A DNS domain list can block some known DoH resolver names, but cannot guarantee all encrypted DNS bypass is prevented. Firefox may use other providers, and VPNs/proxies or direct IP connections can bypass DNS-domain filtering. On managed Firefox clients, a browser policy to disable DoH is more deterministic.  
**Do not block \*\* use-application-dns.net \*\* for the purpose of disabling Firefox DoH**; Firefox uses it as a network signal to disable default DoH behavior under applicable conditions.  
See [firewall-rules.md for your firewall reference and ](file:///home/ubuntu/Downloads/firewall-rules.md)[dns-exceptions.md for device exception notes.](file:///home/ubuntu/Downloads/dns-exceptions.md)  
**7. TVs, games, Discord, and exceptions**  
During earlier DNS enforcement testing, TVs lost connectivity, and some games and Discord had issues. Moving TV exception rules above the blocking rules helped in the prior firewall setup. The exact active rule set and root cause of each application issue have **not** been established for the Technitium migration.  
Troubleshooting sequence:

7. Check the client's configured DNS server(s).

8. Check Advanced Blocking logs for denied domains.

9. Check firewall logs for port 53, 853, and DoH-IP restrictions.

10. Test whether the problem occurs when using the approved Technitium servers.

11. Add narrow, documented exceptions only when evidence identifies a legitimate false positive.  
Allowing a device to query .72/.73 does **not** exempt it from Technitium's Advanced Blocking policy.  
**8. Validation checklist**  
Run from a **client VLAN** with DNS restrictions in place:

# Both approved DNS servers should answer

 dig @192.168.xxx.xx example.com  
 dig @192.168.xxx.xx example.com  
   
 \# Direct external DNS should be blocked if firewall enforcement is active  
 \# A timeout is expected in a DROP configuration; REJECT may fail immediately.  
 dig @8.8.8.8 example.com  
   
 \# Check the currently assigned DNS resolvers on Ubuntu  
 resolvectl status  
   
Also check the Technitium query log for blocked domains and the blocking-app identifier. Test IPv4 and IPv6 separately **if IPv6 is enabled**; this environment was intended to operate without IPv6 on affected clients. Test both UDP and TCP DNS separately. Verify the secondary still resolves queries when the primary is intentionally unavailable.  
**9. Repository structure**  
technitium-dns-lists/  
 ├── README.md  
 ├── blocklist.txt  
 ├── allowlist.txt  
 ├── TECHNITIUM-SETUP.md  
 ├── firewall-rules.md  
 ├── dns-exceptions.md  
 └── firefox-doh-blocklist.txt  \# only if created/committed  
   
\*The tree above is the intended organization, \* ***not a claim that every file is currently committed*** *.*  
**10. Outstanding checks**

- Confirm Advanced Blocking group configuration on both nodes and whether netgear-restricted still exists.

- Confirm exact external list URLs enabled in the live configuration.

- Confirm the contents of the committed GitHub blocklist and allowlist.

- Verify current firewall rules reference .72 and .73, not the older AdGuard .90 address.

- Test DHCP distribution and DNS failover on each client VLAN.

- Revisit certificate validation for the Technitium cluster connection.

- Confirm TV, gaming, and Discord connectivity after the Technitium cutover.  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAACXBIWXMAAA7EAAAPBgFtJdHgAAAAOElEQVRYhe3OwQmAMAwAwCgi3cJls4bLOUcgXaG/ItxNcEdmPlX1BgAAf/Fd3X1GxNg9AQBg2T0BsHMJMebSjScAAAAASUVORK5CYII=)  
*Last documented: October 12, 2026.*

