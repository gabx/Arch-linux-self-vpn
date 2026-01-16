# Arch Linux Self-VPN & Browser Isolation

This project deploys a full **WireGuard VPN (Client + Server)** on a single local Arch Linux machine.

The goal is to isolate a specific application (e.g., Firefox) inside a dedicated **Linux Network Namespace**. Traffic from this application is forced through an encrypted WireGuard tunnel before exiting via the main network interface, ensuring total isolation from the rest of the system.

**Update:** This version includes Systemd integration for automatic startup on boot.

## 🏗 Architecture

1.  **Namespace (Client):** An isolated network bubble (`vpn_client`) containing the browser.
2.  **WireGuard Tunnel:** An encrypted link between the namespace (`10.0.0.2`) and the host (`10.0.0.1`).
3.  **Host (Server):** The physical machine that receives tunnel traffic, applies NAT (Masquerade), and forwards it to the Internet.

## 📋 Prerequisites

* Arch Linux (or similar distribution).
* Root privileges (`sudo`).
* Required packages:
    ```bash
    sudo pacman -S wireguard-tools iptables dnsmasq
    ```

## ⚙️ Installation

### 1. WireGuard Key Generation
Create the configuration directory and generate the keys:
```bash
sudo mkdir -p /etc/wireguard
cd /etc/wireguard
umask 077
wg genkey | tee server_private | wg pubkey > server_public
wg genkey | tee client_private | wg pubkey > client_public
