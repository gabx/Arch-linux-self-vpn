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
```

### 2. Main System Script (`self-vpn`)
Create the main logic script in the system binary path:
```bash
sudo nano /usr/local/bin/self-vpn
```

Paste the following content (Make sure to edit `INTERFACE_INTERNET`):

```bash
#!/bin/bash
# /usr/local/bin/self-vpn

# --- CONFIGURATION ---
INTERFACE_INTERNET="enp3s0"  # Check with 'ip link' (e.g., wlan0, eth0)
WG_DIR="/etc/wireguard"

start() {
    echo "🚀 Starting Self-VPN..."
    # Clean up first
    ip link delete wg0 2>/dev/null
    ip netns delete vpn_client 2>/dev/null
    
    # Server (Host)
    ip link add dev wg0 type wireguard
    ip addr add 10.0.0.1/24 dev wg0
    wg set wg0 private-key $WG_DIR/server_private listen-port 51820
    wg set wg0 peer $(cat $WG_DIR/client_public) allowed-ips 10.0.0.2/32
    ip link set wg0 up
    
    # Namespace (Client)
    ip netns add vpn_client
    ip link add dev wg1 type wireguard
    ip link set wg1 netns vpn_client
    
    # Client Config
    ip netns exec vpn_client ip addr add 10.0.0.2/24 dev wg1
    ip netns exec vpn_client ip link set lo up
    ip netns exec vpn_client ip link set wg1 up
    ip netns exec vpn_client wg set wg1 \
        private-key $WG_DIR/client_private \
        peer $(cat $WG_DIR/server_public) \
        endpoint 10.0.0.1:51820 \
        allowed-ips 0.0.0.0/0
    ip netns exec vpn_client ip route add default dev wg1
    
    # Routing & NAT
    sysctl -w net.ipv4.ip_forward=1 > /dev/null
    sysctl -w net.ipv4.conf.all.rp_filter=0 > /dev/null
    sysctl -w net.ipv4.conf.wg0.rp_filter=0 > /dev/null
    
    iptables -t nat -C POSTROUTING -o $INTERFACE_INTERNET -j MASQUERADE 2>/dev/null || \
    iptables -t nat -A POSTROUTING -o $INTERFACE_INTERNET -j MASQUERADE
    iptables -P FORWARD ACCEPT
    
    # DNS & MTU Fixes
    ip netns exec vpn_client ip link set dev wg1 mtu 1380
    ip netns exec vpn_client bash -c "echo 'nameserver 8.8.8.8' > /etc/resolv.conf"
}

stop() {
    echo "🛑 Stopping Self-VPN..."
    ip netns delete vpn_client 2>/dev/null
    ip link delete wg0 2>/dev/null
    iptables -t nat -D POSTROUTING -o $INTERFACE_INTERNET -j MASQUERADE 2>/dev/null
}

case "$1" in
    start) start ;;
    stop) stop ;;
    restart) stop; sleep 1; start ;;
    *) echo "Usage: $0 {start|stop|restart}"; exit 1 ;;
esac
```

Make it executable:
```bash
sudo chmod +x /usr/local/bin/self-vpn
```

### 3. Systemd Service
Create a system service to start the VPN automatically at boot.
```bash
sudo nano /etc/systemd/system/self-vpn.service
```

Content:
```ini
[Unit]
Description=Self-Hosted WireGuard Namespace VPN
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/self-vpn start
ExecStop=/usr/local/bin/self-vpn stop

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now self-vpn
```

### 4. Application Wrapper (`firefox-vpn`)
This script launches Firefox inside the namespace as your user.
```bash
sudo nano /usr/local/bin/firefox-vpn
```

Content:
```bash
#!/bin/bash
xhost +local: > /dev/null 2>&1
# Replace 'gabx' with your actual username if different
ip netns exec vpn_client sudo -u gabx --preserve-env=DISPLAY,XAUTHORITY,WAYLAND_DISPLAY firefox
```

Make it executable:
```bash
sudo chmod +x /usr/local/bin/firefox-vpn
```

### 5. Sudoers & Desktop Icon
**Sudoers:** To run without password prompt.
Run `sudo EDITOR=nano visudo` and add:
```text
your_username ALL=(ALL) NOPASSWD: /usr/local/bin/firefox-vpn
```

**Desktop Icon:**
Create `~/.local/share/applications/firefox-vpn.desktop`:
```ini
[Desktop Entry]
Name=Firefox VPN (Secure)
Comment=Browse via Local WireGuard VPN
Exec=sudo /usr/local/bin/firefox-vpn
Icon=firefox
Terminal=false
Type=Application
Categories=Network;WebBrowser;
```

## 🚀 Usage

* **Automatic:** The VPN starts automatically at boot via Systemd.
* **Manual Control:**
    * Restart VPN: `sudo systemctl restart self-vpn`
    * Stop VPN: `sudo systemctl stop self-vpn`
* **To Browse:** Simply click the **"Firefox VPN (Secure)"** icon.

## ✅ Verification

Check the route inside the tunnel:
```bash
sudo ip netns exec vpn_client traceroute -n 8.8.8.8
```
The first hop **must** be `10.0.0.1`.

---
*Project created for educational purposes regarding network isolation, Linux Namespaces, and WireGuard.*
