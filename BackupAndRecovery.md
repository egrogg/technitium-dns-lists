# Backup and Recovery

What to back up, how often, how to restore, and what to do when something fails. Menu names can differ slightly between Technitium, Proxmox, and pfSense versions.

> **Never commit backups, certificates, or an unredacted pfSense `config.xml` to this repository.** They contain secrets. Store them off the servers they protect.

---

## 1. What to Back Up

| Item | Why | How | Frequency |
| --- | --- | --- | --- |
| Technitium primary configuration | The cluster's source of truth (settings, zones, apps, Advanced Blocking) | Technitium web UI backup (Settings) | After every config change, plus weekly |
| Primary VM | Fast full restore | Proxmox backup job (vzdump) | Daily or weekly |
| Secondary (Raspberry Pi) | Rebuildable from the guides; it takes its config from the primary when it joins | Keep a copy of `/etc/chrony/chrony.conf` and `/etc/chrony/sources.d/bootstrap-ip.sources`; optional SD card image | After changes |
| pfSense configuration | Rules, aliases, DHCP static mappings | Diagnostics > Backup & Restore | After every change |
| Lists and docs | Block/allow lists, Advanced Blocking config | This GitHub repository | Every commit |
| Query logs | Troubleshooting history only | Local SQLite on each node; not usually worth backing up | Optional |

---

## 2. Technitium

1. In the web interface open **Settings** and use the **Backup** function to download the backup file (the exact label may vary by version).
2. Store it off the DNS servers (another machine or encrypted storage).
3. Do this on the primary after every configuration change.

Restore tips:

- Restoring replaces the configuration on that node.
- **Joining a cluster overwrites a secondary's configuration**, so back up a secondary before joining it to a cluster.
- Test restores on a spare VM before you need them.

---

## 3. Proxmox (Primary VM)

1. **Datacenter > Backup > Add**: choose the VM, a storage target, a schedule, and a retention policy. Snapshot mode is usually fine.
2. **VM > Options**: enable **Start at boot** so DNS returns after a host reboot.
3. Install and enable the QEMU guest agent for clean snapshots:

```
sudo apt install qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

Then enable **QEMU Guest Agent** under the VM's Options.

4. Keep at least one backup on storage that is not the same disk as the VM.

---

## 3a. Raspberry Pi (Secondary)

- The secondary has no unique DNS data: it synchronizes from the primary.
- Keep the Chrony bootstrap settings (documented in `HowToInstallTechnitium.md`) and the DHCP reservation.
- Rebuild steps: reflash Ubuntu Server, apply the time-sync fix if needed, install Technitium, then join the cluster (`Technitium-DNS-Cluster-Setup.md`).
- Optional: image the SD card after a known-good build so a rebuild takes minutes.

---

## 4. pfSense

1. **Diagnostics > Backup & Restore**.
2. Download the configuration as XML. Choose the encryption option when available.
3. Store it off the firewall.
4. Document the rules and aliases in `firewall-rules.md` so the intent survives even if the backup is lost.

---

## 5. Recovery Scenarios

| What failed | Impact | Recovery |
| --- | --- | --- |
| Primary VM lost | Secondary still answers DNS; config changes impossible | Restore the VM from a Proxmox backup, or rebuild and restore the Technitium backup. If it is unrecoverable, Technitium offers promoting the secondary to primary (read its current documentation first), then add a new secondary. |
| Secondary (Pi) lost | Primary still answers; clients with both addresses fail over | Reflash, install, join the cluster. |
| Both DNS servers down | Clients lose DNS | Use the emergency steps in Section 6, then rebuild the primary first. |
| pfSense lost | No routing or DHCP | Restore `config.xml` to replacement hardware. |
| Bad list change on GitHub | Sites wrongly blocked or allowed | Revert the commit, wait for the next list refresh (24 hours) or re-save the Advanced Blocking config on the primary. |
| Bad Advanced Blocking config | Filtering wrong or off | Re-apply `advanced-blocking-config.recommended.json` on the primary. |

---

## 6. Emergency: No DNS

If both Technitium servers are down and the household has no DNS:

1. In pfSense re-enable the DNS Resolver (Services > DNS Resolver) and set the DHCP DNS fields temporarily to the pfSense address, or to a public resolver. Remember that filtering is off while you do this.
2. Temporarily relax the firewall DNS reject rules (`firewall-rules.md`) if clients are being blocked from the fallback.
3. Restore the Technitium servers.
4. **Put every setting back** and renew client leases.

Record what you changed so nothing is left half-reverted.

---

## 7. Test Restores

| Test | Result | Date |
| --- | --- | --- |
| Technitium backup restores on a spare VM | NOT TESTED | |
| Proxmox VM backup restores | NOT TESTED | |
| Secondary rebuilt from scratch and rejoined | NOT TESTED | |
| pfSense config restores on spare hardware or a VM | NOT TESTED | |
| Emergency no-DNS procedure rehearsed | NOT TESTED | |
