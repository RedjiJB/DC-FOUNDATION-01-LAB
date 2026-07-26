# DC-FOUNDATION-01 — Raspberry Pi Node Foundation

**Node:** `dc-node-01` (D-Central node 01) · **Platform:** Raspberry Pi 5 (8GB), Raspberry Pi OS 64-bit (Debian 13 "Trixie") · **Status:** ✅ Complete — 32/32 rubric, all 9 defects closed

Provisioning and hardening of a headless Raspberry Pi 5 into a reproducible, key-only-SSH, default-deny-firewalled server node — the foundation for the [D-Central](../../../..) sovereign-infrastructure cooperative and the CST portfolio.

## What's here

| File | What it is |
|---|---|
| `DC-FOUNDATION-01-LAB-REPORT.md` | Full lab report — Part A (build manual, executed) + Part B (meta report), 64 figures |
| `DC-FOUNDATION-01-STUDY-GUIDE.md` | Command-by-command teaching walkthrough with skill checks |
| `DC-FOUNDATION-01.tex` / `DC-FOUNDATION-01-STUDY-GUIDE.tex` | LaTeX sources (compile with `pdflatex`, twice, from this folder with `evidence/` present) |
| `DC-FOUNDATION-01.pdf` / `DC-FOUNDATION-01-STUDY-GUIDE.pdf` | Compiled PDFs, fully illustrated |
| `evidence/` | 64 screenshots referenced by the report and LaTeX build |

## Highlights

- **Full hardening chain:** key-only SSH (proven by negative test, not just a successful login), `ufw` default-deny inbound on both IPv4/IPv6, `fail2ban` with the journal backend, `unattended-upgrades`, Docker.
- **A real Black Start event:** a wrong-subnet lockout forced a complete re-image; the rebuild is proven by a changed machine ID, not just asserted.
- **A root-caused, not just patched, defect:** SSH hardening was silently overridden by `sshd_config` include-ordering (`50-cloud-init.conf` beat `99-dc-hardening.conf`) — found via `sshd -T`, fixed by sort order, proven by a negative auth test.
- **Nine defects raised, nine closed** (D1–D9), documented honestly including the ones that were embarrassing.

See `DC-FOUNDATION-01-LAB-REPORT.md` §15/§16 for the full defect register, rubric self-assessment, and lessons learned.

## Companion project

[`DC-FOUNDATION-02`](../DC-FOUNDATION-02.md) — Tailscale mesh VPN + Cloudflare Tunnel, building remote access on top of this hardened node.
