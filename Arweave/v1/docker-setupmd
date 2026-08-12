# Arweave Archival Validator Node Setup Guide


This document provides comprehensive instructions for deploying an Arweave archival validator node using Docker. The setup includes full blockchain synchronization, validation capabilities, and optional mining functionality.


---


## Table of Contents


1. [System Requirements](#system-requirements)  
2. [Directory Structure](#directory-structure)  
3. [Docker Installation](#docker-installation) - not mandatory
4. [Directory Creation](#directory-creation)  
5. [Install Arweave Binaries](#install-arweave-binaries)  
6. [Node Configuration](#node-configuration)  
7. [Building the Docker Image](#building-the-docker-image)  
8. [Running the Node](#running-the-node)  
9. [Monitoring the Node](#monitoring-the-node)  
10. [Troubleshooting](#troubleshooting)  
11. [Node Management Commands](#node-management-commands)  
12. [Reference](#reference)


---


## System Requirements


### Hardware Specifications


| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 4 cores | 8+ cores |
| RAM | 8 GB | 16+ GB (24 GB recommended) |
| Disk (data_dir) | 200 GB SSD | 500+ GB NVMe SSD |
| Disk (storage) | 3.6 TB per partition | Full weave ~373 TB HDDs |
| Network | 10 Mbps | 100+ Mbps |


### Operating System


- Ubuntu 22.04 LTS (tested)
- Ubuntu 24.04 LTS
- Rocky 9


### Docker Requirements


| Component | Minimum Version |
|-----------|-----------------|
| Docker Engine | 20.10+ |
| Docker Compose | 2.0+ |


---


## Directory Structure


Create the following directory structure before starting:


```
/1Disk73/
├── arweave/                      # Binary installation (from official releases)
│   ├── bin/
│   ├── lib/
│   └── releases/
└── node-arweave/               # Docker volume mounts
    ├── data/                     # Node data (wallet, indices, chain data)
    ├── storage/                  # Storage modules (HDDs for archive)
    │   └── disk{1..16}/          # Individual disks for storage modules
    └── config/
        └── config.json           # Node configuration file
```


---


## Docker Installation ( non mandatory Step )


Install Docker Engine and Docker Compose plugin:


```bash
sudo apt update && sudo apt upgrade -y


# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh


# Add user to docker group
sudo usermod -aG docker $USER


# Install Docker Compose plugin
sudo apt install -y docker-compose-plugin


# Reboot to apply group changes
sudo reboot
```


After reboot, verify Docker is working:


```bash
docker --version
docker compose version
```


**Expected output:**


```
Docker version 24.0.7, build ...
Docker Compose version v2.23.0
```


---


## Directory Creation


Create the necessary directories:


```bash
# Create main directories
sudo mkdir -p /1Disk73/node-arweave/data
sudo mkdir -p /1Disk73/node-arweave/storage/disk{1..16}
sudo mkdir -p /1Disk73/node-arweave/config


# Set permissions
sudo chmod -R 755 /1Disk73/node-arweave
```


**Expected output:**


```
No errors. Directories created successfully.
```


---


## Install Arweave Binaries


Before generating a wallet or building the Docker image, you must download and extract the official Arweave binaries. This guide uses version **2.9.5.1**.


**Note:** Version 2.9.5.1 has a known Prometheus metric bug that may cause crashes in some environments. If you encounter repeated crashes with this version, consider downgrading to 2.9.4.1 (the stable version).


```bash
cd /1Disk73


# Download the release for Ubuntu 22.04 (adjust if using a different OS)
sudo wget https://github.com/ArweaveTeam/arweave/releases/download/N.2.9.5.1/arweave-2.9.5.1.ubuntu22.x86_64.tar.gz


# Extract the archive (this may create a folder or extract files directly)
sudo tar -xzf arweave-2.9.5.1.ubuntu22.x86_64.tar.gz


# Determine if a folder was created and organise the files
if [ -d "arweave-2.9.5.1" ]; then
    # Folder exists – rename it to arweave
    sudo mv arweave-2.9.5.1 arweave
else
    # Files extracted into the current directory – create arweave/ and move them
    sudo mkdir -p arweave
    # Move all extracted files (excluding the archive) into arweave/
    sudo mv bin lib releases erts-* genesis_data arweave/ 2>/dev/null
    sudo mv .erlang arweave/ 2>/dev/null || true
fi


# Remove the archive
sudo rm arweave-2.9.5.1.ubuntu22.x86_64.tar.gz


# Verify installation
cd /1Disk73/arweave
sudo ./bin/arweave version
```


**Expected output:**


```
arweave 2.9.5.1 ()
   erts 14.2.5.11
```


---


## Node Configuration


### Step 1: Generate Wallet and Capture Address


The wallet generation requires `sudo` because the data directory may have restricted permissions. We generate the wallet and immediately capture the address printed to the console.


```bash
cd /1Disk73/arweave


# Generate wallet with sudo and capture the address from the output
sudo ./bin/arweave wallet create rsa /1Disk73/arweave-docker/data/ 2>&1 | sudo tee /tmp/wallet_output.txt


# Extract the address from the output
WALLET_ADDR=$(sudo grep -oP 'Created a wallet with address \K[a-zA-Z0-9_-]+' /tmp/wallet_output.txt)
if [ -z "$WALLET_ADDR" ]; then
  echo "ERROR: Could not extract wallet address."
  exit 1
fi
echo "Wallet address: $WALLET_ADDR"


# Find the generated wallet file (owned by root)
WALLET_FILE=$(sudo find /1Disk73/arweave-docker/data/wallets -name "*.json" -type f | head -1)
if [ -z "$WALLET_FILE" ]; then
  echo "ERROR: Wallet file not found."
  exit 1
fi
echo "Wallet file: $WALLET_FILE"


# Rename the file to wallet.json
sudo mv "$WALLET_FILE" /1Disk73/arweave-docker/data/wallet.json


# Change ownership to your user (use primary group)
sudo chown $USER:$(id -gn) /1Disk73/arweave-docker/data/wallet.json


# Set proper permissions
sudo chmod 644 /1Disk73/arweave-docker/data/wallet.json


# Remove the now-empty wallets directory
sudo rmdir /1Disk73/arweave-docker/data/wallets 2>/dev/null


# Verify
sudo ls -la /1Disk73/arweave-docker/data/wallet.json
```


**Expected output:**


```
Created a wallet with address [wallet_address].
Wallet file: /1Disk73/arweave-docker/data/wallets/arweave_keyfile_[address].json
-rw-r--r-- 1 user group 3163 Aug  7 06:20 /1Disk73/arweave-docker/data/wallet.json
```


### Step 2: Verify the Wallet Address (Optional)


If you want to double‑check, you can print the variable:


```bash
echo "Wallet address: $WALLET_ADDR"
```


**Expected output:**


```
Wallet address: [alphanumeric_address]
```


### Step 3: Create Configuration File


Create the node configuration file with the recommended settings. The `mining_addr` is automatically set to the captured wallet address. Alternatively, you may use a pre‑existing wallet address if you already have one.


```bash
cd /1Disk73/arweave-docker


# Use sudo tee to write the config file
cat << EOF | sudo tee config/config.json
{
    "peers": [
        "asia.peers.arweave.xyz",
        "europe.peers.arweave.xyz",
        "india.peers.arweave.xyz",
        "north-america.peers.arweave.xyz",
        "oceania.peers.arweave.xyz",
        "peers.arweave.xyz"
    ],
    "data_dir": "/data",
    "vdf_server_trusted_peers": ["vdf-server-3.arweave.xyz"],
    "transaction_blacklist_urls": ["https://public_shepherd.arweave.net"],
    "mining_addr": "$WALLET_ADDR",
    "sync_jobs": 20,
    "port": 1984,
    "mine": false
}
EOF


# Verify the file
cat config/config.json
```


**Configuration notes:**
- `sync_jobs`: 20 provides a good balance for initial sync.
- `mine`: false – the node acts as an archival validator, not a miner.
- The `mining_addr` is your wallet address, used to credit any future mining rewards if you later enable mining.
- If you already have a wallet address and prefer not to generate a new one, you can replace `$WALLET_ADDR` with your existing address.


---


## Building the Docker Image


### Step 1: Copy Binaries to Build Context


```bash
cd /1Disk73/arweave-docker
sudo cp -r /1Disk73/arweave/ ./arweave-binaries/
```


### Step 2: Create Dockerfile


The following Dockerfile has been tested and confirmed working. It uses Ubuntu 22.04 and copies the binaries from the build context.


```bash
cd /1Disk73/arweave-docker


cat << 'EOF' | sudo tee Dockerfile
FROM ubuntu:22.04


# Install dependencies
RUN apt-get update && apt-get install -y \
    wget \
    curl \
    screen \
    libsodium-dev \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*


# Set working directory
WORKDIR /opt/arweave


# Copy existing Arweave binaries from build context
COPY --chown=root:root ./arweave-binaries/ /opt/arweave/


# Ensure binaries are executable
RUN chmod +x /opt/arweave/bin/*


# Create data directories
RUN mkdir -p /data /config /storage


# Set environment variables
ENV ARNODE=arweave@127.0.0.1
ENV ARCOOKIE=arweave


# Expose port
EXPOSE 1984


# Default command
ENTRYPOINT ["/opt/arweave/bin/arweave"]
CMD ["foreground", "config_file", "/config/config.json"]
EOF
```


### Step 3: Build the Image


```bash
sudo docker build -t arweave-custom:latest .
```


**Expected output:**


```
[+] Building 94.1s (11/11) FINISHED
=> [internal] load build definition from Dockerfile
=> [internal] load metadata for docker.io/library/ubuntu:22.04
=> [1/6] FROM docker.io/library/ubuntu:22.04
=> [2/6] RUN apt-get update && apt-get install -y ...
=> [3/6] WORKDIR /opt/arweave
=> [4/6] COPY --chown=root:root ./arweave-binaries/ /opt/arweave/
=> [5/6] RUN chmod +x /opt/arweave/bin/*
=> [6/6] RUN mkdir -p /data /config /storage
=> exporting to image
=> => naming to docker.io/library/arweave-custom:latest
```


---


## Running the Node


### Step 1: Configure Huge Pages (Required for RandomX)


```bash
# Check current huge pages
sudo sysctl vm.nr_hugepages


# Set huge pages to 3500
sudo sysctl -w vm.nr_hugepages=3500


# Make permanent
echo "vm.nr_hugepages=3500" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```


**Expected output:**


```
vm.nr_hugepages = 0
vm.nr_hugepages = 3500
vm.nr_hugepages = 3500
```


### Step 2: Start the Docker Container


Use the following command to start the container. The port mapping `-p 1984:1984` can be changed to a different host port if needed (e.g., `-p 8000:1984` for external access via a pre‑opened port).


```bash
sudo docker run -d \
  --name arweave-archive-node \
  --restart unless-stopped \
  -p 8000:1984 \
  --dns 8.8.8.8 \
  --dns 1.1.1.1 \
  --memory=24g \
  --memory-swap=24g \
  --shm-size=4g \
  -v /1Disk73/arweave-docker/data:/data \
  -v /1Disk73/arweave-docker/storage:/storage \
  -v /1Disk73/arweave-docker/config:/config \
  arweave-custom:latest \
  foreground config_file /config/config.json
```


**Expected output:**


```
[container_id_hash]
```


**Note on port mapping:** If the default port 1984 is blocked by a corporate firewall, map to an allowed port (e.g., `-p 8000:1984`). All `curl` examples in this document should then use the new port.


### Step 3: Verify Container is Running


```bash
sudo docker ps
```


**Expected output:**


```
CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS         PORTS                              NAMES
[hash]         arweave-custom:latest    "/opt/arweave/bin/ar…"   5 seconds ago   Up 5 seconds   0.0.0.0:1984->1984/tcp             arweave-archive-node
```


---


## Monitoring the Node


### View Node Logs


```bash
# Follow logs in real-time
sudo docker logs -f arweave-archive-node


# View last 50 lines
sudo docker logs --tail=50 arweave-archive-node


# Search for sync progress
sudo docker logs arweave-archive-node | grep -E "Added block|Synced up to"
```
**Expected output:**


```bash
The account tree has been initialized at the block height 1975020.
Joined the Arweave network successfully at the block WqHdDfEEVH5MOptRRqTSshHwcHdOw3v_lGaAakIppckUrfFXFEzBISiWyISiQ4hC, height 1975020.
# NOTE: This takes 5-10 mins to initialize
```


### Monitor Sync Progress


```bash
# Monitor blocks vs height
watch -n 30 'curl -s http://localhost:1984/info | jq "{height, blocks}"'
```


**Expected output:**


```json
{
  "height": 1972319,
  "blocks": 208
}
```


### Check Node Status


```bash
curl -s http://localhost:1984/info | jq '.'
```


**Expected output (during sync):**


```json
{
  "version": 5,
  "release": 89,
  "queue_length": 0,
  "peers": 5,
  "node_state_latency": 2,
  "network": "arweave.N.1",
  "height": 1972319,
  "current": "block_hash_here",
  "blocks": 208
}
```


### Check Storage Modules


```bash
curl -s http://localhost:1984/storage_modules | jq '.'
```


### Check Connected Peers


```bash
curl -s http://localhost:1984/peers | jq '.[0:10]'
```


### Monitor Resource Usage


```bash
# Container resource usage
sudo docker stats arweave-archive-node


# Disk usage
df -h /1Disk73/arweave-docker/data


# Memory usage
free -h
```


### Query Blocks


```bash
# Get current block
curl -s http://localhost:1984/block/current | jq '.'


# Get block by height
curl -s http://localhost:1984/block/hash/1 | jq '.'


# Get block by height (when synced)
curl -s http://localhost:1984/block/hash/1972319 | jq '.'
```


---


## Troubleshooting


### Issue 1: RandomX Memory Allocation Failure


**Symptom:**
```
error: "randomx_alloc_cache failed"
```


**Cause:** Huge pages not configured or insufficient container memory.


**Resolution:**


```bash
# Configure huge pages
sudo sysctl -w vm.nr_hugepages=3500
echo "vm.nr_hugepages=3500" | sudo tee -a /etc/sysctl.conf


# Increase container memory
sudo docker stop arweave-archive-node
sudo docker rm arweave-archive-node
sudo docker run -d ... --memory=24g ...
```


### Issue 2: Node Cannot Join Network


**Symptom:**
```
Peer [IP]:1984 is not available.
```


**Cause:** DNS resolution failure or firewall blocking outbound port 1984.


**Resolution:**


```bash
# Use public DNS
sudo docker run ... --dns 8.8.8.8 --dns 1.1.1.1 ...


# Add regional peers
sudo cat > config/config.json << 'EOF'
{
    "peers": [
        "asia.peers.arweave.xyz",
        "europe.peers.arweave.xyz",
        "india.peers.arweave.xyz",
        "north-america.peers.arweave.xyz",
        "oceania.peers.arweave.xyz"
    ],
    ...
}
EOF
```


### Issue 3: Configuration Parsing Errors


**Symptom:**
```
Failed to parse config: unknown: {<<"mining">>,false}
```


**Cause:** Invalid configuration key. The correct key is `mine` (boolean), not `mining`.


**Resolution:**


```bash
# Use correct key
sudo cat > config/config.json << 'EOF'
{
    ...
    "mine": false
}
EOF
```


### Issue 4: Node Crash During Sync


**Symptom:**
```
[os_mon] cpu supervisor port (cpu_sup): Erlang has closed
```


**Cause:** Memory exhaustion or version 2.9.5.1 Prometheus metric bug.


**Resolution:**


- Increase memory limits as above.
- Add `"start_from_latest_state": true` to the config to resume from the last persisted state.
- If crashes persist, downgrade to 2.9.4.1 (replace the download URL in the "Install Arweave Binaries" section).


### Issue 5: Block 1 Not Available


**Symptom:**
```
curl -s http://localhost:1984/block/hash/1 | jq '.'
parse error: Invalid numeric literal at line 1, column 6
```


**Cause:** Node has not completed initial sync.


**Resolution:** Monitor `blocks` counter in `/info`:


```bash
curl -s http://localhost:1984/info | jq '{height, blocks}'
```


When `blocks == height`, block 1 will be queryable.


### Issue 6: Slow Sync Performance


**Resolution:** Increase sync jobs:


```bash
sudo cat > config/config.json << 'EOF'
{
    ...
    "sync_jobs": 40
}
EOF
```


### Issue 7: GraphQL Endpoint Returns Empty Results


**Symptom:**


```bash
curl -X POST "http://localhost:1984/graphql" ... # returns null or empty data
```


**Cause:** The node has not yet synced the blocks you are querying.


**Resolution:**


- Wait for the archive sync to complete (`blocks == height`).
- For immediate testing, query only blocks up to the current `blocks` count.
- Verify GraphQL is enabled (it is enabled by default; if not, add `"enable": ["graphql"]` to the config and restart the container).


### Issue 8: External Access Fails with ECONNRESET


**Symptom:** Node responds locally (`curl localhost:1984`) but external requests fail with `ECONNRESET` or `ECONNREFUSED`.


**Cause:** Firewall or network security group blocking inbound traffic.


**Resolution:**


- Use a port that is already allowed in the corporate network (e.g., 8000, 8080).
- Remap the container port: `-p 8000:1984`.
- Ensure UFW/iptables allows the port: `sudo ufw allow 8000/tcp`.
- Use SSH tunnelling as a workaround: `ssh -L 1984:localhost:1984 user@<vm_ip>`.


### Issue 9: Wallet File Not Found


**Symptom:** The wallet generation command does not create a file.


**Cause:** The Arweave binary creates a `wallets` subdirectory with a uniquely named file.


**Resolution:** Use the `find` command to locate the generated file and rename it as shown in [Step 1 of Node Configuration](#step-1-generate-wallet-and-capture-address).


### Issue 10: Permission Denied When Writing Wallet


**Symptom:** `Failed to create a wallet, reason: ["eacces"]`


**Cause:** The target directory is not writable by the user.


**Resolution:** Use `sudo` to generate the wallet as shown in Step 1. Then adjust ownership to your user.


### Issue 11: Repeated Prometheus Metric Errors (2.9.5.1)


**Symptom:** Logs show `badarg` in `ets:lookup` for `prometheus_gauge_table` or similar.


**Cause:** A known bug in version 2.9.5.1.


**Resolution:** Downgrade to version 2.9.4.1 by replacing the download URL in the "Install Arweave Binaries" section with:


```
https://github.com/ArweaveTeam/arweave/releases/download/N.2.9.4.1/arweave-2.9.4.1.ubuntu22.x86_64.tar.gz
```


Then rebuild the image and restart the container. Version 2.9.4.1 is stable and does not suffer from these crashes.


---


## Node Management Commands


### Start/Stop Container


```bash
# Stop
sudo docker stop arweave-archive-node


# Start
sudo docker start arweave-archive-node


# Restart
sudo docker restart arweave-archive-node


# Remove (stops and deletes container)
sudo docker rm arweave-archive-node
```


### Update Configuration


```bash
# Update config file
sudo vi /1Disk73/arweave-docker/config/config.json


# Restart to apply
sudo docker restart arweave-archive-node
```


### Check Node Health


```bash
# Ping the node
sudo docker exec arweave-archive-node /opt/arweave/bin/arweave ping


# Check status
sudo docker exec arweave-archive-node /opt/arweave/bin/arweave status
```


---


## Reference


### Configuration Options


| Option | Description | Default |
|--------|-------------|---------|
| `peers` | List of peer addresses to connect to | `[]` |
| `data_dir` | Directory for chain data and wallet | Required |
| `mining_addr` | Wallet address for mining rewards | Required for mining |
| `sync_jobs` | Number of parallel sync workers | 100 |
| `port` | HTTP API port | 1984 |
| `mine` | Enable/disable mining | false |
| `vdf_server_trusted_peers` | VDF server peers | `[]` |
| `start_from_latest_state` | Resume from last persisted state | false |
| `packing_workers` | Number of packing threads | CPU cores |
| `hashing_threads` | Number of hashing threads | 15 |


### API Endpoints


| Endpoint | Description |
|----------|-------------|
| `/info` | Node status and network info |
| `/blocks` | Recent blocks |
| `/block/current` | Latest block |
| `/block/hash/{height}` | Block by height |
| `/tx/{id}` | Transaction by ID |
| `/tx/{id}/status` | Transaction status |
| `/peers` | Connected peers |
| `/storage_modules` | Storage module status |
| `/graphql` | GraphQL query endpoint |


### Useful Docker Commands


| Command | Purpose |
|---------|---------|
| `sudo docker ps` | List running containers |
| `sudo docker logs -f arweave-archive-node` | Follow logs |
| `sudo docker stats arweave-archive-node` | View resource usage |
| `sudo docker exec -it arweave-archive-node /bin/bash` | Open shell in container |


### Support Resources


- Official Documentation: https://docs.arweave.org
- GitHub: https://github.com/ArweaveTeam/arweave
- Discord: https://discord.gg/GHB4fxVv8B
- Block Explorer: https://viewblock.io/arweave


---


## Conclusion


This guide provides a complete setup for an Arweave archival validator node. The node will synchronize the full blockchain from genesis, validate transactions, and serve as a permanent archive. The sync process can take several days to complete. Once `blocks` reaches `height` in the `/info` endpoint, the node is fully operational.


**Key features of this runbook:**
- Uses version **2.9.5.1** as requested.
- Wallet generation with `sudo` directly into the data directory, with automatic address capture.
- Ownership adjustment to allow user management of the wallet file.
- Recommended configuration with `sync_jobs: 20` and mining disabled.
- Comprehensive troubleshooting covering all common issues, including the known Prometheus bug and downgrade instructions.


Following this guide will yield a production‑ready archival validator node with minimal manual intervention.

