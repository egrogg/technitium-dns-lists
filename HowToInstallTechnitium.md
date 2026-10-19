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

Reboot if required:

```bash
sudo reboot
```

After reboot, log back in and get the current IP and mac ID.  Then set the static IP (make a DHCP reservation) in your gateway or where your IP's are being assigned.

```bash
IP a
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
2. Launch Raspberry Pi Imager.
3. Select Raspberry Pi 4.
4. Select a supported 64-bit Raspberry Pi OS Lite image.
5. Select the storage device.
6. Configure the hostname, username, and SSH access.
7. Write the operating system to the storage device.

## Step 3 - Boot the Raspberry Pi

1. Insert the microSD card into the Raspberry Pi.
2. Connect the Raspberry Pi to the network.
3. Connect power.
4. Wait for the Raspberry Pi to boot.

## Step 4 - Connect Using SSH

From another Linux computer:

```bash
ssh username@192.168.101.73
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

Open:

```text
http://192.168.101.73:5380
```

Complete the initial setup wizard.

---

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
dig @192.168.101.72 google.com
```

Test the secondary Technitium server:

```bash
dig @192.168.101.73 google.com
```

A successful DNS query should return:

```text
status: NOERROR
```

And an ANSWER SECTION containing DNS records.

---

# Troubleshooting

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
dig @192.168.101.72 google.com
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
