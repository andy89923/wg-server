# WireGuard VPN Server

A self-hosted [WireGuard](https://www.wireguard.com/) VPN server running in Docker, powered by the [linuxserver/wireguard](https://docs.linuxserver.io/images/docker-wireguard/) image. This lets you route your devices' internet traffic through your own server — great for privacy, remote access, or accessing your home/office network from anywhere.

---

## Table of Contents

- [What is WireGuard?](#what-is-wireguard)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Starting the Server](#starting-the-server)
- [Showing Peer Info / QR Code](#showing-peer-info--qr-code)
- [Adding or Removing Peers](#adding-or-removing-peers)
- [Connecting a Client Device](#connecting-a-client-device)
- [Useful Commands](#useful-commands)

---

## What is WireGuard?

WireGuard is a modern, fast, and secure VPN protocol. Unlike traditional VPNs (OpenVPN, IPSec), it is lightweight and easy to configure. When you connect a device (called a **peer**) to this server, all of that device's internet traffic is encrypted and tunnelled through your server.

---

## Prerequisites

Before you begin, make sure the following are installed on your **server machine**:

| Requirement | Notes |
|---|---|
| Linux (Ubuntu/Debian recommended) | WireGuard kernel module must be available |
| [Docker](https://docs.docker.com/engine/install/) | v20.10+ |
| [Docker Compose](https://docs.docker.com/compose/install/) | v2+ (`docker compose` command) |
| A **public IP address** or domain name | So clients can reach your server from the internet |

> **Tip:** If your server is behind a router/NAT, you need to forward **UDP port 51820** to your server's local IP address in your router settings.

---

## Configuration

Open `compose.yml` and update the following environment variables before the first run:

```yaml
environment:
  # Comma-separated list of peer names (one per device you want to connect)
  - PEERS=macbookair,ipadair,iphone

  - PUID=1000         # User ID to run the container as (use `id -u` on your server)
  - PGID=1000         # Group ID (use `id -g` on your server)
  - TZ=Etc/UTC        # Your timezone, e.g. America/New_York
  - SERVERURL=<YOUR_SERVER_PUBLIC_IP>  # ← Replace with your server's public IP or domain
  - SERVERPORT=51820  # UDP port WireGuard listens on
  - INTERNAL_SUBNET=10.60.163.0       # Private IP range used inside the VPN tunnel
  - ALLOWEDIPS=0.0.0.0/0,::/0         # Route ALL traffic through the VPN (full tunnel)
```

### Key settings explained

| Setting | What it does |
|---|---|
| `PEERS` | Names of the devices (clients) you want to pre-configure. Each name generates a config file and QR code automatically. |
| `SERVERURL` | The public IP or domain name of your server. Clients use this to connect. |
| `SERVERPORT` | The UDP port WireGuard listens on (default `51820`). Must be open in your firewall. |
| `INTERNAL_SUBNET` | The VPN's private subnet (e.g. `10.60.163.0/24`). Each peer gets an IP in this range (e.g. `10.60.163.2`, `10.60.163.3`, …). |
| `ALLOWEDIPS` | Which traffic to route through the VPN. `0.0.0.0/0,::/0` means **all** traffic (recommended for a full VPN). |

---

## Starting the Server

```bash
# Start the WireGuard server in the background
sudo docker compose up -d

# Verify the container is running
sudo docker compose ps

# View live logs (useful for troubleshooting)
sudo docker compose logs -f wireguard
```

On the first start the container will automatically generate:
- A server key pair
- Individual configuration files and QR codes for every peer listed in `PEERS`

All generated files are stored in the `./config/` directory on your host.

---

## Showing Peer Info / QR Code

Use the `peer-qr.sh` helper script to display the configuration for a peer as a **QR code** (scan it with the WireGuard mobile app) or as plain text.

```bash
# Show the QR code for a peer (replace "iphone" with the peer name)
bash peer-qr.sh iphone

# Show the QR code for another peer
bash peer-qr.sh macbookair
```

You can also run the underlying container command directly:

```bash
# Show peer info inside the container
sudo docker compose exec wireguard /app/show-peer iphone
```

The peer name must match one of the names you set in the `PEERS` environment variable.

---

## Adding or Removing Peers

Peer names are managed through the `PEERS` variable in `compose.yml`. Edit the file and then recreate the container.

### Add a new peer

1. Open `compose.yml` and add the new peer name to the `PEERS` list:

    ```yaml
    - PEERS=macbookair,ipadair,iphone,android-phone
    ```

2. Restart the container to generate the new peer's config:

    ```bash
    sudo docker compose up -d --force-recreate
    ```

3. Show the new peer's QR code or config file:

    ```bash
    bash peer-qr.sh android-phone
    ```

### Remove a peer

1. Remove the peer's name from the `PEERS` list in `compose.yml`.
2. Delete the peer's config folder:

    ```bash
    sudo rm -rf ./config/peer_android-phone
    ```

3. Restart the container:

    ```bash
    sudo docker compose up -d --force-recreate
    ```

> **Note:** Removing a peer only prevents the server from accepting new connections from it. If the peer already has a config file on their device, revoke access by restarting the server — the server will no longer have the peer's public key in its configuration.

---

## Connecting a Client Device

### Mobile (iOS / Android)

1. Install the **WireGuard** app from the [App Store](https://apps.apple.com/app/wireguard/id1441195209) or [Google Play](https://play.google.com/store/apps/details?id=com.wireguard.android).
2. Open the app → tap **+** → **Scan QR code**.
3. Run `bash peer-qr.sh <peer-name>` on your server and scan the QR code shown in the terminal.
4. Toggle the tunnel on. Your device is now connected to the VPN.

### Desktop (Windows / macOS / Linux)

1. Install the [WireGuard client](https://www.wireguard.com/install/).
2. Find your peer's config file on the server:

    ```bash
    # Config files are stored here (e.g. for peer "macbookair"):
    cat ./config/peer_macbookair/peer_macbookair.conf
    ```

3. Copy the contents of the `.conf` file to your client machine.
4. In the WireGuard client, click **Import tunnel(s) from file** (or **Add empty tunnel** and paste the config).
5. Activate the tunnel.

---

## Useful Commands

```bash
# Start the server
sudo docker compose up -d

# Stop the server
sudo docker compose down

# Restart the server
sudo docker compose restart wireguard

# View logs
sudo docker compose logs -f wireguard

# Show a peer's QR code
bash peer-qr.sh <peer-name>

# Open a shell inside the container (for advanced debugging)
sudo docker compose exec wireguard bash

# Check active WireGuard connections inside the container
sudo docker compose exec wireguard wg show
```
