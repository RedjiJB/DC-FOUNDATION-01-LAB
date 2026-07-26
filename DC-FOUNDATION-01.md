# DC-FOUNDATION-01 — Raspberry Pi Node Foundation: Manual + Meta Report

**Project ID:** P00 (Foundation) · **Node:** dc-node-01 · **Platform:** Raspberry Pi 5 (8GB), NVMe/USB SSD boot, Raspberry Pi OS Bookworm (64-bit) · **Author:** Redji Jean Baptiste · **Tier:** T2 (Standard Lab Report) · **Date:** ____

> This document is two things: **Part A** is the reproducible setup manual (do this). **Part B** is the meta report — the portfolio deliverable that documents it to the CST-99 standard (submit this). Nothing is "done" until Part B is filled with your real evidence.

---

# PART A — THE SETUP MANUAL

> Status assumed: heatsink installed, SSD flashed, first boot complete, SSH access working. Each step ends with a **✓ Verify**. Do not skip verification — it's what makes the meta report defensible.

## 0. Capture your baseline (evidence for the report)
Before changing anything, record the starting state:
```bash
hostnamectl                      # OS, kernel, hardware
uname -a
ip a                             # current IP
df -h /                          # confirm booting from SSD, not SD
cat /proc/device-tree/model      # confirms Pi 5
```
**✓ Verify:** `df -h /` shows your SSD (e.g. `/dev/nvme0n1p2` or `/dev/sda2`) as `/`, not `mmcblk0`. Screenshot this — it's Figure 1 in the report.

## 1. Full system update + firmware
```bash
sudo apt update && sudo apt full-upgrade -y
sudo rpi-eeprom-update -a        # bootloader/firmware
sudo reboot
```
**✓ Verify:** after reboot, `sudo rpi-eeprom-update` reports up to date.

## 2. Base configuration (raspi-config)
```bash
sudo raspi-config
```
Set, in order:
- **System Options → Hostname** → `dc-node-01` (deliberate naming; this becomes your fleet convention).
- **Localisation** → timezone (America/Toronto), locale, keyboard.
- **Advanced Options → Expand Filesystem** (if not already full-disk).
- **Advanced Options → Boot Order** → confirm NVMe/USB boot priority.
- **Performance → GPU Memory** → 16 (headless server needs no GPU RAM).
Finish → reboot if prompted.
**✓ Verify:** `hostname` returns `dc-node-01`; `timedatectl` shows correct timezone.

## 3. Static IP address (Bookworm uses NetworkManager — not dhcpcd)
A server must not move. Find your interface and set a static address via `nmcli`:
```bash
nmcli device status                          # find your connection (e.g. "Wired connection 1")
nmcli connection show
# Replace values for your network:
sudo nmcli connection modify "Wired connection 1" \
  ipv4.addresses 192.168.1.50/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "1.1.1.1,9.9.9.9" \
  ipv4.method manual
sudo nmcli connection up "Wired connection 1"
```
(Or run `sudo nmtui` for a menu-driven version.)
**✓ Verify:** `ip a` shows 192.168.1.50; `ping -c3 1.1.1.1` works; reconnect SSH to the new static IP.

## 4. User + SSH key hardening *(CST8207 / CST8323)*
> The Imager likely created your user at flash time. Confirm it has sudo, then move to key-only auth.

**4a. Confirm sudo user** (skip creation if your flashed user already works):
```bash
groups                            # should include 'sudo'
```
If you need a fresh admin user:
```bash
sudo adduser redji
sudo usermod -aG sudo redji
```

**4b. Install your SSH public key** — from your **local machine** (not the Pi):
```bash
ssh-copy-id redji@192.168.1.50
```
Then confirm you can log in **with the key** in a new terminal before locking anything down.

**4c. Harden the SSH daemon** — on the Pi, edit `/etc/ssh/sshd_config`:
```bash
sudo nano /etc/ssh/sshd_config
```
Set:
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```
Apply:
```bash
sudo systemctl restart ssh
```
**✓ Verify:** open a NEW terminal, `ssh redji@192.168.1.50` logs in via key with no password prompt. **Do not close your working session until the new one succeeds** — this is how you avoid locking yourself out (documented as a real risk in the report).

## 5. Host firewall — ufw *(CST8323 / CST8249)*
```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH            # keep your management path open
sudo ufw enable
sudo ufw status verbose
```
**✓ Verify:** `ufw status` shows default-deny inbound with SSH allowed. Screenshot = Figure 2.

## 6. Brute-force protection — fail2ban
```bash
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```
**✓ Verify:** `fail2ban-client status sshd` returns an active jail.

## 7. Automatic security updates
```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades   # choose "Yes"
```
**✓ Verify:** `/etc/apt/apt.conf.d/20auto-upgrades` exists with update lists enabled.

## 8. Reliable time sync (needed for logs, TLS, TOTP, NATS later)
```bash
timedatectl                       # check NTP active
sudo timedatectl set-ntp true
```
**✓ Verify:** `timedatectl` shows "System clock synchronized: yes".

## 9. Core tooling
```bash
sudo apt install -y git vim htop curl wget net-tools dnsutils \
  build-essential ca-certificates gnupg tmux tree
```
**✓ Verify:** `git --version`, `docker` (next step), `tree` all resolve.

## 10. Portfolio workflow — Git + GitHub *(CST8206 / CST8300 — the standard)*
```bash
git config --global user.name  "Redji Jean Baptiste"
git config --global user.email "you@example.com"
ssh-keygen -t ed25519 -C "dc-node-01"      # add the public key to GitHub → Settings → SSH keys
ssh -T git@github.com                        # verify auth
```
Create the portfolio skeleton (per the deliverable standard §5):
```bash
mkdir -p ~/cst-portfolio && cd ~/cst-portfolio
git init
mkdir -p L1-CST8207-linux/p00-node-foundation/{artifacts,evidence}
# Save THIS document as the README of p00:
# ~/cst-portfolio/L1-CST8207-linux/p00-node-foundation/README.md
git add . && git commit -m "P00: node foundation baseline"
# git remote add origin git@github.com:redjijb/cst-portfolio.git
# git push -u origin main
```
**✓ Verify:** `ssh -T git@github.com` greets you by username; first commit exists.

## 11. Container substrate — Docker *(DC-PI-BUILD #1)*
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# log out and back in for group to apply, then:
docker run --rm hello-world
docker compose version
```
**✓ Verify:** `hello-world` runs without sudo; `docker compose version` resolves. This is the platform layer every later project deploys onto.

## 12. Final verification checklist
- [ ] Booting from SSD (`df -h /`)
- [ ] Hostname + timezone + static IP correct
- [ ] Key-only SSH; root + password login disabled
- [ ] ufw active, default-deny inbound
- [ ] fail2ban sshd jail active
- [ ] unattended-upgrades enabled
- [ ] NTP synchronized
- [ ] Docker runs rootless-group; compose present
- [ ] Portfolio repo initialized + pushed
- [ ] `sudo reboot` → everything survives a reboot

## 13. Troubleshooting log (fill as you go — this feeds §8 of the report)
| Symptom | Cause | Fix |
|---|---|---|
| Locked out after SSH change | Key not installed before disabling passwords | Re-enable via direct console/keyboard; install key; retry |
| Static IP didn't hold | Edited dhcpcd on Bookworm (wrong tool) | Use `nmcli`/`nmtui` (NetworkManager) |
| `docker` needs sudo | Group change not applied | Log out/in or `newgrp docker` |

---

# PART B — META REPORT (the deliverable, to CST-99 standard)

**Project ID:** P00 · **Course mapping:** CST8207 GNU/Linux System Support · CST8182 Networking Fundamentals · CST8323/8249 Network Security · CST8206 Foundation of IT Service Management · **Program/Level:** 1560X/0156X, L1 foundation · **Tier:** T2 · **Environment:** Pi 5 8GB, SSD boot, Bookworm 64-bit, static 192.168.1.50/24.

## 1. Objective & scope
Provision a Raspberry Pi 5 from first boot into a hardened, reproducible server node that serves as the foundation for the DC-PI and CST project portfolios. Scope: OS baseline, static addressing, SSH key hardening, host firewall, intrusion protection, automatic patching, container substrate, and portfolio workflow. Out of scope: individual service deployments (subsequent projects).

## 2. Competencies demonstrated
- **Linux administration** (users, permissions, systemd, package mgmt) — CST8207.
- **Network configuration** (static IP, gateway, DNS, verification) — CST8182.
- **Host hardening** (key auth, firewall, fail2ban, least-privilege) — CST8323/8249.
- **Service management + documentation** (reproducibility, change control) — CST8206.
- **Resume bullet:** "Provisioned and hardened a Linux server node from bare metal — key-only SSH, host firewall, fail2ban, automated patching, and containerization — documented to a reproducible standard."
- **Job-target relevance:** baseline sysadmin/SecOps competency expected by DND/CSE/private-sector infrastructure roles.

## 3. Background
A server node's security posture is set at provisioning; retrofitting hardening after services are exposed is error-prone. Bookworm's move to NetworkManager changes the correct method for static addressing, a common failure point. Key-only SSH plus default-deny firewalling establishes least privilege before any attack surface exists.

## 4. Environment & tools
Pi 5 (8GB), SSD boot, Raspberry Pi OS Bookworm 64-bit; tools: raspi-config, nmcli, OpenSSH, ufw, fail2ban, unattended-upgrades, Docker, git. [Insert topology: Pi → switch → gateway → internet.]

## 5. Methodology
Per Part A, steps 0–12 (numbered, reproducible). Summarize here; full commands live in Part A / Appendix A.

## 6. Results & evidence *(attach to /evidence)*
- **Fig 1:** `df -h /` — SSD as root filesystem.
- **Fig 2:** `ufw status verbose` — default-deny inbound, SSH allowed.
- **Fig 3:** new-session key-only SSH login (no password prompt).
- **Fig 4:** `fail2ban-client status sshd` — active jail.
- **Fig 5:** `docker run hello-world` success + `docker compose version`.
- **Fig 6:** `timedatectl` — clock synchronized.

## 7. Analysis & discussion
Explain *why* each control matters: default-deny + key auth removes the two most common intrusion vectors (exposed passwords, open ports) before any service exists; static addressing guarantees the node is reachable for the fleet conventions later projects assume; the container substrate is what allows 8–12 services to coexist. Note the RAM ceiling (~8–12 always-on services) as a capacity limit informing later scheduling.

## 8. Challenges & troubleshooting
Record what actually happened from the Part A §13 log (e.g., NetworkManager vs dhcpcd confusion; the near-lockout risk when disabling password auth and how sequencing the key install first mitigated it). This section proves the work was real.

## 9. Security & risk considerations
Least privilege (non-root user, sudo, key-only SSH); reduced attack surface (default-deny ufw); resilience (fail2ban, auto-patching); management-path preservation (allow SSH before enabling firewall). Residual risks: physical access to the node; single-node = single point of failure (addressed later by backups + a second node).

## 10. Conclusion & lessons learned
A hardened, documented, container-ready node now exists as the portfolio foundation. Key lesson: on Bookworm, network config must go through NetworkManager — using the legacy tool silently fails and is the #1 setup trap.

## 11. References
Raspberry Pi OS documentation; OpenSSH manual; ufw/fail2ban docs; Docker install docs. [IEEE format.]

## 12. Appendices
- **Appendix A:** full command history (`history > artifacts/setup-history.txt`).
- **Appendix B:** final `sshd_config`, `ufw status`, `nmcli connection show` output.

## Rubric self-assessment (threshold: ≥3 each, ≥28/32)
| Dimension | Score /4 | Note |
|---|---|---|
| Objective clarity |  |  |
| Reproducibility |  | Another student can rebuild from Part A alone |
| Evidence quality |  | Figs 1–6 attached |
| Technical correctness |  |  |
| Analysis depth |  |  |
| Professional communication |  |  |
| Competency mapping |  | §2 complete |
| Security consideration |  | §9 complete |
| **Total** | **/32** |  |

## Deliverable checklist
- [ ] Part A executed, all ✓ Verify passed
- [ ] Part B filled with real evidence (Figs 1–6)
- [ ] Artifacts committed (`setup-history.txt`, configs)
- [ ] Rubric self-scored ≥28/32
- [ ] Pushed to `redjijb/cst-portfolio`
