# Quick Start Guide

Initialize a new peer-observer infrastructure using the template:

```bash
nix flake init --template github:0xb10c/peer-observer-infra-library
```

This creates a basic template with configuration files to get you started.

## What This Library Provides

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

## Project Structure

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
│   └── grafana-admin-password-web01.age
└── wireguard-keys/                    # Temporary key storage (gitignored)
    ├── web01-private.key
    ├── web01-public.key
    └── ....
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
TODO: https://github.com/nix-community/disko 
This is especially useful for unattended installations, re-installation after a system crash or for setting up more than one identical server

##### Create hosts/web01/disko.nix
**IMPORTANT:** Use `/dev/nvme0n1` directly, not `/dev/disk/by-id/...` to avoid kexec issues.

```nix
let
  swap = "4G";
in
{
  disko.devices = {
    disk = {
      main = {
        type = "disk";
        device = "/dev/nvme0n1";  # Direct device path
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

```bash
# Generate WireGuard key pair
wg genkey | tee wireguard-keys/web01-private.key | wg pubkey > wireguard-keys/web01-public.key

# View public key (update infra.nix with this)
cat wireguard-keys/web01-public.key
```

#### 5.2. Get SSH Host Key

After initial deployment, get the server's SSH host key:

```bash
ssh youruser@web01 "cat /etc/ssh/ssh_host_ed25519_key.pub"
```

Example output:
```
ssh-ed25519 AAAAC3N......... root@web01
```

#### 5.3. Update `secrets/secrets.nix`

**CRITICAL:** Use SSH keys, not age keys, for agenix recipients.

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

```bash
cd secrets

# Encrypt WireGuard private key
cat ../wireguard-keys/web01-private.key | agenix -e wireguard-private-key-web01.age -i ~/.ssh/id_ed25519

# Create and encrypt Grafana password
EDITOR=vi agenix -e grafana-admin-password-web01.age -i ~/.ssh/id_ed25519
# Enter a secure password, save and close

# Verify secrets exist
ls -lh *.age
```

### Step 6: Configure Local SSH

Add web01 to your local `/etc/hosts`:

```bash
sudo vi /etc/hosts
```

Add:
```
3.214.XXX.XXX    web01
```

Or configure `~/.ssh/config`:

```bash
Host web01
    HostName 3.214.XXX.XXX
    User youruser
    IdentityFile ~/.ssh/peer-observer-key.pem
```

### Step 7: Initial NixOS Installation

Use `nixos-anywhere` to convert Ubuntu to NixOS:

```bash
nix run github:nix-community/nixos-anywhere -- \
  --flake .#web01 \
  --build-on-remote \
  root@3.214.15.113 \
  --ssh-option "IdentityFile=/home/youruser/.ssh/peer-observer-key.pem"
```

**What happens:**
1. Uploads install SSH keys
2. Switches to kexec (NixOS installer in RAM)
3. Partitions disk according to `disko.nix`
4. Builds NixOS system configuration
5. Installs NixOS and GRUB bootloader
6. Reboots into NixOS

**Duration:** 15-30 minutes depending on network and CPU.


### Step 8: Verify Initial Installation

After reboot, SSH into the server:

```bash
ssh youruser@web01

# Verify NixOS
nixos-version

# Check basic services
systemctl status sshd
```
At this point, only basic services are running (SSH, networking).

### Step 9: Deploy Full Services

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


### Step 10: Verify Services

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

### Step 11: Access Dashboard

Open in browser:
```
https://observer.yourdomain.xyz
```

You should see:
- Dashboard homepage with node information
- SSL certificate valid (Let's Encrypt)
- No browser warnings


## Security Considerations

- **Limited Access Mode**: By default, webservers run in LIMITED_ACCESS mode to prevent IP address leakage
- **SSH Key Authentication**: Password login is disabled; only SSH key authentication is allowed
- **WireGuard VPN**: All inter-host communication is encrypted via WireGuard
- **ACME Certificates**: Automatic SSL certificate generation and renewal
- **Firewall**: Restrictive firewall rules with only necessary ports open

## Available Commands

When in the development shell (`nix develop`):

- `deploy <host>` - Deploy configuration to a specific host
- `build-vm <host>` - Build a VM for testing configuration

## Troubleshooting

### Common Issues

1. **Missing hardware configuration**: Ensure hardware-configuration.nix exists for each host
2. **WireGuard connection failures**: Verify public/private key pairs and IP addresses
3. **Domain not resolving**: Ensure DNS records point to webserver IP addresses
4. **Secret decryption errors**: Check that age keys are properly configured

### Getting Help

- Review the [module documentation](https://0xb10c.github.io/peer-observer-infra-library/)
- Check NixOS logs: `journalctl -u <service-name>`
- Validate configuration: `nix flake check`
