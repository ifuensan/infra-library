# Setup Guide

## Introduction
The peer-observer-infra-library enables you to deploy and manage Bitcoin peer-observer 
infrastructure using NixOS. 

It provides:

### Node Infrastructure
- **Bitcoin Core nodes** with customizable configurations
- **Peer-observer extractors and tools** for monitoring Bitcoin network activity
- **NATS server** for inter-component communication
- **WireGuard VPN** for secure communication between nodes and webservers
- **Tor and I2P support** for enhanced privacy and connectivity

### Web Infrastructure  
- **Nginx webserver** with automatic SSL certificates
- **Prometheus and Grafana** for metrics collection and visualization
- **Peer-observer frontend tools** including WebSocket interfaces
- **Fork-observer** to monitor blockchain forks across all connected nodes
- **Addrman-observer** for address manager analysis

### Project Structure
Be carefull with you private configuration and secret key don't push it to git.
```
peer-infra-library/
├── flake.nix                          # Main flake configuration
├── infra.nix                          # Infrastructure definitions
├── hosts/                             # nodes and frontend
│   └── web01/
│       ├── disko.nix                  # Disk partitioning config
│       └── hardware-configuration.nix # Hardware-specific config
│   └── node01/
│       └── ...
├── secrets/
│   ├── secrets.nix                    # Secret definitions (public keys)
│   ├── wireguard-private-key-web01.age
│   ├── ...
│   ├── web01-private.key
│   ├── web01-public.key
│   └── grafana-admin-password-web01.age
```

## Setting Up Your Infrastructure

### 1. Initialize Your Configuration

```bash
nix flake init --template github:0xb10c/peer-observer-infra-library
```

### 2. Configure Your Infrastructure
Define your setup:
- **Global settings**: Admin user, SSH keys, and shared configuration
- **Nodes**: Bitcoin observation nodes with unique IDs and WireGuard configuration
- **Webservers**: Frontend interfaces with domains and Grafana access

Key areas to configure:
- Admin username and SSH public keys
- Node descriptions and architecture (x86_64-linux or aarch64-linux)  
- WireGuard IP addresses and public keys
- Domain names for webservers
- Hardware configuration modules

#### 2.1 Create Configuration Files

##### Create hosts/web01/disko.nix
We are using for Declarative disk partitioning [disko.nix](https://github.com/nix-community/disko). This is especially useful for unattended installations, re-installation after a system crash or for setting up more than one identical server.

Use `sudo fdisk -l` in destination host to review wich is your principal disk device (i.e. `/dev/sda` o `/dev/nvme0n1` , etc.).

```nix
let
  swap = "4G";
in
{
  disko.devices = {
    disk = {
      main = {
        type = "disk";
        device = "/dev/nvme0n1";  # Direct device path, replice with yours.
        content = {
          type = "gpt";
          partitions = {
            boot = {
              size = "1M";
              type = "EF02";  # GRUB MBR
            };
            ESP = {
              size = "512M";
              type = "EF00";
              content = {
                type = "filesystem";
                format = "vfat";
                mountpoint = "/boot";
                mountOptions = [ "umask=0077" ];
              };
            };
            swap = {
              size = swap;
              type = "8200";
              content = {
                type = "swap";
              };
            };
            root = {
              size = "100%";
              content = {
                type = "filesystem";
                format = "ext4";
                mountpoint = "/";
              };
            };
          };
        };
      };
    };
  };
}
```

#### 3.2. Create `hosts/web01/hardware-configuration.nix`

```nix
{ config, lib, pkgs, modulesPath, ... }:
{
  imports = [
    (modulesPath + "/profiles/qemu-guest.nix")
  ];

  boot.initrd.availableKernelModules = [ "nvme" "xhci_pci" "ahci" "sd_mod" ];
  boot.initrd.kernelModules = [ ];
  boot.kernelModules = [ ];
  boot.extraModulePackages = [ ];

  boot.loader.grub = {
    enable = true;
    efiSupport = true;
    efiInstallAsRemovable = true;
    device = "nodev";
  };

  networking.useDHCP = lib.mkDefault true;
  nixpkgs.hostPlatform = lib.mkDefault "x86_64-linux";
}
```

**Key differences from auto-generated config:**
- Includes `qemu-guest.nix` profile (not `hypervGuest`)
- Includes `nvme` in kernel modules
- Properly configured GRUB for EFI boot

### Step 5: Generate and Encrypt Secrets

#### 5.1. Generate WireGuard Keys

Install the wireguard-tools package.

```bash
# Generate WireGuard key pair
wg genkey | tee secrets/web01-private.key | wg pubkey > secrets/web01-public.key

# View public key (update infra.nix with this)
cat secrets/web01-public.key
```

You need generate the Wireguard keys for all your nodes (node01, node02, etc.)

#### 5.2. Get SSH Host Key

### Step 6: Configure your local

Add your servers name and IP to your local `/etc/hosts` or Or configure `~/.ssh/config`.

Get the server's SSH host key:

```bash
ssh youruser@web01 "cat /etc/ssh/ssh_host_ed25519_key.pub"
```

Example output:
```
ssh-ed25519 AAAAC3N......... root@web01
```

Get your personal SSH Key:
```bash
cat ~/.ssh/id_ed25519.pub
```
If you don't have one you'll need to generate.

#### 5.3. Update `secrets/secrets.nix`

Use here you SSH keys, not age keys, for agenix recipients. Setup the names of your nodes and webserver as you need it. Read the instruccionts in the secret.nix file.

```nix
let
  # Your personal SSH key (for encrypting/decrypting)
  user = "ssh-ed25519 AAAAC3N..... youruser@yourdomain.com";
  
  # Server SSH host keys (obtained after deployment)
  web01 = "ssh-ed25519 AAAAC3N.....;
in
{
  "wireguard-private-key-web01.age".publicKeys = [
    web01
    user
  ];
  
  "grafana-admin-password-web01.age".publicKeys = [
    web01
    user
  ];
}
```

#### 5.4. Encrypt Secrets

Use `nix develop` to enter the Nix development shell.

```bash
cd secrets

# Encrypt WireGuard private key
cat web01-private.key | agenix -e wireguard-private-key-web01.age -i ~/.ssh/id_ed25519

# Create and encrypt Grafana password
EDITOR=vi agenix -e grafana-admin-password-web01.age -i ~/.ssh/id_ed25519
# Enter a secure password, save and close

# Verify secrets exist
ls -lh *.age

# Go to you root project directory
cd ..
```
### Step 7: Setup `infra.nix`

Follow the instruction inside the `infra.nix` template. Be carefull with:
- Setup your admin user:
```bash
    admin = {
      username = "placeholder";
      sshPubKeys = [
        "ssh-rsa AAAAAAA..."
        "ssh-ed25519 AAAAC..."
      ];
    };
```
- Configure your infrastructure definition with your specific values:
```bash
    node01 = {
      id = 1;
      wireguard = {
        ip = "10.21.0.1";
        pubkey = "fakekH7xb/DdO...";
      };
      setup = true;
      arch = "x86_64-linux";
      # FIXME:
      # Feel free to set this to whatever you like. Note that this might be shown
      # publicly.
      description = ''
        This is a placeholder description for node01. HTML is <b>supported<b>.
      '';
      # Bitcoin node configuration
      bitcoind = {
        net = {
           useTor = false;
           useI2P = false;
           useASMap = true;
        };
      };
      extraConfig = {  };
      extraModules = [
        disko.nixosModules.disko
        ./hosts/node01-disko.nix
        ./hosts/node01-hardware.nix
      ];
    };
```
Key values to update:
- wgPublicKey: Output from cat secrets/web01-public.key
- domain: Your actual domain with DNS pointing to the server
- security.acme.defaults.email: Valid email for Let's Encrypt notifications

Repeat similar configuration for each node (node01, node02, etc.) with their respective WireGuard addresses and keys.

### Step 8: Initial NixOS Installation

Use `nixos-anywhere` to convert Ubuntu to NixOS:

```bash
nix run github:nix-community/nixos-anywhere -- \
  --flake .#web01 \
  --build-on-remote \
  [user]@web01
```
Use `--ssh-option "IdentityFile=/home/youruser/.ssh/peer-observer-key.pem"` for VPS with this requierment, like AWS.

**What happens:**
1. Uploads install SSH keys
2. Switches to kexec (NixOS installer in RAM)
3. Partitions disk according to `disko.nix`
4. Builds NixOS system configuration
5. Installs NixOS and GRUB bootloader
6. Reboots into NixOS

### Step 9: Verify Initial Installation

After reboot, SSH into the server:

```bash
ssh youruser@web01

# Verify NixOS
nixos-version

# Check basic services
systemctl status sshd
```
At this point, only basic services are running (SSH, networking).

### Step 10: Deploy Full Services

Change `setup = false` in `infra.nix` to activate all services:

```nix
web01 = {
  setup = false;  # Activate nginx, grafana, wireguard, etc.
  ...
}
```

Deploy the full configuration:

```bash
# Enter development shell
nix develop

# Deploy using the built-in function
deploy web01
```

**What happens:**
1. Copies new system configuration to web01
2. Decrypts secrets using agenix
3. Activates services: WireGuard, nginx, Grafana, Prometheus
4. Reloads systemd units

**Duration:** 5-10 minutes.


### Step 11: Verify Services

```bash
ssh youruser@web01

# Check WireGuard
sudo wg show

# Check nginx
systemctl status nginx
sudo ss -tlnp | grep -E "(80|443)"

# Check Grafana
systemctl status grafana

# Check SSL certificate
systemctl status acme-observer.yourdomain.xyz.service

# Test connectivity to node01 (if configured)
ping -c 3 10.21.0.1
```

Expected output for WireGuard:
```
interface: wg-peerobserver
  public key: IByMint............
  private key: (hidden)
  listening port: 51820

peer: WYVP74.........  # node01
  endpoint: 172.31.XX.XX:44192
  allowed ips: 10.21.0.1/32
  latest handshake: X seconds ago
  transfer: XX KiB received, XX KiB sent
```

### Step 12: Access Dashboard

Open in browser and use your configured domain.

By default, webserver is configured with `access_DANGER = "LIMITED_ACCESS"` which restricts certain features for security.
To access Grafana, monitoring dashboards, debug logs, and real-time WebSocket data without exposing them publicly, use an SSH tunnel.

```bash
# From your local machine
ssh -f -N -L 8002:localhost:8002 youruser@web01
```

Then access in your browser:
```
http://localhost:8002/monitoring      # Grafana dashboards
http://localhost:8002/addrman         # Address manager visualization
http://localhost:8002/debug-logs      # Bitcoin Core debug logs
http://localhost:8002/websocket       # Real-time WebSocket data
```

| Route | Description | Port |
|-------|-------------|------|
| `/` | Main dashboard | 8002 |
| `/monitoring` | Grafana dashboards (port 9321 proxied) | 8002 |
| `/addrman` | Address manager visualization | 8002 |
| `/debug-logs` | Bitcoin Core debug logs from nodes | 8002 |
| `/debug-logs/node01/` | Debug logs from node01 | 8002 |
| `/forks` | Fork observer | 8002 |
| `/websocket` | WebSocket interface | 8002 |
| `/websocket/node01/` | Real-time data from node01 | 8002 |


