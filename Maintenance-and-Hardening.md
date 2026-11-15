# Maintenance and Hardening

Routine upkeep and security hardening for the Technitium servers, the Proxmox VM, the Raspberry Pi, and this repository.

---

## 1. Updates

### Ubuntu

```
sudo apt update
sudo apt upgrade -y
```

Automatic security updates:

```
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Reboot **one node at a time**, and confirm DNS and cluster status (`Technitium-DNS-Cluster-Setup.md`) before touching the other.

### Technitium

- Read the release notes before updating.
- Update one node at a time and keep both nodes on the same version.
- After updating, check the cluster status, run a `dig` test against each server, and confirm Advanced Blocking still loads its lists.

### Proxmox, pfSense

Apply updates in a maintenance window, take a backup first (`BackupAndRecovery.md`), and recheck DNS afterwards.

---

## 2. Server Firewall (UFW)

UFW is installed on Ubuntu Server but may be inactive. Allow SSH first, or you can lock yourself out.

```
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH from your admin network only
sudo ufw allow from 192.168.XXX.0/24 to any port 22 proto tcp

# DNS from client networks
sudo ufw allow from 192.168.XXX.0/24 to any port 53

# Admin UI and cluster traffic from admin hosts and the peer node only
sudo ufw allow from 192.168.XXX.XX to any port 5380 proto tcp
sudo ufw allow from 192.168.XXX.XX to any port 53443 proto tcp

sudo ufw enable
sudo ufw status verbose
```

Adjust the addresses and add one line per client VLAN that needs DNS. Do not expose ports 5380 or 53443 to the internet. If you use DNS-over-TLS or DNS-over-HTTPS to your own servers, open those ports deliberately.

If time sync breaks after enabling the firewall, check outbound UDP 123 and TCP 4460 (see the time-synchronization section of `HowToInstallTechnitium.md`).

---

## 3. HTTPS for the Admin Interface

The default web interface on port 5380 is plain HTTP. For ongoing administration:

1. In **Settings > Web Service** enable HTTPS (port 53443) and choose or upload a certificate.
2. Browse to `https://<server>:53443` and confirm it works.
3. Once it does, restrict or disable the HTTP port.

Clustering also uses the HTTPS port, so a valid certificate helps there too.

---

## 4. Accounts and SSH

- Use a unique, strong administrator password for each server's Technitium account, and enable two-factor authentication if your version offers it.
- Use SSH keys on the Proxmox VM as well as the Pi. After confirming key login works, disable password logins:

```
sudo nano /etc/ssh/sshd_config      # set: PasswordAuthentication no
sudo systemctl restart ssh
```

Keep a second session open while testing so a mistake does not lock you out.

---

## 5. Installing Technitium

The guides use `curl -sSL https://download.technitium.com/dns/install.sh | sudo bash`. If you prefer to review the script before running it:

```
curl -sSLO https://download.technitium.com/dns/install.sh
less install.sh
sudo bash install.sh
```

---

## 6. Raspberry Pi Storage

SQLite query logging writes continuously to a microSD card, which wears out.

- Keep query-log retention modest (30 days in the guide) and watch free space: `df -h`.
- Consider booting the Pi from a USB SSD instead of an SD card.
- Check for storage errors occasionally: `sudo dmesg | grep -i -E 'mmc|i/o error'`.

---

## 7. Routine Checks

| Frequency | Check |
| --- | --- |
| Weekly | Cluster status Connected; both servers answer DNS |
| Weekly | Advanced Blocking lists updated without errors |
| Monthly | `df -h` on both nodes; `chronyc tracking` on the Pi |
| Monthly | Review query logs for blocked-site spikes and bypass attempts (DoH hostnames, canary domain) |
| Monthly | Review firewall logs for unexpected DNS or DoH hits |
| Quarterly | Failover test from a representative client on each VLAN |
| Quarterly | Test a restore (`BackupAndRecovery.md`) |
| After every change | Export configs and update the docs in this repository |

---

## 8. Repository Hygiene

- Turn on two-factor authentication for the GitHub account. The block and allow lists are loaded straight from this repository, so control of it is control of your filtering.
- Add a repository description and topics, and add a license file.
- Keep placeholders (`192.168.XXX.XX`, `dns.MYDOMAIN.com`) consistent across files.
- Never commit credentials, certificates, backups, or an unredacted pfSense `config.xml`.
- Optionally keep a short `CHANGELOG.md` of list and config changes.
