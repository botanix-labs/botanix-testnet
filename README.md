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