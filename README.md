# Arch Linux Self-VPN & Browser Isolation

This project deploys a full **WireGuard VPN (Client + Server)** on a single local Arch Linux machine.

The goal is to isolate a specific application (e.g., Firefox) inside a dedicated **Linux Network Namespace**. Traffic from this application is forced through an encrypted WireGuard tunnel before exiting via the main network interface, ensuring total isolation from the rest of the system.

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
```

### 2. Main Script (`start_vpn.sh`)
Place the `start_vpn.sh` script (provided in this repository) into your scripts folder and make it executable:
```bash
chmod +x start_vpn.sh
```
*Note: Edit the `INTERFACE_INTERNET` variable (e.g., `enp3s0` or `wlan0`) and `USER_NAME` inside the script to match your configuration.*

### 3. Launch Wrapper (`firefox-vpn`)
This script launches Firefox inside the namespace without typing complex commands.
Copy the file to `/usr/local/bin/`:
```bash
sudo cp firefox-vpn /usr/local/bin/
sudo chmod +x /usr/local/bin/firefox-vpn
```

### 4. Sudoers Configuration (Optional but recommended)
To launch the browser without typing the root password every time:
```bash
sudo EDITOR=nano visudo
```
Add this line at the very end:
```text
your_username ALL=(ALL) NOPASSWD: /usr/local/bin/firefox-vpn
```

### 5. Desktop Entry (.desktop)
To add an icon to your application menu.
Copy the `firefox-vpn.desktop` file to `~/.local/share/applications/`.

## 🚀 Usage

1.  **Start the VPN** (Required after every reboot):
    ```bash
    sudo ./start_vpn.sh
    ```

2.  **Launch the Browser**:
    * Via terminal: `firefox-vpn`
    * Or via the **"Firefox VPN (Secure)"** icon in your application menu.

## ✅ Verification

To verify that traffic is passing through the tunnel (10.0.0.1) and not directly through your router:

```bash
sudo ip netns exec vpn_client traceroute -n 8.8.8.8
```
The first line **must** be: `1  10.0.0.1 ...`

---
