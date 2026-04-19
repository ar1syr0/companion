---
# Project COMPANION — What Is This Thing?
So there I was, staring at my ever-growing pile of hard drives, laptops, and the creeping existential dread that comes with realising "what if my main machine dies in a hotel in the middle of nowhere?"

The answer, obviously, was to buy another computer.

# The Mission
Project COMPANION was born out of three very specific holiday anxieties:

"What if I lose my laptop data on vacation?"
Solution: Build a tiny machine that silently sucks up files from every device in the room like a digital vacuum cleaner and shoots them home to TrueNAS while you sleep.

"What if my main laptop dies abroad?"
Solution: The same tiny machine is also secretly a full desktop computer. Pop in a keyboard, plug it into the hotel TV, and carry on like nothing happened.

"The hotel TV is just sitting there doing nothing."
Solution: ...RetroArch. Obviously.

# The Design Philosophy
The machine had to meet one golden rule:

"It must fit in my cable organiser."

That was it. That was the entire engineering brief. Everything else — the Docker stack, the Tailscale VPN, the Ansible playbook, the Debian 13 install — all of it existed in service of one man's refusal to carry a large bag.

The result is a candybar-style mini PC :

- Draws less power than your phone charger (or same around 15-17W max recorded)

- Creates its own private WiFi hotspot in a hotel room

- Silently backs up every device connected to it

- Tunnels all that data home over an encrypted VPN

- Runs Portainer, Syncthing, and Pi-hole in Docker

- Plays PlayStation 1, SNES, GBA, and Sega Genesis on the hotel TV

- Can impersonate your main workstation in an emergency

![the mini pc](/assets/minipc_collage.png "The Mini PC")
![travel mode](/assets/travel_mode.jpg "It Fits !")


And it was, at various points during development, broken by Debian 13 refusing to package libretro-pcsx-rearmed — a fight that was eventually won by making the playbook simply shrug and move on, like a seasoned traveller who has missed one too many connecting flights to bother getting upset about it anymore.

Welcome to Project COMPANION. It's small. It's quiet. It's got your back — and your ROMs. For all the serious staff see below ⬇️


## Table of Contents

1. [Project Description](#1-project-description)
2. [Core Design](#2-core-design)
3. [Requirements](#3-requirements)
4. [Deployment Overview](#4-deployment-overview)
5. [Base OS Installation](#5-base-os-installation)
6. [SSH Server Setup](#6-ssh-server-setup)
7. [Headless / Travel Stability](#7-headless--travel-stability)
8. [Core Networking](#8-core-networking)
9. [Docker Engine](#9-docker-engine)
10. [Directory Layout](#10-directory-layout)
11. [Service: Portainer](#11-service-portainer)
12. [Service: Syncthing](#12-service-syncthing)
13. [Service: Pi-hole](#13-service-pi-hole)
14. [Syncthing Topology & TrueNAS Integration](#14-syncthing-topology--truenas-integration)
15. [XFCE Failsafe Desktop](#15-xfce-failsafe-desktop)
16. [RetroArch](#16-retroarch)
17. [Daily Access Quick Reference](#17-daily-access-quick-reference)
18. [Ansible Automated Setup](#18-ansible-automated-setup)
19. [Recommended Build Order](#19-recommended-build-order)
20. [Final Outcome](#20-final-outcome)

---

## Quick Reference — Ports & Services

| Service            | Port          | Protocol | URL                                 |
| ------------------ | ------------- | -------- | ----------------------------------- |
| SSH                | 22            | TCP      | `ssh user@companion.local`          |
| Portainer          | 9000 / 9443   | HTTP/S   | `http://companion.local:9000`       |
| Syncthing Web UI   | 8384          | HTTP     | `http://companion.local:8384`       |
| Syncthing Sync     | 22000         | TCP/UDP  | —                                   |
| Syncthing Discovery| 21027         | UDP      | —                                   |
| Pi-hole Admin      | 8081          | HTTP     | `http://companion.local:8081/admin` |
| DNS (Pi-hole)      | 53            | TCP/UDP  | —                                   |
| Portainer Edge     | 8000          | TCP      | —                                   |

---

# 1. Project Description

Project Companion is a portable mini PC built to complement a laptop-based workflow while travelling.

Its main purpose is to act as a **local edge node**:

- Devices in the room connect to the mini PC locally. I utilize a GLiNet Travel Router.
- Laptops and mobile devices dump files to the mini PC first.
- The mini PC then forwards those backups securely to the home TrueNAS server.
- If the main laptop fails, the mini PC can become a fallback workstation.
- When plugged into a hotel TV, it can also act as an HDMI retro-gaming box.

Project Companion is a hybrid of:

- **Travel backup hub** — central collection point for all device data.
- **Portable private network** — self-contained Wi-Fi hotspot for data dump.
- **Docker services host** — lightweight containerised services (Syncthing, Pi-hole, Portainer).
- **Remote bridge to home infrastructure** — encrypted Tailscale tunnel to TrueNAS.
- **Local emergency desktop** — full XFCE GUI fallback if the main laptop is unavailable.

---

# 2. Core Design

## 2.1 Backup Logic

The backup flow :

```
┌──────────────┐     local Wi-Fi     ┌───────────────────┐    Tailscale     ┌──────────┐
│   Laptops    │ ──────────────────► │                   │ ───────────────► │          │
│   Phones     │   file dump         │ Project Companion │  encrypted sync  │ TrueNAS  │
│   Tablets    │ ──────────────────► │   (local hub)     │ ───────────────► │  (home)  │
└──────────────┘                     └───────────────────┘                  └──────────┘
```

1. **Laptops & mobile devices** connect locally to Project Companion.
2. Those devices **dump files onto the mini PC** via Syncthing (Send Only mode).
3. The mini PC acts as the **local intake hub** (Send & Receive mode).
4. The mini PC then **forwards/syncs backup data to TrueNAS** over **Tailscale** (TrueNAS in Receive Only mode).

> **Key principle:** End devices are **not** responsible for backing up directly to TrueNAS. They only need to reach the mini PC.

## 2.2 Networking Logic

Project Companion supports three network modes:

| Mode | When to Use | How It Works |
| ---- | ----------- | ------------ |
| **Local LAN** | Mini PC is on the same network as your devices | All devices discover each other via mDNS (`companion.local`) |
| **Private Hotspot** | No trusted Wi-Fi available (e.g., hotel) | Mini PC creates its own Wi-Fi AP (`CompanionHub`); devices connect directly |
| **Remote Secure** | Syncing backups to home | Mini PC uses Tailscale to reach TrueNAS over an encrypted mesh VPN |

## 2.3 Main Services

| Service | Type | Purpose |
| ------- | ---- | ------- |
| **Debian 13 + XFCE** | Base OS | Lightweight desktop environment for fallback GUI use |
| **OpenSSH Server** | System | Secure remote shell access for headless management |
| **Avahi (mDNS)** | System | Makes the box discoverable as `companion.local` |
| **Wi-Fi Hotspot / AP** | System | Creates a private room-level network |
| **Tailscale** | System | Encrypted mesh VPN for home server connectivity |
| **Docker** | Runtime | Container engine for services |
| **Portainer** | Container | Web UI for managing Docker containers |
| **Syncthing** | Container | Continuous file synchronisation between devices |
| **Pi-hole** | Container | DNS-level ad/tracker blocking (optional) |
| **RetroArch** | Native | Retro game emulator for HDMI output |

---

# 3. Requirements

## 3.1 Hardware

**Minimum recommended specs:** (its what I builded on)

- Intel 12th gen (or equivalent) mini PC
- 4 cores / 4 threads minimum
- 12 GB RAM
- Built-in Wi-Fi (must support AP mode — check your chipset)
- Built-in Bluetooth (for controller pairing)
- HDMI output

## 3.2 Home Infrastructure

Before building Project Companion, ensure you have:

- A working **TrueNAS server** with internet access
- **Tailscale** installed and authenticated on the TrueNAS box
- **Syncthing** installed on TrueNAS (either native or as a plugin/jail)
- A dedicated **storage dataset or directory** on TrueNAS ready for Companion backups

## 3.3 Software

You will need:

- **Debian 13 installer** on a bootable USB (download from [debian.org](https://www.debian.org/download))
- A user account with **sudo access** (created during install)
- **Internet connection** during the first setup (for package downloads)
- Optional: another laptop/device to SSH into the mini PC during setup or you can simply plug it in a monitor

---

# 4. Deployment Overview

Project Companion is built in these layers:

| Phase | Step | Section |
| ----- | ---- | ------- |
| **OS** | Install Debian 13 + XFCE | [§5](#5-base-os-installation) |
| **SSH** | Configure and harden SSH server | [§6](#6-ssh-server-setup) |
| **Stability** | Disable sleep/suspend for headless use | [§7](#7-headless--travel-stability) |
| **Network** | Avahi, hotspot mode, Tailscale | [§8](#8-core-networking) |
| **Docker** | Install Docker Engine + Compose plugin | [§9](#9-docker-engine) |
| **Layout** | Create project directory structure | [§10](#10-directory-layout) |
| **Services** | Deploy Portainer, Syncthing, Pi-hole | [§11](#11-service-portainer), [§12](#12-service-syncthing), [§13](#13-service-pi-hole) |
| **Backup** | Configure Syncthing topology + TrueNAS relay | [§14](#14-syncthing-topology--truenas-integration) |
| **Desktop** | Install XFCE failsafe apps | [§15](#15-xfce-failsafe-desktop) |
| **Gaming** | Install RetroArch + ROM folders | [§16](#16-retroarch) |
| **Automation** | Ansible playbook for full rebuilds | [§18](#18-ansible-automated-setup) |

---

# 5. Base OS Installation

## 5.1 BIOS / UEFI

Before installing Debian, enter BIOS/UEFI and configure:

| Setting | Value | Why |
| ------- | ----- | --- |
| **Restore on AC / Power Loss** | `Power On` | Ensures the mini PC boots automatically after power interruptions |
| **Boot from USB** | Enabled | Required for the Debian installer |
| **Secure Boot** | Optional | Disable if you encounter driver issues during install |
| **Primary Boot Device** | Internal SSD | Set this after OS installation is complete |

## 5.2 Install Debian 13 + XFCE

Install **Debian 13 (Trixie)** and select the following during the tasksel screen:

- [x] XFCE desktop environment
- [x] Standard system utilities
- [x] SSH server

Set the hostname to:

```
companion
```

After Avahi is configured, the machine will be reachable as:

```
companion.local
```

## 5.3 First Boot — System Update

After the first login, update the system and install essential utilities:

```bash
# Full system upgrade
sudo apt update && sudo apt full-upgrade -y

# Install essential tools
sudo apt install -y \
  sudo \
  curl \
  wget \
  git \
  ca-certificates \
  gnupg \
  lsb-release \
  software-properties-common

# Reboot to apply kernel updates
sudo reboot
```

> **Note:** The `sudo` package is included because minimal Debian installs may not include it by default. The `apt-transport-https` package is no longer needed — HTTPS support is built into `apt` natively since Debian 11.

---

# 6. SSH Server Setup

SSH is essential for remote/headless management of Project Companion.

## 6.1 Install and Enable OpenSSH Server

If you selected "SSH server" during the Debian installer, this is already done. Otherwise:

```bash
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

**Verify SSH is running:**

```bash
sudo systemctl status ssh
```

**Test from another device on the same network:**

```bash
ssh youruser@companion.local
```

## 6.2 Generate SSH Keys (on your client machine)

On the laptop or device you'll SSH from, generate an SSH key pair if you don't already have one:

```bash
# Generate an Ed25519 key (recommended — fast and secure)
ssh-keygen -t ed25519 -C "youruser@yourlaptop"
```

When prompted:
- **File location:** Press Enter to accept the default (`~/.ssh/id_ed25519`)
- **Passphrase:** Set a passphrase for extra security (recommended)

**Copy the public key to Project Companion:**

```bash
ssh-copy-id youruser@companion.local
```

This adds your public key to `~/.ssh/authorized_keys` on the mini PC, enabling passwordless login.

**Test key-based login:**

```bash
ssh youruser@companion.local
# Should log you in without asking for the account password
```

## 6.3 Harden SSH Configuration

Once key-based auth is working, lock down the SSH server:

```bash
sudo nano /etc/ssh/sshd_config
```

Find and set the following directives (uncomment lines if needed):

```sshd_config
# Disable root login — always use your normal user + sudo
PermitRootLogin no

# Disable password authentication — keys only
PasswordAuthentication no

# Disable empty passwords
PermitEmptyPasswords no

# Use only Protocol 2 (Protocol 1 is insecure and deprecated)
Protocol 2

# Limit login attempts
MaxAuthTries 3

# Limit concurrent unauthenticated connections
MaxStartups 3:30:10

# Disable X11 forwarding (not needed for headless use)
X11Forwarding no

# Set an idle timeout (disconnect after 15 minutes of inactivity)
ClientAliveInterval 300
ClientAliveCountMax 3
```

**Restart SSH to apply changes:**

```bash
sudo systemctl restart ssh
```

> ** Warning:** Before restarting SSH, make sure key-based login works. If you disable password auth and your key isn't set up, you'll lock yourself out of remote access. Keep a local console session open as a fallback.

## 6.4 Firewall Considerations (Optional)

If you install a firewall (e.g., `ufw`), ensure SSH is allowed:

```bash
sudo apt install -y ufw
sudo ufw allow ssh
sudo ufw enable
```

---

# 7. Headless / Travel Stability

Because this is a mini PC, the default power management can cause problems — the system may suspend when monitors are disconnected or when idle timers expire.

**Mask all sleep-related systemd targets:**

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

**Verify they are masked:**

```bash
systemctl status sleep.target
# Expected: "Loaded: masked (Reason: Unit sleep.target is masked.)"

systemctl status suspend.target
# Expected: "Loaded: masked (Reason: Unit suspend.target is masked.)"
```

> **Why this matters:** Without masking these targets, the mini PC may go to sleep when running headless (no monitor), which kills SSH sessions and stops Docker services.

---

# 8. Core Networking

## 8.1 Avahi (mDNS)

Avahi enables **multicast DNS** so the system is discoverable on the local network as `companion.local` without needing to know its IP address.

**Install and enable:**

```bash
sudo apt install -y avahi-daemon
sudo systemctl enable --now avahi-daemon
```

**Test from another device on the same network:**

```bash
ping companion.local
```

> **Note:** mDNS only works on the local subnet. It won't work across VLANs or over Tailscale.

## 8.2 Wi-Fi Hotspot / AP Mode

Project Companion can create its own Wi-Fi network so devices can connect and dump data directly, even without existing infrastructure.

**Install NetworkManager:**

```bash
sudo apt install -y network-manager
sudo systemctl enable --now NetworkManager
```

**Check available interfaces:**

```bash
nmcli device status
```

Look for your Wi-Fi interface (usually `wlan0` or `wlp*`).

**Verify your chipset supports AP mode:**

```bash
iw list | grep -A 10 "Supported interface modes"
# Look for "AP" in the output
```

**Create the hotspot:**

```bash
# Replace 'wlan0' with your actual Wi-Fi interface name
# Replace the password with something secure
nmcli device wifi hotspot ifname wlan0 ssid CompanionHub password 'ChangeThisPassword123'
```

**Managing the hotspot:**

```bash
# List all connections
nmcli connection show

# Start the hotspot
nmcli connection up Hotspot

# Stop the hotspot
nmcli connection down Hotspot
```

> **Important:** Not all Wi-Fi chipsets support AP mode. If the hotspot command fails, check your chipset compatibility.

## 8.3 Tailscale

Tailscale is the **secure bridge** between Project Companion and your home TrueNAS. It creates an encrypted WireGuard mesh VPN without port forwarding.

**Install:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

**Authenticate (one-time — requires a browser):**

```bash
sudo tailscale up
```

This prints a URL. Open it in a browser to authenticate with your Tailscale account.

**Verify connectivity:**

```bash
# Show connected devices
tailscale status

# Show this device's Tailscale IP
tailscale ip
```

Ensure your TrueNAS server also appears in `tailscale status`.

---

# 9. Docker Engine

## 9.1 Install Docker

**Install Docker Engine using the official convenience script:**

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

**Add your user to the `docker` group** (run `docker` without `sudo`):

```bash
sudo usermod -aG docker $USER
```

> **Note:** You must **log out and back in** (or run `newgrp docker`) for the group change to take effect.

**Install the Compose plugin:**

```bash
sudo apt install -y docker-compose-plugin
```

**Verify:**

```bash
docker version
docker compose version
```

---

# 10. Directory Layout

All Docker services live under `~/companion/services/`. Each service gets its own subdirectory with its own `docker-compose.yml` and data folders, making services independently manageable.

**Create the full directory structure:**

```bash
# Main project root
mkdir -p ~/companion

# ── Services (Docker) ────────────────────────────────────
# Parent directory for all containerised services.
# Each service has its own folder, compose file, and data.

# Portainer — Docker management web UI
mkdir -p ~/companion/services/portainer/data

# Syncthing — file sync engine
mkdir -p ~/companion/services/syncthing/config

# Pi-hole — DNS ad/tracker blocker (optional)
mkdir -p ~/companion/services/pihole/etc-pihole
mkdir -p ~/companion/services/pihole/etc-dnsmasq.d

# ── Backups ──────────────────────────────────────────────
# Intake directories for device file dumps.
mkdir -p ~/companion/backups/laptops
mkdir -p ~/companion/backups/mobile
mkdir -p ~/companion/backups/staging

# ── ROMs ─────────────────────────────────────────────────
# RetroArch game ROMs, organised by console.
mkdir -p ~/companion/roms/ps1
mkdir -p ~/companion/roms/snes
mkdir -p ~/companion/roms/gba
mkdir -p ~/companion/roms/genesis
```

**Resulting directory tree:**

```
~/companion/
├── services/                          # All Docker services
│   ├── portainer/
│   │   ├── docker-compose.yml         # Portainer stack definition
│   │   └── data/                      # Portainer persistent data
│   ├── syncthing/
│   │   ├── docker-compose.yml         # Syncthing stack definition
│   │   └── config/                    # Syncthing config + database
│   └── pihole/
│       ├── docker-compose.yml         # Pi-hole stack definition
│       ├── etc-pihole/                # Pi-hole configuration
│       └── etc-dnsmasq.d/             # Pi-hole DNS overrides
├── backups/                           # Device file dump intake
│   ├── laptops/                       # Laptop data (documents, projects)
│   ├── mobile/                        # Phone/tablet data (photos, videos)
│   └── staging/                       # Shared transient import area
└── roms/                              # RetroArch ROM files
    ├── ps1/                           # PlayStation 1
    ├── snes/                          # Super Nintendo
    ├── gba/                           # Game Boy Advance
    └── genesis/                       # Sega Genesis / Mega Drive
```

---

# 11. Service: Portainer

Portainer provides a web UI for managing all Docker containers, images, volumes, and networks.

## 11.1 Create the Compose File

```bash
nano ~/companion/services/portainer/docker-compose.yml
```

```yaml
---
# Portainer — Docker Management Web UI
# Web UI:  http://companion.local:9000  (HTTP)
#          https://companion.local:9443 (HTTPS)
# Start:   docker compose up -d
# Stop:    docker compose down

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "8000:8000"   # Edge agent communication
      - "9000:9000"   # HTTP web UI
      - "9443:9443"   # HTTPS web UI
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/data
```

## 11.2 Start Portainer

```bash
cd ~/companion/services/portainer
docker compose up -d
```

## 11.3 Verify

```bash
docker compose ps
```

**Access the web UI:**

```
http://companion.local:9000
```

> **Important:** On first access, Portainer prompts you to create an admin account. Do this **immediately** — Portainer locks itself out if you wait longer than 5 minutes.

---

# 12. Service: Syncthing

Syncthing handles continuous file synchronisation between devices and the TrueNAS home server.

## 12.1 Create the Compose File

```bash
nano ~/companion/services/syncthing/docker-compose.yml
```

```yaml
---
# Syncthing — Continuous File Synchronisation
# Web UI:       http://companion.local:8384
# Sync:         TCP/UDP 22000
# Discovery:    UDP 21027
# Start:        docker compose up -d
# Stop:         docker compose down

services:
  syncthing:
    image: lscr.io/linuxserver/syncthing:latest
    container_name: syncthing
    restart: unless-stopped
    environment:
      - PUID=1000           # Match your host user's UID (run 'id' to check)
      - PGID=1000           # Match your host user's GID
      - TZ=Europe/London    # Set to your timezone
    ports:
      - "8384:8384"         # Web UI
      - "22000:22000/tcp"   # Sync protocol (TCP)
      - "22000:22000/udp"   # Sync protocol (UDP)
      - "21027:21027/udp"   # Local discovery
    volumes:
      - ./config:/config
      - ../../backups:/data/backups
```

> **Note:** The backups volume mounts `~/companion/backups/` into the container at `/data/backups/`, so Syncthing can see the laptop/mobile/staging directories.

## 12.2 Start Syncthing

```bash
cd ~/companion/services/syncthing
docker compose up -d
```

## 12.3 Verify

```bash
docker compose ps
```

**Access the web UI:**

```
http://companion.local:8384
```

---

# 13. Service: Pi-hole

Pi-hole provides DNS-level ad and tracker blocking. **This service is optional** — it is not required for the core backup workflow.

## 13.1 Create the Compose File

```bash
nano ~/companion/services/pihole/docker-compose.yml
```

```yaml
---
# Pi-hole — DNS Ad/Tracker Blocker
# Admin UI:  http://companion.local:8081/admin
# DNS:       Port 53 TCP/UDP
# Start:     docker compose up -d
# Stop:      docker compose down
#
# This service is OPTIONAL. Remove or comment out if not needed.

services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: unless-stopped
    environment:
      TZ: Europe/London                # Set to your timezone
      WEBPASSWORD: ChangeThisPassword  # Change before first start
    ports:
      - "53:53/tcp"     # DNS (TCP)
      - "53:53/udp"     # DNS (UDP)
      - "8081:80/tcp"   # Web UI (mapped to 8081 to avoid conflicts)
    volumes:
      - ./etc-pihole:/etc/pihole
      - ./etc-dnsmasq.d:/etc/dnsmasq.d
```

## 13.2 Start Pi-hole

```bash
cd ~/companion/services/pihole
docker compose up -d
```

## 13.3 Verify

```bash
docker compose ps
```

**Access the admin panel:**

```
http://companion.local:8081/admin
```

> **Note:** If port 53 is already in use (e.g., by `systemd-resolved`), Pi-hole will fail to start. Fix with: `sudo sed -i 's/#DNSStubListener=yes/DNSStubListener=no/' /etc/systemd/resolved.conf && sudo systemctl restart systemd-resolved`

> ** Hotspot + Pi-hole:** If running both hotspot mode and Pi-hole, ensure hotspot-connected devices use Companion's IP as their DNS server.

---

# 14. Syncthing Topology & TrueNAS Integration

## 14.1 Purpose

Syncthing acts as a **local intake hub** plus **home relay path**:

```
End Devices (Send Only) ──► Project Companion (Send & Receive) ──► TrueNAS (Receive Only)
```

## 14.2 Folder Roles

| Node | Syncthing Folder Type | Role |
| ---- | --------------------- | ---- |
| **End devices** (laptops, phones) | Send Only | Push data to Companion; never pull |
| **Project Companion** | Send & Receive | Accept from devices, forward to TrueNAS |
| **TrueNAS** | Receive Only | Final backup destination; optionally enable file versioning |

## 14.3 Suggested Folder Mapping

On the Project Companion Syncthing instance:

| Folder Path (in container) | Shared With | Purpose |
| -------------------------- | ----------- | ------- |
| `/data/backups/laptops` | Laptops + TrueNAS | Laptop project/document dumps |
| `/data/backups/mobile` | Phones + TrueNAS | Phone photo/video dumps |
| `/data/backups/staging` | All devices | Shared transient import area |

## 14.4 Device Flow

1. Laptops / mobile devices **dump data** to Project Companion over local Wi-Fi (or hotspot)
2. Project Companion **stores that data locally** on SSD
3. Project Companion **syncs that data to TrueNAS** over the encrypted Tailscale tunnel

> **Key benefit:** Travel devices stay simple. The mini PC is the sole node responsible for uplinking to home.

## 14.5 TrueNAS Requirements

Your TrueNAS box needs:

- Tailscale installed and authenticated (same Tailscale account as Companion)
- Syncthing available (native, plugin, jail, or Docker)
- A dedicated target dataset or directory for Companion backups
- Firewall rules allowing Syncthing traffic from the Tailscale interface (if applicable)

## 14.6 Initial Syncthing Configuration

Access at `http://companion.local:8384`, then:

1. **Add each laptop/phone** as a remote device (exchange device IDs)
2. **Add the TrueNAS** Syncthing instance as a remote device
3. **Share laptop/mobile backup folders** to Project Companion (Send Only on devices)
4. **Share Companion backup folders** onward to TrueNAS (Receive Only on TrueNAS)
5. **Confirm folder types** match the table in §14.2
6. Optionally enable **file versioning** on TrueNAS (staggered or simple) for safety

---

# 15. XFCE Failsafe Desktop

The XFCE desktop serves as a **fallback workstation** if your main laptop is unavailable.

**Essential desktop apps:**

```bash
sudo apt install -y firefox-esr thunar vlc filezilla
```

**Terminal productivity tools:**

```bash
sudo apt install -y htop btop tmux
```

| App | Purpose |
| --- | ------- |
| `firefox-esr` | Web browser (Portainer, Syncthing UIs, general browsing) |
| `thunar` | Graphical file manager |
| `vlc` | Media playback (video, audio, streaming) |
| `filezilla` | FTP/SFTP file transfer client |
| `htop` / `btop` | Interactive system resource monitors |
| `tmux` | Terminal multiplexer (persistent SSH sessions) |

---

# 16. RetroArch

RetroArch is installed **natively** (not in Docker) for best HDMI output performance and direct hardware access.

## 16.1 Install RetroArch and Emulation Cores

```bash
# RetroArch base
sudo apt install -y retroarch libretro-core-info retroarch-assets

# Emulation cores
sudo apt install -y \
  libretro-pcsx-rearmed \
  libretro-snes9x \
  libretro-mgba \
  libretro-genesisplusgx
```

| Core | System |
| ---- | ------ |
| `libretro-pcsx-rearmed` | PlayStation 1 |
| `libretro-snes9x` | Super Nintendo |
| `libretro-mgba` | Game Boy Advance |
| `libretro-genesisplusgx` | Sega Genesis / Mega Drive |

## 16.2 ROM Folders

ROM directories were created in §10. Place your ROM files in:

```
~/companion/roms/ps1/
~/companion/roms/snes/
~/companion/roms/gba/
~/companion/roms/genesis/
```

## 16.3 Launch

```bash
retroarch --menu --fullscreen
```

## 16.4 Bluetooth Controller Pairing

Since Bluetooth is only used for pairing game controllers, use the **Blueman** GUI manager that integrates into the XFCE desktop — no terminal commands needed.

**Install Blueman:**

```bash
sudo apt install -y blueman
```

> **Note:** `blueman` automatically pulls in `bluez` (the Bluetooth stack) as a dependency.

**Pairing a controller:**

1. Open **Bluetooth Manager** from the XFCE system tray (or launch `blueman-manager`)
2. Put your controller into **pairing mode**
3. Click **Search** in Blueman to scan for devices
4. Select your controller from the list
5. Click **Pair**, then **Trust** and **Connect**

The controller will auto-reconnect on future boots as long as it's trusted.

---

# 17. Daily Access Quick Reference

## Local Access (same network or hotspot)

| Service | URL |
| ------- | --- |
| Portainer | `http://companion.local:9000` |
| Syncthing | `http://companion.local:8384` |
| Pi-hole | `http://companion.local:8081/admin` |

## SSH Access

```bash
ssh youruser@companion.local
```

## Hotspot Mode

1. Start the hotspot: `nmcli connection up Hotspot`
2. Connect devices to `CompanionHub` Wi-Fi
3. Access all services via the URLs above

## Service Management

Each service is independently managed from its own directory:

```bash
# Portainer
cd ~/companion/services/portainer && docker compose up -d
cd ~/companion/services/portainer && docker compose down

# Syncthing
cd ~/companion/services/syncthing && docker compose up -d
cd ~/companion/services/syncthing && docker compose down

# Pi-hole
cd ~/companion/services/pihole && docker compose up -d
cd ~/companion/services/pihole && docker compose down

# Start ALL services at once
for svc in ~/companion/services/*/; do (cd "$svc" && docker compose up -d); done

# Stop ALL services at once
for svc in ~/companion/services/*/; do (cd "$svc" && docker compose down); done
```

## Remote Home Bridge

- Tailscale handles all Companion ↔ TrueNAS communication automatically
- No port forwarding or dynamic DNS required

---

# 18. Ansible Automated Setup

## 18.1 Why Use Ansible

Ansible automates the **entire setup** after a fresh Debian install into a single repeatable command.

**What the playbook automates:**

- System update and full upgrade
- All package installation (utilities, desktop apps, RetroArch, Bluetooth tools)
- SSH server installation and hardening
- Service enablement (Avahi, NetworkManager)
- Sleep target masking
- Tailscale installation
- Docker + Compose plugin installation
- Docker group membership
- Full project directory structure (`~/companion/services/...`, backups, roms)
- Per-service `docker-compose.yml` file creation
- Portainer container deployment

**What requires manual steps after the playbook:**

| Step | Why |
| ---- | --- |
| `sudo tailscale up` | Requires browser-based authentication |
| Start Syncthing + Pi-hole | Run `docker compose up -d` in each service directory |
| Wi-Fi hotspot creation | Password is user-chosen |
| Syncthing device pairing | Requires device IDs from each endpoint |
| SSH key setup | Keys are generated on the client, not the server |
| RetroArch Cores | Some may fail due to Debian 13 beeing an asshole |

## 18.2 Install Ansible

```bash
sudo apt update && sudo apt install -y ansible
```

## 18.3 Playbook

**Create the playbook file:**

```bash
nano ~/companion-playbook.yml
```

**Paste the following:**

```yaml

# ╔══════════════════════════════════════════════════════════════╗
# ║  Run:    ansible-playbook ~/companion-playbook.yml \         ║
# ║            --ask-become-pass                                 ║
# ║                                                              ║
# ║  Full Automated Setup tested and builded for:                ║
# ║  Debian 13 Trixie                                            ║
# ║                                                              ║
# ║  This single file handles:                                   ║
# ║    - System packages & updates                               ║
# ║    - SSH server & hardening                                  ║
# ║    - Network services (Avahi, NetworkManager)                ║
# ║    - Sleep target masking                                    ║
# ║    - Firewall (UFW)                                          ║
# ║    - Tailscale VPN                                           ║
# ║    - Docker Engine + Compose plugin                          ║
# ║    - Project directory structure                             ║
# ║    - Docker Compose stack (Portainer, Syncthing, Pi-hole)    ║
# ║    - RetroArch (fault-tolerant, skips unavailable cores)     ║
# ╚══════════════════════════════════════════════════════════════╝

- name: Project Companion — Full Setup
  hosts: localhost
  connection: local
  become: true

  vars:
    # ── User Configuration ───────────────────────────────────────
    companion_user: "{{ ansible_env.SUDO_USER | default(ansible_user_id) }}"
    companion_home: "/home/{{ companion_user }}"
    companion_root: "{{ companion_home }}/companion"

    # ── Base System Utilities ────────────────────────────────────
    base_packages:
      - sudo
      - curl
      - wget
      - git
      - ca-certificates
      - gnupg
      - lsb-release
      - apt-transport-https

    # ── SSH Server ───────────────────────────────────────────────
    ssh_packages:
      - openssh-server

    # ── Network Services ─────────────────────────────────────────
    network_packages:
      - avahi-daemon
      - network-manager

    # ── Firewall ─────────────────────────────────────────────────
    firewall_packages:
      - ufw

    # ── Bluetooth ────────────────────────────────────────────────
    bluetooth_packages:
      - blueman

    # ── Docker Compose Plugin ────────────────────────────────────
    docker_packages:
      - docker-compose-plugin

    # ── XFCE Failsafe Desktop Apps ───────────────────────────────
    desktop_packages:
      - firefox-esr
      - thunar
      - vlc
      - filezilla
      - htop
      - btop
      - tmux

    # ── RetroArch Base (Must succeed) ────────────────────────────
    retroarch_base:
      - retroarch
      - libretro-core-info
      - retroarch-assets

    # ── RetroArch Cores (Optional — skips if unavailable) ────────
    # NOTE: Any core that fails to install via APT can be added
    # manually later via RetroArch > Online Updater > Core Downloader
    retroarch_cores:
      - libretro-snes9x
      - libretro-mgba
      - libretro-genesisplusgx
      - libretro-pcsx-rearmed

    # ── Sleep Targets to Mask ────────────────────────────────────
    sleep_targets:
      - sleep.target
      - suspend.target
      - hibernate.target
      - hybrid-sleep.target

    # ── SSH Hardening Settings ───────────────────────────────────
    sshd_settings:
      - { key: "PermitRootLogin",       value: "no"  }
      - { key: "PasswordAuthentication", value: "no"  }
      - { key: "PermitEmptyPasswords",  value: "no"  }
      - { key: "MaxAuthTries",          value: "3"   }
      - { key: "X11Forwarding",         value: "no"  }
      - { key: "ClientAliveInterval",   value: "300" }
      - { key: "ClientAliveCountMax",   value: "3"   }

    # ── Project Directory Structure ──────────────────────────────
    companion_directories:
      - "{{ companion_root }}/services/portainer/data"
      - "{{ companion_root }}/services/syncthing/config"
      - "{{ companion_root }}/services/pihole/etc-pihole"
      - "{{ companion_root }}/services/pihole/etc-dnsmasq.d"
      - "{{ companion_root }}/backups/laptops"
      - "{{ companion_root }}/backups/mobile"
      - "{{ companion_root }}/backups/staging"
      - "{{ companion_root }}/roms/ps1"
      - "{{ companion_root }}/roms/snes"
      - "{{ companion_root }}/roms/gba"
      - "{{ companion_root }}/roms/genesis"

    # ── Docker Compose Stack ─────────────────────────────────────
    docker_compose_content: |
      ---
      # Project Companion — Docker Compose Stack
      # Start:  docker compose up -d
      # Stop:   docker compose down
      # Status: docker compose ps

      services:

        portainer:
          image: portainer/portainer-ce:latest
          container_name: portainer
          restart: always
          ports:
            - "8000:8000"
            - "9000:9000"
            - "9443:9443"
          volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            - ./services/portainer/data:/data

        syncthing:
          image: lscr.io/linuxserver/syncthing:latest
          container_name: syncthing
          restart: unless-stopped
          environment:
            - PUID=1000
            - PGID=1000
            - TZ=Europe/London
          ports:
            - "8384:8384"
            - "22000:22000/tcp"
            - "22000:22000/udp"
            - "21027:21027/udp"
          volumes:
            - ./services/syncthing/config:/config
            - ./backups:/data/backups

        pihole:
          image: pihole/pihole:latest
          container_name: pihole
          restart: unless-stopped
          environment:
            TZ: Europe/London
            WEBPASSWORD: ChangeThisPassword
          ports:
            - "53:53/tcp"
            - "53:53/udp"
            - "8081:80/tcp"
          volumes:
            - ./services/pihole/etc-pihole:/etc/pihole
            - ./services/pihole/etc-dnsmasq.d:/etc/dnsmasq.d

  tasks:

    # ─────────────────────────────────────────────────────────────
    # Phase 1: System Update
    # ─────────────────────────────────────────────────────────────

    - name: Update APT cache
      apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Full system upgrade
      apt:
        upgrade: full

    # ─────────────────────────────────────────────────────────────
    # Phase 2: Package Installation
    # ─────────────────────────────────────────────────────────────

    - name: Install base system utilities
      apt:
        name: "{{ base_packages }}"
        state: present

    - name: Install SSH server
      apt:
        name: "{{ ssh_packages }}"
        state: present

    - name: Install network service packages
      apt:
        name: "{{ network_packages }}"
        state: present

    - name: Install firewall
      apt:
        name: "{{ firewall_packages }}"
        state: present

    - name: Install Bluetooth GUI manager
      apt:
        name: "{{ bluetooth_packages }}"
        state: present

    - name: Install desktop environment apps
      apt:
        name: "{{ desktop_packages }}"
        state: present

    # ─────────────────────────────────────────────────────────────
    # Phase 3: SSH Hardening
    # ─────────────────────────────────────────────────────────────

    - name: Enable and start SSH
      systemd:
        name: ssh
        enabled: true
        state: started

    - name: Harden sshd_config settings
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?{{ item.key }}\\s'
        line: "{{ item.key }} {{ item.value }}"
        state: present
        validate: "sshd -t -f %s"
      loop: "{{ sshd_settings }}"
      notify: restart ssh

    # ─────────────────────────────────────────────────────────────
    # Phase 4: Network Services
    # ─────────────────────────────────────────────────────────────

    - name: Enable and start Avahi (mDNS)
      systemd:
        name: avahi-daemon
        enabled: true
        state: started

    - name: Enable and start NetworkManager
      systemd:
        name: NetworkManager
        enabled: true
        state: started

    - name: Mask sleep targets for headless stability
      systemd:
        name: "{{ item }}"
        masked: true
      loop: "{{ sleep_targets }}"

    # ─────────────────────────────────────────────────────────────
    # Phase 5: Firewall
    # ─────────────────────────────────────────────────────────────

    - name: Allow SSH through UFW
      community.general.ufw:
        rule: allow
        name: OpenSSH
      ignore_errors: true

    - name: Enable UFW firewall
      community.general.ufw:
        state: enabled
      ignore_errors: true

    # ─────────────────────────────────────────────────────────────
    # Phase 6: Tailscale
    # ─────────────────────────────────────────────────────────────

    - name: Install Tailscale
      shell: curl -fsSL https://tailscale.com/install.sh | sh
      args:
        creates: /usr/bin/tailscale
      # NOTE: After playbook completes run: sudo tailscale up
      # Browser authentication required on first run.

    # ─────────────────────────────────────────────────────────────
    # Phase 7: Docker Engine
    # ─────────────────────────────────────────────────────────────

    - name: Install Docker Engine
      shell: curl -fsSL https://get.docker.com | sh
      args:
        creates: /usr/bin/docker

    - name: Install Docker Compose plugin
      apt:
        name: "{{ docker_packages }}"
        state: present

    - name: Ensure docker group exists
      group:
        name: docker
        state: present

    - name: Add user to docker group
      user:
        name: "{{ companion_user }}"
        groups: docker
        append: true

    # ─────────────────────────────────────────────────────────────
    # Phase 8: RetroArch (Fault-Tolerant)
    # ─────────────────────────────────────────────────────────────

    - name: Install RetroArch base system (required)
      apt:
        name: "{{ retroarch_base }}"
        state: present

    - name: Install RetroArch emulation cores (skips unavailable packages)
      apt:
        name: "{{ item }}"
        state: present
      loop: "{{ retroarch_cores }}"
      ignore_errors: true
      register: core_results

    - name: Report any skipped cores
      debug:
        msg: >
          WARNING: Core '{{ item.item }}' could not be installed via APT.
          You can install it manually inside RetroArch via:
          Main Menu > Online Updater > Core Downloader
      when: item.failed is defined and item.failed
      loop: "{{ core_results.results }}"

    - name: Ensure RetroArch config directory exists
      file:
        path: "{{ companion_home }}/.config/retroarch"
        state: directory
        owner: "{{ companion_user }}"
        group: "{{ companion_user }}"
        mode: '0755'

    - name: Configure RetroArch to use Debian 13 core paths
      copy:
        dest: "{{ companion_home }}/.config/retroarch/retroarch.cfg"
        content: |
          libretro_directory = "/usr/lib/x86_64-linux-gnu/libretro"
          libretro_info_path = "/usr/share/libretro/info"
        owner: "{{ companion_user }}"
        group: "{{ companion_user }}"
        mode: '0644'
        force: no

    # ─────────────────────────────────────────────────────────────
    # Phase 9: Project Directory Structure
    # ─────────────────────────────────────────────────────────────

    - name: Create companion project directory tree
      file:
        path: "{{ item }}"
        state: directory
        owner: "{{ companion_user }}"
        group: "{{ companion_user }}"
        mode: '0755'
      loop: "{{ companion_directories }}"

    # ─────────────────────────────────────────────────────────────
    # Phase 10: Docker Compose Stack
    # ─────────────────────────────────────────────────────────────

    - name: Write docker-compose.yml to companion root
      copy:
        dest: "{{ companion_root }}/docker-compose.yml"
        content: "{{ docker_compose_content }}"
        owner: "{{ companion_user }}"
        group: "{{ companion_user }}"
        mode: '0644'

    - name: Start all Docker services
      shell: docker compose up -d
      args:
        chdir: "{{ companion_root }}"

    # ─────────────────────────────────────────────────────────────
    # Done
    # ─────────────────────────────────────────────────────────────

    - name: Setup Complete — Post-Install Checklist
      debug:
        msg: |
         Project Companion setup complete.

          Manual steps remaining:
          ─────────────────────────────────────────
          1. Log out and back in (docker group takes effect)
          2. Run: sudo tailscale up  (browser auth required)
          3. Configure WiFi Hotspot:
             nmcli device wifi hotspot ifname wlan0 ssid CompanionHub password 'YourPassword'
          4. Access Portainer at http://companion.local:9000
             → Create admin account immediately
          5. Change Pi-hole password in:
             ~/companion/docker-compose.yml (WEBPASSWORD)
          6. Open Syncthing at http://companion.local:8384
             → Pair your devices and set folder roles
          ─────────────────────────────────────────

  # ── Handlers ───────────────────────────────────────────────────
  handlers:
    - name: restart ssh
      systemd:
        name: ssh
        state: restarted
```

## 18.4 Run the Playbook

```bash
ansible-playbook ~/companion-playbook.yml --ask-become-pass
```

Enter your sudo password when prompted.

> ** Warning:** The playbook hardens SSH by disabling password authentication (§6.3). Make sure you have a **local console session open** or set up your SSH keys **before** running the playbook a second time.

## 18.5 After the Playbook Completes

1. **Log out and back in** (or reboot) for the docker group change
2. **Set up SSH keys** from your client machine
3. **Authenticate Tailscale:**
   ```bash
   sudo tailscale up
   ```
4. **Start remaining services:**
   ```bash
   cd ~/companion/services/syncthing && docker compose up -d
   cd ~/companion/services/pihole && docker compose up -d
   ```
5. **Access Portainer** at `http://companion.local:9000` — create your admin account
6. **Configure Wi-Fi hotspot**
7. **Configure Syncthing** device pairing
8. **Change default passwords** in Pi-hole's `docker-compose.yml`

---

# 19. Recommended Build Order

**Manual build** (without Ansible):

| Step | Action | Section |
| ---- | ------ | ------- |
| 1 | Install Debian 13 + XFCE 
| 2 | Update system and install base packages 
| 3 | Install and configure SSH server 
| 4 | Mask sleep targets 
| 5 | Install and enable Avahi 
| 6 | Configure Wi-Fi hotspot mode 
| 7 | Install and authenticate Tailscale 
| 8 | Install Docker + Compose plugin 
| 9 | Create directory structure 
| 10 | Deploy Portainer 
| 11 | Deploy Syncthing 
| 12 | Deploy Pi-hole (optional) 
| 13 | Configure Syncthing topology + TrueNAS relay 
| 14 | Install desktop apps 
| 15 | Install RetroArch + cores 

**Ansible build** (automated):

| Step | Action |
| ---- | ------ |
| 1 | Install Debian 13 + XFCE (manual) |
| 2 | Install Ansible: `sudo apt install -y ansible` |
| 3 | Run: `ansible-playbook ~/companion-playbook.yml --ask-become-pass` |
| 4 | Complete manual post-steps |

---

# 20. Final Outcome

When fully configured, Project Companion will:

- Create its own **local Wi-Fi environment** for room-level device connectivity
- Accept **data dumps from laptops and mobile devices** via Syncthing
- Act as the **local file intake hub** — the single collection point for all backups
- Forward backups to **TrueNAS securely over Tailscale** — fully encrypted
- Provide secure **SSH access** with key-based auth and hardened configuration
- Provide a **GUI fallback desktop** (XFCE) if the main laptop is unavailable
-  Host independently managed **Docker services** (Portainer, Syncthing, Pi-hole)
-  Block ads and trackers via **Pi-hole** when needed
-  Output **retro games directly over HDMI** with Bluetooth controller support
-  Be fully **rebuildable from a single Ansible playbook**
