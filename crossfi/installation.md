### **Recommended Hardware:**

4 Cores, 8GB RAM, 200GB of storage (NVME)

# **Installation**

```bash
# install dependencies, if needed
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y

# install go, if needed
cd $HOME
VER="1.21.3"
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
echo "export CROSSFI_CHAIN_ID="crossfi-evm-testnet-1"" >> $HOME/.bash_profile
echo "export CROSSFI_PORT="18"" >> $HOME/.bash_profile
source $HOME/.bash_profile
 
# download binary
cd $HOME
wget https://github.com/crossfichain/crossfi-node/releases/download/v0.3.0-prebuild3/crossfi-node_0.3.0-prebuild3_linux_amd64.tar.gz && tar -xf crossfi-node_0.3.0-prebuild3_linux_amd64.tar.gz
tar -xvf crossfi-node_0.3.0-prebuild3_linux_amd64.tar.gz
chmod +x $HOME/bin/crossfid
mv $HOME/bin/crossfid /usr/local/bin/
rm -rf crossfi-node_0.3.0-prebuild3_linux_amd64.tar.gz $HOME/bin
 
# config and init app
crossfid init $MONIKER
sed -i -e "s|^node *=.*|node = \"tcp://localhost:${CROSSFI_PORT}657\"|" $HOME/.mineplex-chain/config/client.toml
 
# download genesis and addrbook
wget -O $HOME/.mineplex-chain/config/genesis.json https://snapshot.val1gators.net/crossfi-testnet/genesis.json
wget -O $HOME/.mineplex-chain/config/addrbook.json https://snapshot.val1gators.net/crossfi-testnet/addrbook.json
 
# set seeds and peers
SEEDS=""
$PEERS"de1e9221a35de41e08653ef0aca405c1f2ad500e@crossfi-testnet-peer.val1gators.net:19656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.mineplex-chain/config/config.toml
 
# set custom ports in app.toml
sed -i.bak -e "s|:1317|:${CROSSFI_PORT}317|g;
s|:8080|:${CROSSFI_PORT}080|g;
s|:9090|:${CROSSFI_PORT}090|g;
s|:9091|:${CROSSFI_PORT}091|g;
s|:8545|:${CROSSFI_PORT}545|g;
s|:8546|:${CROSSFI_PORT}546|g;
s|:6065|:${CROSSFI_PORT}065|g" $HOME/.mineplex-chain/config/app.toml
 
# set custom ports in config.toml file
sed -i.bak -e "s|:26658|:${CROSSFI_PORT}658|g;
s|:26657|:${CROSSFI_PORT}657|g;
s|:6060|:${CROSSFI_PORT}060|g;
s|:26656|:${CROSSFI_PORT}656|g;
s|^external_address = \"\"|external_address = \"$(wget -qO- eth0.me):${CROSSFI_PORT}656\"|g;
s|:26660|:${CROSSFI_PORT}660|g" $HOME/.mineplex-chain/config/config.toml
 
# config pruning
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.mineplex-chain/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.mineplex-chain/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.mineplex-chain/config/app.toml
 
# set minimum gas price, enable prometheus and disable indexing
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "10000000000000mpx"|g' $HOME/.mineplex-chain/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.mineplex-chain/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.mineplex-chain/config/config.toml
 
# create service file
sudo tee /etc/systemd/system/crossfid.service > /dev/null <<EOF
[Unit]
Description=Crossfi node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.mineplex-chain
ExecStart=$(which crossfid) start --home $HOME/.mineplex-chain
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
 
# reset and download snapshot
crossfid tendermint unsafe-reset-all --home $HOME/.mineplex-chain
 
# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable crossfid
sudo systemctl restart crossfid && sudo journalctl -u crossfid -f
```

# **Wallet**

### **Create a wallet**

```bash
crossfid keys add $WALLET
```

### **To restore your keys you have created, do this**

```bash
crossfid keys add $WALLET --recover
```

### **Save wallet and validator address**

```bash
WALLET_ADDRESS=$(crossfid keys show $WALLET -a)
VALOPER_ADDRESS=$(crossfid keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### **Check sync status, once your node is fully synced, the output from above will print "false"**

```bash
crossfid status 2>&1 | jq
```

### **After creating a validator, make sure the fund has been funded sucessfully**

```bash
crossfid query bank balances $WALLET_ADDRESS
```

# **Validator**

To create a validator at airchain node, you should import the data of your validator to the **validator.json** file

```bash
crossfid tx staking create-validator \
--amount 1000000mpx \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(crossfid tendermint show-validator) \
--moniker $MONIKER \
--identity "" \
--website "" \
--details "a validator blockchain" \
--chain-id crossfi-evm-testnet-1 \
--gas auto \
--gas-adjustment 1.5 \
--gas-prices 10000000000000mpx \
-y
```
