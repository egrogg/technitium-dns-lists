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
> are examples from my home network.
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
> I have used the following so you know to replace with your
> own IP's:
> 192.168.xxx.xx

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

Get the current IP and mac ID.  Then set the static IP (make a DHCP reservation) in your gateway or where your IP's are being assigned.  

```bash
ip a
```
To identify the network interface and MAC address more clearly, you can also use:  
```bash
ip -br addr
ip link show
```
These commands help identify the network interface, assigned IP address, and MAC address.  

Reboot so that the new IP that has been assigned will take effect:  

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

## Step 5 - Update Raspberry Pi OS

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

Log in to the Technitium web interface.

1. Navigate to **Settings**.
2. Locate the DNS Server settings.
3. Review the configured listening addresses.
4. Confirm that Technitium is listening on the
   intended network interfaces.
5. Save changes only if modifications are necessary.

### Verify DNS Port 53

From the Ubuntu Server terminal, run:

```bash
sudo ss -tulpn | grep ':53 '
```

Technitium should be listening on DNS port `53`.

DNS uses:

- UDP port 53 for standard DNS queries.
- TCP port 53 for DNS queries requiring TCP transport.

> **Important:** Ensure that no other DNS service
> is conflicting with Technitium on port 53.

Repeat this verification on both servers.

---

## Step 4 - Configure DNS Forwarders

Technitium can resolve DNS queries recursively or
forward them to another DNS resolver.

For this example, we will configure DNS forwarding.

### Configure Forwarders

1. Log in to the Technitium web interface.
2. Navigate to **Settings**.
3. Navigate to **ProxyAndForwarders**
4. Locate the **Forwarders** section.
5. Select the desired forwarding protocol. You can use the quick select to add. I have added two.
6. Enter the DNS forwarder addresses.
7. Save the configuration.

### Example DNS Providers

| Provider | DNS Address | Protocol |
|----------|-------------|----------|
| dns.adguard-dns.com | 94.140.14.14:853 | DNS |
| dns.adguard-dns.com | 94.140.14.15:853 | DNS |

> **Note:** These are examples only.
> Choose DNS forwarders according to your
> security, privacy, and filtering requirements.

### DNS over TLS
This is the **Forwarder Protocol**
Technitium also supports encrypted upstream
DNS forwarding using DNS over TLS (DoT). --This is what I have selected

DNS over TLS typically uses TCP port `853`.

For encrypted forwarding, configure the provider's
supported hostname and protocol according to
the Technitium settings.

> **Important:** Do not assume that entering an
> IP address automatically enables encrypted DNS.
> The forwarding protocol must be configured correctly.

I have also selected **Concurrent Forwarding**
And **Forwarder Concurrency** as 2

### Verify Forwarder Configuration

After saving the settings, test DNS resolution:

```bash
dig @192.168.101.72 google.com
```

Repeat for the secondary server:

```bash
dig @192.168.101.73 google.com
```

Both servers should return successful DNS responses.

---

## Step 5 - Verify DNS Resolution

From another computer on your network,
open a terminal.

### Test Primary DNS

```bash
dig @192.168.101.72 google.com A
```

### Test Secondary DNS

```bash
dig @192.168.101.73 google.com A
```

### Expected Result

A successful query should contain:

```text
status: NOERROR
```

The response should also include an ANSWER SECTION
containing one or more IPv4 addresses.

### Verify TCP DNS

Test the primary server:

```bash
dig +tcp @192.168.101.72 google.com A
```

Test the secondary server:

```bash
dig +tcp @192.168.101.73 google.com A
```

Both queries should complete successfully.

---

## Step 6 - Verify DNS Server Availability

From another computer, verify that both servers
are reachable.

### Primary DNS

```bash
ping -c 4 192.168.101.72
```

### Secondary DNS

```bash
ping -c 4 192.168.101.73
```

> **Note:** Some networks block ICMP traffic.
> A failed ping does not necessarily mean
> the DNS server is unavailable.

The DNS queries from Step 5 are the more
important tests.

---

## Step 7 - Verify Technitium Services

On each Ubuntu Server, run:

```bash
sudo systemctl status dns.service
```

Expected:

```text
Active: active (running)
```

To verify that Technitium starts automatically:

```bash
sudo systemctl is-enabled dns.service
```

Expected:

```text
enabled
```

If the service is not enabled, run:

```bash
sudo systemctl enable dns.service
```

---

## Step 8 - Verify Configuration After Reboot

After completing the initial configuration,
reboot each DNS server individually.

> **Important:** Reboot one server at a time
> to avoid unnecessary DNS interruptions.

### Reboot Ubuntu Server

```bash
sudo reboot
```

After reconnecting, verify the service:

```bash
sudo systemctl status dns.service
```

### Verify DNS Resolution

Primary server:

```bash
dig @192.168.101.72 google.com
```

Secondary server:

```bash
dig @192.168.101.73 google.com
```

Both servers should return successful responses.

### Raspberry Pi Time Synchronization

On the Raspberry Pi, also verify Chrony:

```bash
timedatectl
chronyc tracking
chronyc sources -v
```

Confirm that the system clock is synchronized
before relying on DNSSEC validation.

---

## Initial Configuration Complete

At this point:

- Both Technitium DNS servers are installed.
- Administrator credentials have been configured.
- DNS listening settings have been reviewed.
- DNS forwarding has been configured if desired.
- UDP and TCP DNS resolution have been tested.
- Technitium services are running.
- Both servers have been tested after reboot.
- Raspberry Pi time synchronization has been verified.

**The initial Technitium installation and configuration
are complete.**

> **Next Step:** Configure Technitium DNS clustering
> to connect the primary and secondary servers.
>
> Clustering will be documented separately.

---

# Part 4 - Verify DNS Resolution

Install DNS utilities if necessary:

```bash
sudo apt install dnsutils -y
```

Test the primary Technitium server:

```bash
dig @192.168.xxx.xx google.com
```

Test the secondary Technitium server:

```bash
dig @192.168.xxx.xx google.com
```

A successful DNS query should return:

```text
status: NOERROR
```

And an ANSWER SECTION containing DNS records.

## DNSSEC Validation Verification

After verifying normal DNS resolution, test how the
Technitium server handles invalid DNSSEC signatures.

### Test Secondary DNS Server

Run:

```bash
dig @192.168.xxx.xx dnssec-failed.org A
```

### Expected Result

```text
status: SERVFAIL
```

The response may also contain:

```text
EDE: 6 (DNSSEC Bogus)
```

### Actual Test Results

Test Date: October 19, 2026

```text
status: SERVFAIL
EDE: 6 (DNSSEC Bogus)
EDE: 22 (No Reachable Authority)
EDE: 13 (Cached Error)
```

### Explanation

The domain `dnssec-failed.org` is intentionally
configured with invalid DNSSEC signatures.

A DNSSEC-validating resolver should reject the
invalid response.

**Important:** A validating upstream DNS provider
may also reject the response.

This test confirms that invalid DNSSEC data was
rejected somewhere in the resolution path.
It does not independently prove that Technitium
performed the validation.

### Result

DNSSEC-invalid domain rejected: PASS


---

# Part 5 - Troubleshooting

## Technitium Service Not Running

Check the service:

```bash
sudo systemctl status dns.service
```

View recent service logs:

```bash
sudo journalctl -u dns.service -n 50 --no-pager

```

## DNS Resolution Fails

Check DNS resolution:

```bash
dig @192.168.xxx.xx google.com
```

## DNSSEC errors after reboot

```bash
timedatectl
chronyc tracking
chronyc sources -v
```

## DNS port 53 not responding  
Check whether Technitium is listening on port 53:  
```bash
sudo ss -tulpn | grep ':53 '
```

## Web Interface Not Accessible
Technitium web interface not responding  
Verify that Technitium is listening on port 5380:

```bash
sudo ss -tulpn | grep ':5380 '
```
For remote access problems, also verify that the server firewall and your gateway rules permit the intended traffic.
Check firewall rules and network connectivity.

---

# References

- [Technitium DNS Server](https://technitium.com/dns/)
- [Technitium DNS GitHub](https://github.com/TechnitiumSoftware/DnsServer)
- [Ubuntu Server](https://ubuntu.com/download/server)
- [Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/)
- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)

---

**Documentation maintained for the Technitium DNS home network project.**
