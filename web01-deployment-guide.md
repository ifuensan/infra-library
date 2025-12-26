# Deployment Guide: web01 (Dashboard Server)

## Overview

This guide documents the complete deployment process for the **web01** server in the peer-observer infrastructure. The web01 server hosts the public-facing dashboard, Grafana, and nginx, connecting to observation nodes via WireGuard.

**Environment:**
- Platform: AWS EC2
- Instance Type: t3.small
- Base OS: Ubuntu 24.04 (converted to NixOS during deployment)
- Region: us-east-X
- Final OS: NixOS 24.05

**Server Details:**
- Hostname: web01
- Public IP: 3.214.XXX.XXX
- Domain: observer.yourdomain.xyz
- WireGuard IP: 10.21.1.1/24

---

## Prerequisites

### 1. Local Development Environment

```bash
# Required tools
- nix (with flakes enabled)
- SSH access to AWS
- agenix (for secret management)
- AWS CLI configured
```

### 2. AWS Infrastructure

- AWS EC2 instance running Ubuntu 24.04
- Security Group configured with:
  - Port 22 (SSH) - Your IP or 0.0.0.0/0
  - Port 80 (HTTP) - 0.0.0.0/0
  - Port 443 (HTTPS) - 0.0.0.0/0
  - Port 51820/UDP (WireGuard) - 0.0.0.0/0

### 3. DNS Configuration

- Domain with A record pointing to server IP:
  ```
  observer.yourdomain.xyz → 3.214.XXX.XXX
  ```

---

## Project Structure

```
test-peer-infra-library/
├── flake.nix                          # Main flake configuration
├── infra.nix                          # Infrastructure definitions
├── hosts/
│   └── web01/
│       ├── disko.nix                  # Disk partitioning config
│       └── hardware-configuration.nix # Hardware-specific config
├── secrets/
│   ├── secrets.nix                    # Secret definitions (public keys)
│   ├── wireguard-private-key-web01.age
│   └── grafana-admin-password-web01.age
└── wireguard-keys/                    # Temporary key storage (gitignored)
    ├── web01-private.key
    └── web01-public.key
```

---

## Step-by-Step Deployment

### Step 1: Provision EC2 Instance

Create an Ubuntu 24.04 instance in AWS:

```bash
aws ec2 run-instances \
  --image-id ami-dddd \  # Ubuntu 24.04 LTS
  --instance-type t3.small \
  --key-name peer-observer-key \
  --security-group-ids sg-bbbb \
  --subnet-id subnet-XXXXXXXX \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":50,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web01}]'
```

Wait for instance to be running and note the public IP.

### Step 2: Obtain Disk ID

SSH into the Ubuntu instance to get the disk identifier:

```bash
ssh -i ~/.ssh/peer-observer-key.pem ubuntu@3.214.XXX.XXX "ls -la /dev/disk/by-id/"
```

Example output:
```
nvme-Amazon_Elastic_Block_Store_volxxxxxxx -> ../../nvme0n1
```

### Step 3: Create Configuration Files

#### 3.1. Create `hosts/web01/disko.nix`

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

### Step 4: Configure Infrastructure

Update `infra.nix` to define web01:

```nix
webservers = {
  web01 = {
    id = 1;
    setup = false;  # false = full services, true = basic infrastructure only
    arch = "x86_64-linux";
    description = "Peer Observer Dashboard - HackNodes Lab";
    domain = "observer.yourdomain.xyz";

    wireguard = {
      ip = "10.21.1.1";
      pubkey = "IByMint...............";
    };

    grafana.admin_user = "ifuensan";
    
    access_DANGER = "LIMITED_ACCESS";  # or "FULL_ACCESS" for public demo

    index = {
      limitedAccessNotice = ''
        <div class="alert alert-info" role="alert">
          <h2>🔒 HackNodes Lab - Peer Observer</h2>
          <p><strong>Bitcoin Network Monitoring Dashboard</strong></p>
          <p>Real-time insights into Bitcoin P2P network behavior.</p>
        </div>
      '';
    };

    extraConfig = {
      security.acme.acceptTerms = true;
      security.acme.defaults.email = "youremail@yourdomain.com";
    };

    extraModules = [
      disko.nixosModules.disko
      ./hosts/web01/disko.nix
      ./hosts/web01/hardware-configuration.nix
    ];       
  };
};
```

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
EDITOR=nano agenix -e grafana-admin-password-web01.age -i ~/.ssh/id_ed25519
# Enter a secure password, save and close

# Verify secrets exist
ls -lh *.age
```

### Step 6: Configure Local SSH

Add web01 to your local `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Add:
```
3.214.XXX.XXX    web01
```

Or configure `~/.ssh/config`:

```bash
Host web01
    HostName 3.214.XXX.XXX
    User ifuensan
    IdentityFile ~/.ssh/peer-observer-key.pem
```

### Step 7: Initial NixOS Installation

Use `nixos-anywhere` to convert Ubuntu to NixOS:

```bash
nix run github:nix-community/nixos-anywhere -- \
  --flake .#web01 \
  --build-on-remote \
  root@3.214.15.113 \
  --ssh-option "IdentityFile=/home/ifuensan/.ssh/peer-observer-key.pem"
```

**What happens:**
1. Uploads install SSH keys
2. Switches to kexec (NixOS installer in RAM)
3. Partitions disk according to `disko.nix`
4. Builds NixOS system configuration
5. Installs NixOS and GRUB bootloader
6. Reboots into NixOS

**Duration:** 15-30 minutes depending on network and CPU.

**Common Issues:**

- **"Problem opening /dev/disk/by-id/..."**: Use direct device path (`/dev/nvme0n1`) in disko.nix
- **Password prompt**: Ensure correct SSH key path with full absolute path
- **Boot failure**: Check that hardware-configuration.nix has proper GRUB config with EFI support

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

---

## Troubleshooting

### Issue: Agenix Fails to Decrypt Secrets

**Symptom:**
```
age: error: no identity matched any of the recipients
```

**Solution:**
1. Get server's SSH host key: `ssh youruser@web01 "cat /etc/ssh/ssh_host_ed25519_key.pub"`
2. Update `secrets/secrets.nix` with correct key
3. Delete old encrypted secrets: `rm secrets/*-web01.age`
4. Re-encrypt: `cat wireguard-keys/web01-private.key | agenix -e secrets/wireguard-private-key-web01.age -i ~/.ssh/id_ed25519`
5. Deploy again: `deploy web01`

### Issue: WireGuard Not Connecting to node01

**Symptom:**
```
ping: sendmsg: Destination address required
```

**Solution:**
1. Verify node01 has WireGuard running: `ssh node01 "sudo wg show"`
2. Check that node01 has web01 as peer
3. Verify secrets are decrypted: `ls -la /run/agenix/`
4. Check WireGuard logs: `journalctl -u wireguard-wg-peerobserver -n 50`

### Issue: Certificate Not Valid

**Symptom:**
Browser shows "Not Secure" or self-signed certificate warning.

**Solution:**
1. Verify DNS resolves: `dig observer.yourdomain.xyz`
2. Check ACME service: `systemctl status acme-observer.yourdomain.xyz.service`
3. View logs: `journalctl -u acme-observer.yourdomain.xyz.service`
4. Restart ACME: `sudo systemctl restart acme-observer.yourdomain.xyz.service`

### Issue: Cannot SSH After Deployment

**Symptom:**
```
ssh: connect to host web01 port 22: Connection timed out
```

**Solution:**
1. Check Security Group allows SSH from your IP
2. Verify instance is running in AWS console
3. Check system logs in AWS: Actions → Monitor and troubleshoot → Get system log
4. Use EC2 Instance Connect from AWS console

---

## Configuration Updates

To update web01 configuration after deployment:

```bash
# 1. Make changes to infra.nix or host configs
nano infra.nix

# 2. Enter development shell
nix develop

# 3. Deploy changes
deploy web01

# 4. Verify changes
ssh youruser@web01
systemctl status <changed-service>
```

---

## Security Considerations

1. **Secrets Management**
   - Never commit unencrypted keys to git
   - Store WireGuard private keys in `wireguard-keys/` (gitignored)
   - Use agenix for all sensitive data
   - Rotate secrets periodically

2. **Network Security**
   - Limit SSH access to known IPs in Security Group
   - WireGuard provides encrypted tunnel to nodes
   - Let's Encrypt provides HTTPS encryption for public access

3. **Access Control**
   - Configure `access_DANGER = "LIMITED_ACCESS"` for production
   - Use strong Grafana admin password
   - Regularly update NixOS: `nix flake update && deploy web01`

---

## Maintenance

### Update NixOS

```bash
# Update flake inputs
nix flake update

# Deploy updates
nix develop
deploy web01
```

### Check Service Health

```bash
ssh youruser@web01

# All services status
systemctl --failed

# Specific service logs
journalctl -u nginx -f
journalctl -u grafana -f
journalctl -u wireguard-wg-peerobserver -f
```

### Monitor Disk Space

```bash
df -h /
# If pruned Bitcoin node fills up, increase prune target in node01 config
```

---

## Key Learnings

1. **Use Direct Device Paths**: `/dev/nvme0n1` works better than `/dev/disk/by-id/...` in kexec environment
2. **SSH Keys for Agenix**: Use `ssh-ed25519` keys, not `age` keys, as recipients
3. **Hardware Config Matters**: Auto-generated configs may have wrong virtualization settings
4. **setup = false vs true**: 
   - `setup = true`: Basic NixOS only (SSH, networking)
   - `setup = false`: Full application services
5. **DNS Before SSL**: Let's Encrypt requires DNS to be configured before issuing certificates

---

## Resources

- [peer-observer-infra-library](https://github.com/0xB10C/peer-observer-infra-library)
- [nixos-anywhere documentation](https://github.com/nix-community/nixos-anywhere)
- [agenix documentation](https://github.com/ryantm/agenix)
- [NixOS manual](https://nixos.org/manual/nixos/stable/)

---

## Deployment Checklist

- [ ] EC2 instance provisioned with Ubuntu
- [ ] Security Group configured (22, 80, 443, 51820)
- [ ] DNS A record configured
- [ ] Disk ID obtained
- [ ] `hosts/web01/disko.nix` created with direct device path
- [ ] `hosts/web01/hardware-configuration.nix` created with proper GRUB config
- [ ] WireGuard keys generated
- [ ] `infra.nix` updated with web01 configuration
- [ ] Server SSH host key obtained
- [ ] `secrets/secrets.nix` updated with SSH keys (not age keys)
- [ ] Secrets encrypted with agenix
- [ ] `/etc/hosts` or SSH config updated
- [ ] Initial NixOS installation completed
- [ ] Full services deployed (`setup = false`)
- [ ] WireGuard tunnel verified
- [ ] Dashboard accessible via HTTPS
- [ ] SSL certificate valid