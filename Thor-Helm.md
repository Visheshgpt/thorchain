# THORChain Node Setup Guide


 


**Setup Date:** July 16, 2026 


**Environment:** Ubuntu 22.04 LTS on Azure VM 


**Documentation Sources:**


- [THORNode Docs](https://docs.thorchain.org/thornodes/joining)


- [Node Launcher Repository](https://gitlab.com/thorchain/devops/node-launcher)


 


---


 


## Table of Contents


 


1. [Prerequisites & System Requirements](#prerequisites--system-requirements)


2. [Phase 1: System Preparation](#phase-1-system-preparation)


3. [Phase 2: Kubernetes Installation (K3s)](#phase-2-kubernetes-installation-k3s)


4. [Phase 3: Helm & Tools Installation](#phase-3-helm--tools-installation)


5. [Phase 4: THORNode Deployment](#phase-4-thornode-deployment)


6. [Phase 5: Post-Deployment & Monitoring](#phase-5-post-deployment--monitoring)


7. [Troubleshooting](#troubleshooting)


8. [Commands Reference](#commands-reference)


9. [Environment Variables Reference](#environment-variables-reference)


 


---


 


## Prerequisites & System Requirements


 


### Hardware Requirements


 


| Resource | Minimum | Recommended | Notes |


|----------|---------|-------------|-------|


| **CPU** | 4 cores | 8+ cores | THORNode is CPU-intensive during sync |


| **RAM** | 16 GB | 32+ GB | Multiple services run simultaneously |


| **Storage** | 1.5 TB SSD | 4 TB SSD | Blockchain data grows continuously |


| **Network** | 100 Mbps | 1 Gbps | Fast sync requires high bandwidth |


| **OS Disk** | 30 GB | 50+ GB | For system + K3s binaries |


 


### Software Requirements


 


| Tool | Version | Purpose | Mandatory |


|------|---------|---------|-----------|


| **Ubuntu** | 22.04 LTS | Operating System | Yes |


| **kubectl** | Latest | Kubernetes CLI | Yes |


| **Helm** | v3.21+ | Kubernetes package manager | Yes |


| **K3s** | v1.36+ | Lightweight Kubernetes | Yes |


| **Go** | 1.23+ | Required for CRD generation | Yes |


| **jq** | 1.6+ | JSON processor | Yes |


| **make** | 4.3+ | Build automation | Yes |


| **git** | 2.34+ | Source control | Yes |


| **curl/wget** | Latest | HTTP client | Yes |


 


### Network Requirements


 


- Outbound HTTPS (443) to Docker registries and GitHub


- Outbound to THORChain P2P network (port 27146)


- Inbound port 27147 (RPC) if exposing publicly


- Inbound port 27146 (P2P) for peer connections


 


---


 


## Phase 1: System Preparation


 


### Step 1.1: Update System Packages


 


```bash


sudo apt-get update


sudo apt-get install -y curl wget git make jq


```


 


| Detail | Value |


|--------|-------|


| **Tool** | apt-get |


| **Purpose** | Install base system dependencies |


| **Mandatory** | Yes |


| **Time** | 1-2 minutes |


 


### Step 1.2: Install kubectl


 


```bash


curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"


sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl


kubectl version --client


```


 


| Detail | Value |


|--------|-------|


| **Tool** | kubectl |


| **Purpose** | Kubernetes command-line interface |


| **Mandatory** | Yes |


| **Verification** | `kubectl version --client` shows version info |


 


### Step 1.3: Install Go


 


```bash


cd /1Disk73


wget https://go.dev/dl/go1.23.5.linux-amd64.tar.gz


sudo rm -rf /usr/local/go


sudo tar -C /usr/local -xzf go1.23.5.linux-amd64.tar.gz


export PATH=$PATH:/usr/local/go/bin


go version


```


 


| Detail | Value |


|--------|-------|


| **Tool** | Go programming language |


| **Purpose** | Required for building Cosmos Operator CRD tools |


| **Mandatory** | Yes |


| **Verification** | `go version` shows go1.23.5 |


 


### Step 1.4: Clone Node Launcher Repository


 


```bash


cd /1Disk73


git clone https://gitlab.com/thorchain/devops/node-launcher.git


cd node-launcher


```


 


| Detail | Value |


|--------|-------|


| **Tool** | git |


| **Purpose** | Official THORChain deployment automation repository |


| **Mandatory** | Yes |


 


---


 


## Phase 2: Kubernetes Installation (K3s)


 


### Why K3s?


 


K3s is a lightweight, production-ready Kubernetes distribution. THORNode requires Kubernetes for orchestrating its multi-container stack.


 


### Step 2.1: Create Data Directories


 


```bash


sudo mkdir -p /1Disk73/k3s


sudo chmod 755 /1Disk73/k3s


sudo mkdir -p /1Disk73/tmp


sudo chmod 777 /1Disk73/tmp


```


 


| Detail | Value |


|--------|-------|


| **Purpose** | Store K3s data on large data disk instead of OS disk |


| **Mandatory** | Yes (if OS disk is small) |


 


### Step 2.2: Install K3s with Custom Data Path


 


```bash


export TMPDIR=/1Disk73/tmp


curl -sfL https://get.k3s.io | sudo INSTALL_K3S_EXEC="--data-dir=/1Disk73/k3s" sh -


```


 


| Detail | Value |


|--------|-------|


| **Purpose** | Installs lightweight Kubernetes with data on large disk |


| **Key Flag** | `--data-dir=/1Disk73/k3s` redirects all K3s data to large disk |


| **Time** | 30-60 seconds |


 


### Step 2.3: Wait for K3s to Start


 


```bash


echo "Waiting 30 seconds for K3s to start..."


sleep 30


sudo systemctl status k3s --no-pager


```


 


**Expected Output:**


```


● k3s.service - Lightweight Kubernetes


 	Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor preset: enabled)


 	Active: active (running) since ...


```


 


### Step 2.4: Configure kubectl Access


 


```bash


mkdir -p ~/.kube 2>/dev/null || sudo mkdir -p ~/.kube


sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config


sudo chown $USER:$(id -gn) ~/.kube/config


chmod 600 ~/.kube/config


export KUBECONFIG=~/.kube/config


```


 


| Detail | Value |


|--------|-------|


| **Purpose** | Allows current user to interact with K3s cluster |


| **Security** | Config file permissions set to 600 |


 


### Step 2.5: Verify Kubernetes Cluster


 


```bash


kubectl get nodes


kubectl cluster-info


```


 


**Expected Output:**


```


NAME          	STATUS   ROLES       	AGE   VERSION


usazuadvdl00514   Ready	control-plane   44s   v1.36.2+k3s1


```


 


---


 


## Phase 3: Helm & Tools Installation


 


### Step 3.1: Install Helm Binary


 


```bash


cd /1Disk73


wget https://get.helm.sh/helm-v3.21.2-linux-amd64.tar.gz


tar -zxvf helm-v3.21.2-linux-amd64.tar.gz


sudo mv linux-amd64/helm /usr/local/bin/helm


sudo chmod +x /usr/local/bin/helm


```


 


| Detail | Value |


|--------|-------|


| **Tool** | Helm |


| **Purpose** | Kubernetes package manager for deploying THORNode charts |


| **Version** | v3.21.2 |


 


### Step 3.2: Configure Helm Data Directories


 


```bash


sudo mkdir -p /1Disk73/helm-config /1Disk73/helm-cache /1Disk73/helm-data


sudo chown -R $USER:$(id -gn) /1Disk73/helm-config /1Disk73/helm-cache /1Disk73/helm-data


 


export HELM_CONFIG_HOME=/1Disk73/helm-config


export HELM_CACHE_HOME=/1Disk73/helm-cache


export HELM_DATA_HOME=/1Disk73/helm-data


```


 


| Detail | Value |


|--------|-------|


| **Purpose** | Redirect Helm config to writable location |


| **Why** | Domain-joined systems often have restricted home directories |


 


### Step 3.3: Verify Helm Installation


 


```bash


which helm


helm version


```


 


**Expected Output:**


```


/usr/local/bin/helm


version.BuildInfo{Version:"v3.21.2", ...}


```


 


### Step 3.4: Install Helm Diff Plugin


 


```bash


helm plugin install https://github.com/databus23/helm-diff


helm diff version


```


 


| Detail | Value |


|--------|-------|


| **Purpose** | Shows differences before applying Helm upgrades |


| **Mandatory** | Yes (node-launcher Makefile uses it) |


 


### Step 3.5: Configure Go Environment


 


```bash


export GOPATH=/1Disk73/go


export GOCACHE=/1Disk73/go-cache


mkdir -p $GOPATH $GOCACHE


```


 


### Step 3.6: Install Cosmos Operator CRDs Manually


 


The node-launcher uses a custom approach that bypasses the cosmos-operator's problematic init containers. Instead of using `make tools`, we manually install the CRDs:


 


```bash


cd /1Disk73/node-launcher


rm -rf co


git clone https://github.com/strangelove-ventures/cosmos-operator.git co


cd co


git checkout 5f01b1dec1d2f9b7976e0fae05feea02ab05ec22


kubectl apply -f config/crd/bases/


```


 


**Expected Output:**


```


customresourcedefinition.apiextensions.k8s.io/cosmosfullnodes.cosmos.strange.love created


customresourcedefinition.apiextensions.k8s.io/scheduledvolumesnapshots.cosmos.strange.love created


```


 


### Step 3.7: Verify CRDs Installed


 


```bash


kubectl get crd | grep cosmos


```


 


**Expected Output:**


```


cosmosfullnodes.cosmos.strange.love      	2026-07-08T07:45:39Z


scheduledvolumesnapshots.cosmos.strange.love 2026-07-08T07:45:39Z


```


 


---


 


## Phase 4: THORNode Deployment (Standalone Mode)


 


**Important:** We deploy THORNode in standalone mode (without the cosmos-operator) to avoid issues with init containers and genesis file handling.


 


### Step 4.1: Set Environment Variables


 


```bash


export KUBECONFIG=~/.kube/config


export HELM_CONFIG_HOME=/1Disk73/helm-config


export HELM_CACHE_HOME=/1Disk73/helm-cache


export HELM_DATA_HOME=/1Disk73/helm-data


export GOPATH=/1Disk73/go


export GOCACHE=/1Disk73/go-cache


export PATH=$PATH:/usr/local/go/bin


```


 


### Step 4.2: Create External IP Configmap


 


```bash


VM_IP=$(hostname -I | awk '{print $1}')


kubectl create configmap thornode-external-ip -n thornode --from-literal=externalIP=$VM_IP


```


 


| Detail | Value |


|--------|-------|


| **Purpose** | Provides external IP for the node |


| **Mandatory** | Yes (on VMs without cloud LoadBalancer) |


 


### Step 4.3: Create PVC for Chain Data


 


```bash


cat <<EOF | kubectl apply -f -


apiVersion: v1


kind: PersistentVolumeClaim


metadata:


  name: thornode


  namespace: thornode


spec:


  accessModes:


	- "ReadWriteOnce"


  resources:


	requests:


  	storage: "100Gi"


  storageClassName: "local-path"


EOF


 


# Wait for PVC to bind


kubectl get pvc -n thornode -w


```


 


**Expected Output:**


```


NAME   	STATUS   VOLUME                                 	CAPACITY   ACCESS MODES   STORAGECLASS   AGE


thornode   Bound	pvc-xxx-xxx-xxx                        	100Gi  	RWO        	local-path 	10s


```


 


### Step 4.4: Create and Fix Genesis File


 


```bash


cat <<EOF | kubectl apply -f -


apiVersion: v1


kind: Pod


metadata:


  name: genesis-setup


  namespace: thornode


spec:


  restartPolicy: Never


  volumes:


	- name: chain-home


  	persistentVolumeClaim:


    	claimName: thornode


  containers:


	- name: setup


  	image: alpine


  	command:


    	- sh


    	- -c


    	- |


      	apk add curl jq


      	mkdir -p /home/operator/cosmos/.thornode/config


      	echo "Downloading genesis..."


      	curl -s https://gateway.liquify.com/chain/thorchain_v1_rpc/genesis | jq -r .result.genesis > /home/operator/cosmos/.thornode/config/genesis.json


      	echo "Fixing YggdrasilVault..."


      	sed -i 's/YggdrasilVault/AsgardVault/g' /home/operator/cosmos/.thornode/config/genesis.json


      	sed -i '/STOPFUNDYGGDRASIL/d' /home/operator/cosmos/.thornode/config/genesis.json


      	echo "Genesis created:"


      	ls -lh /home/operator/cosmos/.thornode/config/genesis.json


  	volumeMounts:


    	- name: chain-home


      	mountPath: /home/operator/cosmos


EOF


 


# Wait for completion


kubectl wait --for=condition=complete pod/genesis-setup -n thornode --timeout=120s


kubectl logs -n thornode genesis-setup


```


 


**Expected Output:**


```


Downloading genesis...


Fixing YggdrasilVault...


Genesis created:


-rw-r--r-- 1 root root 23.4M /home/operator/cosmos/.thornode/config/genesis.json


```


 


### Step 4.5: Deploy THORNode Pod


 


```bash


cat <<EOF | kubectl apply -f -


apiVersion: v1


kind: Pod


metadata:


  name: thornode-0


  namespace: thornode


  labels:


	app-name: thornode


spec:


  volumes:


	- name: chain-home


  	persistentVolumeClaim:


    	claimName: thornode


  containers:


	- name: node


  	image: registry.gitlab.com/thorchain/thornode:mainnet-3.6.1@sha256:a40c63b1d2c3523aeb9bc2a42f92094ffddce1ffd00d5e888863302448f8cace


  	command:


    	- thornode


    	- start


    	- --home


    	- /home/operator/cosmos/.thornode


  	volumeMounts:


    	- name: chain-home


      	mountPath: /home/operator/cosmos


  	env:


    	- name: HOME


      	value: /home/operator/cosmos


    	- name: CHAIN_ID


      	value: thorchain-1


    	- name: EXTERNAL_IP


      	valueFrom:


        	configMapKeyRef:


          	name: thornode-external-ip


          	key: externalIP


  	ports:


    	- name: rpc


      	containerPort: 27147


    	- name: p2p


      	containerPort: 27146


    	- name: api


      	containerPort: 1317


EOF


```


 


### Step 4.6: Verify Pod Status


 


```bash


kubectl get pods -n thornode


```


 


**Expected Output:**


```


NAME        	READY   STATUS  	RESTARTS   AGE


genesis-setup   0/1 	Completed   0      	2m


thornode-0  	1/1 	Running 	0      	30s


```


 


### Step 4.7: Add Seeds for Peer Discovery


 


```bash


cat <<EOF | kubectl apply -f -


apiVersion: v1


kind: Pod


metadata:


  name: add-seeds


  namespace: thornode


spec:


  restartPolicy: Never


  volumes:


	- name: chain-home


  	persistentVolumeClaim:


    	claimName: thornode


  containers:


	- name: fix


  	image: alpine


  	command:


    	- sh


    	- -c


    	- |


      	cd /home/operator/cosmos/.thornode/config


      	sed -i 's/^seeds = .*/seeds = \"15f9d663d23709473333f4d14d2a0a8a8b0cf1c9@54.37.217.220:26656,4c3c9ca8b04f675c7977c56f80ed8a5d97a56346@54.37.218.253:26656\"/' config.toml


      	echo "Seeds added successfully"


  	volumeMounts:


    	- name: chain-home


      	mountPath: /home/operator/cosmos


EOF


 


kubectl wait --for=condition=complete pod/add-seeds -n thornode --timeout=60s


kubectl logs -n thornode add-seeds


kubectl delete pod add-seeds -n thornode


 


# Restart the node to apply changes


kubectl delete pod thornode-0 -n thornode


sleep 10


kubectl get pods -n thornode -w


```


 


---


 


## Phase 5: Post-Deployment & Monitoring


 


### Step 5.1: Check Sync Status


 


```bash


# Check if RPC is responding (try both ports)


kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/status | jq '.result.sync_info' 2>/dev/null || echo "Using port 26657"


 


# Or use the THORChain RPC port


kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:27147/status | jq '.result.sync_info' 2>/dev/null || echo "Using port 27147"


```


 


**Expected Output:**


```json


{


  "latest_block_height": "0",


  "latest_block_time": "1970-01-01T00:00:00Z",


  "catching_up": true


}


```


 


### Step 5.2: Monitor Sync Progress


 


```bash


# Monitor block height every 10 seconds


watch -n 10 'kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/status 2>/dev/null | jq ".result.sync_info | {height: .latest_block_height, catching_up: .catching_up}"'


```


 


### Step 5.3: Check Peers


 


```bash


# Check number of connected peers


kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/net_info | jq '.result.peers | length'


```


 


**Expected Output:** Should show > 0 once connected to peers.


 


### Step 5.4: Create Monitoring Script


 


```bash


cat > /1Disk73/monitor-thornode.sh << 'EOF'


#!/bin/bash


export KUBECONFIG=~/.kube/config


 


echo "========================================"


echo " 	THORNODE MONITORING"


echo "========================================"


echo ""


 


echo "📊 POD STATUS:"


kubectl get pods -n thornode


echo ""


 


echo "🔄 SYNC STATUS:"


kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/status 2>/dev/null | jq '.result.sync_info | {height: .latest_block_height, catching_up: .catching_up}' || echo "Node not ready"


echo ""


 


echo "👥 PEERS:"


kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/net_info 2>/dev/null | jq '.result.peers | length' || echo "0"


echo ""


 


echo "📝 RECENT LOGS (errors only):"


kubectl logs -n thornode thornode-0 -c node --tail=20 2>/dev/null | grep -E "(error|fatal|panic)" || echo "No errors found"


echo "========================================"


EOF


 


chmod +x /1Disk73/monitor-thornode.sh


```


 


### Step 5.5: Log Monitoring Commands


 


```bash


# Follow node logs


kubectl logs -n thornode thornode-0 -c node -f


 


# Check last 50 lines


kubectl logs -n thornode thornode-0 -c node --tail=50


 


# Check for errors


kubectl logs -n thornode thornode-0 -c node --tail=100 | grep -E "error|fatal|panic"


```


 


---


 


## Troubleshooting


 


### Problem 1: Genesis File Has YggdrasilVault


 


**Symptom:**


```


panic: invalid vault type value: YggdrasilVault


```


 


**Root Cause:** The genesis file from the API contains deprecated YggdrasilVault type.


 


**Fix:** Replace YggdrasilVault with AsgardVault:


 


```bash


# Create a fix pod


cat <<EOF | kubectl apply -f -


apiVersion: v1


kind: Pod


metadata:


  name: fix-genesis


  namespace: thornode


spec:


  restartPolicy: Never


  volumes:


	- name: chain-home


  	persistentVolumeClaim:


    	claimName: thornode


  containers:


	- name: fix


  	image: alpine


  	command:


    	- sh


    	- -c


    	- |


      	apk add curl jq


      	cd /home/operator/cosmos/.thornode/config


      	curl -s https://gateway.liquify.com/chain/thorchain_v1_rpc/genesis | jq -r .result.genesis > genesis.json


      	sed -i 's/YggdrasilVault/AsgardVault/g' genesis.json


      	sed -i '/STOPFUNDYGGDRASIL/d' genesis.json


      	echo "Genesis fixed"


  	volumeMounts:


    	- name: chain-home


      	mountPath: /home/operator/cosmos


EOF


kubectl wait --for=condition=complete pod/fix-genesis -n thornode --timeout=120s


kubectl delete pod fix-genesis -n thornode


kubectl delete pod thornode-0 -n thornode


```


 


### Problem 2: Pod Stuck in Pending or Init


 


**Symptom:**


```


thornode-0   0/1   Pending   0   5m


```


 


**Root Cause:** PVC not bound or resource issues.


 


**Fix:**


```bash


# Check PVC status


kubectl get pvc -n thornode


 


# Check pod events


kubectl describe pod thornode-0 -n thornode | grep -A 10 "Events:"


 


# Check node resources


kubectl describe node | grep -A 5 "Allocated resources"


```


 


### Problem 3: No Peers Connected


 


**Symptom:**


```


kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/net_info | jq '.result.peers | length'


0


```


 


**Root Cause:** Seed nodes unreachable or DNS resolution issues.


 


**Fix 1: Add seeds with IP addresses directly:**


```bash


cat <<EOF | kubectl apply -f -


apiVersion: v1


kind: Pod


metadata:


  name: add-seeds-ip


  namespace: thornode


spec:


  restartPolicy: Never


  volumes:


	- name: chain-home


  	persistentVolumeClaim:


    	claimName: thornode


  containers:


	- name: fix


  	image: alpine


  	command:


    	- sh


    	- -c


    	- |


      	cd /home/operator/cosmos/.thornode/config


      	# Use IP addresses directly (from dig seed.thorchain.info)


      	sed -i 's/^seeds = .*/seeds = \"15f9d663d23709473333f4d14d2a0a8a8b0cf1c9@54.37.217.220:26656,4c3c9ca8b04f675c7977c56f80ed8a5d97a56346@54.37.218.253:26656\"/' config.toml


      	echo "Seeds with IPs added"


  	volumeMounts:


    	- name: chain-home


      	mountPath: /home/operator/cosmos


EOF


kubectl wait --for=condition=complete pod/add-seeds-ip -n thornode --timeout=60s


kubectl delete pod add-seeds-ip -n thornode


kubectl delete pod thornode-0 -n thornode


```


 


**Fix 2: Check DNS resolution from the pod:**


```bash


kubectl exec -n thornode thornode-0 -c node -- nslookup seed.thorchain.info 2>/dev/null || echo "DNS failed"


kubectl exec -n thornode thornode-0 -c node -- ping -c 3 8.8.8.8 2>/dev/null || echo "Network unreachable"


```


 


### Problem 4: RPC Not Responding


 


**Symptom:**


```


command terminated with exit code 7


```


 


**Root Cause:** RPC port not listening or wrong port.


 


**Fix:**


```bash


# Check which port the node is using for RPC


kubectl exec -n thornode thornode-0 -c node -- cat /home/operator/cosmos/.thornode/config/config.toml | grep "laddr" | grep -v "p2p"


 


# Try both ports


kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/status


kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:27147/status


```


 


### Problem 5: State Sync Errors


 


**Symptom:**


```


failed to start state sync: failed to set up light client state provider: invalid TrustOptions


```


 


**Root Cause:** State sync is enabled but trust height/hash not properly configured.


 


**Fix: Disable state sync:**


```bash


cat <<EOF | kubectl apply -f -


apiVersion: v1


kind: Pod


metadata:


  name: disable-statesync


  namespace: thornode


spec:


  restartPolicy: Never


  volumes:


	- name: chain-home


  	persistentVolumeClaim:


    	claimName: thornode


  containers:


	- name: fix


  	image: alpine


  	command:


    	- sh


    	- -c


    	- |


      	cd /home/operator/cosmos/.thornode/config


      	sed -i 's/^enable = .*/enable = false/' config.toml


      	echo "State sync disabled"


  	volumeMounts:


    	- name: chain-home


      	mountPath: /home/operator/cosmos


EOF


kubectl wait --for=condition=complete pod/disable-statesync -n thornode --timeout=60s


kubectl delete pod disable-statesync -n thornode


kubectl delete pod thornode-0 -n thornode


```


 


---


 


## Commands Reference


 


### Deployment Commands


 


| Command | Purpose |


|---------|---------|


| `kubectl apply -f <file>` | Deploy resources from YAML |


| `kubectl delete pod thornode-0 -n thornode` | Restart THORNode pod |


| `kubectl get pods -n thornode -w` | Watch pod status changes |


| `kubectl logs -n thornode thornode-0 -c node -f` | Follow node logs |


 


### Monitoring Commands


 


| Command | Purpose |


|---------|---------|


| `kubectl get pods -n thornode` | Check all pod statuses |


| `kubectl describe pod thornode-0 -n thornode` | Detailed pod info + events |


| `kubectl get events -n thornode --sort-by='.lastTimestamp'` | Recent events |


| `kubectl get pvc -n thornode` | Check storage volumes |


| `kubectl get svc -n thornode` | Check services |


 


### Block Height & Sync Commands


 


| Command | Purpose |


|---------|---------|


| `kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/status \| jq '.result.sync_info'` | Full sync info |


| `kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/status \| jq -r '.result.sync_info.latest_block_height'` | Just block height |


| `watch -n 10 'kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/status \| jq ".result.sync_info.latest_block_height"'` | Continuous monitoring |


 


### Network Commands


 


| Command | Purpose |


|---------|---------|


| `kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/net_info \| jq '.result.peers | length'` | Check number of peers |


| `kubectl exec -n thornode thornode-0 -c node -- nslookup seed.thorchain.info` | Test DNS resolution |


| `kubectl exec -n thornode thornode-0 -c node -- ping -c 3 8.8.8.8` | Test network connectivity |


 


### Log Commands


 


| Command | Purpose |


|---------|---------|


| `kubectl logs -n thornode thornode-0 -c node --tail=100` | Last 100 lines of node logs |


| `kubectl logs -n thornode thornode-0 -c node -f` | Follow node logs |


| `kubectl logs -n thornode thornode-0 -c node --tail=100 \| grep -E "error\|fatal\|panic"` | Check for errors |


| `kubectl logs -n thornode thornode-0 -c node --previous --tail=100` | Previous container logs |


 


### Debug Commands


 


| Command | Purpose |


|---------|---------|


| `kubectl describe pod thornode-0 -n thornode` | Detailed pod information |


| `kubectl describe pvc thornode -n thornode` | PVC details |


| `kubectl exec -n thornode thornode-0 -c node -- cat /home/operator/cosmos/.thornode/config/config.toml` | View config |


| `kubectl exec -n thornode thornode-0 -c node -- ls -la /home/operator/cosmos/.thornode/config/` | Check config directory |


 


---


 


## Environment Variables Reference


 


### Required Environment Variables


 


Set these in **EVERY new terminal session**:


 


```bash


export KUBECONFIG=~/.kube/config


export HELM_CONFIG_HOME=/1Disk73/helm-config


export HELM_CACHE_HOME=/1Disk73/helm-cache


export HELM_DATA_HOME=/1Disk73/helm-data


export GOPATH=/1Disk73/go


export GOCACHE=/1Disk73/go-cache


export PATH=$PATH:/usr/local/go/bin


```


 


### Persist Environment Variables


 


```bash


cat > /1Disk73/thornode-env.sh << 'EOF'


#!/bin/bash


export KUBECONFIG=~/.kube/config


export HELM_CONFIG_HOME=/1Disk73/helm-config


export HELM_CACHE_HOME=/1Disk73/helm-cache


export HELM_DATA_HOME=/1Disk73/helm-data


export GOPATH=/1Disk73/go


export GOCACHE=/1Disk73/go-cache


export PATH=$PATH:/usr/local/go/bin


echo "THORNode environment loaded."


EOF


 


chmod +x /1Disk73/thornode-env.sh


```


 


**Usage:**


```bash


source /1Disk73/thornode-env.sh


```


 


---


 


## Sync Timeline


 


| Phase | Time | Block Range | Notes |


|-------|------|-------------|-------|


| **Init containers** | 0-5 min | N/A | Pod initialization |


| **Starting sync** | 5-10 min | 0 - 100 | Node starts syncing |


| **Early blocks** | 10 min - 6 hrs | 100 - 5,000,000 | Faster (smaller blocks) |


| **Middle blocks** | 6-18 hrs | 5M - 15,000,000 | Medium speed |


| **Recent blocks** | 18-36 hrs | 15M - 26,000,000+ | Slower (larger blocks) |


| **Fully synced** | 24-48 hrs total | Current height | `catching_up: false` |


 


**Expected Block Heights:**


- Current mainnet height: ~26,000,000+


- Initial height from genesis: 4,786,560


- Time to sync: 24-48 hours


 


---


 


## File System Layout


 


```


/1Disk73/


├── k3s/                	# K3s data (container images, volumes)


├── node-launcher/      	# THORChain deployment repository


│   ├── Makefile        	# Main automation


│   ├── thornode-stack/ 	# THORNode Helm charts


│   └── co/             	# Cosmos Operator (cloned)


├── helm-config/        	# Helm configuration


├── helm-cache/         	# Helm chart cache


├── helm-data/          	# Helm plugins and data


├── go/                 	# Go modules


├── go-cache/           	# Go build cache


├── tmp/                	# Temporary files


├── thornode-env.sh     	# Environment variables script


└── monitor-thornode.sh 	# Monitoring script


 


K3s Storage:


/1Disk73/k3s/storage/


├── pvc-xxx_thornode_thornode/       	# THORNode chain data


├── pvc-xxx_thornode_zcash-daemon/   	# Zcash daemon data


└── pvc-xxx_thornode_data-midgard-*/ 	# Midgard data


```


 


---


 


## Quick Start (TL;DR)


 


```bash


# 1. Source environment


source /1Disk73/thornode-env.sh


 


# 2. Check node status


kubectl get pods -n thornode


 


# 3. Monitor sync


watch -n 10 'kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/status 2>/dev/null | jq ".result.sync_info | {height: .latest_block_height, catching_up: .catching_up}"'


 


# 4. Check peers


kubectl exec -n thornode thornode-0 -c node -- curl -s localhost:26657/net_info | jq '.result.peers | length'


 


# 5. Follow logs


kubectl logs -n thornode thornode-0 -c node -f


 


# 6. Restart node if needed


kubectl delete pod thornode-0 -n thornode


```


 


---


 


**Document Version:** 2.0 


**Last Updated:** July 16, 2026 


**Status:** Deployed and Syncing 


**Author:** DevOps Tea

