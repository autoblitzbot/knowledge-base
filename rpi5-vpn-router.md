# Raspberry Pi 5 VPN Router — via Tailscale Exit Node

Build a home VPN router from a Raspberry Pi 5, using OpenWrt and Tailscale. All LAN client traffic exits through a remote Tailscale exit node — no VPN client needed on the clients.

## Parts List

| Part | Recommendation | Notes |
|------|---------------|-------|
| **Raspberry Pi 5** (4GB or 8GB) | [Official resellers](https://www.raspberrypi.com/products/raspberry-pi-5/) | 4GB is more than enough for a router |
| **USB 3.0 Gigabit Ethernet adapter** | RTL8153 chipset adapter | See below |
| **microSD card** (16GB+) | Samsung EVO Plus, SanDisk Extreme | Class 10 / A1 minimum |
| **Power supply** | Official Pi 5 USB-C 27W (5.1V/5A) | The USB GbE adapter draws extra power, don't skimp on the PSU |
| **Cooler** | [Official Active Cooler](https://www.raspberrypi.com/products/active-cooler/) or metal case with fan | Router runs continuously, cooling is essential |
| **Ethernet cable** (x2) | Cat5e or Cat6 | WAN + LAN |

### USB Ethernet Adapter — Which One?

**Look for the RTL8153 chipset**, not AX88179. The Realtek driver has better Linux/OpenWrt support — more stable and faster on the Pi.

Specific recommendations:
- **Ugreen USB 3.0 Gigabit Ethernet Adapter** — RTL8153, widely available, cheap
- **Cable Matters USB 3.0 to Gigabit Ethernet** — RTL8153, reliable
- **TP-Link UE300** — RTL8153, though sometimes has boot-compatibility issues on Pi

> **Tip:** Plug in the adapter *before* powering on the Pi — some adapters can cause a reboot if hot-plugged.

## Network Topology

```
Internet ←→ ISP Router ←→ [eth0 WAN] RPi5 [eth1 LAN] ←→ Switch/AP ←→ Clients
                                          ↕
                                    Tailscale tunnel
                                          ↕
                                    Remote Exit Node
```

- **eth0** (built-in) → WAN (facing ISP router, DHCP client)
- **eth1** (USB adapter) → LAN (internal network, DHCP server)
- The Pi NATs LAN traffic and routes it through the Tailscale tunnel to a remote exit node

## 1. Installing OpenWrt

### Download the Image

1. Go to the [OpenWrt Firmware Selector](https://firmware-selector.openwrt.org/?version=24.10.1&target=bcm27xx/bcm2712&id=rpi-5)
2. Select the **24.10.1** stable release (or newer if available)
3. Download the **Factory (SQUASHFS)** image (`.img.gz`)

> **Why SquashFS and not EXT4?** SquashFS uses a read-only root + writable overlay. Advantage: you can restore factory state anytime with `firstboot` (useful if you lock yourself out with the firewall), and it means fewer SD card writes → longer lifespan. EXT4 images don't have a factory reset option.

### Write to SD Card

Linux/macOS:
```bash
# Extract and write (replace /dev/sdX with your card's actual device)
gunzip openwrt-*.img.gz
sudo dd if=openwrt-*.img of=/dev/sdX bs=4M status=progress
sync
```

Or use [Raspberry Pi Imager](https://www.raspberrypi.com/software/): "Use custom" → select the `.img` file.

### First Boot

1. Insert the microSD into the Pi
2. Plug in the USB Ethernet adapter **first**
3. Connect **eth0** (built-in) to your laptop or switch
4. Power on
5. Wait ~1 minute for boot
6. In your browser: **http://192.168.1.1** → LuCI web interface
7. First step: set a root password: **System → Administration → Router Password**

## 2. Network Configuration

### Identify Interfaces

Via SSH (`ssh root@192.168.1.1`):

```bash
# List interfaces
ip link show
# Usually: eth0 = built-in, eth1 = USB adapter
# Verify:
dmesg | grep -i eth
```

### WAN Interface (eth0 → facing ISP)

**Network → Interfaces → Add new interface:**

| Field | Value |
|-------|-------|
| Name | `wan` |
| Protocol | DHCP client |
| Device | `eth0` |
| Firewall zone | `wan` |

### LAN Interface (eth1 → internal network)

The default `lan` interface is on the `br-lan` bridge. Modify it:

**Network → Interfaces → LAN → Edit:**

| Field | Value |
|-------|-------|
| Protocol | Static address |
| IPv4 address | `192.168.2.1` |
| Netmask | `255.255.255.0` |
| Device | `eth1` (USB adapter) |
| DHCP | Enabled (default range is fine) |
| Firewall zone | `lan` |

> **Important:** If your ISP router also uses 192.168.1.x, assign a different subnet to the LAN (e.g., 192.168.2.0/24), otherwise there will be a conflict.

### Firewall

**Network → Firewall:** The default rules are fine:
- `lan → wan`: **ACCEPT** (LAN clients can reach the internet)
- `wan → lan`: **REJECT** (nothing comes in from outside)
- Masquerading (NAT): **ON** for the `wan` zone

Save: **Save & Apply**.

### Test

A device connected to the LAN port should receive a 192.168.2.x address via DHCP and be able to reach the internet through the ISP router.

```bash
# From the Pi:
ping -c 3 1.1.1.1
# From a LAN client:
ping -c 3 8.8.8.8
```

## 3. Installing Tailscale

### Package Installation

```bash
opkg update
opkg install tailscale
```

> **Note:** The Tailscale package takes ~22 MB of space. No issue with a 16GB SD card.

### Start and Log In

```bash
# Enable and start the service
/etc/init.d/tailscale enable
/etc/init.d/tailscale start

# Log in
tailscale up --accept-routes
```

This gives you a link — open it in your browser and log in with your Tailscale account. The Pi will appear in the admin console: [https://login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines)

### Firewall Zone for Tailscale

**Network → Firewall → Add zone:**

| Field | Value |
|-------|-------|
| Name | `tailscale` |
| Input | ACCEPT |
| Output | ACCEPT |
| Forward | ACCEPT |
| Covered devices | `tailscale0` |
| Allow forward from | `lan` |
| Allow forward to | `lan`, `wan` |

**Save & Apply.**

## 4. Connecting to a Remote Exit Node

### Prerequisite: An Exit Node in Your Tailscale Network

You need another machine in your tailnet running as an exit node. This can be:
- A VPS (e.g., Hetzner, DigitalOcean) somewhere in the world
- Another home machine in a different country
- Any device running Tailscale

On that machine:
```bash
tailscale up --advertise-exit-node
```

Then in the [Tailscale Admin Console](https://login.tailscale.com/admin/machines), approve the exit node:
**Machine → "..." menu → Edit route settings → Use as exit node ✓**

### Point the Pi to the Exit Node

```bash
# List available exit nodes
tailscale exit-node list

# Connect (replace <HOSTNAME> with the exit node's name)
tailscale set --exit-node=<HOSTNAME>

# Verify — the Pi's IP should now be the exit node's IP
curl -s https://ifconfig.me
```

### Why Does This Work for LAN Clients Too?

LAN clients are NATed through the Pi. When the Pi's traffic goes through the exit node, **all LAN client traffic automatically goes there too** — no need to install Tailscale on the clients.

### Disable Exit Node

```bash
# Revert to normal routing (exit via ISP)
tailscale set --exit-node=
```

## 5. Auto-Start and Optimization

### CPU Performance Governor

A router is under constant load; the powersave governor slows down crypto operations:

```bash
# Set immediately
echo performance > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor

# Set at boot — add to /etc/rc.local (before exit 0):
echo performance > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```

### Tailscale Exit Node Setting After Reboot

`tailscale set --exit-node=<HOSTNAME>` **persists across reboots** — Tailscale stores the setting in its state file (`/var/lib/tailscale/`), and the daemon applies it automatically on restart. No extra boot script needed.

### Watchdog

If the Tailscale tunnel goes down, LAN clients lose internet. A simple watchdog cron job:

```bash
cat << 'EOF' > /etc/tailscale-watchdog.sh
#!/bin/sh
# If internet is unreachable via exit node, reconnect
if ! ping -c 2 -W 5 1.1.1.1 > /dev/null 2>&1; then
    logger -t tailscale-watchdog "Connection lost, reconnecting..."
    tailscale set --exit-node=
    sleep 5
    tailscale set --exit-node=<HOSTNAME>
fi
EOF
chmod +x /etc/tailscale-watchdog.sh
```

Add to crontab (every 5 minutes):

```bash
echo '*/5 * * * * /etc/tailscale-watchdog.sh' >> /etc/crontabs/root
/etc/init.d/cron restart
```

## Performance

| Measurement | Expected Value |
|-------------|---------------|
| WireGuard (Tailscale) throughput | ~800–900 Mbps |
| One-direction max (CPU-limited) | ~1 Gbps |
| Latency overhead | +1–3 ms (within LAN) |

The Pi 5's Cortex-A76 cores have ARMv8 crypto extensions, which provide hardware acceleration for WireGuard. On a 1 Gbps connection, the CPU is the bottleneck, not the NIC.

## Troubleshooting

### USB Adapter Not Showing Up

```bash
dmesg | grep -i eth
lsusb
ip link show
```

If `eth1` is not visible, try plugging in the adapter with the Pi powered off, then boot.

### Tailscale Won't Connect

```bash
tailscale status
# If "Logged out":
tailscale up --accept-routes
```

### LAN Clients Not Getting IP Addresses

```bash
# Is the DHCP server running?
/etc/init.d/dnsmasq status
# Check logs:
logread | grep dhcp
```

### No Internet via Exit Node

```bash
# Exit node status
tailscale exit-node list
# Ping test directly from the Pi
ping -c 3 1.1.1.1
# If it fails, temporarily disable:
tailscale set --exit-node=
# If it works without exit node, the issue is with the exit node
```

## References

- [OpenWrt Firmware Selector — RPi 5](https://firmware-selector.openwrt.org/?version=24.10.1&target=bcm27xx/bcm2712&id=rpi-5)
- [Tailscale Exit Nodes documentation](https://tailscale.com/kb/1103/exit-nodes)
- [Tailscale Subnet Routers documentation](https://tailscale.com/kb/1019/subnets)
- [Tailscale on OpenWrt — WunderTech](https://www.wundertech.net/how-to-set-up-tailscale-on-openwrt/)
- [OpenWrt Forum — Tailscale exit node on 24.10](https://forum.openwrt.org/t/how-to-configure-tailscale-18-02-on-24-10-to-access-remote-exit-node/226561)
