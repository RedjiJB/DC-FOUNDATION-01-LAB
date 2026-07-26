# DC-FOUNDATION-01 — Complete Study Guide
### Understand every command, concept, and screenshot in the lab, from zero

> **How to use this.** This document assumes you executed the lab by following the manual line-by-line without necessarily understanding it. It goes back over everything and explains *what each piece does, why it's there, what the output means, and where it can bite you.* Every section ends with a **Skill Check** — cover the answers and test yourself. If you can answer them, you own the material rather than having merely typed it.
>
> Read it with the report open beside you; figure numbers here match the report.

---

## Table of contents

0. [Mental model: what you actually built](#0)
1. [Foundations you need first](#1) — SSH, terminals, root, the shell
2. [Imaging and first boot](#2)
3. [Baseline capture](#3) — `hostnamectl`, `uname`, `ip a`, `df`, device tree
4. [Updating the system](#4) — `apt`, repositories, `&&`
5. [`raspi-config`](#5)
6. [Networking deep dive](#6) — IP, CIDR, DHCP vs static, NetworkManager, DNS, APIPA
7. [Users and permissions](#7) — `/etc/passwd`, `/etc/shadow`, `sudo`, groups
8. [SSH keys and hardening](#8) — asymmetric crypto, `authorized_keys`, `sshd_config`
9. [The firewall — `ufw`](#9)
10. [`fail2ban`](#10)
11. [Automatic updates](#11)
12. [Time synchronisation](#12)
13. [Git and GitHub](#13)
14. [Docker](#14)
15. [The nine defects as teaching cases](#15)
16. [Master skill-check exam](#16)

---

<a name="0"></a>
## 0. Mental model: what you actually built

You turned a bare Raspberry Pi into a **hardened headless Linux server**. Break that phrase down, because it's the whole lab in four words:

- **Server** — a computer that runs continuously and provides services to other machines over a network, rather than being sat in front of. It has no monitor of its own in normal use.
- **Headless** — no screen, no keyboard. You reach it entirely over the network with SSH. Everything in this lab was done without ever plugging a display into the Pi.
- **Hardened** — deliberately configured to resist attack: only key-based logins, a firewall that denies everything by default, automatic patching, and brute-force protection.
- **Linux** — specifically Raspberry Pi OS, which is Debian (a Linux distribution) built for the Pi's ARM processor.

Everything else is detail supporting those four ideas. Keep this picture in mind: a small always-on computer you talk to remotely, locked down so that only you can, kept patched so it doesn't rot.

---

<a name="1"></a>
## 1. Foundations you need first

### 1.1 What a shell / terminal actually is

When you see `admin@dc-node-01:~ $` and type commands, you are talking to a **shell** — a program (here, **bash**) that reads what you type, runs the corresponding program, and shows you its output. The prompt itself is information:

```
admin @ dc-node-01 : ~ $
 │        │          │  │
 │        │          │  └─ "$" = ordinary user (a "#" would mean root)
 │        │          └──── current directory ("~" means your home folder)
 │        └───────────────  the computer's hostname
 └────────────────────────  the user you are logged in as
```

**Why this matters:** throughout the lab, the prompt tells you *which machine you are on* and *who you are*. Several of your mistakes came from running a PowerShell command on the Pi (bash) or vice-versa. The prompt is how you tell them apart:

| Prompt looks like | You are on | Language |
|---|---|---|
| `admin@dc-node-01:~ $` | the Raspberry Pi | bash |
| `PS C:\Users\jredj>` | your Windows laptop | PowerShell |

### 1.2 Root, `sudo`, and why you're not always root

Linux has a superuser called **root** who can do anything — delete any file, change any setting. Operating as root all the time is dangerous: one typo can destroy the system, and any program you run has unlimited power. So you log in as a normal user (`admin`) with limited power, and borrow root's power only for specific commands by prefixing them with **`sudo`** ("superuser do").

```bash
apt update          # fails or does nothing — normal users can't change system packages
sudo apt update     # works — you temporarily act as root for just this command
```

This is the **principle of least privilege**: hold the minimum power needed, escalate deliberately. It's a theme of the entire lab.

### 1.3 SSH in one paragraph

**SSH (Secure Shell)** is the protocol that lets you get a shell on a remote machine over an encrypted connection. When you ran `ssh admin@172.20.10.10`, your laptop opened an encrypted tunnel to the Pi and gave you its shell. Everything typed travels encrypted, so nobody on the network can read your commands or the Pi's responses. SSH listens on **port 22** by default (a "port" is a numbered doorway on a machine; different services use different numbers).

### Skill Check 1
1. In `admin@dc-node-01:~ $`, what does the `~` mean, and what would `#` instead of `$` tell you?
2. Why is it safer to run as `admin` + `sudo` than to log in as root directly?
3. What does SSH give you, and on what port does it listen by default?

<details><summary>Answers</summary>

1. `~` is your home directory (`/home/admin`). A `#` prompt means you're operating as root.
2. Least privilege — a normal user can't accidentally (or via a compromised program) damage the whole system; root power is borrowed per-command and is auditable.
3. An encrypted remote shell (and file transfer); port 22.
</details>

---

<a name="2"></a>
## 2. Imaging and first boot

### 2.1 What "flashing an image" means

An **OS image** is a byte-for-byte copy of a fully installed operating system. The Raspberry Pi Imager writes that image onto the storage card so the Pi has something to boot. You didn't install Linux step-by-step; you copied a pre-made installation onto the card.

### 2.2 First-boot customisation — the clever part

Normally you'd boot a new machine, attach a monitor, and configure it. The Imager instead lets you set the **hostname, user, Wi-Fi, locale, and SSH** *before* writing, injecting those settings into the boot partition. The result: the Pi comes up already configured and already on the network, with no monitor ever attached. This is why, later, `raspi-config` showed the hostname field **already filled in** — you'd set it at flash time.

**Key insight:** this converts configuration into verification, and it's what made your rebuild cheap. When you had to re-image after the lockout, you just re-ran the same wizard.

### 2.3 The Windows "damaged drive" false alarm

After writing, Windows offered to "scan and fix" the drive. **Ignore it always.** Windows can't read Linux's `ext4` filesystem, so it thinks the card is corrupt. Running the repair can actually corrupt the freshly written boot partition.

### 2.4 mDNS — how `dc-node-01.local` works

You reached the Pi at `dc-node-01.local` before it had a known IP address. That `.local` suffix uses **mDNS (multicast DNS)**, a system where devices on the same local network announce their own names. It's a convenience for humans — but *not* dependable for services, which is exactly why you later set a static IP.

### Skill Check 2
1. Why didn't you need a monitor to configure the Pi?
2. Windows says the new card is damaged. What do you do and why?
3. How did `dc-node-01.local` resolve to the Pi with no DNS server configured?

<details><summary>Answers</summary>

1. First-boot customisation injected hostname/user/Wi-Fi/SSH into the boot partition, so it self-configured and joined the network on first boot.
2. Ignore it. Windows misreads the Linux `ext4` partition; repairing can corrupt the image.
3. mDNS/Avahi — the Pi announces its own `.local` name on the local segment.
</details>

---

<a name="3"></a>
## 3. Baseline capture

You ran a set of read-only commands *before changing anything*. This is deliberate: it's the "before" photo that every later "after" is compared against. (Recall the lockout — the baseline `ip a` is what let you prove the static IP had actually changed things.)

### 3.1 `hostnamectl`

```bash
hostnamectl
```
Prints system identity. Key fields:

| Field | Meaning |
|---|---|
| `Static hostname` | the machine's name — `dc-node-01` |
| `Machine ID` | a random unique ID generated on first install |
| `Operating System` | `Debian GNU/Linux 13 (trixie)` |
| `Kernel` | `Linux 6.18.34+rpt-rpi-2712` — the core of the OS; `-rpi-2712` marks the Pi 5 chip |
| `Architecture` | `arm64` — 64-bit ARM processor |

**Why the Machine ID matters:** it's generated fresh on each install. When you re-imaged, it *changed* — which is unforgeable proof you did a genuine rebuild, not a repair. Everything else (hostname, OS) stayed the same, showing the build is reproducible.

### 3.2 `uname -a`

```bash
uname -a
```
"Unix name" — prints kernel details. `-a` means "all." It confirms the kernel version and architecture. Overlaps with `hostnamectl` but is universal across all Unix systems.

### 3.3 `ip a`

```bash
ip a          # short for: ip address show
```
Lists every network interface and its addresses. You had three:

| Interface | What it is | State in baseline |
|---|---|---|
| `lo` | loopback — the machine talking to itself (`127.0.0.1`) | always up |
| `eth0` | the wired Ethernet port | `NO-CARRIER` = nothing plugged in |
| `wlan0` | the Wi-Fi radio | up, with a `dynamic` address |

Reading a `wlan0` entry:
```
inet 10.0.0.249/24 ... dynamic ... valid_lft 172252sec
```
- `inet 10.0.0.249` — the IPv4 address.
- `/24` — the subnet size (explained in §6).
- `dynamic` — this address came from **DHCP** (auto-assigned), not set by hand.
- `valid_lft 172252sec` — the "lease" expires in ~2 days; the network can reclaim it.

**This one line is the reason §6 exists.** A server shouldn't have an address that can change out from under it.

### 3.4 `df -h`

```bash
df -h          # "disk free", -h = human-readable (G/M instead of raw blocks)
```
Shows how full each filesystem is. The critical line:
```
/dev/mmcblk0p2   58G  6.8G  49G  13%  /
```
- `/dev/mmcblk0p2` — the device. **`mmcblk` = "MultiMediaCard block" = an SD card.** This is how you discovered you were booting from microSD, not an SSD (which would be `nvme0n1` or `sda`).
- `/` — this filesystem is mounted as the **root** of everything (the top of the directory tree).
- `13%` used — plenty of space.

### 3.5 `cat /proc/device-tree/model`

```bash
cat /proc/device-tree/model
```
- `cat` prints a file's contents.
- `/proc/` is a **virtual filesystem** — not real files on disk, but a window the kernel exposes into live hardware/system info.
- `/proc/device-tree/model` is the board's own description of itself, handed to the kernel by the firmware. It printed `Raspberry Pi 5 Model B Rev 1.1` — authoritative, not a guess.

### Skill Check 3
1. Why capture a baseline before changing anything?
2. In `inet 10.0.0.249/24 ... dynamic ... valid_lft 172252sec`, which two words tell you this is DHCP and not static, and how?
3. How did you know from `df -h` that you were on an SD card, not an SSD?
4. What is `/proc` and why is it trustworthy for hardware identity?

<details><summary>Answers</summary>

1. It's the reference every later verification compares against (e.g. proving the static IP took effect).
2. `dynamic` (assigned by DHCP) and `valid_lft ...sec` (a finite lease). A static address shows neither — it reads `valid_lft forever` with no `dynamic`.
3. The device was `/dev/mmcblk0p2`; `mmcblk` is the kernel's name for an SD/MMC card. An SSD would be `nvme...` or `sda...`.
4. A virtual filesystem the kernel exposes with live system/hardware data; `model` is populated by firmware from the board's device tree, so it's the hardware's own claim.
</details>

---

<a name="4"></a>
## 4. Updating the system

```bash
sudo apt update && sudo apt full-upgrade -y
```

### 4.1 What `apt` is
`apt` is Debian's **package manager** — it installs, removes, and updates software from online repositories, handling dependencies for you. Two sub-commands here:

- **`apt update`** — refreshes the *list* of available packages and versions from the repositories. It downloads no software; it just learns what's out there. Think "check for updates."
- **`apt full-upgrade`** — actually installs the newer versions. `full-upgrade` (vs plain `upgrade`) is allowed to *remove* packages when needed to resolve dependency changes — the right choice on a rolling distribution.
- **`-y`** — auto-answer "yes" to prompts, so it runs unattended.

### 4.2 The `&&` — why it's not just "run both"
`A && B` means **"run B only if A succeeded."** If `apt update` fails, the upgrade is skipped. This is why your `supd` typo (Fig 16) was harmless: `sudo apt update && supd apt full-upgrade` ran the update fine, then the misspelled half failed on its own — nothing was left half-done. **Lesson: read the whole chain, not just the first command.**

### 4.3 Repositories and "Hit" vs "Get"
The output showed four sources on `trixie`: Debian base, `-updates`, `-security`, and Raspberry Pi's own archive. `Hit:` means the local list was already current; `Get:` means it downloaded fresh metadata. "60 packages can be upgraded" on a weeks-old image is the concrete argument for automatic updates (§11).

### 4.4 Firmware
```bash
sudo rpi-eeprom-update -a
```
Updates the Pi's **bootloader firmware** (stored in EEPROM, a tiny chip). `-a` = apply all available. Because firmware updates are *staged* and applied at the next boot, you can only confirm success **after a reboot** — which is why the report checks it post-reboot and sees `CURRENT = LATEST`.

### Skill Check 4
1. Difference between `apt update` and `apt full-upgrade`?
2. In `A && B`, what does `&&` guarantee, and how did that make your `supd` typo harmless?
3. Why must you reboot before verifying an EEPROM/firmware update?

<details><summary>Answers</summary>

1. `update` refreshes the package *catalog* (downloads nothing installable); `full-upgrade` installs newer versions and may remove packages to satisfy dependencies.
2. B runs only if A succeeded. `apt update` (A) succeeded, so the catalog refreshed; the misspelled upgrade half (B) failed independently, leaving nothing partially applied.
3. Firmware is staged and only applied at boot, so the "up to date" confirmation is only meaningful after restarting.
</details>

---

<a name="5"></a>
## 5. `raspi-config`

```bash
sudo raspi-config
```
A menu-driven configuration tool specific to Raspberry Pi OS. It's a friendly front-end that edits config files for you. In the lab you used it mainly to **confirm** settings (hostname, localisation) that the Imager had already applied — and its title bar independently reported `Raspberry Pi 5 Model B Rev 1.1, 8GB`, corroborating your hardware.

The important conceptual point: because first-boot customisation had already set these, `raspi-config` was a *verification* step, not a *configuration* step. That's a property of a good reproducible build.

Deferred items you noted for later:
- **GPU Memory → 16** — a headless server renders no graphics, so give the GPU the minimum and leave more RAM for real work.
- **Boot Order** — only relevant if you later move to SSD booting.

### Skill Check 5
1. Why was `raspi-config` mostly a verification step in this lab rather than a configuration step?
2. Why set GPU memory to the minimum on this node?

<details><summary>Answers</summary>

1. The Imager's first-boot customisation had already applied hostname/locale/etc., so you were confirming, not setting.
2. It's headless — no display — so GPU RAM is wasted; minimising it frees RAM for server workloads.
</details>

---

<a name="6"></a>
## 6. Networking deep dive

This is the hardest section and the one that broke your build, so it gets the most detail.

### 6.1 IP addresses and what `/24` means
An **IP address** like `10.0.0.2` identifies a machine on a network. The `/24` after it is the **subnet mask in CIDR notation** — it says how many bits of the address identify the *network* versus the *host*.

- `/24` = the first 24 bits (first three numbers, `10.0.0`) are the **network**; the last 8 bits (last number) identify the **host**. So every device on `10.0.0.x` is on the same local network, and they can talk directly.
- `/28` (your iPhone hotspot) = 28 network bits, leaving 4 host bits → only 16 addresses, 14 usable. That's why a hotspot holds few devices.

**Why the lockout happened:** you set `192.168.1.50/24` on a node whose network was `10.0.0.0/24`. `192.168.1.x` and `10.0.0.x` are *different networks*. The Pi could no longer reach your laptop or the gateway — it was shouting into a network that didn't exist there. Instant disconnect.

### 6.2 Gateway and DNS
- **Gateway** (e.g. `10.0.0.1`) — the address of your router, the "door" out of your local network to the wider internet. Without a correct gateway, the Pi can reach local machines but not the internet.
- **DNS (Domain Name System)** — translates names like `github.com` into IP addresses. You set `8.8.8.8` (Google's resolver) as the DNS server. Your typo `8.4.4.4` (should be `8.8.4.4`) was a broken *secondary* — invisible while the primary worked, dangerous exactly when it didn't.

### 6.3 DHCP vs static — the core distinction
- **DHCP (Dynamic Host Configuration Protocol):** a server (usually your router) hands out addresses automatically, on a time-limited *lease*. Convenient, but the address can change. Fine for laptops, wrong for servers.
- **Static:** you assign a fixed address by hand. It never changes. A server must be findable at a known address, so servers get static addresses (or, better, a **DHCP reservation** — see below).

You can tell them apart in `ip a`: DHCP shows `dynamic` + `valid_lft <seconds>`; static shows `noprefixroute` + `valid_lft forever`.

### 6.4 NetworkManager, `nmcli`, and Netplan — the layers
On modern Debian, networking is managed by **NetworkManager**. You control it with **`nmcli`** (NetworkManager command-line interface).

```bash
nmcli connection show        # list saved connection "profiles"
nmcli device status          # show devices (wlan0, eth0) and their state
```

A **connection profile** is a saved bundle of settings (address, gateway, DNS) that gets applied to a device. Crucially, the profile has a *name*, and **you must discover that name, never assume it** — yours was `netplan-wlan0-Calypso`, then `Calypso`, then `"Wired connection 1"` across the build. The manual's assumed name didn't exist, which is fault #1 in the saga.

The `netplan-` prefix revealed a second layer: **Netplan** was *rendering* the profiles that NetworkManager applied. That means a change made only with `nmcli` could be silently overwritten when Netplan re-rendered — the same class of trap as editing the wrong config file.

To set a static address:
```bash
sudo nmcli connection modify "Calypso" \
  ipv4.addresses 10.0.0.2/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  ipv4.method manual
sudo nmcli connection up "Calypso"
```
- `modify` edits the saved profile (doesn't apply it yet).
- `ipv4.method manual` — the switch from DHCP to static. Without this, the address settings are ignored.
- `connection up` — **deactivates and re-activates** the profile so changes take effect. This is not a gentle reload; it drops the connection and rebuilds it. That's why one of your `up` commands failed when DHCP didn't answer on the way back — and why doing this over the very connection you're using can disconnect you.

### 6.5 APIPA — the `169.254.x.x` flood
When your laptop showed hundreds of `169.254.x.x` addresses (Fig 14), that's **APIPA (Automatic Private IP Addressing)**, also called link-local. A machine self-assigns a `169.254.x.x` address when **it asked for DHCP and got no answer.** Seeing it told you the *whole segment* had no working DHCP — a network fault, not a Pi fault. That distinction decides whether you fix the node or the network.

### 6.6 The durable fix for a portable node
Your node moved across three networks, and a host-side static address is glued to one subnet. The right answer for "always reachable at a known address" is a **DHCP reservation** on the router: the router always hands the *same* address to the Pi's network card (identified by its **MAC address**, a hardware serial number like `88:a2:9e:c2:77:d8`). The Pi keeps DHCP's flexibility but always gets the same address. Carry this into P01.

### Skill Check 6
1. Explain why `192.168.1.50/24` on a `10.0.0.0/24` network disconnected the Pi.
2. What does `/24` vs `/28` tell you about how many hosts fit?
3. How do you distinguish a DHCP address from a static one in `ip a` output?
4. What is the gateway for, and what is DNS for?
5. You see `169.254.x.x` everywhere. What does that mean and what does it rule out?
6. Why must you always run `nmcli connection show` before modifying a connection?
7. Why is a router DHCP reservation better than a host static IP for a node that moves?

<details><summary>Answers</summary>

1. Those are two different networks. With an address on `192.168.1.x`, the Pi had no route to the `10.0.0.x` gateway or your laptop, so all traffic (including your SSH session) died.
2. `/24` = 256 addresses (254 usable hosts); `/28` = 16 addresses (14 usable). More mask bits = fewer hosts.
3. DHCP: `dynamic` + finite `valid_lft`. Static: no `dynamic`, `valid_lft forever`, usually `noprefixroute`.
4. Gateway = the router, the exit to other networks/the internet. DNS = name-to-IP translation (`github.com` → an address).
5. APIPA/link-local self-assignment = no DHCP server answered. It rules out "the node alone is broken" and points at the whole segment/DHCP.
6. Profile names are not fixed — yours changed three times. Assuming the name gives "unknown connection" errors; discovery is one cheap command.
7. A host static breaks the moment the node changes subnet; a reservation keyed to the MAC gives a stable address on the home network while still working (via DHCP) elsewhere.
</details>

---

<a name="7"></a>
## 7. Users and permissions

### 7.1 Where users live
- **`/etc/passwd`** — the list of user accounts (username, ID, home directory, shell). World-readable; contains no passwords despite the name.
- **`/etc/shadow`** — the actual password hashes. Readable **only by root**. This is the key to the next point.

### 7.2 The permission lesson you hit
```bash
passwd redji            # → "You may not view or modify password information for redji"
sudo passwd redji       # → success
```
Why did the first fail? Changing a password writes to `/etc/shadow`, which only root can touch. As the `admin` user you may change **your own** password (there's a special path for that) but not someone else's — that needs root, hence `sudo`. **The error was the permission system working correctly, not a malfunction.**

### 7.3 Groups and `usermod -aG`
A **group** bundles users so permissions can be granted collectively. Membership in the `sudo` group is what lets `admin` use `sudo` at all. Membership in `docker` lets you run Docker without `sudo`.

```bash
sudo usermod -aG docker admin
```
- `usermod` — modify a user account.
- `-G` — set the user's **supplementary groups**.
- **`-a`** — *append*. This flag is critical: **without `-a`, `-G` replaces the entire group list**, which can strip a user of `sudo` and other groups in one shot. Always `-aG` together.

Group changes only take effect in a **new login session** (or after `newgrp docker`), which is why Docker needed a re-login before it worked without `sudo`.

### 7.4 `groups`
```bash
groups              # your own groups
groups testuser1    # someone else's
```
This is how you caught that `testuser1` had `sudo` when it shouldn't. You revoked it:
```bash
sudo gpasswd -d testuser1 sudo    # remove testuser1 from the sudo group
```

### Skill Check 7
1. Why is `/etc/shadow` readable only by root while `/etc/passwd` is world-readable?
2. `passwd redji` failed but `sudo passwd redji` worked. Explain precisely why.
3. What does the `-a` in `usermod -aG` prevent, and what goes wrong without it?
4. Why did Docker still need `sudo` immediately after `usermod -aG docker`?

<details><summary>Answers</summary>

1. `/etc/passwd` holds only account metadata (safe to read); `/etc/shadow` holds password hashes, so restricting it to root prevents offline password cracking.
2. Setting another user's password writes to root-only `/etc/shadow`; an unprivileged user can't. `sudo` supplies root privilege for that write.
3. `-a` appends to the group list. Without it, `-G` overwrites all supplementary groups, potentially removing `sudo`/hardware groups.
4. Group membership applies only to new sessions; the current shell predates the change. A re-login or `newgrp docker` activates it.
</details>

---

<a name="8"></a>
## 8. SSH keys and hardening — the heart of the lab

### 8.1 Asymmetric cryptography in plain terms
Password login means a shared secret both sides know — guessable, phishable, reused. **Key-based login** uses a **keypair**: two mathematically linked files.

- **Private key** (`id_ed25519`) — stays on your laptop, secret, never shared.
- **Public key** (`id_ed25519.pub`) — copied to the server; safe to share.

The magic: something the public key can verify can only have been produced by the matching private key. So the server can confirm "this is really the holder of the private key" **without the private key ever leaving your laptop.** No secret crosses the network. You cannot brute-force it the way you can a password.

### 8.2 Ed25519 vs RSA
You had both key types; you deployed **Ed25519**. It gives ~128-bit security in a tiny key (68 bytes vs RSA's thousands), with faster verification and no risky parameter choices. Prefer it.

### 8.3 `ssh-keygen`
```bash
ssh-keygen -t ed25519 -C "dc-node-01"
```
- `-t ed25519` — key type.
- `-C "dc-node-01"` — a comment label, purely for you to identify the key later in `authorized_keys`. (The workstation key was labelled `jredj@MSI`; the Pi's GitHub key `dc-node-01`. Labels make revocation possible without guesswork.)

### 8.4 Getting the public key onto the server
```bash
scp .\id_ed25519.pub admin@172.20.10.10:~/.ssh    # copy the file up
```
- `scp` = secure copy (file transfer over SSH).
- Direction matters: you push the **public** key up. The private key never moves.

Then on the Pi, the key must land in **`~/.ssh/authorized_keys`** — the list of public keys allowed to log in as this user:
```bash
cat ./id_ed25519.pub >> ~/.ssh/authorized_keys
```
**`>>` vs `>` — a critical distinction:**
- `>>` **appends** to the file.
- `>` **overwrites** it — wiping every existing key.

You used `>` and it was harmless only because the file was effectively empty. On a server with existing keys, `>` would silently revoke them all — an instant lockout on a password-disabled host. Build the `>>` habit, or use `ssh-copy-id`, which appends and fixes permissions in one step.

### 8.5 The permissions trap (D5)
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```
- `chmod` sets file permissions. The numbers are octal: `700` = owner may read/write/execute, nobody else anything; `600` = owner read/write, nobody else anything.
- **Why it matters:** `sshd` **silently ignores** `authorized_keys` if the file or `~/.ssh` is writable by anyone but the owner — a security measure. The symptom is "my key just doesn't work" with no useful error. Fixing permissions is mandatory, not optional.

### 8.6 Host keys and `known_hosts`
The first time you SSH to a machine you see:
```
The authenticity of host '172.20.10.10' can't be established.
ED25519 key fingerprint is SHA256:KjzNAN5t3U+...
Are you sure you want to continue connecting?
```
This is the reverse direction: the **server** proves its identity to **you** with its own host key. On first contact there's nothing to compare against, so SSH asks you to vouch for it and then saves the fingerprint in `~/.ssh/known_hosts`. If it ever changes, SSH screams — that's how it detects a man-in-the-middle. **Accepting blindly is the one weak moment**; ideally you compare the fingerprint against the server's real one.

### 8.7 Hardening `sshd` — the config and the drop-in
The SSH **server** daemon is `sshd`; its config is `/etc/ssh/sshd_config`. The three directives you needed:
```
PermitRootLogin no            # root may never log in over SSH
PasswordAuthentication no     # passwords refused; keys only
PubkeyAuthentication yes      # keys allowed
```

You applied these via a **drop-in file** in `/etc/ssh/sshd_config.d/` rather than editing the main file. Why: the main file already had a syntax error and gets overwritten by upgrades; a drop-in is cleaner and upgrade-safe.

### 8.8 The include-ordering trap (D3) — the most important lesson
Your drop-in was named `99-dc-hardening.conf`, but `PasswordAuthentication` stayed `yes`. The reason:

- `sshd_config` uses **the FIRST value it reads** for each setting (the opposite of most config systems, where the last wins).
- Debian includes drop-ins in **alphabetical order**, and cloud-init had already placed `50-cloud-init.conf` saying `PasswordAuthentication yes`.
- `50-` sorts before `99-`, so cloud-init's `yes` was read first and won.

Fix: rename yours to `00-dc-hardening.conf` so it sorts first and its values are read first.

```bash
sudo mv .../99-dc-hardening.conf .../00-dc-hardening.conf
sudo sshd -t                    # validate config syntax BEFORE restarting
sudo systemctl restart ssh
```
- `sshd -t` — test the config for syntax errors without applying it. Always run this before restarting; a broken config means `sshd` won't start, and on a headless box that's an unrecoverable lockout.
- `systemctl restart ssh` — restart the SSH service so the new config loads.

### 8.9 `sshd -T` and the negative test — proving it worked
```bash
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication'
```
- `sshd -T` prints the **effective, fully-resolved** configuration — what the daemon *actually* decided after reading every file. This is how you caught that `passwordauthentication` was still `yes`: reading the drop-in wasn't enough; you had to read the *result*.
- `| grep -E '...'` — pipe the output into `grep`, which filters to lines matching the pattern (`-E` = extended regex, `|` inside means "or").

Then the **negative test** — the single most important proof in the lab. Run from your laptop:
```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no admin@172.20.10.10
```
- `-o` sets an option.
- `PubkeyAuthentication=no` — don't offer your key.
- `PreferredAuthentications=password` — force the password method.
- Together: "try to log in with a password only." The correct result is **`Permission denied (publickey)`** — the server *refuses* passwords and says keys are the only way.

**Why this is the point:** every earlier check proved something *works* (your key logs in). Only this proves something is *refused*. A working key never demonstrated that passwords were disabled — the two are independent. **In security, you must test the refusal, not just the success.** This is the deepest lesson in the whole lab: *configuration that is present is not configuration that is effective; prove the negative.*

### Skill Check 8
1. In one sentence, how does key-based login prove your identity without sending a secret?
2. Why is Ed25519 preferred over RSA?
3. What's the difference between `>` and `>>`, and why did it nearly not matter for you but usually would?
4. `sshd` ignores your key and gives no clear error. What do you check first?
5. Explain the include-ordering trap: why did `99-dc-hardening.conf` lose to `50-cloud-init.conf`?
6. What does `sshd -T` show that reading the config file doesn't?
7. Write the negative test and state exactly what output means "pass."
8. Why is a successful key login NOT proof that password login is disabled?

<details><summary>Answers</summary>

1. The server verifies a signature that only the matching private key could have produced, so the private key never leaves your machine.
2. ~128-bit security in a tiny fast key with no risky parameters.
3. `>` overwrites, `>>` appends. Your `authorized_keys` was effectively empty so overwriting lost nothing; on a populated file `>` would delete all existing keys → lockout.
4. Permissions: `~/.ssh` must be `700` and `authorized_keys` `600`; `sshd` silently ignores them if group/world-writable.
5. `sshd_config` uses the first value read; drop-ins load alphabetically; `50-` (cloud-init, `yes`) was read before `99-` (yours, `no`), so `yes` won. Renaming to `00-` fixes it.
6. The effective, fully-resolved configuration after merging all files — the actual decision, not just one file's request.
7. `ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no admin@<ip>` → pass = `Permission denied (publickey).`
8. Pubkey and password auth are independent; a working key says nothing about whether passwords are also accepted. Only forcing password-only and being refused proves it.
</details>

---

<a name="9"></a>
## 9. The firewall — `ufw`

### 9.1 What a firewall does
A **firewall** filters network traffic by rules — which connections are allowed in or out. Linux's kernel firewall is `iptables`/`nftables`; **`ufw` ("Uncomplicated Firewall")** is a friendly front-end to it.

### 9.2 The commands, in order
```bash
sudo ufw default deny incoming     # block ALL inbound by default
sudo ufw default allow outgoing    # allow the Pi to reach out
sudo ufw allow OpenSSH             # carve one exception: let SSH in
sudo ufw enable                    # turn it on
sudo ufw status verbose            # show the result
```
- **`default deny incoming`** — the security posture: nothing gets in unless explicitly allowed. This is "default-deny," the correct stance.
- **`allow OpenSSH`** — an **application profile** (not a raw port number). It resolves to "TCP port 22" but keeps the intent readable — the rule says *why* the port is open.

### 9.3 The ordering that saved you
`allow OpenSSH` came **before** `enable`. If reversed, `enable` would have applied default-deny with no SSH exception and cut your session instantly — the same self-inflicted lockout class as the static-IP disaster, from a different tool. **Always open the management path before turning the firewall on.**

### 9.4 IPv4 and IPv6
The status showed rules for both `22/tcp` and `22/tcp (v6)`. Your node has real IPv6 addresses, so a v4-only rule would leave it reachable over IPv6 while *looking* firewalled. `ufw` handling both is correct — and checking for both is the step people skip.

### Skill Check 9
1. What does "default-deny incoming" mean and why is it the right posture?
2. Why must `allow OpenSSH` come before `enable`?
3. Why does it matter that `ufw` created both v4 and v6 rules?
4. What's the advantage of the `OpenSSH` application profile over writing `allow 22`?

<details><summary>Answers</summary>

1. All inbound connections are blocked unless explicitly permitted; it minimises attack surface — you allow only what you need.
2. Enabling first would apply default-deny with no SSH exception and drop your remote session, locking you out.
3. The node has global IPv6; a v4-only rule would leave IPv6 open, so the firewall would be bypassable.
4. It's self-documenting (states intent) and resilient if the underlying port/definition changes.
</details>

---

<a name="10"></a>
## 10. `fail2ban`

### 10.1 What it does
`fail2ban` watches logs for repeated failed logins and **temporarily bans** the offending IP by adding a firewall rule. It blunts brute-force attacks: even with passwords off, it stops noisy attackers from hammering the port.

```bash
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

### 10.2 `jail.conf` → `jail.local`
A **jail** is a watch-and-ban ruleset for one service (here, `sshd`). You copy `jail.conf` to `jail.local` and edit the copy because **package upgrades overwrite `jail.conf`** but leave `jail.local` alone. Editing the wrong one means your settings vanish at the next update — silently. (Same principle as the SSH drop-in.)

### 10.3 `systemctl enable --now`
- `systemctl` controls system services.
- `enable` — start it automatically at every boot.
- `--now` — also start it right now.
- Together: "run it now and every boot." Without `enable`, it wouldn't survive a reboot.

### 10.4 The journal backend
The status showed `Journal matches: _SYSTEMD_UNIT=ssh.service`. This means the jail reads the **systemd journal** (the modern binary log) rather than tailing a text file like `/var/log/auth.log`. On a journal-only system, a file-based filter would watch a file that never fills — a jail that looks healthy and bans nothing. All counters at zero is the correct *baseline* (no attacks yet), not proof of effectiveness.

### Skill Check 10
1. What does `fail2ban` actually do when it detects repeated failures?
2. Why copy `jail.conf` to `jail.local` instead of editing `jail.conf`?
3. What does `systemctl enable --now` do, split into its two parts?
4. Why does the journal backend matter?

<details><summary>Answers</summary>

1. Adds a temporary firewall rule banning the source IP after N failed attempts within a window.
2. Upgrades overwrite `jail.conf` but preserve `jail.local`, so your customisations survive updates.
3. `enable` = start on every boot; `--now` = start immediately as well.
4. It reads systemd's journal (the actual log source on this system); a file-based filter would watch a file that never gets written, so it would never ban anything.
</details>

---

<a name="11"></a>
## 11. Automatic updates

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```
Installs and enables a service that **automatically applies security updates** on a schedule. The install created `/etc/apt/apt.conf.d/20auto-upgrades` (how often) and `50unattended-upgrades` (what qualifies — security only by default), and a systemd symlink proving it runs at boot.

This is the answer to "60 packages pending on a weeks-old image." A node you don't log into daily will rot; this keeps the security posture you built today from decaying by next month. `dpkg-reconfigure` re-runs a package's setup questions.

### Skill Check 11
1. What problem does `unattended-upgrades` solve, in terms of the 60-pending-packages observation?
2. What's the difference in role between the `20auto-upgrades` and `50unattended-upgrades` files?

<details><summary>Answers</summary>

1. An unattended node drifts out of patch currency; this applies security updates automatically so the hardened state doesn't decay.
2. `20auto-upgrades` sets frequency (enable update/upgrade runs); `50unattended-upgrades` sets which origins/updates qualify (security by default).
</details>

---

<a name="12"></a>
## 12. Time synchronisation

```bash
timedatectl                                  # show clock/NTP status
sudo timedatectl set-timezone America/Toronto
```
Output to understand:
- `System clock synchronized: yes` — the clock is being kept accurate.
- `NTP service: active` — **NTP (Network Time Protocol)** is running, which sets the clock from internet time servers.
- `Time zone: America/Toronto (EDT, -0400)` — correct summer offset, proving the timezone *rules* are applied.
- `RTC in local TZ: no` — the **hardware clock (RTC)** runs in UTC; local time is derived. This is correct — an RTC in local time breaks across daylight-saving changes.

### Why time is load-bearing
- `fail2ban` correlates log timestamps to decide what's a burst of failures.
- TLS certificates are valid only within a time window — a wrong clock breaks HTTPS.
- Any future two-factor (TOTP) codes fail on a skewed clock.

Note the subtlety you hit: `NTP service: active` describes the *daemon running*, not that it *succeeded*. When DNS was broken, NTP couldn't reach its servers and `NTPSynchronized` was `no` despite the service being "active" — the same present-vs-effective trap as the SSH config.

### Skill Check 12
1. What does `NTP service: active` mean, and why is it not the same as "the clock is correct"?
2. Why should the RTC run in UTC rather than local time?
3. Name two things that break silently if the clock is wrong.

<details><summary>Answers</summary>

1. It means the time-sync daemon is running; it can be running yet failing to reach servers (e.g. DNS broken), so the clock may still be unsynchronised. Check `NTPSynchronized`.
2. UTC avoids daylight-saving ambiguities; local time on the RTC breaks across DST transitions.
3. TLS/HTTPS certificate validation and TOTP two-factor codes (also `fail2ban`/log correlation).
</details>

---

<a name="13"></a>
## 13. Git and GitHub

### 13.1 What Git is
**Git** is version control — it tracks changes to files over time in a **repository** ("repo"), letting you commit snapshots and push them to a remote server like **GitHub**. It's how your portfolio is stored and published.

```bash
git init                      # create a new repo in this folder
git add .                     # stage all changes for the next snapshot
git commit -m "message"       # record a snapshot with a description
git push -u origin main       # upload commits to the remote named "origin", branch "main"
```

### 13.2 SSH auth to GitHub
GitHub authenticates you with an SSH key (same idea as §8). The Pi generated its *own* key (separate from the laptop's), and you added the public half to GitHub.
```bash
ssh -T git@github.com
```
- `-T` — no terminal (GitHub gives no shell). Expected success: `Hi RedjiJB! You've successfully authenticated, but GitHub does not provide shell access.`
- The earlier `Permission denied (publickey)` before you registered the key is **expected** — the key existed but GitHub hadn't seen it yet.

### 13.3 `master` vs `main` — your push failure
`git init` created a branch called **`master`**; you pushed **`main`**, which didn't exist → `src refspec main does not match any`. Fix:
```bash
git branch -M main            # rename current branch to "main"
git push -u origin main
```

### 13.4 "Repository not found" ≠ permission problem
This appeared because the repo **didn't exist on GitHub yet** — `git remote add` only records a URL locally; it creates nothing on the server. GitHub deliberately says "not found" (not "denied") for repos you can't see. You created it on github.com, then the push succeeded.

### 13.5 The evidence-sanitisation lesson (D9)
**Deleting a file in a later commit does not remove it from Git history** — the data stays in the repo forever and can be recovered by anyone. That's why you had to keep the banking screenshot *out of the first commit*, not delete it afterward. Prevention is the only clean control. You verified nothing sensitive was published with:
```bash
git ls-files | grep -E "071704|184342|183550"    # empty output = clean
```

### Skill Check 13
1. What do `git add`, `git commit`, and `git push` each do?
2. Why did your first `git push -u origin main` fail with "src refspec main does not match any"?
3. Why does GitHub say "Repository not found" instead of "permission denied"?
4. Why is deleting a sensitive file in a later commit NOT enough, and what's the correct approach?

<details><summary>Answers</summary>

1. `add` stages changes; `commit` records a snapshot with a message; `push` uploads commits to the remote.
2. `git init` created `master`, not `main`; pushing a non-existent branch fails. `git branch -M main` renames it.
3. GitHub won't confirm the existence of repos you can't access, so it returns "not found" rather than leaking that it exists. Here it literally didn't exist yet.
4. History retains the blob; it's recoverable by SHA. Keep it out of the first commit entirely; removing published secrets needs history rewriting (filter-repo/BFG) + force-push, and caches may persist.
</details>

---

<a name="14"></a>
## 14. Docker

### 14.1 What containers are
A **container** is a lightweight, isolated package of an application plus everything it needs to run, sharing the host's kernel but otherwise sealed off. **Docker** builds and runs containers. This is the "container substrate" — the platform every later service will deploy onto.

```bash
curl -fsSL https://get.docker.com | sh
```
- `curl` downloads a URL; `-fsSL` = fail quietly on errors, silent, show errors, follow redirects.
- `| sh` — pipe the downloaded script straight into a shell to run it.

**This is a supply-chain risk you accepted knowingly:** piping a remote script to a shell runs whatever that URL serves. Mitigation: the script echoes every privileged action, and it installs a **GPG-signed** Docker repo (`signed-by=`) so subsequent packages are cryptographically verified through APT's normal trust.

### 14.2 Verifying it works
```bash
docker run --rm hello-world
```
- `run` — download (if needed) and run a container.
- `--rm` — delete the container after it exits (don't leave clutter).
- `hello-world` — a tiny test image.
- Note it pulled the **`arm64v8`** variant — Docker auto-selected the image built for your ARM CPU. This is the practical payoff of choosing the 64-bit OS.

```bash
docker compose version
```
**Compose** orchestrates multi-container apps from a config file — you confirmed it's present for later.

### 14.3 The `docker` group = root (D4)
You added yourself to the `docker` group so `docker` runs without `sudo`. **Understand the trade-off:** anyone in the `docker` group can start a container that mounts the host's entire filesystem and edit anything — it is **equivalent to root**. This is an accepted, documented trade-off of running Docker, not an oversight — but it belongs in your risk register, which is where the report puts it.

### Skill Check 14
1. What is a container, and how does it differ from a full virtual machine (one line)?
2. What's the risk in `curl ... | sh`, and what mitigates it here?
3. Why did `hello-world` pull an `arm64v8` image, and why is that a good sign?
4. Why is membership in the `docker` group a security consideration?

<details><summary>Answers</summary>

1. An isolated app bundle sharing the host kernel; a VM virtualises whole hardware and runs its own kernel (heavier).
2. It runs arbitrary remote code as root; mitigated by the script echoing its actions and installing a GPG-signed repo verified by APT.
3. Docker selected the image built for the ARM64 CPU — multi-arch resolution working, vindicating the 64-bit OS choice.
4. It's root-equivalent — a member can mount the host root filesystem in a container and modify anything.
</details>

---

<a name="15"></a>
## 15. The nine defects as teaching cases

Each defect is a lesson compressed into an incident. Learn the *pattern*, not just the fix.

| ID | What happened | The transferable lesson |
|---|---|---|
| **D1** | DNS secondary typed `8.4.4.4` instead of `8.8.4.4` | A broken *backup* is invisible until the primary fails — exactly when you need it. Verify redundancy actively. |
| **D2** | An account (`redji`) you didn't create held `sudo` | Audit *who* can escalate; unexplained privilege is a finding. |
| **D3** | Hardening drop-in present but overridden by cloud-init ordering | **Present ≠ effective.** Read the *resolved* config (`sshd -T`) and test the negative. |
| **D4** | Docker needed `sudo` | A convenience fix (`docker` group) is a security decision (root-equivalent) — state it. |
| **D5** | `~/.ssh` permissions unverified | `sshd` silently ignores world-writable key files — the failure has no obvious error. |
| **D6** | `testuser1` (a demo account) had `sudo` | Lab artifacts accumulate privilege; audit and revoke before service. |
| **D7** | Reboot survival unverified | *Configured ≠ persistent.* A reboot is the only proof; it caught the clock regression others would miss. |
| **D8** | Clock unsynced after reboot | Root cause was D1's stalled network, not NTP. One low-level fault fakes several high-level symptoms. |
| **D9** | Publishing evidence with private data | Git history is permanent; sanitise before the commit, never after. |

**The two meta-lessons that recur:**
1. **Present ≠ effective** (D3, D8): a config file, a running daemon, a green status can all be true while the thing they imply is false. Verify the *outcome*.
2. **Fix the lowest layer first** (D8): a dead clock, a dead resolver, and dropping SSH were *one* stalled DHCP activation. Diagnose bottom-up.

---

<a name="16"></a>
## 16. Master skill-check exam

No hints. If you can answer all of these, you understand the lab end to end.

1. Trace exactly what happened, at the network level, when you applied `192.168.1.50/24` to a node on `10.0.0.0/24`, and why `nmcli` gave no error.
2. You re-imaged the Pi. What single piece of evidence proves it was a genuine fresh install, and why can't it be faked?
3. Explain, in terms of "first value wins," why `99-dc-hardening.conf` failed and `00-dc-hardening.conf` succeeded.
4. Write the negative SSH test from memory and state the exact output that means "pass."
5. Why is a successful key login insufficient evidence that the node is hardened?
6. Your laptop shows hundreds of `169.254.x.x` addresses. Name the mechanism and say what it rules in and rules out.
7. Why does `ufw allow OpenSSH` have to precede `ufw enable`?
8. Give the general rule the SSH drop-in, the `fail2ban` `jail.local`, and editing `sshd_config` via a temp file all share.
9. Why is a router DHCP reservation the correct long-term fix instead of the static IP you set?
10. Why must sensitive screenshots be kept out of the *first* commit rather than deleted later?
11. `NTP service: active` but `NTPSynchronized: no`. Explain how both are true at once and what the real root cause was.
12. State the two meta-lessons (present≠effective; fix the lowest layer) and give one incident illustrating each.

<details><summary>Answers</summary>

1. The interface moved to a subnet (`192.168.1.x`) with no gateway or peers reachable on the actual `10.0.0.x` LAN; all routes (including your SSH session) died. `nmcli` validated the address *syntax*, not its *reachability*, so it returned success.
2. The Machine ID changed (`c8624351…`→`9e98c91d…`); it's generated randomly on first boot of a fresh install, so a changed value can't be produced by a mere repair.
3. `sshd_config` uses the first value read for each key; drop-ins load alphabetically; cloud-init's `50-…yes` was read before `99-…no`, so `yes` won. Renaming to `00-` makes yours read first.
4. `ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no admin@<ip>` → pass = `Permission denied (publickey).`
5. Key and password auth are independent; a working key says nothing about whether passwords are still accepted. Only the negative test proves the refusal.
6. APIPA/link-local: a host self-assigns `169.254.x.x` when no DHCP answers. Rules in "the whole segment has no working DHCP"; rules out "only the node is misconfigured."
7. Otherwise enabling applies default-deny with no SSH exception and severs your remote session.
8. Put your changes in a separate override that survives package upgrades and is validated before replacing the live config — never edit the vendor file directly.
9. A host static is bound to one subnet and breaks when the node moves; a reservation keyed to the MAC gives a stable address at home while DHCP still works elsewhere.
10. Git history is permanent and recoverable by SHA; deletion in a later commit leaves the data retrievable. Prevention before the first commit is the only clean control.
11. "Active" describes the running daemon; with DNS broken it couldn't resolve/reach NTP servers, so it never synchronised. Root cause was a stalled DHCP activation (D1) that killed DNS — not NTP itself.
12. *Present≠effective:* D3 — the hardening config existed but was overridden (proven only by `sshd -T` + negative test). *Fix the lowest layer:* D8 — one stalled DHCP activation produced dead DNS, dead clock, and dropped SSH; fixing the network fixed all three.
</details>

---

*Study guide companion to DC-FOUNDATION-01. Work through each Skill Check with the answers covered; when the Master Exam is effortless, the lab is yours to Black Start — rebuild it from zero, no manual, no internet. That is the BSL bar the timeline sets.*
