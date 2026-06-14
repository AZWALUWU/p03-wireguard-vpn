Here is the complete, production-ready engineering runbook summarizing every step, command, and script executed throughout this project. You can save this as your master reference note.

---

# 📑 Master Runbook: Deploying a Self-Hosted WireGuard VPN on GCP

This comprehensive deployment guide covers everything from provisioning a cloud server to hardening, setting up Dynamic DNS, configuring cryptographic keys, and executing network troubleshooting.

---

## Phase 1: Cloud Provisioning & Server Hardening

### 1. GCP VM Instance Specifications

* **Machine Series/Type:** `E2` -> `e2-micro` (2 vCPUs, 1 GB RAM) -> *Always Free Tier*
* **Region:** `us-central1`, `us-east1`, or `us-west1`
* **Boot Disk:** Ubuntu 22.04 LTS (Standard Persistent Disk, 30 GB)
* **IP Configuration:** Promoted public IP from **Ephemeral** to **Static** under *VPC Network > IP Addresses*.

### 2. OS Update & Initial Firewall Hardening

Connect to the VPS via SSH and initialize system maintenance and local network security using UFW (Uncomplicated Firewall):

```bash
# Update local package index and upgrade all system packages
sudo apt update && sudo apt upgrade -y

# Allow SSH traffic so you don't get locked out
sudo ufw allow 22/tcp

# Allow incoming WireGuard VPN traffic
sudo ufw allow 51820/udp

# Enable the firewall (press 'y' and Enter to confirm)
sudo ufw enable

# Verify firewall status
sudo ufw status verbose

```

---

## Phase 2: Dynamic DNS Setup (DuckDNS Integration)

To handle dynamic IP shifts gracefully, a script automatically pings DuckDNS to map your domain to the server's current public IP.

### 1. Automated DDNS Script

Create a directory and write the update script:

```bash
mkdir ~/duckdns && cd ~/duckdns
nano duck.sh

```

**Script Content (`duck.sh`):**

```bash
#!/bin/bash
echo url="https://www.duckdns.org/update?domains=YOUR_SUBDOMAIN&token=YOUR_DUCKDNS_TOKEN&ip=" | curl -k -o ~/duckdns/duck.log -K -

```

*(Replace `YOUR_SUBDOMAIN` with your custom prefix and `YOUR_DUCKDNS_TOKEN` with your secret DuckDNS token).*

### 2. Script Permissions & Cron Automation

```bash
# Grant execution privileges only to the owner
chmod 700 duck.sh

# Open the cron tab scheduler
crontab -e

```

Add the following line at the absolute bottom of the crontab file to execute the script every 5 minutes:

```text
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1

```

**Verification:**
Run the script manually and check for an `OK` log output:

```bash
./duck.sh
cat duck.log

```

---

## Phase 3: WireGuard Installation & Cryptography

### 1. Install WireGuard Package

```bash
sudo apt update
sudo apt install wireguard -y

```

### 2. Cryptographic Keypair Generation

WireGuard relies on asymmetric cryptography. We must generate a public and private key pair for both the server and the client device.

```bash
# Switch to root to secure key management files
sudo -i
cd /etc/wireguard
umask 077

# Generate Server Keypair
wg genkey | tee server_private.key | wg pubkey > server_public.key

# Generate Client Keypair
wg genkey | tee client_private.key | wg pubkey > client_public.key

```

*Note: Use `cat <filename>.key` to view the strings when building configuration profiles.*

### 3. Kernel IP Forwarding Configuration

To turn your Linux server into a functional network router that passes client traffic to the internet, you must enable IPv4 forwarding:

```bash
nano /etc/sysctl.conf

```

Uncomment or append this line:

```text
net.ipv4.ip_forward=1

```

Apply the changes instantly without system reboot:

```bash
sysctl -p

```

---

## Phase 4: Server Network Configuration

### 1. Identify Network Interface

Identify the main interface card name (e.g., `ens4` or `eth0`):

```bash
ip route list default

```

### 2. Create Server Interface Profile (`wg0.conf`)

```bash
nano /etc/wireguard/wg0.conf

```

**Configuration Content (`wg0.conf`):**

```text
[Interface]
Address = 10.0.0.1/24
SaveConfig = true
ListenPort = 51820
PrivateKey = <INSERT_SERVER_PRIVATE_KEY_STRING>

# NAT and Routing tables invocation upon tunnel startup
PostUp = ufw route allow in on wg0 && iptables -t nat -A POSTROUTING -o ens4 -j MASQUERADE

# Clean up routing tables upon tunnel shutdown
PostDown = ufw route delete allow in on wg0 && iptables -t nat -D POSTROUTING -o ens4 -j MASQUERADE

[Peer]
PublicKey = <INSERT_CLIENT_PUBLIC_KEY_STRING>
AllowedIPs = 10.0.0.2/32

```

*(Modify `ens4` if your actual network interface card name differs).*

### 3. Start WireGuard Daemon

```bash
# Bring up the tunnel interface
wg-quick up wg0

# Enable automatic start upon system boot
systemctl enable wg-quick@wg0

# Check WireGuard status
wg show

# Exit root mode back to standard user
exit

```

---

## Phase 5: Client Configuration & Onboarding

### 1. Generate Client Profile

Create a configuration file to pass down to your mobile phone or laptop:

```bash
nano ~/client.conf

```

**Configuration Content (`client.conf`):**

```text
[Interface]
Address = 10.0.0.2/24
PrivateKey = <INSERT_CLIENT_PRIVATE_KEY_STRING>
DNS = 1.1.1.1, 8.8.8.8

[Peer]
PublicKey = <INSERT_SERVER_PUBLIC_KEY_STRING>
Endpoint = YOUR_SUBDOMAIN.duckdns.org:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25

```

*(Setting `AllowedIPs = 0.0.0.0/0` establishes Full Tunneling, ensuring all client internet traffic passes securely through the cloud server).*

### 2. QR Code Generation for Mobile Pairing

```bash
# Install QR rendering tool
sudo apt install qrencode -y

# Render configuration profile to the terminal screen
qrencode -t ansiutf8 < ~/client.conf

```

Scan this QR code using the official WireGuard app on iOS or Android.

---

## Phase 6: Critical Security Troubleshooting (Post-Mortem Analysis)

### The "Connected, but No Internet" Issue

> **Symptom:** The client app claims the tunnel is "Connected", but the device completely loses internet access.

### 🔎 Diagnosis Methodology

Run the monitoring command on the cloud server:

```bash
sudo wg show

```

* **Observation:** The `latest handshake` and `transfer data received` parameters are completely blank or zero.
* **Conclusion:** The tunnel is active locally on the phone, but packets are failing to reach the cloud machine. The cloud provider's network boundary is discarding inbound traffic.

### 🛠️ Resolution Action

The server OS is ready, but the cloud infrastructure layer is blocking traffic. You must configure GCP's external ingress policy:

1. Go to **Google Cloud Console > VPC Network > Firewall**.
2. Click **Create Firewall Rule**.
3. Apply these explicit network parameters:
* **Name:** `allow-wireguard`
* **Targets:** `All instances in the network`
* **Source IPv4 ranges:** `0.0.0.0/0`
* **Protocols and ports:** Check **UDP**, enter port `51820`.


4. Click **Create**.

### 📊 Empirical Verification

Reconnect the WireGuard client profile. Navigate to `whoer.net` or `icanhazip.com` on the client device.

* **Expected Result:** The detected IP address matches your GCP VM Instance's static public IP, and the detected ISP is explicitly listed as **Google LLC**. This confirms full data path encryption and functional server-side NAT.
