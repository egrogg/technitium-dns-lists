# How to Install Technitium DNS Server

This guide provides step-by-step instructions for installing
Technitium DNS Server on two different platforms:

1. Proxmox VE using an Ubuntu Server virtual machine.  This will be the primary DNS Server
2. Raspberry Pi 4 using a Linux operating system. This will be the Secondary DNS Server
3. Setup Both IP's in your Gateway as Static (DHCP reservation).  

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
IP a
```
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
---

### Result

The original Raspberry Pi 4 configuration successfully
recovered its system time after reboot.

Chrony synchronized automatically, allowing Technitium
DNSSEC validation to operate normally.

**Status: Tested successfully on the original installation.**

# Part 3 - Initial Technitium Configuration

After installing Technitium DNS Server:

1. Log in to the Technitium web interface.
2. Configure administrator credentials.
3. Configure DNS settings.
4. Configure DNS forwarders if required.
5. Configure the server's network settings.
6. Verify DNS resolution.

> **Security Note:** Do not expose the Technitium
> administrative web interface directly to the internet.

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

---

# Part 5 - Troubleshooting

## Technitium Service Not Running

Check the service:

```bash
sudo systemctl status dns.service
```

View recent service logs:

```bash
z
```

## DNS Resolution Fails

Check DNS resolution:

```bash
dig @192.168.xxx.xx google.com
```

Check whether Technitium is listening on port 53:

```bash
sudo ss -tulpn | grep ':53 '
```

## Web Interface Not Accessible

Verify that Technitium is listening on port 5380:

```bash
sudo ss -tulpn | grep ':5380 '
```

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
