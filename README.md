Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.22.6"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export PRYSM_CHAIN_ID="prysm-devnet-1"" >> $HOME/.bash_profile
echo "export PRYSM_PORT="25"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
rm -rf prysm
git clone https://github.com/kleomedes/prysm prysm
cd prysm
git checkout v0.1.0-devnet
make install
```

**config and init app**
```
prysmd config node tcp://localhost:${PRYSM_PORT}657
prysmd config keyring-backend os
prysmd config chain-id prysm-devnet-1
prysmd init "test" --chain-id prysm-devnet-1
```

**download genesis and addrbook**
```
wget -O $HOME/.prysm/config/genesis.json https://server-5.itrocket.net/testnet/prysm/genesis.json
wget -O $HOME/.prysm/config/addrbook.json  https://server-5.itrocket.net/testnet/prysm/addrbook.json
```

**set seeds and peers**
```
SEEDS="1b5b6a532e24c91d1bc4491a6b989581f5314ea5@prysm-testnet-seed.itrocket.net:25656"
PEERS="ff15df83487e4aa8d2819452063f336269958d09@prysm-testnet-peer.itrocket.net:25657,e1d4f82aa17c8cfae6e80117f39cf2e2a29eeb0b@37.27.112.131:10156,5f91fc0d6185fe29944bfa4698428f56cedbf0d2@152.53.66.0:25656,a53a6c4abbd4e4e212639dbff72e2c7295fd8cd8@213.199.43.242:29656,6616bf0bccdec74150ed492952879d394e8b4f22@62.171.161.196:17656,c8713da918b6c5ce36fc99b582c5eb9f9f3f2cd3@185.208.206.191:26656,729d9c4fe35efb9200b9775d2dfad0d5f7c7017b@77.237.244.73:29656,0b4b6be062055f913933597fe7d84c65286d444b@144.76.70.103:32656,5773b96f3a618bd1e5f18df04356ca44ee2cce0b@2.59.117.67:29656,f3dd019aafbee0483de1d5133f03e880d96cde6a@65.108.234.158:23656,cb36b0bf594bbb205827efd65fb2be0c82b4f854@176.9.126.78:20656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.prysm/config/config.toml
```

set custom ports in app.toml
sed -i.bak -e "s%:1317%:${PRYSM_PORT}317%g;
s%:8080%:${PRYSM_PORT}080%g;
s%:9090%:${PRYSM_PORT}090%g;
s%:9091%:${PRYSM_PORT}091%g;
s%:8545%:${PRYSM_PORT}545%g;
s%:8546%:${PRYSM_PORT}546%g;
s%:6065%:${PRYSM_PORT}065%g" $HOME/.prysm/config/app.toml

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${PRYSM_PORT}658%g;
s%:26657%:${PRYSM_PORT}657%g;
s%:6060%:${PRYSM_PORT}060%g;
s%:26656%:${PRYSM_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${PRYSM_PORT}656\"%;
s%:26660%:${PRYSM_PORT}660%g" $HOME/.prysm/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.prysm/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.prysm/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.prysm/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0uprysm"|g' $HOME/.prysm/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.prysm/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.prysm/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/prysmd.service > /dev/null <<EOF
[Unit]
Description=Prysm node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.prysm
ExecStart=$(which prysmd) start --home $HOME/.prysm
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
prysmd tendermint unsafe-reset-all --home $HOME/.prysm
if curl -s --head curl https://server-5.itrocket.net/testnet/prysm/prysm_2024-11-12_1829490_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-5.itrocket.net/testnet/prysm/prysm_2024-11-12_1829490_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.prysm
    else
  echo "no snapshot found"
fi
```

**enable and start service**
```
sudo systemctl daemon-reload
sudo systemctl enable prysmd
sudo systemctl restart prysmd && sudo journalctl -u prysmd -f
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/prysm/autoinstall/)
```

Create wallet
**to create a new wallet, use the following command. don’t forget to save the mnemonic**
```
prysmd keys add $WALLET
```

**to restore exexuting wallet, use the following command**
``
prysmd keys add $WALLET --recover
````
**save wallet and validator address**
```
WALLET_ADDRESS=$(prysmd keys show $WALLET -a)
VALOPER_ADDRESS=$(prysmd keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**check sync status, once your node is fully synced, the output from above will print "false"**
```
prysmd status 2>&1 | jq 
```

**before creating a validator, you need to fund your wallet and check balance**
```
prysmd query bank balances $WALLET_ADDRESS
```

**Node Sync Status Checker**
```
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.prysm/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://prysm-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  if [ "$blocks_left" -lt 0 ]; then
    blocks_left=0
  fi

  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"

  sleep 5
done
```

**Create validator**

Create validator.json file
```
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(prysmd comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000uprysm\",
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
```

**Create a validator using the JSON configuration**
```
prysmd tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id prysm-devnet-1 \
	--gas auto --gas-adjustment 1.5
```




**Firewall security**
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port
```
sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${PRYSM_PORT}656/tcp
sudo ufw enable
```

Delete node
sudo systemctl stop prysmd
sudo systemctl disable prysmd
sudo rm -rf /etc/systemd/system/prysmd.service
sudo rm $(which prysmd)
sudo rm -rf $HOME/.prysm
sed -i "/PRYSM_/d" $HOME/.bash_profile
