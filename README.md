Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

Node Name
Wallet
Port
27
Pruning
Pruning Keep Recent
100
Pruning Interval
19
# install go, if needed
cd $HOME
VER="1.22.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin

# set vars
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export ALLORA_CHAIN_ID="allora-testnet-1"" >> $HOME/.bash_profile
echo "export ALLORA_PORT="27"" >> $HOME/.bash_profile
source $HOME/.bash_profile

# download binary
cd $HOME
rm -rf allora-chain
git clone https://github.com/allora-network/allora-chain
cd allora-chain
git checkout v0.8.3
make install

# config and init app
allorad init $MONIKER --chain-id $ALLORA_CHAIN_ID 
allorad config set client chain-id allora-testnet-1
allorad config set client keyring-backend os
sed -i -e "s|^node *=.*|node = \"tcp://localhost:${ALLORA_PORT}657\"|" $HOME/.allorad/config/client.toml

# download genesis and addrbook
wget -O $HOME/.allorad/config/genesis.json https://undefined/testnet/allora/genesis.json
wget -O $HOME/.allorad/config/addrbook.json  https://undefined/testnet/allora/addrbook.json

# set seeds and peers
SEEDS="720d83b52611c64d119adfc4d08d2e85885d8c74@allora-testnet-seed.itrocket.net:27656"
PEERS="a8cde2de31410d896668e53446495a4a68c4c24f@allora-testnet-peer.itrocket.net:27656,b77f7722b89b5c672a7333c5f0c05d3e04e7eb89@15.204.101.32:26656,2339ddf13f6d8ae2c78a0d0bce8001a19ea2f38c@15.204.101.34:26656,0f6b64fcd38872d18a78d89e090a5e6928883d52@8.209.116.116:26656,79af04335a0ac10073ef9342edba78e8fb08f9fc@89.58.0.245:37778,d3880fd88ab17dd677d67dd6e494305f48833014@162.55.199.126:26656,d3c79122924ff477e941ec0ca1ed775cfb01ca20@66.35.84.140:26656,3668a9f02aa50e701d2bb027ec1176100998606b@46.4.95.49:26656,6bd00668f433787d8418f9d9a477ed1f311d494d@15.204.101.36:26656,2eb9f5f80d721be2d37ab72c10a7be6aaf7897a4@15.204.101.92:26656,18fbf5f16f73e216f93304d94e8b79bf5acd7578@15.204.101.152:26656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.allorad/config/config.toml

# set custom ports in app.toml
sed -i.bak -e "s%:1317%:${ALLORA_PORT}317%g;
s%:8080%:${ALLORA_PORT}080%g;
s%:9090%:${ALLORA_PORT}090%g;
s%:9091%:${ALLORA_PORT}091%g;
s%:8545%:${ALLORA_PORT}545%g;
s%:8546%:${ALLORA_PORT}546%g;
s%:6065%:${ALLORA_PORT}065%g" $HOME/.allorad/config/app.toml

# set custom ports in config.toml file
sed -i.bak -e "s%:26658%:${ALLORA_PORT}658%g;
s%:26657%:${ALLORA_PORT}657%g;
s%:6060%:${ALLORA_PORT}060%g;
s%:26656%:${ALLORA_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${ALLORA_PORT}656\"%;
s%:26660%:${ALLORA_PORT}660%g" $HOME/.allorad/config/config.toml

# config pruning
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.allorad/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.allorad/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.allorad/config/app.toml

# set minimum gas price, enable prometheus and disable indexing
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0uallo"|g' $HOME/.allorad/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.allorad/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.allorad/config/config.toml

# create service file
sudo tee /etc/systemd/system/allorad.service > /dev/null <<EOF
[Unit]
Description=Allora node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.allorad
ExecStart=$(which allorad) start --home $HOME/.allorad
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

# reset and download snapshot
allorad tendermint unsafe-reset-all --home $HOME/.allorad
if curl -s --head curl https://undefined/testnet/allora/null | head -n 1 | grep "200" > /dev/null; then
  curl https://undefined/testnet/allora/null | lz4 -dc - | tar -xf - -C $HOME/.allorad
    else
  echo "no snapshot found"
fi

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable allorad
sudo systemctl restart allorad && sudo journalctl -u allorad -fo cat
Automatic Installation
pruning: | indexer:

source <(curl -s https://itrocket.net/api/testnet/allora/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
allorad keys add $WALLET

# to restore exexuting wallet, use the following command
allorad keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(allorad keys show $WALLET -a)
VALOPER_ADDRESS=$(allorad keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
allorad status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
allorad query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.allorad/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://allora-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"
  sleep 5
done
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, uallo
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
cd $HOME
# Create validator.json file
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(allorad comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000uallo\",
    \"moniker\": \"test\",
    \"identity\": \"\",
    \"website\": \"\",
    \"security\": \"\",
    \"details\": \"I love blockchain ❤️\",
    \"commission-rate\": \"0.1\",
    \"commission-max-rate\": \"0.2\",
    \"commission-max-change-rate\": \"0.01\",
    \"min-self-delegation\": \"1\"
}" > validator.json
# Create a validator using the JSON configuration
allorad tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id allora-testnet-1 \
	--gas auto --gas-adjustment 1.5
	
Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

Firewall security
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port

sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${ALLORA_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop allorad
sudo systemctl disable allorad
sudo rm -rf /etc/systemd/system/allorad.service
sudo rm $(which allorad)
sudo rm -rf $HOME/.allorad
sed -i "/ALLORA_/d" $HOME/.bash_profile
