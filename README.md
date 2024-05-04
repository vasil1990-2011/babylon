#!/bin/bash

while true
do

# Logo

echo -e '\e[40m\e[91m'
echo -e '  ____                  _                    '
echo -e ' / ___|_ __ _   _ _ __ | |_ ___  _ __        '
echo -e '| |   |  __| | | |  _ \| __/ _ \|  _ \       '
echo -e '| |___| |  | |_| | |_) | || (_) | | | |      '
echo -e ' \____|_|   \__  |  __/ \__\___/|_| |_|      '
echo -e '            |___/|_|                         '
echo -e '    _                 _                      '
echo -e '   / \   ___ __ _  __| | ___ _ __ ___  _   _ '
echo -e '  / _ \ / __/ _  |/ _  |/ _ \  _   _ \| | | |'
echo -e ' / ___ \ (_| (_| | (_| |  __/ | | | | | |_| |'
echo -e '/_/   \_\___\__ _|\__ _|\___|_| |_| |_|\__  |'
echo -e '                                       |___/ '
echo -e '\e[0m'

sleep 2

# Menu

PS3='Select an action: '
options=(
"Install"
"Create Wallet"
"Create Validator"
"Exit")
select opt in "${options[@]}"
do
case $opt in

"Install")
echo "============================================================"
echo "Install start"
echo "============================================================"

# set vars
if [ ! $NODENAME ]; then
	read -p "Enter node name: " NODENAME
	echo 'export NODENAME='$NODENAME >> $HOME/.bash_profile
fi
if [ ! $WALLET ]; then
	echo "export WALLET=wallet" >> $HOME/.bash_profile
fi
echo "export BABYLON_CHAIN_ID=bbn-test-3" >> $HOME/.bash_profile
source $HOME/.bash_profile

# update
sudo apt update && sudo apt upgrade -y

# packages
apt install curl iptables build-essential git wget jq make gcc nano tmux htop nvme-cli pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev -y

# install go
if ! [ -x "$(command -v go)" ]; then
ver="1.21.3" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile
fi

# download binary
cd $HOME && rm -rf babylon
git clone https://github.com/babylonchain/babylon
cd babylon
git checkout v0.8.3
make install

# config
babylond config chain-id $BABYLON_CHAIN_ID
babylond config keyring-backend test

# init
babylond init $NODENAME --chain-id $BABYLON_CHAIN_ID

# download genesis and addrbook
curl -L https://snapshots-testnet.nodejumper.io/babylon-testnet/genesis.json > $HOME/.babylond/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/babylon-testnet/addrbook.json > $HOME/.babylond/config/addrbook.json

# set minimum gas price
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.00001ubbn\"/" $HOME/.babylond/config/app.toml

# set peers and seeds
SEEDS="49b4685f16670e784a0fe78f37cd37d56c7aff0e@3.14.89.82:26656,9cb1974618ddd541c9a4f4562b842b96ffaf1446@3.16.63.237:26656"
PEERS="0c9f976c92bcffeab19944b83b056d06ea44e124@5.78.110.19:26656,2af11b08ea816acb95d4de33c3de07a32c1cc801@82.208.20.66:26656,c3e82156a0e2f3d5373d5c35f7879678f29eaaad@144.76.28.163:46656,ddd6f401792e0e35f5a04789d4db7dc386efc499@135.181.182.162:26656,bee2d7c8ecf2770eb0da3331be0ebe20b61805d1@161.97.172.173:26656,69c1b7e1eb114703733c3000baa6c008ebc70073@65.109.113.233:20656,5b124ed79f5f0c02ffca4bfb8a73469265f46de1@3.132.112.231:26656,10b483d706782dd53834eca77562e081e52b16dd@18.116.214.166:26656,b1783b0d95ffeeac6c81be47ff8552bbc27bc054@18.191.27.217:26656,8f618f4f40d1c27e27b760ca10246b8b113e94be@3.13.201.13:26656,d2003e6f9f76887738e2b15ae26a4351950fff82@173.249.60.158:26656,94a6b8d058bc3db464ab8ec0b824cd40c09a2385@3.131.193.119:26656,5463943178cdb57a02d6d20964e4061dfcf0afb4@142.132.154.53:20656,b6cacf998c9dd2254c8afffe63646542539c2625@135.181.17.178:27656,90eac330252ff51bf461602e7b8df054ce8583ae@65.109.64.57:26656,179a498904d880587cc37d07ebd1e01ff81a02fe@3.139.215.161:26656,c0ee3e7f140b2de189ce853cfccb9fb2d922eb66@95.217.203.226:26656,26cb133489436035829b6920e89105046eccc841@178.63.95.125:26656,f7acfdd83e5f3a66bf216546f662ccad8c24c3ce@173.212.207.184:26656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.babylond/config/config.toml

# disable indexing
indexer="null"
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.babylond/config/config.toml

# config pruning
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="10"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.babylond/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.babylond/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.babylond/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.babylond/config/app.toml
sed -i "s/snapshot-interval *=.*/snapshot-interval = 0/g" $HOME/.babylond/config/app.toml

# enable prometheus
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.babylond/config/config.toml

# create service
sudo tee /etc/systemd/system/babylond.service > /dev/null << EOF
[Unit]
Description=Babylon node service
After=network-online.target
[Service]
User=$USER
ExecStart=$(which babylond) start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

babylond tendermint unsafe-reset-all --home $HOME/.babylond --keep-addr-book
curl https://snapshots-testnet.nodejumper.io/babylon-testnet/babylon-testnet_latest.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.babylond

# start service
sudo systemctl daemon-reload
sudo systemctl enable babylond
sudo systemctl start babylond

break
;;

"Create Wallet")
babylond keys add $WALLET
echo "============================================================"
echo "Save address and mnemonic"
echo "============================================================"
BABYLON_WALLET_ADDRESS=$(babylond keys show $WALLET -a)
BABYLON_VALOPER_ADDRESS=$(babylond keys show $WALLET --bech val -a)
echo 'export BABYLON_WALLET_ADDRESS='${BABYLON_WALLET_ADDRESS} >> $HOME/.bash_profile
echo 'export BABYLON_VALOPER_ADDRESS='${BABYLON_VALOPER_ADDRESS} >> $HOME/.bash_profile
source $HOME/.bash_profile

break
;;

"Create Validator")
babylond tx staking create-validator \
--amount=1000000ubbn \
--pubkey=$(babylond tendermint show-validator) \
--moniker=$NODENAME \
--chain-id=bbn-test-3 \
--commission-rate=0.10 \
--commission-max-rate=0.20 \
--commission-max-change-rate=0.01 \
--min-self-delegation=1 \
--from=wallet \
--gas-prices=0.00001ubbn \
--gas-adjustment=1.5 \
--gas=auto \
-y
  
break
;;

"Exit")
exit
;;
*) echo "invalid option $REPLY";;
esac
done
done
