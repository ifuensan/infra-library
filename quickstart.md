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
│   └── grafana-admin-password-web01.age
└── wireguard-keys/                    # Temporary key storage (gitignored)
    ├── web01-private.key
    └── web01-public.key
```

## Setting Up Your Infrastructure

### 1. Initialize Your Configuration

```bash
nix flake init --template github:0xb10c/peer-observer-infra-library
```

### 2. Configure Your Infrastructure

Edit `infra.nix` to define your setup:

- **Global settings**: Admin user, SSH keys, and shared configuration
- **Nodes**: Bitcoin observation nodes with unique IDs and WireGuard configuration
- **Webservers**: Frontend interfaces with domains and Grafana access

Key areas to configure:
- Admin username and SSH public keys
- Node descriptions and architecture (x86_64-linux or aarch64-linux)  
- WireGuard IP addresses and public keys
- Domain names for webservers
- Hardware configuration modules

### 3. Generate and Configure Secrets

Create age encryption keys and configure secrets in `secrets/secrets.nix`:

```bash
# Generate an age key for yourself
age-keygen -o mykey.agekey

# Generate WireGuard keys for each host
wg genkey | tee private.key | wg pubkey > public.key
```

Configure encrypted secrets for:
- WireGuard private keys for each host
- Grafana admin passwords
- Any additional sensitive configuration

### 4. Set Up Hardware Configuration

For each host, you'll need hardware configuration. Use one of these approaches:

#### Option A: Generate hardware config on existing NixOS system
```bash
nixos-generate-config hosts/node01/hardware-configuration.nix
```

#### Option B: Use nixos-anywhere for remote deployment
```bash
nix run github:nix-community/nixos-anywhere -- \
  --generate-hardware-config nixos-generate-config hosts/node01/hardware-configuration.nix \
  --flake .#node01 \
  --target-host <host-ip> \
  --disko-mode disko \
  --build-on remote
```

### 5. Deploy Your Infrastructure

Enter the development shell with required tools:

```bash
nix develop
```

Deploy to your hosts:

```bash
# Deploy a specific host
deploy node01

# Or use nixos-rebuild directly
nixos-rebuild switch \
  --flake .#node01 \
  --target-host node01 \
  --build-host node01 \
  --sudo \
  --show-trace
```

### 6. Test Configuration (Optional)

Build VMs for testing before deployment:

```bash
build-vm node01
```

## Architecture Overview

### Infrastructure Layout

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│     Node 01     │    │     Node 02     │    │   Webserver 01  │
│                 │    │                 │    │                 │
│ • Bitcoin Core  │    │ • Bitcoin Core  │    │ • Nginx         │
│ • Peer Observer │    │ • Peer Observer │    │ • Grafana       │
│ • NATS Server   │◄───┤ • NATS Server   │◄───┤ • Prometheus    │
│ • WireGuard     │    │ • WireGuard     │    │ • Fork Observer │
└─────────────────┘    └─────────────────┘    │ • Frontend      │
                                              │ • WireGuard     │
                                              └─────────────────┘
```

### Communication Flow

1. **Bitcoin nodes** run peer-observer extractors that monitor network activity
2. **NATS messaging** coordinates data collection between components
3. **WireGuard VPN** securely connects all infrastructure components
4. **Webservers** aggregate data from all nodes for visualization and analysis
5. **Prometheus/Grafana** provide metrics collection and dashboards

## Configuration Examples

### Basic Node Configuration

```nix
nodes = {
  node01 = {
    id = 1;
    arch = "x86_64-linux";
    description = "Primary observation node";
    wireguard = {
      ip = "10.21.0.1";
      pubkey = "your-wireguard-public-key";
    };
    bitcoind = {
      net = {
        useTor = true;
        useI2P = true;
        useASMap = true;
      };
    };
    extraModules = [
      ./hosts/node01/hardware-configuration.nix
    ];
  };
};
```

### Custom Bitcoin Configuration

```nix
bitcoind = {
  # Use custom Bitcoin Core build
  package = customBitcoind {
    system = "x86_64-linux";
    overrides = {
      gitURL = "https://github.com/bitcoin/bitcoin.git";
      gitBranch = "master";
      gitCommit = "abc123...";
    };
  };
  
  # Network configuration
  net = {
    useTor = true;
    useI2P = true;
    useASMap = true;
  };
  
  # Ban specific IP ranges
  banlistScript = ''
    bitcoin-cli setban 192.168.1.0/24 add 31536000
  '';
};
```

### Webserver with Custom Domain

```nix
webservers = {
  web01 = {
    id = 1;
    arch = "x86_64-linux";
    domain = "observer.yourdomain.com";
    description = "Public observation frontend";
    
    wireguard = {
      ip = "10.21.1.1";
      pubkey = "your-webserver-wireguard-key";
    };
    
    grafana.admin_user = "admin";
    access_DANGER = "LIMITED_ACCESS"; # or "FULL_ACCESS"
    
    extraConfig = {
      security.acme.acceptTerms = true;
      security.acme.defaults.email = "admin@yourdomain.com";
    };
    
    extraModules = [
      ./hosts/web01/hardware-configuration.nix
    ];
  };
};
```

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
