# How to Install Technitium DNS Server

This guide provides step-by-step instructions for installing Technitium DNS Server on two platforms:

1. Proxmox VE using an Ubuntu Server virtual machine (primary DNS server).
2. Raspberry Pi 4 using Ubuntu Server (secondary DNS server).
3. Configuring DHCP reservations and DNS hand-out on pfSense (see [pfSense-DHCP-and-DNS.md](pfSense-DHCP-and-DNS.md)).

### Related Guides

| Guide | Purpose |
| --- | --- |
| [pfSense-DHCP-and-DNS.md](pfSense-DHCP-and-DNS.md) | DHCP reservations, handing out both DNS servers, pfSense's own resolver |
| [Technitium-DNS-Cluster-Setup.md](Technitium-DNS-Cluster-Setup.md) | Joining the two servers into a cluster |
| [AdvancedBlocking.md](AdvancedBlocking.md) | DNS filtering with the Advanced Blocking app |
| [firewall-rules.md](firewall-rules.md) | pfSense rules that restrict DNS bypass |
| [BackupAndRecovery.md](BackupAndRecovery.md) | Backups and recovery |
| [Maintenance-and-Hardening.md](Maintenance-and-Hardening.md) | Updates, host firewall (UFW), HTTPS admin, SSH, routine checks |

## Document Conventions

- **Placeholders:** IP addresses appear as `192.168.XXX.XX`. Replace them with your own. Where a command is run against both servers, a `# Primary` or `# Secondary` comment shows which address to use. **Never run the primary and secondary tests against the same address.**
- **Test results:** record PASS, FAIL, or NOT TESTED. A test that has not been run is never PASS.
- **Versions:** use the same Ubuntu Server LTS release on both servers and record what you used.

| Item | Version used |
| --- | --- |
| Ubuntu Server LTS (both servers) | _(record the release)_ |
| Technitium DNS Server | _(record the version)_ |

## Network Configuration and IP Addresses

This guide uses the following network configuration as an example.

| Device | Role | Example IP Address |
| --- | --- | --- |
| Proxmox VM | Primary Technitium DNS | 192.168.XXX.XX |
| Raspberry Pi 4 | Secondary Technitium DNS | 192.168.XXX.XX |
| Network Gateway | pfSense router / firewall / DHCP | 192.168.XXX.XX |

> **Important:** The IP addresses shown throughout this guide are illustrative, not the addresses of a real home network.
>
> Replace them with addresses from your own network. For example, if your network uses `10.0.0.0/24`, you might use `10.0.0.10` for the primary and `10.0.0.11` for the secondary.
>
> Ensure that both addresses are available and do not conflict with other devices.

### DHCP Reservations

DHCP reservations are configured on the network gateway (pfSense) so each DNS server always receives the same IP address. A DHCP reservation associates a device's MAC address with a specific IP address, so the server keeps using DHCP but always receives the same address.

> **Note:** A DHCP reservation is different from manually configuring a static IP address in Ubuntu. Either method gives a consistent address, but the configuration is different.

Step-by-step pfSense instructions, including handing out both DNS servers to clients, are in [pfSense-DHCP-and-DNS.md](pfSense-DHCP-and-DNS.md).

---

## Table of Contents

- [Part 1 - Proxmox VM Installation](#part-1---proxmox-vm-installation)
- [Part 2 - Raspberry Pi 4 Installation](#part-2---raspberry-pi-4-installation)
- [Part 2A - Raspberry Pi 4 Time Synchronization (Optional)](#part-2a---raspberry-pi-4-time-synchronization-optional)
- [Part 3 - Initial Technitium Configuration](#part-3---initial-technitium-configuration)
- [Part 4 - Verify DNS Resolution](#part-4---verify-dns-resolution)
- [Part 5 - Troubleshooting](#part-5---troubleshooting)
- [Part 6 - Tested Ubuntu Security Configuration](#part-6---tested-ubuntu-security-configuration)

---

# Part 1 - Proxmox VM Installation

## Step 1 - Download Ubuntu Server

Download the Ubuntu Server LTS ISO. Use the same release on the Raspberry Pi so both servers behave the same way (see Document Conventions).

**Download:** [https://ubuntu.com/download/server](https://ubuntu.com/download/server)

## Step 2 - Upload Ubuntu Server ISO to Proxmox

1. Log in to the Proxmox web interface.
2. Select your Proxmox node.
3. Select **local (proxmox)**.
4. Click **ISO Images**.
5. Click **Upload**.
6. Select the Ubuntu Server ISO.
7. Click **Upload**.

## Step 3 - Create the Virtual Machine

1. Click **Create VM**.
2. Enter the VM name.
3. Select the Ubuntu Server ISO.
4. Configure the virtual machine hardware.

### Recommended VM Configuration

| Setting | Configuration |
| --- | --- |
| VM Name | technitium-dns |
| Operating System | Ubuntu Server LTS |
| CPU | 2 Cores |
| Memory | 4096 MB |
| Disk | 32 GB |
| Network Model | VirtIO |
| Network Bridge | vmbr0 |
| Start at boot | Enabled (VM > Options, after creation) |
| QEMU Guest Agent | Enabled after installing `qemu-guest-agent` (see [BackupAndRecovery.md](BackupAndRecovery.md)) |

> **Note:** These are suggested starting resources. Adjust them for your environment.

## Step 4 - Install Ubuntu Server

1. Start the virtual machine.
2. Open the Proxmox console.
3. Follow the Ubuntu Server installation wizard.
4. Configure the server hostname.
5. Create an administrator account.
6. Install OpenSSH Server.
7. Complete the installation.
8. Reboot the virtual machine.

## Step 5 - Update Ubuntu Server

Log in to Ubuntu Server.

Update the package repositories:

```
sudo apt update
```

Upgrade installed packages:

```
sudo apt upgrade -y
```

Identify the current IP address and MAC address. If using a DHCP reservation, create it on pfSense ([pfSense-DHCP-and-DNS.md](pfSense-DHCP-and-DNS.md)). A DHCP reservation is not the same as setting a static IP inside Ubuntu.

```
ip a
```

To identify the network interface and MAC address more clearly:

```
ip -br addr
ip link show
```

After creating the reservation, renew the DHCP lease or reboot to request the reserved IP:

```
sudo reboot
```

## Step 6 - Install Technitium DNS Server

Run the official Technitium installation script:

```
curl -sSL https://download.technitium.com/dns/install.sh | sudo bash
```

Follow the installation prompts.

> **Tip:** Piping a script straight into `sudo bash` runs it without reviewing it. To read the script first, download it, inspect it, then run it:
>
> ```
> curl -sSLO https://download.technitium.com/dns/install.sh
> less install.sh
> sudo bash install.sh
> ```

## Step 7 - Verify Technitium Service

Check whether Technitium DNS is running:

```
sudo systemctl status dns.service
```

The expected service status is:

```
Active: active (running)
```

> **Note:** After a fresh installation or reboot, Ubuntu Server may initially show `System clock synchronized: no`. This is normal while the NTP service establishes synchronization. Wait a few minutes and run `timedatectl` again.

```
timedatectl
```

Before proceeding with Technitium configuration, verify that:

- `System clock synchronized: yes`
- `NTP service: active`

Accurate time is important for DNSSEC validation and Technitium cluster communication. If either still reports `no` after a few minutes, work through Steps 7.1 to 7.4 below.

### Step 7.1 - Identify the time synchronization service

Ubuntu Server may use either `systemd-timesyncd` or `chrony`.

```
systemctl is-active chrony
systemctl is-active systemd-timesyncd
```

Example:

```
active
inactive
```

This indicates that Chrony is running. If the results are reversed, `systemd-timesyncd` is running instead.

> **Important:** You only need one active time synchronization service.

### Step 7.2 - Check the synchronization service

If using Chrony:

```
chronyc tracking
```

Check the configured NTP sources:

```
chronyc sources -v
```

Look for a source marked `^*`, which is the currently selected synchronization source.

If using `systemd-timesyncd`:

```
timedatectl timesync-status
```

Check the service:

```
systemctl status systemd-timesyncd
```

### Step 7.3 - Restart the time synchronization service

If synchronization is not working, restart the active service.

For Chrony:

```
sudo systemctl restart chrony
```

For `systemd-timesyncd`:

```
sudo systemctl restart systemd-timesyncd
```

Wait a minute or two, then check:

```
timedatectl
```

The expected result is `System clock synchronized: yes`. If it still says `no`, continue to Step 7.4.

### Step 7.4 - Check DNS and firewall connectivity

> **Note:** If a host firewall was turned on and rules were added, temporarily turning it off and checking again may fix the problem. If it does, review the firewall rules.

If restarting the service doesn't resolve the problem, check whether the server can resolve NTP hostnames:

```
resolvectl query ntp.ubuntu.com
```

Check whether the host firewall allows outbound NTP traffic:

```
sudo ufw status verbose
```

| Protocol | Port | Purpose |
| --- | --- | --- |
| UDP | 123 | NTP time synchronization |
| TCP | 4460 | Network Time Security (NTS) key exchange |

Ubuntu 26.04 uses Chrony by default, with NTS-enabled time sources, so both ports may be needed. Older releases that default to `systemd-timesyncd` need only UDP 123.

Also check the gateway: in pfSense, review **Status > System Logs > Firewall** to see whether outbound NTP (UDP 123) or NTS (TCP 4460) from the DNS server is being blocked. See [firewall-rules.md](firewall-rules.md) for the rules to allow if the DNS servers sit on a restricted VLAN.

Check once more after any firewall rule has been changed. If it still does not work, see Part 5 - Troubleshooting for links to Ubuntu help.

---

## Step 8 - Access the Technitium Web Interface

Open a web browser and go to the primary DNS server:

```
http://192.168.XXX.XX:5380
```

Replace the IP address with your Technitium server's address.

Complete the initial setup wizard.

---

# Part 2 - Raspberry Pi 4 Installation

## Step 1 - Download Raspberry Pi Imager

Download Raspberry Pi Imager:

[https://www.raspberrypi.com/software/](https://www.raspberrypi.com/software/)

## Step 2 - Prepare the Raspberry Pi

1. Insert the microSD card into your computer.
2. Launch **Raspberry Pi Imager**.
3. Under **Raspberry Pi Device**, select **Raspberry Pi 4**.
4. Under **Operating System**, select:
   - **Other general-purpose OS**
   - **Ubuntu**
   - **Ubuntu Server LTS (64-bit)**, the **same release** you used on the Proxmox VM (see Document Conventions)
5. Under **Storage**, select the microSD card.
6. Click **Next**.
7. If prompted, select **Edit Settings** or **OS Customisation** to configure:
   - **Hostname:** `technitium-secondary`
   - **Username:** Create an administrator username.
   - **Password:** Create a strong password.
   - **SSH:** Enable SSH for remote administration.
   - **SSH Authentication:** Prefer public-key authentication.
   - **Wi-Fi:** Configure only if not using Ethernet.
8. Save the customization settings, if available.
9. Confirm the selected storage device.
10. Click **Write** to install Ubuntu Server onto the microSD card.
11. Wait for the writing and verification process to complete.
12. Safely remove the microSD card.

> **Storage note:** SQLite query logging (Part 3, Step 6) writes continuously, which wears a microSD card. See [Maintenance-and-Hardening.md](Maintenance-and-Hardening.md) for options such as booting from a USB SSD.

## Step 3 - Boot the Raspberry Pi

1. Insert the microSD card into the Raspberry Pi.
2. Connect the Raspberry Pi to the network.
3. Connect power.
4. Wait for the Raspberry Pi to boot.

## Step 4 - Connect Using SSH

From another Linux computer (this is the secondary Technitium DNS server):

```
ssh username@192.168.XXX.XX
```

Replace the username and IP address with your configuration. Create a DHCP reservation for the Pi as well ([pfSense-DHCP-and-DNS.md](pfSense-DHCP-and-DNS.md)).

## Step 5 - Update Ubuntu Server

Update the package repositories:

```
sudo apt update
```

Upgrade installed packages:

```
sudo apt upgrade -y
```

## Step 6 - Install Technitium DNS Server

Run:

```
curl -sSL https://download.technitium.com/dns/install.sh | sudo bash
```

To review the script before running it, see the tip in Part 1, Step 6.

## Step 7 - Verify Technitium Service

```
sudo systemctl status dns.service
```

Expected result:

```
Active: active (running)
```

## Step 8 - Access the Web Interface

Open (this is the secondary Technitium DNS server):

```
http://192.168.XXX.XX:5380
```

Complete the initial setup wizard.

---

# Part 2A - Raspberry Pi 4 Time Synchronization (Optional)

## Overview

> **Optional troubleshooting procedure:** This section records a specific time synchronization problem encountered during the original installation. First check whether the default Ubuntu time synchronization works after a reboot. Apply the workaround only if needed, after considering the security implications of unauthenticated NTP bootstrap sources and the `trust` directive.

The Raspberry Pi 4 does not include a battery-backed real-time clock (RTC). When it loses power or reboots, Ubuntu Server may start with an incorrect system date and time. This can cause problems with Technitium DNS Server, particularly when DNSSEC validation is enabled.

### Environment

| Component | Configuration |
| --- | --- |
| Device | Raspberry Pi 4 |
| Operating System | Ubuntu Server |
| DNS Server | Technitium DNS |
| Server Role | Secondary DNS |
| IP Address | 192.168.XXX.XX |
| Time Synchronization | Chrony |

---

## Problem

After rebooting, the Raspberry Pi started with an incorrect system clock. Chrony was configured to use authenticated Network Time Security (NTS) servers. Because the system clock was incorrect, TLS certificate validation failed.

The resulting sequence was:

```
Raspberry Pi reboots
        |
        v
System clock is incorrect
        |
        v
NTS certificate validation fails
        |
        v
Chrony cannot synchronize
        |
        v
System clock remains incorrect
        |
        v
DNSSEC validation fails
        |
        v
Technitium returns SERVFAIL
```

Technitium DNS reported:

```
Signature Not Yet Valid
```

---

## Solution - Configure Chrony Bootstrap Sources

The following configuration was used to resolve the boot-time synchronization issue.

> **Security Note:** The bootstrap sources below use unauthenticated NTP. The `trust` directive explicitly trusts these sources, so their integrity is important. This configuration restores time synchronization but does not provide the same authentication guarantees as NTS. Use only trusted time sources. The two noted below are the ones used to get it working.

### Step 2A.1 - Configure Bootstrap NTP Sources

Create the Chrony bootstrap source file:

```
sudo nano /etc/chrony/sources.d/bootstrap-ip.sources
```

Add:

```
server 91.189.91.111 iburst trust
server 185.125.190.121 iburst trust
```

Save the file, then verify it:

```
cat /etc/chrony/sources.d/bootstrap-ip.sources
```

> **Note:** These are the IP addresses used during the original repair. Verify that they remain valid before deploying this configuration on another system.

### Step 2A.2 - Modify Chrony Authentication Selection

Open the Chrony configuration:

```
sudo nano /etc/chrony/chrony.conf
```

Add the following directive:

```
authselectmode ignore
```

Verify that the configuration also contains:

```
makestep 1 3
```

The relevant settings should look like:

```
authselectmode ignore
makestep 1 3
```

#### Explanation

**authselectmode ignore** prevents authentication status from automatically controlling Chrony's source-selection process. This permits the configured bootstrap NTP sources to participate in time synchronization.

**makestep 1 3** allows Chrony to step the system clock when the offset exceeds one second during the first three clock updates.

### Step 2A.3 - Restart Chrony

Restart the service:

```
sudo systemctl restart chrony
```

Wait approximately 20 to 30 seconds, then check:

```
date
timedatectl
chronyc tracking
chronyc sources -v
```

A successful synchronization should eventually show:

```
System clock synchronized: yes
NTP service: active
Leap status: Normal
```

> **Note:** Synchronization may take longer than 30 seconds depending on network availability and the initial clock offset.

### Step 2A.4 - Verify After Reboot

Reboot the Raspberry Pi:

```
sudo reboot
```

After reconnecting through SSH, run:

```
date
timedatectl
chronyc tracking
```

Verify the following:

1. The system date and time are correct.
2. Chrony has synchronized successfully.
3. The NTP service is active.
4. The Chrony leap status is Normal.

Example of correct output:

```
Mon Oct 19 10:39:11 EDT 2026
               Local time: Mon 2026-10-19 10:39:12 EDT
           Universal time: Mon 2026-10-19 14:39:12 UTC
                 RTC time: n/a
                Time zone: America/New_York (EDT, -0400)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
Reference ID    : 5BBD5B71 (ntp-nts-3.ps6.canonical.com)
Stratum         : 3
Ref time (UTC)  : Mon Oct 19 14:34:07 2026
System time     : 0.000008470 seconds slow of NTP time
Last offset     : +0.000000339 seconds
RMS offset      : 0.000197682 seconds
Frequency       : 7.755 ppm fast
Residual freq   : +0.000 ppm
Skew            : 0.016 ppm
Root delay      : 0.063948907 seconds
Root dispersion : 0.002417034 seconds
Update interval : 1038.5 seconds
Leap status     : Normal
```

### Step 2A.5 - Verify Technitium DNS

After confirming that the system clock is synchronized, test DNS resolution against the **secondary** server:

```
dig @192.168.XXX.XX google.com    # Secondary
```

Verify that the response contains:

```
status: NOERROR
```

Check that the DNSSEC time-validation error is no longer occurring. If resolution still fails, review the Technitium logs and Chrony synchronization status.

### Important Maintenance Considerations

The bootstrap NTP sources use hard-coded IP addresses. If these addresses change or become unavailable, the Raspberry Pi may fail to synchronize its clock. Periodically verify:

```
cat /etc/chrony/sources.d/bootstrap-ip.sources
chronyc sources -v
chronyc tracking
```

The file should still show:

```
server 91.189.91.111 iburst trust
server 185.125.190.121 iburst trust
```

### Result

The original Raspberry Pi 4 configuration recovered its system time after reboot. Chrony synchronized automatically, allowing Technitium DNSSEC validation to operate normally.

**Status: Tested successfully on the original installation.**

---

# Part 3 - Initial Technitium Configuration

After installing Technitium DNS Server on both the Proxmox Ubuntu Server VM and the Raspberry Pi 4, configure and verify each server.

| Server | Role | IP Address |
| --- | --- | --- |
| Proxmox Ubuntu Server VM | Primary DNS | 192.168.XXX.XX |
| Raspberry Pi 4 Ubuntu Server | Secondary DNS | 192.168.XXX.XX |

> **Important:** Replace these IP addresses with the addresses assigned to your own DNS servers.

> **Plan for clustering before you configure the secondary.** The next guide, [Technitium-DNS-Cluster-Setup.md](Technitium-DNS-Cluster-Setup.md), joins the secondary to the primary. Joining **overwrites the secondary's Allowed, Blocked, Apps, Settings, and Administration configuration** with the primary's, and afterward those sections synchronize from the primary.
>
> What this means for this part:
>
> - **Configure the primary completely** (Steps 3 to 6). Forwarders, logging, and apps set there are what the cluster will use.
> - On the **secondary**, you only need Steps 1 to 3 (web access, administrator password, network settings) and Step 7 (service checks) before joining. Settings you enter on the secondary beyond that are replaced at join.
> - **Query logs stay local to each node.** The SQLite query-log database is not synchronized between cluster members.
> - **Back up the secondary before joining** ([BackupAndRecovery.md](BackupAndRecovery.md)).
>
> If you run the servers **without** clustering, repeat every step on both servers.

---

## Step 1 - Access the Technitium Web Interface

From a computer on your local network, open a web browser.

**Primary DNS server:**

```
http://192.168.XXX.XX:5380      # Primary
```

**Secondary DNS server:**

```
http://192.168.XXX.XX:5380      # Secondary
```

Technitium uses TCP port `5380` for its default HTTP administrative web interface. Complete the initial setup wizard if prompted.

> **Security Note:** The HTTP administrative interface is not encrypted. Use it only on a trusted management network during initial setup. Configure HTTPS for ongoing administration ([Maintenance-and-Hardening.md](Maintenance-and-Hardening.md), Section 3).

---

## Step 2 - Configure Administrator Credentials

When accessing Technitium DNS Server for the first time, complete the initial administrator setup.

1. Open the Technitium web interface.
2. Follow the initial setup prompts.
3. Configure a strong administrator password.
4. Complete the setup.
5. Log in using the administrator account.

### Security Recommendations

- Use a unique administrator password.
- Do not reuse passwords from other services.
- Restrict access to the administrative interface.
- Do not expose administrative ports directly to the internet.
- Use HTTPS for ongoing administration.

Repeat this process on both DNS servers.

---

## Step 3 - Verify DNS Server Network Settings

After logging in, verify that the DNS server is listening on the correct network interfaces.

### Access DNS Server Settings

1. Log in to the Technitium web interface.
2. Navigate to **Settings**.
3. Locate **DNS Server Local End Points**.
4. Review the configured listening addresses.

### DNS Server Local End Points

By default, Technitium may display:

```
0.0.0.0:53
[::]:53
```

| Setting | Description |
| --- | --- |
| `0.0.0.0:53` | Listen on all IPv4 network interfaces |
| `[::]:53` | Listen on all IPv6 network interfaces |

### IPv4-Only Configuration (Optional)

If your network does not use IPv6, you may configure Technitium to listen only on IPv4. Change:

```
0.0.0.0:53
[::]:53
```

To:

```
0.0.0.0:53
```

> **Important:** Removing the IPv6 listening address does not disable IPv6 in Ubuntu Server. It only stops Technitium from listening for DNS requests over IPv6. If IPv6 stays enabled on your client VLANs, clients can still learn other DNS servers over IPv6; see [pfSense-DHCP-and-DNS.md](pfSense-DHCP-and-DNS.md), Section 5.

### DNS Server IPv4 Source Addresses

Locate **DNS Server IPv4 Source Addresses**. The default is `0.0.0.0`, which lets the operating system choose the source address for outbound DNS requests. For a server with one network interface this can stay unchanged.

### DNS Server IPv6 Source Addresses

Locate **DNS Server IPv6 Source Addresses**. The default is `::`. If IPv6 is not used, no change is normally needed.

### Save Configuration

If you made changes:

1. Save the Technitium settings.
2. Verify that the DNS service remains operational.
3. Test DNS resolution from another computer (Part 4).

### Verify DNS Port 53

From the Ubuntu Server terminal, run:

```
sudo ss -tulpn | grep ':53 '
```

Verify that Technitium is listening on UDP and TCP port 53.

**Result:** record the outcome in the Part 3 checklist (Step 8).

---

## Step 4 - Configure DNS Forwarders

> **Primary only when clustering.** These settings synchronize to the secondary after it joins the cluster.

Technitium can resolve DNS queries recursively or forward them to another resolver. This installation forwards to AdGuard DNS using DNS over TLS (DoT), which encrypts queries between Technitium and the upstream provider.

### Configure Forwarders

1. Log in to the Technitium web interface.
2. Navigate to **Settings**.
3. Select **Proxy & Forwarders**.
4. Locate the **Forwarders** section.
5. Use **Quick Select** to add the desired DNS providers, or manually enter the forwarder addresses.
6. Configure the forwarder addresses.
7. Set **Forwarder Protocol** to **DNS-over-TLS**.
8. Enable **Concurrent Forwarding**.
9. Set **Forwarder Concurrency** to `2`.
10. Save the configuration.

### DNS Forwarder Configuration

The following AdGuard DNS-over-TLS forwarders are configured:

```
dns.adguard-dns.com (94.140.14.14:853)
dns.adguard-dns.com (94.140.15.15:853)
```

| Setting | Configuration |
| --- | --- |
| Forwarder Protocol | DNS-over-TLS |
| Concurrent Forwarding | Enabled |
| Forwarder Concurrency | 2 |

With concurrent forwarding enabled and concurrency set to 2, Technitium can query both configured forwarders simultaneously and use the fastest response.

```
Technitium DNS Server
         |
         v
Concurrent Forwarding
         |
    +----+----+
    |         |
    v         v
AdGuard 1   AdGuard 2
94.140.14.14 94.140.15.15
    |         |
    +----+----+
         |
         v
     DNS Response
```

> **Note:** Concurrent forwarding may send the same query to both upstream endpoints. That can improve availability but may increase upstream traffic.

> **Note:** DNS over TLS encrypts traffic between Technitium and AdGuard DNS. It does not encrypt traffic between client devices and Technitium.

### DNS over TLS (DoT)

Under **Forwarder Protocol**, select **DNS-over-TLS**. DoT uses TCP port `853` by default. For AdGuard DNS the TLS hostname is `dns.adguard-dns.com`, and the IPv4 addresses are `94.140.14.14` and `94.140.15.15`.

> **Important:** Configuring an upstream IP address does not by itself enable encrypted DNS. The **Forwarder Protocol** must be set to **DNS-over-TLS**, and the correct TLS hostname must be used for certificate validation.

> **Firewall note:** If the Technitium servers sit on a VLAN that blocks outbound TCP 853, they cannot reach AdGuard. Keep the servers on an unrestricted VLAN, or allow this traffic explicitly ([firewall-rules.md](firewall-rules.md), Section 3).

### Verify Forwarder Configuration

After saving, test resolution and upstream TLS as described in **Part 4** (Steps 2, 3, and 5).

**Status: Configuration documented.** Record test results in the Part 4 checklist.

---

## Step 5 - Configure Built-in DNS Server Logging

> **Primary only when clustering,** unless you want different logging on a node. Settings synchronize from the primary after joining.

Technitium includes built-in logging for server errors, warnings, and administrative events. It does not require installing an additional app.

### Step 5.1 - Access Logging Settings

1. Log in to the Technitium web interface.
2. Navigate to **Settings**.
3. Select the **Logging** section.
4. Review the existing configuration.

### Step 5.2 - Configure Logging

| Setting | Configuration |
| --- | --- |
| Enable Logging To | File |
| Ignore Resolver Error Logs | Unchecked |
| No Stack Trace | Unchecked |
| Log All Queries | Unchecked |
| Use Local Time | Checked |
| Log Folder Path | `/var/log/technitium/dns` |
| Max Log File Days | 30 |

**Note:** Leave **Log All Queries** disabled when using the Query Logs (SQLite) app, to avoid duplicating detailed query records.

Click **Save Settings**.

### Step 5.3 - Verify Built-in Logging

On Ubuntu Server, run:

```
sudo ls -lah /var/log/technitium/dns
```

Verify that the directory contains Technitium log files. To identify recently modified log files:

```
sudo find /var/log/technitium/dns -maxdepth 1 -type f -printf '%T@ %p\n' | sort -nr | head
```

Check this on **both** servers, since each node writes its own log files.

---

## Step 6 - Install and Configure Query Logs (SQLite)

The Query Logs (SQLite) app provides persistent DNS query history so you can investigate requests, client activity, and blocked domains.

> **Clustering:** Install and configure the app on the primary. After the secondary joins, confirm that the app is present there (Apps are synchronized from the primary). The **database itself is local to each node**; query history is not synchronized.

### Step 6.1 - Install the App

1. Log in to the Technitium web interface.
2. Navigate to **Apps**.
3. Open **App Store**.
4. Locate **Query Logs (Sqlite)**.
5. Click **Install**.
6. Wait for installation to complete.

### Step 6.2 - Configure SQLite Logging

Open **Apps > Query Logs (Sqlite) > Config** and use:

```
{
  "enableLogging": true,
  "maxQueueSize": 200000,
  "maxLogDays": 30,
  "maxLogRecords": 100000,
  "enableVacuum": false,
  "useInMemoryDb": false,
  "sqliteDbPath": "querylogs.db",
  "connectionString": "Data Source='{sqliteDbPath}'; Cache=Shared;"
}
```

Click **Save**.

### Configuration Explanation

| Setting | Purpose |
| --- | --- |
| enableLogging | Enables DNS query logging |
| maxQueueSize | Maximum queued log entries |
| maxLogDays | Maximum retention age |
| maxLogRecords | Maximum retained query records |
| enableVacuum | Controls SQLite database vacuuming |
| useInMemoryDb | Stores the database on disk when false |
| sqliteDbPath | SQLite database filename |
| connectionString | SQLite database connection configuration |

**Important:** The database may reach the record limit before 30 days have elapsed. Actual retention depends on DNS query volume.

> **Privacy note:** Query logs record which sites each device looks up. Keep retention short, restrict who can open the admin interface, and never publish logs.

> **Storage note:** SQLite logging generates ongoing disk writes. On a Raspberry Pi with microSD storage, monitor free space and storage health, or boot from a USB SSD ([Maintenance-and-Hardening.md](Maintenance-and-Hardening.md), Section 6).

### Step 6.3 - Verify Query Logging

Test logging on each server as described in **Part 4, Step 6**.

---

## Step 7 - Verify Technitium Services

After the servers respond to DNS, verify that Technitium runs and starts automatically at boot. Do this on **both** servers.

### Step 7.1 - Service Status

```
sudo systemctl status dns.service
```

Expected: `Active: active (running)`.

### Step 7.2 - Automatic Startup

```
sudo systemctl is-enabled dns.service
```

Expected: `enabled`. If it is not enabled:

```
sudo systemctl enable dns.service
sudo systemctl is-enabled dns.service
```

> **Note:** Enabling a service configures it to start at boot; it does not necessarily start it immediately.

### Step 7.3 - Start Technitium if It Is Not Running

```
sudo systemctl start dns.service
sudo systemctl status dns.service
```

### Step 7.4 - Troubleshoot Startup Problems

```
sudo journalctl -u dns.service -n 50 --no-pager
```

Check for startup failures, port 53 conflicts, configuration errors, permission problems, or missing dependencies.

> **Important:** Do not repeatedly restart the service without reading the error messages. Identify the cause first.

---

## Step 8 - Verify Configuration After Reboot

Verify that both servers start automatically and work after a reboot. This confirms that Ubuntu boots, Technitium starts on its own, DNS works, and (on the Raspberry Pi) the clock synchronizes so DNSSEC problems do not return.

> **Important:** Reboot one server at a time. If the servers already serve network clients, confirm that the other server is working before you reboot the first.

### Step 8.1 - Reboot the Primary

```
sudo reboot
```

After the server returns, reconnect over SSH and run:

```
sudo systemctl status dns.service
systemctl is-enabled dns.service
```

Expected: `active (running)` and `enabled`. Then, from another computer:

```
dig @192.168.XXX.XX google.com A      # Primary
```

Expected: `status: NOERROR` with an ANSWER SECTION.

### Step 8.2 - Reboot the Secondary

After the primary checks out, repeat on the Raspberry Pi:

```
sudo reboot
```

After it returns:

```
sudo systemctl status dns.service
systemctl is-enabled dns.service
```

Then, from another computer:

```
dig @192.168.XXX.XX google.com A      # Secondary
```

### Step 8.3 - Verify Raspberry Pi Time Synchronization

The Raspberry Pi 4 has no battery-backed RTC, so it may boot with an incorrect clock. **Do not manually set the time or restart Chrony before checking.** On the Pi:

```
date
timedatectl
chronyc tracking
chronyc sources -v
```

Expected:

```
System clock synchronized: yes
NTP service: active
Leap status     : Normal
```

and a selected source marked `^*`.

> **Note:** Synchronization may take a while after boot. If it does not synchronize, see [Part 2A](#part-2a---raspberry-pi-4-time-synchronization-optional).

### Step 8.4 - Verify DNSSEC Behavior After Reboot

Because an incorrect clock previously caused DNSSEC errors on the Pi, run the DNSSEC tests in **Part 4, Step 4** against the secondary after the reboot.

### Step 8.5 - Part 3 Checklist

| Verification | Primary | Secondary |
| --- | --- | --- |
| Ubuntu boots successfully | NOT TESTED | NOT TESTED |
| Technitium service running | NOT TESTED | NOT TESTED |
| Automatic startup enabled | NOT TESTED | NOT TESTED |
| Listening on UDP and TCP 53 | NOT TESTED | NOT TESTED |
| DNS resolution after reboot | NOT TESTED | NOT TESTED |
| Chrony synchronized after reboot | n/a | NOT TESTED |
| Built-in logging files present | NOT TESTED | NOT TESTED |
| SQLite query log shows test query | NOT TESTED | NOT TESTED |

> **Note:** Replace NOT TESTED with PASS or FAIL after you run each test. A test that has not been run is never PASS.

---

## Initial Configuration Complete

After the checks above pass:

- Both Technitium servers are installed and reachable.
- Administrator credentials are configured.
- DNS listening settings are reviewed.
- Forwarders, logging, and Query Logs (SQLite) are configured on the primary.
- Both servers have been tested after reboot, including Raspberry Pi time synchronization.

> **Next steps:**
>
> 1. Create DHCP reservations and hand out both DNS servers: [pfSense-DHCP-and-DNS.md](pfSense-DHCP-and-DNS.md).
> 2. Join the servers in a cluster: [Technitium-DNS-Cluster-Setup.md](Technitium-DNS-Cluster-Setup.md).
> 3. Configure filtering: [AdvancedBlocking.md](AdvancedBlocking.md).

---

# Part 4 - Verify DNS Resolution

This is the single place where the DNS tests are defined. Run them after the initial configuration (Part 3), after the cluster is built, and after any firewall or forwarder change.

> **Important:** Replace the placeholder addresses with your actual servers. Test each server by its **own** address and mark each result separately.

## Step 1 - Install DNS Utilities

From another Linux computer on your network:

```
sudo apt update
sudo apt install dnsutils -y
```

The `dig` utility sends DNS queries directly to a server you choose.

## Step 2 - Resolution over UDP

```
dig @192.168.XXX.XX google.com A      # Primary
dig @192.168.XXX.XX google.com A      # Secondary
```

Expected for both: `status: NOERROR` with an ANSWER SECTION containing one or more IPv4 addresses, for example:

```
;; ANSWER SECTION:
google.com.    300    IN    A    142.250.190.14
```

(The address and TTL will differ.) At the bottom of the output, check the SERVER line, such as `;; SERVER: 192.168.XXX.XX#53`, to confirm which server answered.

## Step 3 - Resolution over TCP and Reachability

DNS normally uses UDP, but TCP port 53 must also work.

```
dig +tcp @192.168.XXX.XX google.com A      # Primary
dig +tcp @192.168.XXX.XX google.com A      # Secondary
```

Expected: `status: NOERROR` with an answer. If a TCP query fails, check the host firewall and the pfSense rules for TCP 53.

Optional reachability test:

```
ping -c 4 192.168.XXX.XX      # Primary
ping -c 4 192.168.XXX.XX      # Secondary
```

Look for `0% packet loss`.

> **Note:** Some networks block ICMP. A failed ping does not prove a DNS server is down; the `dig` tests are more meaningful.

## Step 4 - Verify DNSSEC Behavior

### A valid domain

```
dig @192.168.XXX.XX cloudflare.com A +dnssec      # Primary
dig @192.168.XXX.XX cloudflare.com A +dnssec      # Secondary
```

Expected: `status: NOERROR`. The response may include the `ad` flag (`flags: qr rd ra ad`), which indicates DNSSEC authentication. The absence of `ad` does not necessarily mean a failure, since the resolver configuration and upstream provider affect it.

### An intentionally invalid domain

`dnssec-failed.org` has deliberately broken DNSSEC signatures; a validating resolver should reject it.

```
dig @192.168.XXX.XX dnssec-failed.org A      # Primary
dig @192.168.XXX.XX dnssec-failed.org A      # Secondary
```

Expected:

```
status: SERVFAIL
```

The response may also include `EDE: 6 (DNSSEC Bogus)`.

### Previously recorded result

**Test date:** October 19, 2026, secondary server:

```
status: SERVFAIL
EDE: 6 (DNSSEC Bogus)
EDE: 22 (No Reachable Authority)
EDE: 13 (Cached Error)
```

The invalid domain was rejected. This is expected when DNSSEC validation succeeds somewhere in the resolution path.

> **Important:** If your upstream provider validates DNSSEC, the rejection may happen upstream instead of in Technitium. This test shows that invalid DNSSEC data is rejected somewhere in the path; it does not independently prove Technitium validated it.

**Recorded result:** `Secondary DNS - Invalid DNSSEC Domain: PASS` (October 19, 2026 test). Record other results after running them.

## Step 5 - Verify DNS-over-TLS Forwarding

### Configured forwarders

| Setting | Configuration |
| --- | --- |
| DNS Provider | AdGuard DNS |
| Primary Forwarder | 94.140.14.14 |
| Secondary Forwarder | 94.140.15.15 |
| Forwarder Hostname | dns.adguard-dns.com |
| Forwarder Protocol | DNS-over-TLS |
| Port | TCP 853 |
| Concurrent Forwarding | Enabled |
| Forwarder Concurrency | 2 |

> **Note:** Other providers use different addresses, hostnames, and protocols.

### Step 5.1 - Review the configuration

1. Open **Settings > Proxy & Forwarders**.
2. Confirm the forwarder addresses and hostnames.
3. Confirm the protocol is **DNS-over-TLS**.
4. Confirm the settings were saved.

If the servers are clustered, this is defined on the primary and synchronized; still confirm it appears on the secondary.

### Step 5.2 - Test TLS connectivity to the upstream

On each Ubuntu server:

```
openssl s_client \
  -connect 94.140.14.14:853 \
  -servername dns.adguard-dns.com \
  -verify_hostname dns.adguard-dns.com \
  -verify_return_error </dev/null
```

Repeat with `94.140.15.15`. Look for `Verification: OK` or `Verify return code: 0 (ok)`.

> **Important:** This proves the server can make a TLS connection and validate the certificate. It does not prove that Technitium used that connection for a particular query.

### Step 5.3 - Verify actual forwarding traffic

Install `tcpdump` if needed:

```
sudo apt install tcpdump -y
```

On the Technitium server:

```
sudo tcpdump -ni any \
  'tcp port 853 and (host 94.140.14.14 or host 94.140.15.15)'
```

While it runs, query a domain that is not already cached from another computer:

```
dig @192.168.XXX.XX example.com A
```

Expected: TCP traffic between the Technitium server and one or both AdGuard endpoints. With concurrent forwarding enabled, either or both may appear, and cached answers may produce no new upstream traffic. Press `Ctrl+C` to stop.

## Step 6 - Verify DNS Query Logging

After configuring SQLite query logging, generate a query against each server:

```
dig @192.168.XXX.XX example.com A      # Primary
dig @192.168.XXX.XX example.com A      # Secondary
```

In each server's web interface open **Logs > Query Logs**, select the SQLite source, and confirm the request appears with the expected client IP, domain, timestamp, and response code. If `example.com` is intentionally blocked by policy, `NXDOMAIN` is expected; the purpose is to confirm the query is recorded.

**Note:** The query-log databases are local to each server, so check each one. Record NOT TESTED for any server you have not verified.

## Step 7 - Final Verification Checklist

| Verification | Primary | Secondary |
| --- | --- | --- |
| Resolution over UDP | NOT TESTED | NOT TESTED |
| Resolution over TCP | NOT TESTED | NOT TESTED |
| Valid domain resolves | NOT TESTED | NOT TESTED |
| Invalid DNSSEC domain rejected | NOT TESTED | PASS (Oct 19, 2026) |
| DNS-over-TLS configuration reviewed | NOT TESTED | NOT TESTED |
| Upstream TLS connectivity | NOT TESTED | NOT TESTED |
| Forwarding traffic observed | NOT TESTED | NOT TESTED |
| Test query appears in SQLite log | NOT TESTED | NOT TESTED |

> **Important:** Use NOT TESTED for anything you have not performed. Never mark an untested item PASS.

Record errors, unexpected responses, and configuration changes here:

```
(notes)
```

If DNS resolution fails, go to **Part 5 - Troubleshooting**.

## DNS Verification Complete

When the required tests have passed and been recorded:

- Both servers resolve external domains over UDP and TCP.
- Invalid DNSSEC data is rejected somewhere in the resolution path.
- DNS-over-TLS forwarders are reviewed and reachable.
- Forwarding traffic has been observed.

> **Next step:** Continue to Part 5 for common problems, then to [Technitium-DNS-Cluster-Setup.md](Technitium-DNS-Cluster-Setup.md).

---

# Part 5 - Troubleshooting

Use these procedures when a verification step fails. Run server-side commands on the affected Ubuntu host, and replace `192.168.XXX.XX` with that host's actual address. Record the original symptoms before changing settings.

---

## 1. Technitium Service Not Running

```
sudo systemctl status dns.service
systemctl is-enabled dns.service
```

Expected: `active (running)` and `enabled`. If stopped or disabled:

```
sudo systemctl start dns.service
sudo systemctl enable dns.service
```

If startup fails, inspect recent logs:

```
sudo journalctl -u dns.service -n 50 --no-pager
```

Look for port conflicts, invalid configuration, permission problems, or startup errors. Diagnose the cause rather than repeatedly restarting.

---

## 2. DNS Resolution Fails

From a separate client, query the affected server explicitly:

```
dig @192.168.XXX.XX google.com A
dig +tcp @192.168.XXX.XX google.com A
```

A healthy response shows `status: NOERROR` and an `ANSWER SECTION`. If the request times out, verify the service and listening ports:

```
sudo systemctl status dns.service
sudo ss -tulpn | grep ':53 '
sudo ufw status verbose
```

Check that UDP **and** TCP port 53 are permitted from authorized clients, and review the pfSense rules for that VLAN ([firewall-rules.md](firewall-rules.md)). If the server responds but external names do not resolve, inspect **Settings > Proxy & Forwarders** and review the query logs.

> **Note:** `NOERROR` alone does not guarantee an address was returned; check the `ANSWER SECTION` and the queried record type.

---

## 3. DNSSEC Errors After Reboot / Raspberry Pi Clock Problems

Incorrect system time can break DNSSEC validation and TLS certificate checks. On the Raspberry Pi, check the clock **before** manually restarting Chrony:

```
date
timedatectl
sudo systemctl status chrony
chronyc tracking
chronyc sources -v
```

Expected after synchronization: `System clock synchronized: yes`, `NTP service: active`, `Leap status: Normal`, and a selected source marked `^*`. Synchronization may take time after boot.

If the clock does not synchronize:

```
sudo journalctl -u chrony -n 50 --no-pager
```

Check network reachability, NTP/NTS sources, and certificate errors. See **[Part 2A - Raspberry Pi 4 Time Synchronization](#part-2a---raspberry-pi-4-time-synchronization-optional)** for the optional bootstrap repair used in the original installation. The bootstrap sources are unauthenticated and should not be deployed automatically.

After the clock is correct, retest normal DNS and the invalid-DNSSEC domain (Part 4, Step 4).

---

## 4. Port 53 Not Responding

```
sudo ss -tulpn | grep ':53 '
```

Technitium should listen on its configured IPv4/IPv6 addresses for UDP and TCP. If another process owns the port, investigate conflicts with `systemd-resolved`, `dnsmasq`, BIND, or another DNS service.

> **Important:** `systemd-resolved` may use the loopback stub address `127.0.0.53`. Do not disable it blindly; determine whether it actually conflicts with Technitium and whether the host depends on it.

If Technitium is listening but remote clients time out, review UFW and the network firewall policies for UDP/TCP port 53.

---

## 5. Web Interface Not Accessible

```
sudo systemctl status dns.service
sudo ss -tulpn | grep ':5380 '
```

From another device, test reachability if ICMP is allowed:

```
ping -c 4 192.168.XXX.XX
```

A failed ping does **not** prove the web service is down; ICMP may be blocked. Review UFW and the pfSense policies for the actual management port. Restrict management access to trusted networks, and use HTTPS for ongoing administration ([Maintenance-and-Hardening.md](Maintenance-and-Hardening.md)).

---

## 6. DNS-over-TLS Forwarding Problems

| Endpoint | IP address | TLS hostname | Port |
| --- | --- | --- | --- |
| Primary upstream | `94.140.14.14` | `dns.adguard-dns.com` | TCP 853 |
| Secondary upstream | `94.140.15.15` | `dns.adguard-dns.com` | TCP 853 |

Verify **Settings > Proxy & Forwarders** on each server. Test certificate validation and outbound connectivity from the affected host:

```
openssl s_client \
  -connect 94.140.14.14:853 \
  -servername dns.adguard-dns.com \
  -verify_hostname dns.adguard-dns.com \
  -verify_return_error </dev/null
```

Repeat with `94.140.15.15`. Look for `Verify return code: 0 (ok)`. If it fails, check time synchronization, internet routing, certificate validation, and outbound TCP 853 rules. A common cause is a pfSense reject rule for port 853 that also covers the DNS servers' own VLAN ([firewall-rules.md](firewall-rules.md), Section 3).

> **Important:** A successful `openssl` test confirms TLS connectivity, not that Technitium used that upstream for a given query. Use the packet-capture procedure in Part 4, Step 5.3.

---

## 7. One DNS Server Works but the Other Does Not

Query each server by its **distinct** address from the same client:

```
dig @192.168.XXX.XX google.com A      # Primary
dig @192.168.XXX.XX google.com A      # Secondary
```

On the failing server, verify the service, port 53, firewall, forwarders, clock, and logs. Remember that two working DNS servers are not automatically a cluster ([Technitium-DNS-Cluster-Setup.md](Technitium-DNS-Cluster-Setup.md)). If they are clustered, compare the two nodes' settings and Advanced Blocking config.

---

## 8. TVs, Gaming Systems, Discord, or Other Apps Stop Working

Earlier DNS enforcement testing caused connectivity problems on some devices. The precise causes were **not** established for this installation, so troubleshoot with evidence:

1. Confirm the device's assigned DNS servers and network/VLAN.
2. Query the approved Technitium servers directly from a test client.
3. Inspect Technitium query logs for failures or blocked domains.
4. If Advanced Blocking is enabled, review the device's assigned group ([AdvancedBlocking.md](AdvancedBlocking.md)).
5. In pfSense, check **Status > System Logs > Firewall** for DNS (53), DoT (853), and DoH/QUIC (443, including UDP) rejects. Rules 5 and 6 in [firewall-rules.md](firewall-rules.md) are the usual suspects.
6. Test a narrow, temporary exception only after identifying a legitimate false positive; document the change in the exceptions table and retest.

> **Important:** Allowing a device to reach Technitium does not exempt it from Advanced Blocking. Avoid broad bypass rules without evidence.

---

## 9. Collect Diagnostic Information

Before changing settings, capture:

```
hostnamectl
ip -br addr
ip route
sudo systemctl status dns.service
sudo ss -tulpn
sudo ufw status verbose
timedatectl
sudo journalctl -u dns.service -n 100 --no-pager
dig @192.168.XXX.XX google.com A
```

On hosts using Chrony, also run:

```
chronyc tracking
chronyc sources -v
```

Record the affected server, time, error messages, recent changes, and tests performed. **Redact credentials, sensitive hostnames, and private network information before sharing diagnostic output publicly.**

---

## 10. Identify Blocked DNS Queries

When a legitimate domain fails to resolve, Technitium Advanced Blocking may be responsible.

### Step 1 - Review Query Logs

Navigate to **Logs > Query Logs** and locate the affected domain. Review the client IP, requested domain, query type, response code, and blocked status.

### Step 2 - Test DNS Resolution

```
dig @192.168.XXX.XX example.com
```

A blocked response may include:

```
EDE: 15 (Blocked): (source=advanced-blocking-app; group=everyone else; domain=example.com)
```

This identifies the blocking component and policy group, but not necessarily the exact list.

### Step 3 - Check Advanced Blocking

1. Navigate to **Apps > Advanced Blocking > Config** (on the primary when clustered).
2. Locate the affected group.
3. Review the `blocked` array and the configured `blockListUrls`.
4. Determine which rule or list matches the domain.

For example, this explicitly blocks `example.com`:

```
"allowed": [],
"blocked": [
  "example.com"
]
```

If it was added by mistake, remove it:

```
"allowed": [],
"blocked": []
```

**Important:** Preserve other legitimate entries in the `blocked` array; only empty it if it contains no other domains. Save the configuration. If the domain still resolves as blocked, check the external lists and the custom lists in [AdvancedBlocking.md](AdvancedBlocking.md).

### Step 4 - Verify Resolution

```
dig @192.168.XXX.XX example.com
```

Expected, provided no other rule applies: `status: NOERROR` with an answer.

### Recorded investigation

This procedure was used to find a manually configured block for `example.com` in the Advanced Blocking group named `everyone else`. It shows how SQLite query logs, Extended DNS Errors (EDE), and the Advanced Blocking config work together to explain a blocked request.

---

## Troubleshooting Summary

| Symptom | First checks |
| --- | --- |
| Service stopped | `systemctl status dns.service`, journal logs |
| DNS timeout | Port 53, UFW, pfSense/VLAN rules |
| DNSSEC error after reboot | Clock, Chrony, NTP/NTS sources |
| Web interface inaccessible | Service, management port, firewall |
| Upstream resolution fails | Forwarder settings, TCP 853, TLS certificate, pfSense reject rules |
| Only one server fails | Test each IP and compare configurations |
| TV/game/app problems | Client DNS, query logs, blocking policies, pfSense logs |
| Cluster node Unreachable | [Technitium-DNS-Cluster-Setup.md](Technitium-DNS-Cluster-Setup.md), Section 7 |

After correcting a problem, rerun the relevant tests in **Part 4** and record the actual results. Do not mark a test as PASS unless it was performed.

---

# Part 6 - Tested Ubuntu Security Configuration

This section documents security work completed **November 16, 2026** on a **standalone Proxmox Ubuntu test server**, `test-technitium-github-setup` (`192.168.xxx.xx`), running **Technitium DNS Server v15.6.2**. It is **not** the production primary/secondary cluster. Replace the example IPs, username, interface, and key name for a new deployment. Do not copy these addresses blindly.

## Step 1 - Record the Test Environment

| Component | Verified value |
| --- | --- |
| Ubuntu Technitium test host | `192.168.xxx.xx` |
| SSH management desktop | `192.168.xxx.xx` |
| Authorized DNS client subnet | `192.168.xxx.0/24` |
| Wazuh manager | `192.168.xxx.xx` |
| Technitium version | `15.6.2` |
| Technitium service | `dns.service` |
| Technitium application | `/opt/technitium/dns` |
| Technitium configuration | `/etc/dns` |
| SSH username | `username` |

> **Scope:** This host is standalone and may join a **different future cluster**. Do not add a cluster-peer firewall rule until its actual peer address and requirements are known. The original guide's production primary/secondary test results remain unchanged.

## Step 2 - Configure UFW Without Losing SSH

**Before enabling UFW**, identify the IP address of your active SSH client and keep that SSH session open:

```bash
echo "$SSH_CONNECTION"
sudo ss -tulpen
sudo ufw status numbered
grep '^DEFAULT_' /etc/default/ufw
```

The test host used default **deny incoming** and **allow outgoing**. Confirm your actual management address and DNS client networks before using these commands:

```bash
# Inbound SSH only from the management desktop
sudo ufw allow from 192.168.xxx.xx to any port 22 proto tcp comment 'Management SSH'

# Inbound DNS from the authorized LAN
sudo ufw allow from 192.168.xxx.0/24 to any port 53 proto udp comment 'Technitium DNS UDP'
sudo ufw allow from 192.168.xxx.0/24 to any port 53 proto tcp comment 'Technitium DNS TCP'

# Outbound Wazuh agent communication
sudo ufw allow out to 192.168.xxx.xx port 1514 proto tcp comment 'Wazuh Agent Communication'

# Explicit outbound resolver permissions (redundant with allow-outgoing default)
sudo ufw allow out 53/udp comment 'Outbound DNS UDP'
sudo ufw allow out 53/tcp comment 'Outbound DNS TCP'
sudo ufw allow out 853/tcp comment 'Outbound DNS over TLS'

# Reproduces the tested HTTP web-management access rule
sudo ufw allow 5380/tcp comment 'Technitium Web Management'
```

> **Security warning:** The last command permits HTTP management on TCP 5380 from **any source that can route to the host**, including IPv6 where enabled. This matched the tested configuration at the user's request, but is **not recommended as a hardened default**. Prefer a management subnet or single trusted management IP, and use HTTPS for administration. For a new deployment, use instead: `sudo ufw allow from 192.168.xxx.xx to any port 5380 proto tcp comment 'Technitium Web Management'` (adjust the IP). Do not expose either administrative port directly to the internet.

> **Port clarification:** On the tested v15.6.2 host, TCP **5380** was the HTTP Web UI and TCP **53443** was the HTTPS Web UI. **53443 is not exclusively a cluster port.** It was listening but **not opened in UFW** during this test. If you enable remote HTTPS management, restrict its inbound firewall rule to trusted administrators and verify the configured certificate. See [Maintenance-and-Hardening.md](Maintenance-and-Hardening.md).

Verify the DHCP client exception before enabling UFW (important if the host receives a pfSense DHCP reservation):

```bash
sudo grep -nE '67|68|dhcp' /etc/ufw/before.rules
```

The tested host had a rule permitting DHCP replies from UDP source port 67 to destination port 68. Then enable and inspect UFW:

```bash
sudo ufw enable
sudo ufw status verbose
sudo ufw status numbered
```

Expected defaults: `deny (incoming), allow (outgoing)`. **Keep the existing SSH connection open** until a separate new SSH session succeeds. Because the outbound policy allows traffic by default, the explicit outbound rules are descriptive rather than restrictions; to restrict outbound access, design and test a separate egress policy.

### Verify firewall behavior

From an authorized LAN client:

```bash
dig @192.168.xxx.xx google.com A
dig @192.168.xxx.xx google.com A +tcp
ssh username@192.168.xxx.xx
```

On the Technitium host:

```bash
sudo systemctl status dns.service --no-pager
sudo systemctl status wazuh-agent --no-pager
sudo ss -tnp | grep '192.168.xxx.xx:1514'
sudo ufw status verbose
```

The Wazuh socket should show `ESTAB` to `192.168.xxx.xx:1514`. An old "connected" log entry alone does not establish that the agent is currently connected.

## Step 3 - Confirm DNS-over-TLS Upstreams

In **Technitium > Settings > Proxy & Forwarders**, the tested configuration used:

| Setting | Value |
| --- | --- |
| Forwarder 1 | `dns.adguard-dns.com (94.140.14.14:853)` |
| Forwarder 2 | `dns.adguard-dns.com (94.140.15.15:853)` |
| Forwarder protocol | DNS-over-TLS |
| Concurrent forwarding | Enabled |
| Forwarder concurrency | 2 |

From the Technitium host, verify TLS reachability and the certificate hostname for **each** upstream:

```bash
openssl s_client -connect 94.140.14.14:853 \
  -servername dns.adguard-dns.com \
  -verify_hostname dns.adguard-dns.com \
  -verify_return_error </dev/null

openssl s_client -connect 94.140.15.15:853 \
  -servername dns.adguard-dns.com \
  -verify_hostname dns.adguard-dns.com \
  -verify_return_error </dev/null
```

Look for `Verify return code: 0 (ok)`. An empty result from `sudo ss -tnp | grep ':853'` is **not** proof of failure; connections may be short-lived or absent while responses are cached.

**Separate uncompleted verification:** These TLS tests verify reachability and certificate validation, **not** that Technitium actually forwarded a DNS query over TLS. To observe real forwarding, follow **Part 4, Step 5.3** (`tcpdump` during an uncached lookup). Mark that specific check **NOT TESTED** until performed.

## Step 4 - Recover a Forgotten Technitium Web UI Password

The following procedure was used successfully on **Technitium v15.6.2**. It temporarily interrupts DNS service and may affect authentication state. Use console/SSH access and back up the file first. Do **not** publish `auth.config`, backups, or credentials.

1. Identify the actual service configuration directory:

   ```bash
   sudo systemctl cat dns.service
   ```

   The tested service ran `DnsServerApp.dll /etc/dns` as user `dns-server`.

2. Back up authentication data, preserving ownership and permissions:

   ```bash
   sudo cp -a /etc/dns/auth.config /etc/dns/auth.config.bak
   sudo ls -l /etc/dns/auth.config*
   ```

3. Stop Technitium and confirm it is inactive:

   ```bash
   sudo systemctl stop dns.service
   sudo systemctl is-active dns.service
   ```

   Expected: `inactive`.

4. Rename the authentication configuration as directed by the **password-recovery instructions displayed on the Technitium login screen**:

   ```bash
   sudo mv /etc/dns/auth.config /etc/dns/resetadmin.config
   sudo ls -l /etc/dns/resetadmin.config /etc/dns/auth.config.bak
   ```

5. Restart and verify service:

   ```bash
   sudo systemctl start dns.service
   sudo systemctl status dns.service --no-pager
   ```

6. Refresh the Web UI, follow the recovery/login flow, and **immediately set a strong, unique administrator password**. Verify a fresh login. If 2FA was reset, re-enroll it. Review all accounts and permissions afterward. Keep the backup protected; do not blindly restore it over the recovered configuration.

> **Tested outcome:** Administrator Web UI access was restored on `192.168.xxx.xx`. This is a recovery procedure, not a routine installation step.

## Step 5 - Configure a Dedicated SSH Key and Disable Password Logins

**Perform this from the Ubuntu management desktop**, not on the server, so the private key remains on the client. Substitute your own username, server IP, and key filename.

### Generate a dedicated Ed25519 key

```bash
ssh-keygen -t ed25519 -a 100 \
  -f ~/.ssh/id_ed25519_technitium \
  -C "technitium-192.168.xxx.xx"
```

Use a strong private-key passphrase. If the key file already exists, **do not overwrite it**.

### Install only the public key

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_technitium.pub username@192.168.xxx.xx
```

### Verify key-only access **before disabling passwords**

```bash
ssh -i ~/.ssh/id_ed25519_technitium \
  -o IdentitiesOnly=yes \
  -o PreferredAuthentications=publickey \
  -o PasswordAuthentication=no \
  -o KbdInteractiveAuthentication=no \
  username@192.168.xxx.xx

whoami && hostname
```

The tested output was `username` and `test-technitium-github-setup`. A prompt for the **private key passphrase** is normal; it is not an SSH account-password prompt.

### Set server-side SSH policy

On the **Technitium server**, keep the original SSH session open and create:

```bash
sudo nano /etc/ssh/sshd_config.d/99-key-only.conf
```

Add:

```ini
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

Validate syntax and inspect effective settings **before reloading**:

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication) '
```

On the tested Ubuntu host, `passwordauthentication` **still showed `yes`** because `/etc/ssh/sshd_config.d/50-cloud-init.conf` contained `PasswordAuthentication yes`, which took precedence. Diagnose any override with:

```bash
sudo grep -RnEi '^[[:space:]]*(Include|PasswordAuthentication|Match)[[:space:]]' \
  /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

For this installation, `/etc/ssh/sshd_config.d/50-cloud-init.conf` was edited to read:

```ini
PasswordAuthentication no
```

> **Cloud-init caution:** A cloud-init-managed file may be regenerated during reprovisioning. Review the image's cloud-init settings and confirm key-only SSH persists after applicable provisioning changes. Do not assume editing this file permanently changes cloud-init's source configuration.

Check again:

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication) '
```

Required output:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

Apply the verified configuration:

```bash
sudo systemctl reload ssh
sudo systemctl status ssh --no-pager
```

From the **management desktop**, open a **new terminal** and verify the dedicated key still works:

```bash
ssh -i ~/.ssh/id_ed25519_technitium \
  -o IdentitiesOnly=yes \
  -o PreferredAuthentications=publickey \
  -o PasswordAuthentication=no \
  -o KbdInteractiveAuthentication=no \
  username@192.168.xxx.xx
```

Then test that the server rejects password-only authentication:

```bash
ssh -o PubkeyAuthentication=no \
  -o PreferredAuthentications=password \
  -o PasswordAuthentication=yes \
  -o KbdInteractiveAuthentication=no \
  -o NumberOfPasswordPrompts=1 \
  username@192.168.xxx.xx
```

Expected: `Permission denied (publickey).` **Do not close your original session until the new key-based login succeeds.** SSH password authentication and `sudo` password prompts are separate; disabling the former does not disable the latter.

Protect and securely back up `~/.ssh/id_ed25519_technitium` (the **private** key). Never commit it to GitHub.

## Step 6 - November 16, 2026 Verification Record (Standalone Test Host Only)

| Check | Test result |
| --- | --- |
| Technitium v15.6.2 service running | PASS |
| Web UI administrator access restored | PASS |
| UFW active, default deny inbound / allow outbound | PASS |
| DNS from authorized desktop over UDP 53 | PASS |
| DNS from authorized desktop over TCP 53 | PASS |
| Wazuh agent established TCP 1514 connection | PASS |
| AdGuard primary TCP 853 TLS/certificate test | PASS (reported) |
| AdGuard secondary TCP 853 TLS/certificate test | PASS (reported) |
| Actual Technitium DoT forwarding observed by packet capture | NOT TESTED |
| Fresh SSH login after UFW enable | PASS |
| Dedicated Ed25519 SSH key login after SSH reload | PASS |
| SSH password-only login rejected | PASS |
| Effective SSH root login disabled | PASS |
| Reboot persistence of latest firewall/SSH changes | NOT TESTED |
| Production primary/secondary cluster verification | NOT TESTED by these steps |

> **Follow-up hardening:** Restrict HTTP management to trusted IPs and move to HTTPS where feasible; verify Wazuh and DNS after reboot; test actual DoT forwarding; and review cloud-init persistence. Do not mark these checks PASS until performed.

---

# References

- [Technitium DNS Server](https://technitium.com/dns/)
- [Technitium DNS GitHub](https://github.com/TechnitiumSoftware/DnsServer)
- [Technitium blog: Understanding Clustering and How to Configure It](https://blog.technitium.com/2025/11/understanding-clustering-and-how-to.html)
- [Ubuntu Server](https://ubuntu.com/download/server)
- [Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/)
- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
- [Ubuntu Chrony Time Synchronization](https://ubuntu.com/server/docs/how-to/networking/chrony-client/)
- [Ubuntu systemd-timesyncd Time Synchronization](https://ubuntu.com/server/docs/use-timedatectl-and-timesyncd/)

---

**Documentation maintained for the Technitium DNS home network project.**
