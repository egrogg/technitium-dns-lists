# How to Install Technitium DNS Server

This guide provides step-by-step instructions for installing
Technitium DNS Server on two different platforms:

1. Proxmox VE using an Ubuntu Server virtual machine.
2. Raspberry Pi 4 using Ubuntu Server.
3. Configure DHCP reservations in your network gateway.

## Network Configuration and IP Addresses

This installation guide uses the following network configuration
as an example.

| Device | Role | Example IP Address |
|--------|------|--------------------|
| Proxmox VM | Primary Technitium DNS | 192.168.10.10 |
| Raspberry Pi 4 | Secondary Technitium DNS | 192.168.10.11 |
| Network Gateway | Router / Firewall | 192.168.10.1 |

> **Important:** The IP addresses shown throughout this guide
> are illustrative addresses, not the actual IP addresses of my home network.
>
> Replace these addresses with the IP addresses assigned
> to your own network.
>
> For example, if your network uses `192.168.10.0/24`,
> you might configure:
>
> - Primary DNS: `192.168.10.10`
> - Secondary DNS: `192.168.10.11`
>
> Ensure that both addresses are available and do not
> conflict with other devices.
> Some command examples use `192.168.xxx.xx` as a placeholder.
> Replace it with the appropriate primary or secondary server IP.
> **Do not run both primary and secondary tests against the same address.**

### DHCP Reservations

For this installation, DHCP reservations are configured
on the network gateway to ensure each DNS server
receives a consistent IP address.

A DHCP reservation associates a device's MAC address
with a specific IP address.

This allows the server to continue using DHCP while
receiving the same IP address.

> **Note:** A DHCP reservation is different from
> manually configuring a static IP address in Ubuntu.
> Either method can provide a consistent address,
> but the configuration process is different.

---

## Table of Contents

- [Part 1 - Proxmox VM Installation](#part-1---proxmox-vm-installation)
- [Part 2 - Raspberry Pi 4 Installation](#part-2---raspberry-pi-4-installation)
- [Part 3 - Initial Technitium Configuration](#part-3---initial-technitium-configuration)
- [Part 4 - Verify DNS Resolution](#part-4---verify-dns-resolution)
- [Part 5 - Troubleshooting](#part-5---troubleshooting)

---

# Part 1 - Proxmox VM Installation

## Step 1 - Download Ubuntu Server

Download the latest supported Ubuntu Server LTS ISO.

**Download:**
https://ubuntu.com/download/server

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
|---------|---------------|
| VM Name | technitium-dns |
| Operating System | Ubuntu Server LTS |
| CPU | 2 Cores |
| Memory | 4096 MB |
| Disk | 32 GB |
| Network Model | VirtIO |
| Network Bridge | vmbr0 |

> **Note:** These are suggested starting resources.
> Adjust them based on your environment.

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

```bash
sudo apt update
```

Upgrade installed packages:

```bash
sudo apt upgrade -y
```

Identify the current IP address and MAC address. If using a DHCP reservation,
create it on the gateway or DHCP server. A DHCP reservation is not the same
as setting a static IP inside Ubuntu.  

```bash
ip a
```
To identify the network interface and MAC address more clearly, you can also use:  
```bash
ip -br addr
ip link show
```
These commands help identify the network interface, assigned IP address, and MAC address.  

After creating the reservation, renew the DHCP lease or reboot to request the reserved IP:  

```bash
sudo reboot
```

## Step 6 - Install Technitium DNS Server

Run the official Technitium installation script:

```bash
curl -sSL https://download.technitium.com/dns/install.sh | sudo bash
```

Follow the installation prompts.

## Step 7 - Verify Technitium Service

Check whether Technitium DNS is running:

```bash
sudo systemctl status dns.service
```

The expected service status is:

```text
Active: active (running)
```

## Step 8 - Access the Technitium Web Interface

Open a web browser and navigate to the Primary DNS server IP:

```text
http://192.168.xxx.xx:5380
```

Replace the IP address with your Technitium server's IP address.

Complete the initial setup wizard.

---

# Part 2 - Raspberry Pi 4 Installation

## Step 1 - Download Raspberry Pi Imager

Download Raspberry Pi Imager:

https://www.raspberrypi.com/software/

## Step 2 - Prepare the Raspberry Pi

1. Insert the microSD card into your computer.
2. Launch **Raspberry Pi Imager**.
3. Under **Raspberry Pi Device**, select **Raspberry Pi 4**.
4. Under **Operating System**, select:
   - **Other general-purpose OS**
   - **Ubuntu**
   - **Ubuntu Server 24.04 LTS (64-bit)**
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


## Step 3 - Boot the Raspberry Pi

1. Insert the microSD card into the Raspberry Pi.
2. Connect the Raspberry Pi to the network.
3. Connect power.
4. Wait for the Raspberry Pi to boot.

## Step 4 - Connect Using SSH

From another Linux computer: (this is for the Secondary Technitium DNS server)

```bash
ssh username@192.168.xxx.xx
```

Replace the username and IP address with your configuration.

## Step 5 - Update Ubuntu Server

Update the package repositories:

```bash
sudo apt update
```

Upgrade installed packages:

```bash
sudo apt upgrade -y
```

## Step 6 - Install Technitium DNS Server

Run:

```bash
curl -sSL https://download.technitium.com/dns/install.sh | sudo bash
```

## Step 7 - Verify Technitium Service

```bash
sudo systemctl status dns.service
```

Expected result:

```text
Active: active (running)
```

## Step 8 - Access the Web Interface

Open: (This is for the Secondary Technitium DNS Server)

```text
http://192.168.xxx.xx:5380
```

Complete the initial setup wizard.

---
## Raspberry Pi 4 — Ubuntu Server Time Synchronization

### Overview

> **Optional troubleshooting procedure:** This section records a specific
> time synchronization problem encountered during the original installation.
> First check whether the default Ubuntu time synchronization works after
> reboot. Apply the workaround only if needed, after considering the security
> implications of unauthenticated NTP bootstrap sources and the `trust` directive.

The Raspberry Pi 4 does not include a built-in battery-backed
real-time clock (RTC).

When the Raspberry Pi loses power or reboots, Ubuntu Server
may start with an incorrect system date and time.

This can cause problems with Technitium DNS Server,
particularly when DNSSEC validation is enabled.

### Environment

| Component | Configuration |
|-----------|---------------|
| Device | Raspberry Pi 4 |
| Operating System | Ubuntu Server |
| DNS Server | Technitium DNS |
| Server Role | Secondary DNS |
| IP Address | 192.168.xxx.xx |
| Time Synchronization | Chrony |

---

### Problem

After rebooting, the Raspberry Pi started with an incorrect
system clock.

Chrony was configured to use authenticated Network Time
Security (NTS) servers.

Because the system clock was incorrect, TLS certificate
validation failed.

The resulting sequence was:

```text
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

```text
Signature Not Yet Valid
```

---

### Solution — Configure Chrony Bootstrap Sources

The following configuration was used to resolve the
boot-time synchronization issue.

> **Security Note:** The bootstrap sources below use
> unauthenticated NTP. The `trust` directive explicitly
> trusts these sources, so their integrity is important.
> This configuration restores time synchronization but
> does not provide the same authentication guarantees
> as NTS. Use only trusted time sources. Noted below
> are the two that I used to get it working.

### Step 1 — Configure Bootstrap NTP Sources

Create the Chrony bootstrap source file:

```bash
sudo nano /etc/chrony/sources.d/bootstrap-ip.sources
```

Add:

```text
server 91.189.91.111 iburst trust
server 185.125.190.121 iburst trust
```

Save the file.

Verify the configuration:

```bash
cat /etc/chrony/sources.d/bootstrap-ip.sources
```

> **Note:** These are the IP addresses used during the
> original repair. Verify that they remain valid before
> deploying this configuration on another system.

---

### Step 2 — Modify Chrony Authentication Selection

Open the Chrony configuration:

```bash
sudo nano /etc/chrony/chrony.conf
```

Add the following directive:

```text
authselectmode ignore
```

Verify that the configuration also contains:

```text
makestep 1 3
```

The relevant settings should look like:

```text
authselectmode ignore
makestep 1 3
```

#### Explanation

**authselectmode ignore**

Prevents authentication status from automatically
controlling Chrony's source-selection process.

This permits the configured bootstrap NTP sources
to participate in time synchronization.

**makestep 1 3**

Allows Chrony to step the system clock when the
time offset exceeds one second during the first
three clock updates.

---

### Step 3 — Restart Chrony

Restart the service:

```bash
sudo systemctl restart chrony
```

Wait approximately 20–30 seconds.

Check the system time:

```bash
date
```

Check synchronization status:

```bash
timedatectl
```

Check Chrony tracking:

```bash
chronyc tracking
```

Check available NTP sources:

```bash
chronyc sources -v
```

A successful synchronization should eventually show:

```text
System clock synchronized: yes
NTP service: active
Leap status: Normal
```

> **Note:** Synchronization may take longer than
> 30 seconds depending on network availability
> and the initial clock offset.

---

### Step 4 — Verify After Reboot

Reboot the Raspberry Pi:

```bash
sudo reboot
```

After reconnecting through SSH, run:

```bash
date
timedatectl
chronyc tracking
```

Verify the following:

1. The system date and time are correct.
2. Chrony has synchronized successfully.
3. The NTP service is active.
4. The Chrony leap status is Normal.

Expected results:

```text
System clock synchronized: yes
NTP service: active
Leap status: Normal
```

Correctly reporting will show the following:

```bash
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

---

### Step 5 — Verify Technitium DNS

After confirming that the system clock is synchronized,
test DNS resolution: Use the Secondary Technitium DNS server IP:

```bash
dig @192.168.xxx.xx google.com
```

Verify that the response contains:

```text
status: NOERROR
```

Check that the DNSSEC time-validation error
is no longer occurring.

If DNS resolution still fails, review the
Technitium DNS logs and Chrony synchronization status.

---

### Important Maintenance Considerations

The bootstrap NTP sources use hard-coded IP addresses.

If these addresses change or become unavailable,
the Raspberry Pi may fail to synchronize its clock.

Periodically verify:

```bash
cat /etc/chrony/sources.d/bootstrap-ip.sources
```
Results should  show:

```bash
server 91.189.91.111 iburst trust
server 185.125.190.121 iburst trust

```
You can also run

```bash
chronyc sources -v
```
This displays the NTP sources Chrony knows about and their selection status.

You can also run:
```bash
chronyc tracking
```
This verifies Chrony's current synchronization status.

---

### Result

The original Raspberry Pi 4 configuration successfully
recovered its system time after reboot.

Chrony synchronized automatically, allowing Technitium
DNSSEC validation to operate normally.

**Status: Tested successfully on the original installation.**


# Part 3 - Initial Technitium Configuration

After installing Technitium DNS Server on both the
Proxmox Ubuntu Server VM and Raspberry Pi 4,
the next step is to configure and verify each DNS server.

For this installation:

| Server | Role | IP Address |
|--------|------|------------|
| Proxmox Ubuntu Server VM | Primary DNS | 192.168.xxx.xx |
| Raspberry Pi 4 Ubuntu Server | Secondary DNS | 192.168.xxx.xx |

> **Important:** Replace these IP addresses with
> the addresses assigned to your own DNS servers.

---

## Step 1 - Access the Technitium Web Interface

From a computer connected to your local network,
open a web browser.

### Primary DNS Server

Navigate to:

```text
http://192.168.xxx.xx:5380
```

### Secondary DNS Server

Navigate to:

```text
http://192.168.xxx.xx:5380
```

Technitium uses TCP port `5380` for its default
HTTP administrative web interface.

Complete the initial setup wizard if prompted.

> **Security Note:** The HTTP administrative interface
> is not encrypted. Use it only on a trusted management
> network during initial setup. Configure HTTPS for
> ongoing administration.

---

## Step 2 - Configure Administrator Credentials

When accessing Technitium DNS Server for the first time,
complete the initial administrator setup.

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

After logging in to the Technitium DNS web interface,
verify that the DNS server is listening on the
correct network interfaces.

### Access DNS Server Settings

1. Log in to the Technitium DNS web interface.
2. Navigate to **Settings**.
3. Locate **DNS Server Local End Points**.
4. Review the configured listening addresses.

### DNS Server Local End Points

By default, Technitium may display:

```text
0.0.0.0:53
[::]:53
```

These settings mean:

| Setting | Description |
|---------|-------------|
| `0.0.0.0:53` | Listen on all IPv4 network interfaces |
| `[::]:53` | Listen on all IPv6 network interfaces |

### IPv4-Only Configuration (Optional)

If your network does not use IPv6, you may configure
Technitium to listen only on IPv4.

Change:

```text
0.0.0.0:53
[::]:53
```

To:

```text
0.0.0.0:53
```

> **Important:** Removing the IPv6 listening address
> does not disable IPv6 in Ubuntu Server.
> It only prevents Technitium from listening
> for DNS requests over IPv6.

### DNS Server IPv4 Source Addresses

Locate:

**DNS Server IPv4 Source Addresses**

The default value is:

```text
0.0.0.0
```

This allows the operating system to select the
appropriate IPv4 source address for outbound
DNS requests.

For a server with one network interface,
this setting can normally remain unchanged.

### DNS Server IPv6 Source Addresses

Locate:

**DNS Server IPv6 Source Addresses**

The default value is:

```text
::
```

This setting controls the IPv6 source address
used for outbound DNS requests when IPv6 is enabled.

If IPv6 is not being used, no changes are
normally necessary.

### Save Configuration

If changes were made:

1. Save the Technitium settings.
2. Verify that the DNS service remains operational.
3. Test DNS resolution from another computer.

### Verify DNS Port 53

From the Ubuntu Server terminal, run:

```bash
sudo ss -tulpn | grep ':53 '
```

Verify that Technitium is listening on port 53.

DNS uses:

- UDP port 53
- TCP port 53

### Test DNS Resolution

From another computer, run:

```bash
dig @192.168.xxx.xx google.com A
```

For the secondary server:

```bash
dig @192.168.xxx.xx google.com A
```

### Expected Result

```text
status: NOERROR
```

The response should also contain an ANSWER SECTION
with one or more IPv4 addresses.

> **Note:** Replace the example IP addresses
> with the addresses assigned to your DNS servers.

---

**Result:** DNS server network settings verified.
---

## Step 4 - Configure DNS Forwarders

Technitium DNS Server can resolve DNS queries recursively
or forward requests to another DNS resolver.

For this installation, we will configure DNS forwarding
using AdGuard DNS with DNS over TLS (DoT).

DNS over TLS encrypts DNS queries between Technitium
and the upstream DNS provider.

---

### Configure Forwarders

1. Log in to the Technitium DNS web interface.
2. Navigate to **Settings**.
3. Select **Proxy & Forwarders**.
4. Locate the **Forwarders** section.
5. Use **Quick Select** to add the desired DNS providers,
   or manually enter the forwarder addresses.
6. Configure the forwarder addresses.
7. Set **Forwarder Protocol** to **DNS-over-TLS**.
8. Enable **Concurrent Forwarding**.
9. Set **Forwarder Concurrency** to `2`.
10. Save the configuration.

---

### DNS Forwarder Configuration
The following AdGuard DNS-over-TLS forwarders
are configured in Technitium:

```text
dns.adguard-dns.com (94.140.14.14:853)
dns.adguard-dns.com (94.140.15.15:853)
```

### Forwarder Settings

| Setting | Configuration |
|---------|---------------|
| Forwarder Protocol | DNS-over-TLS |
| Concurrent Forwarding | Enabled |
| Forwarder Concurrency | 2 |

These settings allow Technitium to forward DNS
queries to AdGuard DNS using encrypted connections.

With concurrent forwarding enabled and concurrency
set to 2, Technitium can query both configured
forwarders simultaneously.

The server selects forwarders based on their
average latency and uses the fastest response.

> **Note:** DNS over TLS encrypts DNS traffic
> between Technitium and AdGuard DNS.
> It does not automatically encrypt DNS traffic
> between client devices and Technitium.

For this installation:

```text
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

> **Note:** Concurrent forwarding may send the
> same DNS query to multiple upstream providers.
>
> This can improve availability but may increase
> upstream DNS traffic.

---

### DNS over TLS (DoT)

Under **Forwarder Protocol**, select:

**DNS-over-TLS**

DNS over TLS uses TCP port `853` by default.

This encrypts DNS queries sent from Technitium
to the configured upstream DNS provider.

For AdGuard DNS, the TLS hostname is:

```text
dns.adguard-dns.com
```

The corresponding IPv4 addresses are:

```text
94.140.14.14
94.140.15.15
```

> **Important:** Configuring a DNS server IP address
> does not automatically enable encrypted DNS.
>
> The **Forwarder Protocol** must be set to
> **DNS-over-TLS**.
>
> The correct TLS hostname must also be used
> for certificate validation.

---

### Verify Forwarder Configuration

After saving the settings, verify DNS resolution
from another computer on the network.

#### Test Primary Technitium DNS Server

```bash
dig @192.168.xxx.xx google.com A
```

#### Test Secondary Technitium DNS Server

```bash
dig @192.168.xxx.xx google.com A
```

### Expected Results

Both DNS servers should return:

```text
status: NOERROR
```

The response should also contain an ANSWER SECTION
with one or more IPv4 addresses.

> **Important:** Replace the example IP addresses
> with the IP addresses assigned to your
> Technitium DNS servers.

---

### Verify DNS over TLS Connectivity

On the Ubuntu Server running Technitium,
you can test connectivity to the AdGuard
DNS-over-TLS endpoint.

```bash
openssl s_client \
-connect 94.140.14.14:853 \
-servername dns.adguard-dns.com \
-verify_hostname dns.adguard-dns.com \
-verify_return_error </dev/null
```

Look for:

```text
Verification: OK
Verify return code: 0 (ok)
```

This verifies that the server can establish a TLS
connection and validate the certificate presented
by the upstream endpoint.

> **Note:** This test confirms TLS connectivity
> and certificate validation.
>
> It does not independently prove that Technitium
> is forwarding DNS queries using TLS.

---

### Configuration Complete

The Technitium DNS servers are now configured
to forward DNS queries using encrypted DNS over TLS.

Configuration summary:

- DNS Provider: AdGuard DNS
- Forwarder Protocol: DNS over TLS
- DNS Port: TCP 853
- Concurrent Forwarding: Enabled
- Forwarder Concurrency: 2

**Status: Configuration documented.**

---

## Step 5 - Verify DNS Resolution

After configuring the DNS forwarders, verify that both
Technitium DNS servers can successfully resolve domain names.

Perform these tests from another computer on your network.

> **Important:** Replace the example IP addresses below
> with the actual IP addresses assigned to your
> primary and secondary Technitium DNS servers.

### Test Primary DNS (UDP)

Open a terminal and run:

```bash
dig @192.168.xxx.xx google.com A
```

### Test Secondary DNS (UDP)

```bash
dig @192.168.xxx.xx google.com A
```

### Expected Results

Both DNS servers should return:

```text
status: NOERROR
```

The response should also contain an ANSWER SECTION
with one or more IPv4 addresses.

Example:

```text
;; ANSWER SECTION:
google.com.    300    IN    A    142.250.190.14
```

> **Note:** The returned IP address and TTL may differ.

### Verify DNS Server Response

At the bottom of the `dig` output, locate the SERVER line.

Example for the primary server:

```text
;; SERVER: 192.168.xxx.xx#53
```

This confirms which DNS server responded to the query.

---

### Verify DNS over TCP

DNS normally uses UDP port 53, but TCP port 53
must also be available for proper DNS operation.

#### Test Primary DNS (TCP)

```bash
dig +tcp @192.168.xxx.xx google.com A
```

#### Test Secondary DNS (TCP)

```bash
dig +tcp @192.168.xxx.xx google.com A
```

### Expected Results

Both DNS servers should return:

```text
status: NOERROR
```

The response should also contain an ANSWER SECTION
with one or more IPv4 addresses.

Verify that the SERVER line identifies the
correct Technitium DNS server.

---

### Verification Summary

| Test | Expected Result |
|------|-----------------|
| Primary DNS - UDP | NOERROR + IPv4 address |
| Secondary DNS - UDP | NOERROR + IPv4 address |
| Primary DNS - TCP | NOERROR + IPv4 address |
| Secondary DNS - TCP | NOERROR + IPv4 address |

If all four tests complete successfully,
both Technitium DNS servers are responding
to DNS queries over UDP and TCP.

**Status: DNS resolution verified after successful testing.**

---


## Step 6 - Verify DNS Server Availability

After verifying DNS resolution in Step 5, confirm that both Technitium DNS servers are reachable over the network.

Perform these tests from another computer on your network.

> **Important:** Replace the example IP addresses with the actual addresses assigned to your primary and secondary Technitium DNS servers.

### Test Primary DNS Server

Open a terminal and run:

```bash
ping -c 4 192.168.xxx.xx
```

### Test Secondary DNS Server

```bash
ping -c 4 192.168.xxx.xx
```

### Expected Results

Both servers should respond to the ping requests.

Example:

```text
64 bytes from 192.168.xxx.xx: icmp_seq=1 ttl=64 time=0.532 ms
64 bytes from 192.168.xxx.xx: icmp_seq=2 ttl=64 time=0.487 ms
64 bytes from 192.168.xxx.xx: icmp_seq=3 ttl=64 time=0.501 ms
64 bytes from 192.168.xxx.xx: icmp_seq=4 ttl=64 time=0.493 ms
```

At the bottom of the output, look for:

```text
4 packets transmitted, 4 received, 0% packet loss
```

### Understanding the Results

| Result | Meaning |
|---|---|
| 0% packet loss | All ping requests received responses |
| 25% packet loss | One of four requests did not receive a response |
| 50% packet loss | Two of four requests did not receive responses |
| 100% packet loss | No ping responses were received |
| Destination Host Unreachable | The destination could not be reached from the reporting device |

### Important Notes

> **ICMP Traffic:** Some networks block ICMP traffic through firewall rules.
>
> ICMP has been enabled in my network configuration to allow connectivity testing.
>
> A failed ping does not necessarily mean the Technitium DNS server is unavailable.
>
> The DNS resolution tests performed in Step 5 provide a more meaningful verification of DNS functionality.

### Verification Summary

| Test | Expected Result |
|---|---|
| Primary DNS Server | Ping replies received |
| Secondary DNS Server | Ping replies received |
| Packet Loss | Ideally 0% |

**Status: Network connectivity verified after successful testing.**

---
## Step 7 - Verify Technitium Services

After confirming network connectivity, verify that Technitium DNS Server is running and configured to start automatically when Ubuntu Server boots.

Perform the following steps on **both Technitium DNS servers**:

- Primary DNS — Proxmox Ubuntu Server VM
- Secondary DNS — Raspberry Pi 4 running Ubuntu Server

### Step 7.1 - Verify Technitium Service Status

Connect to the Ubuntu Server using SSH.

Run:

```bash
sudo systemctl status dns.service
```

### Expected Result

Look for the following line:

```text
Active: active (running)
```

This confirms that the Technitium DNS service is currently running.

### Step 7.2 - Verify Automatic Startup

To verify that Technitium starts automatically when Ubuntu Server boots, run:

```bash
sudo systemctl is-enabled dns.service
```

### Expected Result

```text
enabled
```

This confirms that the service is configured to start automatically during the normal system boot process.

### Step 7.3 - Enable Automatic Startup (If Necessary)

If the service is not enabled, run:

```bash
sudo systemctl enable dns.service
```

Verify the setting again:

```bash
sudo systemctl is-enabled dns.service
```

Expected:

```text
enabled
```

> **Note:** Enabling a service does not necessarily start it immediately. It configures the service to start during the normal boot process.

### Step 7.4 - Start Technitium (If Not Running)

If the service is not running, start it using:

```bash
sudo systemctl start dns.service
```

Verify the service status:

```bash
sudo systemctl status dns.service
```

Expected:

```text
Active: active (running)
```

### Step 7.5 - Troubleshoot Service Issues

If Technitium fails to start, review the service logs:

```bash
sudo journalctl -u dns.service -n 50 --no-pager
```

This displays the 50 most recent journal entries associated with the Technitium DNS service.

Check for errors related to:

- Service startup failures.
- Port 53 conflicts.
- Configuration errors.
- File or directory permissions.
- Missing dependencies.

> **Important:** Do not repeatedly restart the service without reviewing the error messages. Identify the cause before making configuration changes.

### Verification Summary

| Verification | Expected Result |
|---|---|
| Primary DNS service | Active (running) |
| Secondary DNS service | Active (running) |
| Primary automatic startup | Enabled |
| Secondary automatic startup | Enabled |
| Service logs | No unresolved critical startup errors |

### Result

If both Technitium DNS services are running and enabled, the service verification is complete.

**Status: Technitium services verified after successful testing.**

---
## Step 8 - Verify Configuration After Reboot

After completing the initial configuration, verify that both Technitium DNS servers start automatically and function correctly after a reboot.

This test confirms that:

- Ubuntu Server boots successfully.
- Technitium DNS starts automatically.
- DNS resolution works after reboot.
- The Raspberry Pi 4 synchronizes its system clock.
- DNSSEC-related time synchronization problems do not return.

> **Important:** Reboot one DNS server at a time to avoid unnecessary DNS interruptions. If these servers are already serving network clients, verify that the remaining DNS server can handle requests during the reboot.

---

### Step 8.1 - Reboot the Primary DNS Server

Connect to the primary Technitium DNS server running on the Proxmox Ubuntu Server VM.

Run:

```bash
sudo reboot
```

Wait for Ubuntu Server to finish rebooting.

Reconnect using SSH.

### Verify Technitium Service

Run:

```bash
sudo systemctl status dns.service
```

Expected:

```text
Active: active (running)
```

Verify automatic startup:

```bash
systemctl is-enabled dns.service
```

Expected:

```text
enabled
```

### Verify Primary DNS Resolution

From another computer on your network, run:

```bash
dig @192.168.xxx.xx google.com A
```

Expected:

```text
status: NOERROR
```

The response should also contain an ANSWER SECTION with one or more IPv4 addresses.

> **Note:** Replace the example IP address with the address assigned to your primary Technitium DNS server.

---

### Step 8.2 - Reboot the Secondary DNS Server

After verifying that the primary server is functioning correctly, connect to the Raspberry Pi 4 running Ubuntu Server.

Run:

```bash
sudo reboot
```

Wait for the Raspberry Pi to finish rebooting.

Reconnect using SSH.

### Verify Technitium Service

Run:

```bash
sudo systemctl status dns.service
```

Expected:

```text
Active: active (running)
```

Verify automatic startup:

```bash
systemctl is-enabled dns.service
```

Expected:

```text
enabled
```

### Verify Secondary DNS Resolution

From another computer on your network, run:

```bash
dig @192.168.xxx.xx google.com A
```

Expected:

```text
status: NOERROR
```

The response should also contain an ANSWER SECTION with one or more IPv4 addresses.

---

### Step 8.3 - Verify Raspberry Pi Time Synchronization

The Raspberry Pi 4 does not have a built-in battery-backed real-time clock (RTC).

Because of this, Ubuntu Server may initially boot with an incorrect system time.

Earlier in this guide, Chrony was configured to correct the system clock automatically.

We must verify that this configuration continues to work after reboot.

**Do not manually set the system time or restart Chrony before performing these checks.**

On the Raspberry Pi, run:

```bash
date
```

Verify that the current date and time are correct.

Next, run:

```bash
timedatectl
```

Expected:

```text
System clock synchronized: yes
NTP service: active
```

Check Chrony tracking:

```bash
chronyc tracking
```

Expected:

```text
Leap status     : Normal
```

Finally, verify the available NTP sources:

```bash
chronyc sources -v
```

Look for a selected time source indicated by:

```text
^*
```

> **Note:** Synchronization may take some time after reboot. If the clock is initially incorrect, wait for Chrony to synchronize and check again.
>
> If synchronization does not occur, review the Chrony boot-time configuration documented earlier in this guide.

---

### Step 8.4 - Verify DNSSEC Behavior After Reboot

Because an incorrect system clock previously caused DNSSEC validation errors on the Raspberry Pi, perform an additional DNSSEC test.

From another computer, run:

```bash
dig @192.168.xxx.xx dnssec-failed.org A
```

Use the IP address of the secondary Technitium DNS server.

### Expected Result

```text
status: SERVFAIL
```

The response may also include:

```text
EDE: 6 (DNSSEC Bogus)
```

The domain `dnssec-failed.org` intentionally contains invalid DNSSEC signatures.

A DNSSEC-validating resolver should reject the response.

> **Important:** If an upstream DNS provider performs DNSSEC validation, the rejection may occur upstream rather than within Technitium itself.
>
> This test confirms that invalid DNSSEC data is rejected somewhere in the DNS resolution path.

### Verify Normal DNS Resolution

Also test a legitimate domain:

```bash
dig @192.168.xxx.xx cloudflare.com A
```

Expected:

```text
status: NOERROR
```

The response should contain one or more IPv4 addresses.

Together, these tests help confirm that normal DNS resolution works and invalid DNSSEC data is rejected after reboot.

---

### Step 8.5 - Final Verification

After both servers have been rebooted and tested, verify the following:

| Verification | Primary DNS | Secondary DNS |
|---|---|---|
| Ubuntu Server boots successfully | PASS / FAIL | PASS / FAIL |
| Technitium service running | PASS / FAIL | PASS / FAIL |
| Technitium automatic startup enabled | PASS / FAIL | PASS / FAIL |
| DNS resolution successful | PASS / FAIL | PASS / FAIL |
| DNSSEC-invalid domain rejected | Not tested | PASS / FAIL |
| Chrony synchronized | Not tested | PASS / FAIL |

> **Note:** Replace PASS / FAIL with the actual results after completing the tests. A test that has not been performed should be marked NOT TESTED.

---

## Initial Configuration Complete

After successfully completing all verification steps:

- Both Technitium DNS servers are installed.
- Administrator credentials have been configured.
- DNS listening settings have been reviewed.
- DNS forwarding has been configured.
- UDP and TCP DNS resolution have been tested.
- Both Technitium services are running.
- Automatic service startup has been verified.
- Both servers have been tested after reboot.
- Raspberry Pi time synchronization has been verified.
- DNSSEC behavior has been tested after reboot.

**The initial Technitium DNS installation and configuration are complete after all required tests pass.**

> **Next Step:** Configure Technitium DNS clustering to connect the primary and secondary servers.
>
> Clustering will be documented separately.

---
# Part 4 - Verify DNS Resolution

After installing and configuring both Technitium DNS servers,
perform the following tests to verify DNS resolution,
network connectivity, DNSSEC behavior, and DNS forwarding.

These tests should be performed after completing the initial
configuration and reboot verification in Part 3.

> **Important:** Replace the example IP addresses throughout
> this section with the actual IP addresses assigned to your
> primary and secondary Technitium DNS servers.

---

## Step 1 - Install DNS Utilities

From another Linux computer on your network,
install the DNS utilities if necessary.

On Ubuntu:

```bash
sudo apt update
sudo apt install dnsutils -y
```

The `dig` utility allows you to send DNS queries directly
to a specified DNS server.

---

## Step 2 - Verify Primary and Secondary DNS Resolution

Verify that both Technitium DNS servers can resolve
external domain names.

### Test Primary DNS Server

```bash
dig @192.168.xxx.xx google.com A
```

### Test Secondary DNS Server

```bash
dig @192.168.xxx.xx google.com A
```

### Expected Results

Both queries should return:

```text
status: NOERROR
```

The response should contain an ANSWER SECTION
with one or more IPv4 addresses.

Example:

```text
;; ANSWER SECTION:
google.com.    300    IN    A    142.250.190.14
```

> **Note:** The returned IP addresses and TTL values
> may differ from the example.

At the bottom of the response, verify the SERVER line:

```text
;; SERVER: 192.168.xxx.xx#53
```

This confirms which DNS server responded.

---

## Step 3 - Verify DNS over TCP

DNS normally uses UDP port 53, but TCP port 53
is also required for proper DNS operation.

Verify that both Technitium servers respond
to DNS queries over TCP.

### Test Primary DNS Server

```bash
dig +tcp @192.168.xxx.xx google.com A
```

### Test Secondary DNS Server

```bash
dig +tcp @192.168.xxx.xx google.com A
```

### Expected Results

Both queries should return:

```text
status: NOERROR
```

An ANSWER SECTION containing one or more IPv4 addresses
should also be present.

If either query fails, verify that TCP port 53 is allowed
through the server firewall and applicable network firewall rules.

---

## Step 4 - Verify DNSSEC Behavior

DNS Security Extensions (DNSSEC) help protect DNS
responses against certain forms of tampering.

The following tests check how the DNS resolution path
handles valid and invalid DNSSEC data.

### Test a Valid Domain

Run the following command against the primary server:

```bash
dig @192.168.xxx.xx cloudflare.com A +dnssec
```

Repeat the test against the secondary server:

```bash
dig @192.168.xxx.xx cloudflare.com A +dnssec
```

### Expected Results

Both queries should return:

```text
status: NOERROR
```

The response should contain an ANSWER SECTION
with one or more IPv4 addresses.

Depending on the resolver configuration,
the response may contain the authenticated data flag:

```text
flags: qr rd ra ad
```

The `ad` flag indicates that the response has been
authenticated using DNSSEC.

> **Note:** The absence of the `ad` flag does not
> necessarily mean that DNS resolution failed.
> The resolver configuration and upstream DNS provider
> can affect whether the flag appears.

### Test an Invalid DNSSEC Domain

The domain `dnssec-failed.org` intentionally contains
invalid DNSSEC signatures.

A DNSSEC-validating resolver should reject its records.

#### Test Primary DNS Server

```bash
dig @192.168.xxx.xx dnssec-failed.org A
```

#### Test Secondary DNS Server

```bash
dig @192.168.xxx.xx dnssec-failed.org A
```

### Expected Results

```text
status: SERVFAIL
```

The response may also contain:

```text
EDE: 6 (DNSSEC Bogus)
```

### Previously Recorded Test Results

**Test Date:** October 19, 2026

The following results were recorded during testing
of the secondary Technitium DNS server:

```text
status: SERVFAIL
EDE: 6 (DNSSEC Bogus)
EDE: 22 (No Reachable Authority)
EDE: 13 (Cached Error)
```

### Explanation

The invalid DNSSEC domain was rejected.

This is the expected behavior when DNSSEC validation
is performed successfully somewhere in the DNS
resolution path.

**Important:** When Technitium uses an upstream DNS
provider that performs DNSSEC validation, the upstream
provider may reject the invalid response.

Therefore, this test does not independently prove
that Technitium performed DNSSEC validation.

### Recorded Result

```text
Secondary DNS - Invalid DNSSEC Domain: PASS
```

> **Note:** The PASS result above refers to the
> October 19, 2026 test. Other verification results
> should be recorded after completing their tests.

---

## Step 5 - Verify DNS-over-TLS Forwarding

If DNS-over-TLS was configured in Part 3,
verify that the configured upstream DNS servers
are reachable.

For this installation, AdGuard DNS was configured
as the upstream provider.

### Configured Forwarders

| Setting | Configuration |
|---|---|
| DNS Provider | AdGuard DNS |
| Primary Forwarder | 94.140.14.14 |
| Secondary Forwarder | 94.140.15.15 |
| Forwarder Hostname | dns.adguard-dns.com |
| Forwarder Protocol | DNS-over-TLS |
| Port | TCP 853 |
| Concurrent Forwarding | Enabled |
| Forwarder Concurrency | 2 |

> **Note:** These are the forwarders used in this
> installation. Other DNS providers may use different
> addresses, hostnames, and protocols.

### Step 5.1 - Verify Forwarder Configuration

Log in to the Technitium DNS web interface.

1. Open **Settings**.
2. Navigate to **Proxy & Forwarders**.
3. Locate the configured forwarders.
4. Verify the forwarder addresses and hostnames.
5. Verify that the forwarder protocol is set to **DNS-over-TLS**.
6. Confirm that the settings have been saved.

Perform this verification on both DNS servers.

### Step 5.2 - Test DNS-over-TLS Connectivity

Connect to the primary Ubuntu DNS server using SSH.

Run:

```bash
openssl s_client \
  -connect 94.140.14.14:853 \
  -servername dns.adguard-dns.com \
  -verify_hostname dns.adguard-dns.com \
  -verify_return_error </dev/null
```

Repeat the test using the secondary AdGuard endpoint:

```bash
openssl s_client \
  -connect 94.140.15.15:853 \
  -servername dns.adguard-dns.com \
  -verify_hostname dns.adguard-dns.com \
  -verify_return_error </dev/null
```

### Expected Results

Look for:

```text
Verification: OK
```

Or:

```text
Verify return code: 0 (ok)
```

A successful result indicates that the server can
establish a TLS connection and validate the certificate
presented by the upstream DNS endpoint.

Repeat these tests from the secondary Technitium server.

> **Important:** These commands test TLS connectivity
> to the upstream DNS provider.
>
> They do not independently prove that Technitium
> is forwarding DNS queries through that connection.

### Step 5.3 - Verify Actual Forwarding Traffic

To confirm that Technitium is communicating with
the configured upstream DNS servers over TCP port 853,
inspect the network traffic.

If `tcpdump` is not installed, install it first:

```bash
sudo apt install tcpdump -y
```

On the Technitium server, run:

```bash
sudo tcpdump -ni any \
  'tcp port 853 and (host 94.140.14.14 or host 94.140.15.15)'
```

While the capture is running, generate a DNS query
from another computer:

```bash
dig @192.168.xxx.xx example.com A
```

You may need to query a domain that is not already
cached by Technitium.

### Expected Results

The packet capture should show TCP traffic
between the Technitium server and one or both
configured AdGuard DNS endpoints.

> **Note:** Because concurrent forwarding is enabled,
> Technitium may communicate with either or both
> configured forwarders.
>
> Cached DNS responses may not generate new
> upstream traffic.
>
> Traffic on TCP port 853 provides evidence of
> DNS-over-TLS connectivity. A packet capture showing
> an established TLS session carrying traffic during
> an uncached DNS query provides stronger evidence
> that forwarding is occurring over TLS.

Press `Ctrl+C` to stop the packet capture.

---

## Step 6 - Final DNS Verification Checklist

After completing the tests, record the results.

| Verification | Primary DNS | Secondary DNS |
|---|---|---|
| DNS resolution over UDP | PASS / FAIL | PASS / FAIL |
| DNS resolution over TCP | PASS / FAIL | PASS / FAIL |
| Valid domain resolves | PASS / FAIL | PASS / FAIL |
| Invalid DNSSEC domain rejected | PASS / FAIL | PASS / FAIL |
| DNS-over-TLS configuration reviewed | PASS / FAIL | PASS / FAIL |
| Upstream TLS connectivity | PASS / FAIL | PASS / FAIL |
| Forwarding traffic observed | PASS / FAIL | PASS / FAIL |

> **Important:** Replace PASS / FAIL with the actual
> test results. Use NOT TESTED for any verification
> that has not been performed.

### Verification Notes

Record any errors, unexpected responses,
or configuration changes discovered during testing.

If DNS resolution fails, proceed to
**Part 5 - Troubleshooting**.

---

## DNS Verification Complete

After successfully completing the required tests,
the following have been verified:

- Both Technitium DNS servers resolve external domains.
- DNS resolution works over UDP and TCP.
- Valid DNS responses are returned.
- Invalid DNSSEC data is rejected somewhere in the resolution path.
- DNS-over-TLS forwarder settings have been reviewed.
- Connectivity to the configured upstream DNS servers has been tested.
- Forwarding traffic has been examined.

**DNS verification is complete when the required tests
have passed and their results have been recorded.**

> **Next Step:** Continue to Part 5 - Troubleshooting
> for information about diagnosing common
> Technitium DNS problems.

---

# Part 5 - Troubleshooting

Use these procedures when a verification step fails. Run server-side commands
on the affected Ubuntu host, and replace `192.168.xxx.xx` with that host's
actual address. Record the original symptoms before changing settings.

---

## 1. Technitium Service Not Running

Check service status and whether it is enabled at boot:

```bash
sudo systemctl status dns.service
systemctl is-enabled dns.service
```

Expected: `active (running)` and `enabled`.

If stopped, start the service; if disabled, enable it:

```bash
sudo systemctl start dns.service
sudo systemctl enable dns.service
```

If startup fails, inspect recent logs:

```bash
sudo journalctl -u dns.service -n 50 --no-pager
```

Look for port conflicts, invalid configuration, permission problems, or
startup errors. Diagnose the cause rather than repeatedly restarting.

---

## 2. DNS Resolution Fails

From a separate client, query the affected server explicitly:

```bash
dig @192.168.xxx.xx google.com A
dig +tcp @192.168.xxx.xx google.com A
```

A healthy response normally shows `status: NOERROR` and an `ANSWER SECTION`.
If the request times out, verify service status and listening ports:

```bash
sudo systemctl status dns.service
sudo ss -tulpn | grep ':53 '
sudo ufw status verbose
```

Check that UDP **and** TCP port 53 are permitted from authorized clients,
and review the gateway/VLAN firewall rules. If the server responds but
external names do not resolve, inspect the forwarders in **Settings → Proxy &
Forwarders** and review Technitium's query/log views.

> **Note:** `NOERROR` alone does not guarantee an address was returned;
> check the `ANSWER SECTION` and the queried record type.

---

## 3. DNSSEC Errors After Reboot / Raspberry Pi Clock Problems

Incorrect system time can break DNSSEC validation and TLS certificate checks.
On the Raspberry Pi, check the clock **before** manually restarting Chrony:

```bash
date
timedatectl
sudo systemctl status chrony
chronyc tracking
chronyc sources -v
```

Expected after synchronization: `System clock synchronized: yes`,
`NTP service: active`, `Leap status: Normal`, and a selected source marked
`^*`. Synchronization may take time after boot.

If the clock does not synchronize:

```bash
sudo journalctl -u chrony -n 50 --no-pager
```

Check network reachability, NTP/NTS sources, and certificate errors. Refer to
**Part 2 — Raspberry Pi 4 — Ubuntu Server Time Synchronization** for the
optional bootstrap repair used in the original installation. The bootstrap
sources are unauthenticated and should not be deployed automatically.

After time synchronization, retest normal DNS and the intentionally invalid
DNSSEC test domain as described in Part 4.

---

## 4. Port 53 Not Responding

Check which process owns DNS port 53:

```bash
sudo ss -tulpn | grep ':53 '
```

Technitium should listen on its configured IPv4/IPv6 addresses for UDP and
TCP. If another process owns the required address/port, investigate possible
conflicts with `systemd-resolved`, `dnsmasq`, BIND, or another DNS service.

> **Important:** `systemd-resolved` may use the loopback stub address
> `127.0.0.53`. Do not disable it blindly; determine whether it actually
> conflicts with Technitium and whether the host depends on it.

If Technitium is listening but remote clients time out, review UFW and
network firewall policies for UDP/TCP port 53.

---

## 5. Web Interface Not Accessible

Check the service and the configured web management port (default HTTP 5380):

```bash
sudo systemctl status dns.service
sudo ss -tulpn | grep ':5380 '
```

From another device, test network reachability if ICMP is allowed:

```bash
ping -c 4 192.168.xxx.xx
```

A failed ping does **not** prove the web service is down; ICMP may be blocked.
Review UFW and gateway policies for the actual management port. Restrict
management access to trusted administration networks, and use HTTPS for
ongoing administration rather than exposing HTTP management publicly.

---

## 6. DNS-over-TLS Forwarding Problems

This installation uses the AdGuard DNS-over-TLS endpoints:

| Endpoint | IP address | TLS hostname | Port |
|---|---|---|---|
| Primary upstream | `94.140.14.14` | `dns.adguard-dns.com` | TCP 853 |
| Secondary upstream | `94.140.15.15` | `dns.adguard-dns.com` | TCP 853 |

Verify the settings in **Settings → Proxy & Forwarders** on each server.
Test certificate validation and outbound connectivity from the affected host:

```bash
openssl s_client \
  -connect 94.140.14.14:853 \
  -servername dns.adguard-dns.com \
  -verify_hostname dns.adguard-dns.com \
  -verify_return_error </dev/null
```

Repeat with `94.140.15.15`. Look for `Verify return code: 0 (ok)`.
If it fails, check time synchronization, internet routing, certificate
validation, and outbound TCP 853 rules.

> **Important:** A successful `openssl` test confirms TLS connectivity,
> not that Technitium actually used that upstream for a particular query.
> Use the packet capture procedure in Part 4 for stronger evidence.

---

## 7. One DNS Server Works but the Other Does Not

Query each server **by its distinct address** from the same client:

```bash
# Replace with the primary server's actual IP:
dig @192.168.10.10 google.com A

# Replace with the secondary server's actual IP:
dig @192.168.10.11 google.com A
```

On the failing server, verify the service, port 53, firewall, forwarders,
clock, and logs. Remember that two working DNS servers are not automatically
a Technitium cluster; clustering is documented separately.

---

## 8. TVs, Gaming Systems, Discord, or Other Apps Stop Working

Earlier DNS enforcement testing caused connectivity problems on some devices.
The precise causes of those issues were **not** established for this
Technitium installation. Use evidence-based troubleshooting:

1. Confirm the device's assigned DNS servers and network/VLAN.
2. Query the approved Technitium servers directly from a test client.
3. Inspect Technitium query logs for failures or blocked domains.
4. If Advanced Blocking is enabled, review the device's assigned policy.
5. Inspect gateway firewall logs for DNS port 53, DoT port 853, and relevant
   encrypted-DNS restrictions (including UDP 443 where applicable).
6. Test a narrow, temporary exception only after identifying a legitimate
   false positive; document the change and retest.

> **Important:** Allowing a device to reach Technitium does not exempt it
> from Technitium Advanced Blocking. Avoid broad bypass rules without evidence.

---

## 9. Collect Diagnostic Information

When an issue persists, capture relevant output before changing settings:

```bash
hostnamectl
ip -br addr
ip route
sudo systemctl status dns.service
sudo ss -tulpn
sudo ufw status verbose
timedatectl
sudo journalctl -u dns.service -n 100 --no-pager
dig @192.168.xxx.xx google.com A
```

On hosts using Chrony, also run:

```bash
chronyc tracking
chronyc sources -v
```

Record the affected server, time, error messages, recent changes, and tests
performed. **Redact credentials, sensitive hostnames, and private network
information before sharing diagnostic output publicly.**

---

## Troubleshooting Summary

| Symptom | First checks |
|---|---|
| Service stopped | `systemctl status dns.service`, journal logs |
| DNS timeout | Port 53, UFW, gateway/VLAN firewall |
| DNSSEC error after reboot | Clock, Chrony, NTP/NTS sources |
| Web interface inaccessible | Service, management port, firewall |
| Upstream resolution fails | Forwarder settings, TCP 853, TLS certificate |
| Only one server fails | Test each IP and compare configurations |
| TV/game/app problems | Client DNS, query logs, blocking policies |

After correcting a problem, rerun the relevant tests in **Part 4** and
record the actual results. Do not mark a test as PASS unless it was performed.

---

# References

- [Technitium DNS Server](https://technitium.com/dns/)
- [Technitium DNS GitHub](https://github.com/TechnitiumSoftware/DnsServer)
- [Ubuntu Server](https://ubuntu.com/download/server)
- [Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/)
- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)

---

**Documentation maintained for the Technitium DNS home network project.**
