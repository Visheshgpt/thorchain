# Nervos CKB — Mainnet Node Setup Guide
---


 


## Ports


 **8114** -> RPC API
 **8115** -> P2P
 **8114** -> P2P outbound to bootnodes
> **Note:** COINIA Node listens for P2P on `8115` (per official CKB docs).
> The official Nervos bootnodes run their P2P on `8114`.
> So coinia firewall must allow **outbound TCP on both 8114 and 8115**.
> If outbound 8114 is blocked → 0 peers → node won't sync.
---




## Step 1 — Install Docker
---
## Step 2 — Create Working Directory


```
mkdir ckb
```
---
## Step 3 — Create Config Files
### 3a. `.env` — version pin




```


cat > .env <<'EOF'


CKB_VERSION=v0.206.0


EOF


```


> Change `CKB_VERSION` here when upgrading. That's the only file you touch for upgrades.


---


### 3b. `docker-compose.yml`


 


```


cat > docker-compose.yml <<'EOF'


services:


  ckb:


    image: nervos/ckb:${CKB_VERSION:-v0.206.0}


    container_name: ckb-archive


    restart: unless-stopped


    command: run


    ports:


      - "8115:8115"               # P2P — must be open to internet


      - "127.0.0.1:8114:8114"     # RPC — localhost only


    volumes:


      - ./ckb.toml:/var/lib/ckb/ckb.toml:ro


      - ckb-data:/var/lib/ckb


    healthcheck:


      test: ["CMD", "ckb", "--version"]


      interval: 30s


      timeout: 10s


      retries: 3


      start_period: 30s


    logging:


      driver: json-file


      options:


        max-size: "100m"


        max-file: "5"


 


volumes:


  ckb-data:


    name: ckb-data


EOF


```


---


### 3c. `ckb.toml` — node configuration


```


cat > ckb.toml <<'EOF'


data_dir = "data"


 


[chain]


spec = { bundled = "specs/mainnet.toml" }


 


[logger]


filter        = "info"


color         = false


log_to_file   = true


log_to_stdout = true


 


[sentry]


dsn = ""


 


[db]


# RocksDB cache — adjust based on your RAM


# 16 GB RAM → 1073741824  (1 GB)


# 32 GB RAM → 2147483648  (2 GB)


cache_size = 1073741824


 


[network]


listen_addresses               = ["/ip4/0.0.0.0/tcp/8115"]


bootnodes = [


  "/ip4/16.163.82.218/tcp/8114/p2p/QmaZMemLXSsxKUrYNucjEbPxVX3rBKsGhWW2muWtWxUWyh",


  "/ip4/35.79.196.111/tcp/8114/p2p/QmYCRVonLfP18LSoz2WCHaXDorUYxuUMfhtcXK1TuZ1iwF",


  "/ip4/13.234.144.148/tcp/8114/p2p/QmbT7QimcrcD5k2znoJiWpxoESxang6z1Gy9wof1rT1LKR",


  "/ip4/34.64.120.143/tcp/8114/p2p/QmejEJEbDcGGMp4D6WtftMMVLkR1ZuBfMgyLFDMJymkDt6",


  "/ip4/3.218.170.86/tcp/8114/p2p/QmShw2vtVt49wJagc1zGQXGS6LkQTcHxnEV3xs6y8MAmQN",


  "/ip4/35.236.107.161/tcp/8114/p2p/QmSRj57aa9sR2AiTvMyrEea8n1sEM1cDTrfb2VHVJxnGuu",


  "/ip4/23.101.191.12/tcp/8114/p2p/QmexvXVDiRt2FBGptgK4gBJusWyyTEEaHeuCAa35EPNkZS",


  "/ip4/20.151.143.237/tcp/8114/p2p/QmNsGNQjYA6iP472bNnNE2GR31kCYBifhY1XcaUxRjZ1py",


  "/ip4/52.59.155.249/tcp/8114/p2p/QmRHqhSGMGm5FtnkW8D6T83X7YwaiMAZXCXJJaKzQEo3rb",


  "/ip4/3.10.216.39/tcp/8114/p2p/QmagxSv7GNwKXQE7mi1iDjFHghjUpbqjBgqSot7PmMJqHA",


  "/ip4/13.37.172.80/tcp/8114/p2p/QmXJg4iKbQzMpLhX75RyDn89Mv7N2H8vLePBR7kgZf6hYk",


  "/ip4/34.118.49.255/tcp/8114/p2p/QmeCzzVmSAU5LNYAeXhdJj8TCq335aJMqUxcvZXERBWdgS",


  "/ip4/40.115.75.216/tcp/8114/p2p/QmW3P1WYtuz9hitqctKnRZua2deHXhNePNjvtc9Qjnwp4q",


  "/ip4/34.176.239.95/tcp/8114/p2p/QmQoWrmuFauCn3zZ2mYYKAciG9opTbjzC2wVEfWveZNDt8",


  "/ip4/13.245.217.98/tcp/8114/p2p/Qmf4t1SzFhRWuGcFcgs7r4pXvkACsz3FcaBMcmMKQMMpn7",


]


max_peers                      = 125


max_outbound_peers             = 8


ping_interval_secs             = 120


ping_timeout_secs              = 1200


connect_outbound_interval_secs = 15


upnp                           = false


discovery_local_address        = false


bootnode_mode                  = false


support_protocols = ["Ping","Discovery","Identify","Feeler","DisconnectMessage","Sync","Relay","Time","Alert","LightClient","Filter","HolePunching"]


 


[rpc]


listen_address        = "0.0.0.0:8114"


max_request_body_size = 10485760


modules = ["Net","Pool","Chain","Stats","Subscription","Experiment","RichIndexer"]


reject_ill_transactions = true


enable_deprecated_rpc   = false


 


[tx_pool]


max_tx_pool_size     = 180_000_000


min_fee_rate         = 1_000


min_rbf_rate         = 1_500


max_tx_verify_cycles = 70_000_000


max_ancestors_count  = 25


 


[indexer_v2]


index_tx_pool = true


EOF


```


> **`RichIndexer`** is required for querying address balances and transaction history. It indexes all blocks from genesis automatically.


---


## Step 4 — Initialize Chain (First Time Only)
This step generates the `specs/` directory inside the Docker volume. Run it once before starting the node.


```
docker run --rm \
  -v ckb-data:/var/lib/ckb \
  nervos/ckb:v0.206.0 \
  init --chain mainnet --force
```


 
---




## Step 5 — Start the Node


 


```bash
docker compose up -d
```




---


## Step 6 — Check Logs
```
docker compose logs -f ckb
```


---


## Step 7 — Verify Node is Working


 
### Check node is alive


```


curl -s -X POST http://127.0.0.1:8114 \


  -H 'Content-Type: application/json' \


  -d '{"id":1,"jsonrpc":"2.0","method":"local_node_info","params":[]}' | python3 -m json.tool


```


 


### Check current block (tip)


```


curl -s -X POST http://127.0.0.1:8114 \


  -H 'Content-Type: application/json' \


  -d '{"id":1,"jsonrpc":"2.0","method":"get_tip_block_number","params":[]}' | \


  python3 -c "import sys,json; r=json.load(sys.stdin)['result']; print('Tip block:', int(r,16))"


```


### Check sync status


```


curl -s -X POST http://127.0.0.1:8114 \


  -H 'Content-Type: application/json' \


  -d '{"id":1,"jsonrpc":"2.0","method":"sync_state","params":[]}' | python3 -m json.tool


```


### Check peer count


```


curl -s -X POST http://127.0.0.1:8114 \


  -H 'Content-Type: application/json' \


  -d '{"id":1,"jsonrpc":"2.0","method":"get_peers","params":[]}' | \


  python3 -c "import sys,json; p=json.load(sys.stdin)['result']; print('Connected peers:', len(p))"


```







