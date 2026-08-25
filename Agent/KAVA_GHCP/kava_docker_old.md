Kava:
The Kava Network is the first Layer-1 blockchain to combine the speed and scalability of the Cosmos SDK with the developer support of Ethereum. The Kava Network will empower developers to build for Web3 and next-gen blockchain technologies through its unique co-chain architecture. KAVA is the native governance and staking token of the Kava Network, enabling its decentralization and security.
Official documentation: https://docs.kava.io/docs/intro 
Github repo of rosetta-based API: https://github.com/Kava-Labs/rosetta-kava 
Discord channel: https://discord.com/invite/JJYnuCx 
Node Setup:
Kava implementation of the Archival node Set-up
Node set-up
Install go 1.18+
Git clone v0.24.0 https://github.com/Kava-Labs/kava 
Then cd kava -> build image from dockerfile-rocksdb
sudo docker build -t rocksdb:7.10.2 .
Initialize with that image-
sudo docker run -it -p 26657:26657 -p 26656:26656 -p 1317:1317 -v /NewDisk121Machine/node-kava2/kava/.kava:/kava/.kava rocksdb:7.10.2 kavad init nodekava --chain-id kava_2222-10
Then in .kava download Snapshot from ChainLayer QuickSync-
https://dl2.quicksync.io/kava_2222-10-archive.20230826.2041.tar.lz4 
Untar it with lz4
Git clone https://github.com/lz4/lz4 
make
make install
lz4 -dc --no-sparse [SNAPSHOT_FILE] | tar xfC - ${KAVAD_HOME}
Then add genesis.json
wget https://kava-genesis-files.s3.us-east-1.amazonaws.com/kava_2222-10/genesis.json 
a
Add Peers in Config.toml
persistent_peers=2a15d9c39eea97b4cf00480b45d4ea32a2e173d0@94.130.78.22:26656,5a933891627e8bde0c4bd0b43c9f99b706e520a2@141.95.99.214:11656,ab3064c37d406245fa2d6e6109395518e8ac8a4c@95.111.255.148:26656,41d88639239c55fd37279d24df507238e1c417ea@85.237.192.104:26656,b23050c89f8cb2f4099688b2bcd60f49b15f41d1@95.214.53.217:26656,d5db8898d40054c07442f3364b32f7fac2752f5e@188.34.178.92:26656,fc34d9a3aff6026a3dbd531a96a50680df61dd91@50.116.3.21:26656,50e4cad7d5e28f7b6495168f92e12bf810e293fd@142.132.152.187:10856,6885971cdb724fa93034fb9e6a11113a6f555d2a@15.235.53.92:11656,7b5a2b519cb5a7d70f0fc5842829d4cce1262585@65.108.121.153:26656,51cfccb07d5a45efdf98d005159c01f0656751ad@54.165.27.59:26656,f7c894901f450b92614fd051d10854d168ec30b5@65.21.94.20:10856,7393ed21b6dc516fcc0ad33c4fe42bdd295d2795@18.206.217.244:26656,508d7ec33c7f3c9c479ca9b845cadbbefee670f7@162.55.133.237:21656,d68410115d7681196651e7fece9e4cafc0456856@3.0.206.176:26656,4cfdd459466cfd492d66b7a5fe26cde96e35d735@182.48.203.7:26656,63ec88e98fc267fb82fa62a51ca5c0a2c115d749@51.38.53.4:27656,ebc272824924ea1a27ea3183dd0b9ba713494f83@185.16.39.172:26656,4efe3caf3b8c0ca197d40756f3bb1bd6081bf18d@51.210.220.20:36656,c124ce0b508e8b9ed1c5b6957f362225659b5343@136.243.248.185:26656,82588f011491c6100d922d133f52fc23460b9231@95.217.91.233:26656,8b5c4a890c8ae7efbbe3360af71be1c3c3a9e12e@121.78.241.68:46656,ce203135031ab08fc0ddff5bd13806e25f21b91d@3.115.125.121:26656,dcd6026ebe5586ed0e94751090f8290b5997666b@5.189.165.172:26656,bc61c26018f65e54232b7e9e99bf7599dffeb78b@13.56.56.180:26656
seeds = 334e291ac361f9a1cf253d290047700b488b679@52.2.147.96:26656
In config.toml make-
db_backend = "rocksdb"
Then change app.toml flags-
minimum-gas-prices = "0.025ukava;1000000000akava"
Api configuration enable it
sudo docker run -d -it -p 26657:26657 -p 26656:26656 -p 1317:1317 -v /NewDisk121Machine/node-kava2/kava/.kava:/kava/.kava rocksdb:7.10.2 kava start --pruning nothing --home /kava/.kava
But For Temporary Using Official API-
https://api.data.kava.io 
Swagger- https://swagger.kava.io/ 
1.Prerequisites:
To run rosetta-kava, docker is required.
2. Features:
Tracking of all native token balance changes for all transaction types
Stateless, offline transaction construction
Historical balance lookup and reconciliation
3. System Requirements:
rosetta-kava has been tested on an AWS c5.2xlarge instance. We recommend 8 vCPU, 16GB of RAM, and at least 2TB of storage for running a dockerized rosetta-kava node.
4. Install the mainnet:
The following commands will build a docker container named rosetta-kava and configure the container for running on the kava-mainnet network.
1. Command to build the image:
docker build . -f Dockerfile.mainnet -t rosetta-kava-mainnet
2. To bring up and run the command:
sudo docker run -it -d -e "MODE=online" -e "NETWORK=kava-mainnet" -e "PORT=8000" -v "/NewDisk73Machine/node-kava/rosetta-kava/kava-data:/data" -p 8080:8000 -p 26657:26657 rosetta-kava-mainnet
Curl Queries:
For latest blocks:
curl -X 'POST' 'http://10.213.73.73:8080/network/status ' -H 'accept: application/json' -H 'Content-Type: application/json' -d '{"network_identifier": {"blockchain": "Kava","network": "kava-mainnet"},"metadata": {}}'
For block at a given height:_
curl --location --request POST '10.213.73.73:8080/block' --header 'Content-Type: application/json' --data-raw '{"network_identifier": {"blockchain": "Kava", "network": "kava-mainnet"}, "block_identifier": {"index": 233333}}'
Curl Query Using Official API
1.For Latest Blocks:
curl -X 'GET'
'http://10.213.120.121:1317/cosmos/base/tendermint/v1beta1/blocks/latest '
-H 'accept: application/json'
2.For a BlockHeight:
curl -X 'GET'
'http://10.213.120.121:1317/cosmos/base/tendermint/v1beta1/blocks/23333 '
-H 'accept: application/json'
For A balance at BlockHeight:
curl --location --request GET 'http://10.213.120.121:1317/cosmos/bank/v1beta1/balances/kava1wyl4l4zmyas5ymxzyevyfj2j054rn56p637ccn '
--header 'x-cosmos-block-height: 67622'
Transactions
curl --location --request GET 'https://api.data.kava.io/cosmos/tx/v1beta1/txs?events=message.action='\''/cosmos.staking.v1beta1.MsgDelegate'\ ''&events=message.sender='''kava1kax995f2n4y7p6kfx66h9kg70nn79s5we9t630'''&page=1&limit=100'
for getting other Transaction Type Just Replace Message.action={transactionType}
a) '/cosmos.staking.v1beta1.MsgDelegate'
b) '/cosmos.staking.v1beta1.MsgBeginRedelegate'
c) '/cosmos.staking.v1beta1.MsgUndelegate'
d) '/kava.hard.v1beta1.MsgWithdraw'
e) '/kava.hard.v1beta1.MsgRepay'
f) '/kava.hard.v1beta1.MsgBorrow'
for Recieve
curl --location --request GET 'https://api.data.kava.io/cosmos/tx/v1beta1/txs?events=message.action='\''/cosmos.staking.v1beta1.MsgDelegate'\ ''&events=transfer.recipient='''kava1kax995f2n4y7p6kfx66h9kg70nn79s5we9t630'''&page=1&limit=100'x

