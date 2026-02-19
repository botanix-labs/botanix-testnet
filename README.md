# Botanix Testnet RPC Guide

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Lightweight testnet environment for Botanix development and testing using Docker.

## Overview

This repository provides an easy-to-use Docker setup for running Botanix testnet nodes. It supports both standard testnet and Mutinynet configurations.

## Prerequisites

- `make`
- `docker`
- `docker-compose`

### Installing Dependencies

#### Linux

```sh
# Install make
sudo apt-get update
sudo apt-get install build-essential make

# Install Docker
sudo apt-get install ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.10.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Add your user to the docker group (optional)
sudo usermod -aG docker $USER
```

#### macOS

```sh
# Install make
xcode-select --install

# Install Docker
# Download and install Docker Desktop for Mac from https://www.docker.com/products/docker-desktop
```

## Quick Start

```sh
# Clone the repository
git clone https://github.com/botanix-labs/botanix-testnet.git
cd botanix-testnet

# Start Mutinynet
make start-mutinynet

# IMPORTANT: Wait for Mutinynet Bitcoin node to sync completely before proceeding!
# You can check sync status with: docker logs -f botanix-mutiny-bitcoind

# OR start Standard Testnet
make start-testnet

# Check node status
make status-testnet
# OR 
make status-mutinynet
```

## Binary + systemd Setup (Linux)

If you want to run from binaries instead of Docker for the Botanix services, use `systemd` to manage `cometbft` and `reth`.

### 1. Install binaries

Download and install Linux (x86_64/amd64) binaries:

```sh
mkdir -p /tmp/botanix-bin && cd /tmp/botanix-bin

# botanix-reth 1.5.5-rc.6
curl -L "https://storage.googleapis.com/botanix-artifact-registry/botanix-releases/botanix-reth/rc/1.5.5-rc.6/botanix-reth-x86_64-unknown-linux-gnu.tar.gz" -o botanix-reth.tar.gz
tar -xzf botanix-reth.tar.gz
sudo install -m 0755 ./botanix-reth /usr/local/bin/reth

# cometbft 1.0.1
curl -L "https://github.com/cometbft/cometbft/releases/download/v1.0.1/cometbft_1.0.1_linux_amd64.tar.gz" -o cometbft.tar.gz
tar -xzf cometbft.tar.gz
sudo install -m 0755 ./cometbft /usr/local/bin/cometbft
```

### 2. Create directories and copy config

```sh
sudo mkdir -p /opt/botanix/reth/botanix_testnet
sudo mkdir -p /opt/botanix/cometbft/config /opt/botanix/cometbft/data
sudo mkdir -p /etc/botanix

sudo cp config/reth/chain.toml /opt/botanix/reth/botanix_testnet/chain.toml
sudo cp -R config/cometbft/* /opt/botanix/cometbft/config/
```

Create a service user:

```sh
sudo useradd --system --home /opt/botanix --shell /usr/sbin/nologin botanix || true
sudo chown -R botanix:botanix /opt/botanix /etc/botanix
```

### 3. Create environment file

Create `/etc/botanix/reth.env`:

```sh
BITCOIND_USER=foo
BITCOIND_PASS=bar
BITCOIND_HOST=http://127.0.0.1:38332
PERSISTENT_PEERS_LIST=a491a5a7b33fad2d52749c273dbd454d2f85b9b8@barcelona.botanix.io:26656,f390c94c8445b3fdf99da24acfa58a39eabddd1b@chelsea.botanix.io:26656,4d81aaee32ed59fe0e66288786365e02af7c258e@liverpool.botanix.io:26656,e90497f8fb232f192d156f5f85a34b863376eef2@madrid.botanix.io:26656,d8bc85c5ed2b155f08bdbe2d2d61717a5f765fff@arsenal.botanix.io:26656,1482fae7da43fcc82873a71d688c8118eeb29027@manchester.botanix.io:26656
MONIKER=your-moniker
```

If you run Mutinynet bitcoind via Docker, `BITCOIND_HOST=http://127.0.0.1:38332` matches the mapped host port.

### 4. Add `systemd` units

Create `/etc/systemd/system/botanix-cometbft.service`:

```ini
[Unit]
Description=Botanix CometBFT Node
After=network-online.target
Wants=network-online.target

[Service]
User=botanix
Group=botanix
EnvironmentFile=/etc/botanix/reth.env
ExecStart=/usr/local/bin/cometbft node \
  --home=/opt/botanix/cometbft \
  --proxy_app=tcp://127.0.0.1:26658 \
  --moniker=${MONIKER} \
  --key-type=secp256k1 \
  --p2p.persistent_peers=${PERSISTENT_PEERS_LIST} \
  --p2p.laddr=tcp://0.0.0.0:26656 \
  --rpc.laddr=tcp://0.0.0.0:26657
Restart=always
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Create `/etc/systemd/system/botanix-reth.service`:

```ini
[Unit]
Description=Botanix Reth RPC Node
After=network-online.target botanix-cometbft.service
Wants=network-online.target botanix-cometbft.service

[Service]
User=botanix
Group=botanix
EnvironmentFile=/etc/botanix/reth.env
ExecStart=/usr/local/bin/reth node \
  --federation-config-path=/opt/botanix/reth/botanix_testnet/chain.toml \
  --datadir=/opt/botanix/reth/botanix_testnet \
  --chain=botanix-testnet \
  --http \
  --http.addr=0.0.0.0 \
  --http.port=8545 \
  --http.api=admin,debug,eth,net,trace,txpool,web3,rpc \
  --http.corsdomain=* \
  --bitcoind.primary_url=${BITCOIND_HOST} \
  --bitcoind.primary_username=${BITCOIND_USER} \
  --bitcoind.primary_password=${BITCOIND_PASS} \
  --p2p-secret-key=/opt/botanix/reth/botanix_testnet/discovery-secret \
  --port=30303 \
  --btc-network=signet \
  --metrics=0.0.0.0:9001 \
  --ipcdisable \
  --abci-port=26658 \
  --abci-host=127.0.0.1 \
  --cometbft-rpc-port=26657 \
  --cometbft-rpc-host=127.0.0.1 \
  --is-testnet \
  --ws \
  --ws.addr=0.0.0.0 \
  --ws.port=8546 \
  --ws.api=debug,eth,net,trace,txpool,web3,rpc \
  -vvv
Restart=always
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

### 5. Enable and manage services

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now botanix-cometbft botanix-reth

sudo systemctl status botanix-cometbft
sudo systemctl status botanix-reth

journalctl -u botanix-cometbft -f
journalctl -u botanix-reth -f
```

## Commands

| Command | Description |
|---------|-------------|
| `make start-mutinynet` | Start the Mutinynet environment |
| `make stop-mutinynet` | Stop the Mutinynet environment |
| `make status-mutinynet` | Check Mutinynet node status |
| `make start-testnet` | Start the standard testnet environment |
| `make stop-testnet` | Stop the standard testnet environment |
| `make status-testnet` | Check standard testnet node status |

## Using CAAS (Compressed Always Available Snapshot)

CAAS provides compressed snapshots of the Reth and CometBFT databases for quick testnet setup.

### Prerequisites

- `lz4`
- `tar`

### Setup Steps

1. Download the snapshots:

```sh
curl -L https://storage.googleapis.com/botanix-rpc-testnet-snapshot/cometbft/cometbft-snapshot-Feb-16-2026-0125PM-EST.tar.lz4 -o consensus.tar.lz4
curl -L https://storage.googleapis.com/botanix-rpc-testnet-snapshot/reth/reth-snapshot-Feb-16-2026-0125PM-EST.tar.lz4 -o poa.tar.lz4
```

2. Decompress and unpack the files:

```sh
lz4 -dc consensus.tar.lz4 | tar -x --strip-components=3
lz4 -dc poa.tar.lz4       | tar -x --strip-components=3
```

3. Copy files to the appropriate directories:


4. Delete cometbft wal:
```sh
sudo rm -rf ./data/cometbft/cs.wal  # adjust if your paths are different
```

5. Start the testnet with the snapshot data in place:

```sh
make start-testnet
```

## Project Structure

```sh
├── config/                    # Configuration files
│   ├── bitcoind/              # Mutiny signet bitcoin configuration
│   ├── cometbft/              # CometBFT configuration
│   └── reth/                  # Reth configuration
├── data/                      # Data directories for nodes
├── docker-compose.yml         
├── mutiny.docker-compose.yml  
└── Makefile                   
```

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
