# initia-Node

| Bileşenler | Minimum Gereksinimler | 
| ------------ | ------------ |
| Ubuntu 20.04.2 |
| CPU |	4 |
| RAM	| 6 GB |
| Storage	| 250 GB SSD |

# Sunucumuzu Güncelliyoruz.

```
sudo apt update
```

```
sudo apt install curl git jq build-essential gcc unzip wget lz4 -y
```

# GO'yu Kuruyoruz.

```
cd $HOME && \
ver="1.21.3" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

# GEREKLİ YERLERİ DEĞİŞTİRİN.
$INITIA_NODENAME
$INITIA_WALLET

```
echo "export INITIA_NODENAME=$INITIA_NODENAME"  >> $HOME/.bash_profile
echo "export INITIA_WALLET=$INITIA_WALLET" >> $HOME/.bash_profile
echo "export INITIA_PORT=11" >> $HOME/.bash_profile
echo "export INITIA_CHAIN_ID=initiation-1" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

# Repoyu Klonluyoruz.

```
git clone https://github.com/initia-labs/initia
```

Klasöre giriyoruz.

```
cd initia
```

```
git checkout v0.2.15
```

```
make install
```

# Olduğu gibi Kopyalayın.
```
initiad config set client chain-id $INITIA_CHAIN_ID
initiad config set client keyring-backend test
initiad init --chain-id $INITIA_CHAIN_ID $INITIA_NODENAME

wget https://testnet.anatolianteam.com/initia/genesis.json -O $HOME/.initia/config/genesis.json
wget https://testnet.anatolianteam.com/initia/addrbook.json -O $HOME/.initia/config/addrbook.json

sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.15uinit,0.01uusdc"|g' $HOME/.initia/config/app.toml

indexer="null"
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.initia/config/config.toml

SEEDS="2eaa272622d1ba6796100ab39f58c75d458b9dbc@34.142.181.82:26656,c28827cb96c14c905b127b92065a3fb4cd77d7f6@testnet-seeds.whispernode.com:25756"
PEERS="0cb7dc2a96dfdf228547ef4a89da838ffc036f39@85.10.200.82:26656,093e1b89a498b6a8760ad2188fbda30a05e4f300@35.240.207.217:26656"
sed -i 's|^seeds *=.*|seeds = "'$SEEDS'"|; s|^persistent_peers *=.*|persistent_peers = "'$PEERS'"|' $HOME/.initia/config/config.toml


sed -i 's|^prometheus *=.*|prometheus = true|' $HOME/.initia/config/config.toml

pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="50"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.initia/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.initia/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.initia/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.initia/config/app.toml

sed -i.bak -e "
s%:26658%:${INITIA_PORT}658%g;
s%:26657%:${INITIA_PORT}657%g;
s%:6060%:${INITIA_PORT}060%g;
s%:26656%:${INITIA_PORT}656%g;
s%:26660%:${INITIA_PORT}660%g
" $HOME/.initia/config/config.toml
sed -i.bak -e "
s%:1317%:${INITIA_PORT}317%g; 
s%:8080%:${INITIA_PORT}080%g; 
s%:9090%:${INITIA_PORT}090%g; 
s%:9091%:${INITIA_PORT}091%g
" $HOME/.initia/config/app.toml
sed -i.bak -e "s%:26657%:${INITIA_PORT}657%g" $HOME/.initia/config/client.toml

PUB_IP=`curl -s -4 icanhazip.com`
sed -e "s|external_address = \".*\"|external_address = \"$PUB_IP:${INITIA_PORT}656\"|g" ~/.initia/config/config.toml > ~/.initia/config/config.toml.tmp
mv ~/.initia/config/config.toml.tmp  ~/.initia/config/config.toml

tee /etc/systemd/system/initiad.service > /dev/null << EOF
[Unit]
Description=Initia Node
After=network-online.target

[Service]
User=$USER
ExecStart=$(which initiad) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

```
systemctl daemon-reload
systemctl enable initiad
systemctl start initiad
```

```
journalctl -u initiad -f -o cat
```

# SNAPSHOT

```
sudo apt update
sudo apt install lz4 -y
```

```
systemctl stop initiad

cp $HOME/.initia/data/priv_validator_state.json $HOME/.initia/priv_validator_state.json.backup 

initiad tendermint unsafe-reset-all --home $HOME/.initia --keep-addr-book
SNAP_NAME=$(curl -s https://testnet.anatolianteam.com/initia/ | egrep -o ">initiation-1.*\.tar.lz4" | tr -d ">")
curl -L https://testnet.anatolianteam.com/initia/${SNAP_NAME} | tar -I lz4 -xf - -C $HOME/.initia

mv $HOME/.initia/priv_validator_state.json.backup $HOME/.initia/data/priv_validator_state.json 

systemctl restart initiad && journalctl -u initiad -f -o cat
```

Eğer Loglar akmıyosa, Şu komutla sync durumuna bakabilirsiniz.

```
initiad status 2>&1 | jq .sync_info
```

# Validator Kuracağız
Not: Eşitlenene Kadar bekliyoruz.
Kontrol Etmek için initia Scanı kullanabilirsiniz.
https://scan.initia.tech/initiation-1

# Cüzdan Oluşturuyoruz.

Kodu Yazdıktan Sonra şifre soracak 2 kere aynı şifreyi girin ve 24 Kelimenizi Kaydedin.

```
initiad keys add $INITIA_WALLET
```

Daha Sonra Cüzdanınıza Faucet alın Faucet Linki:
https://faucet.testnet.initia.xyz/

Validator Oluşturma Komutu,

```
initiad tx mstaking create-validator \
  --amount=1000000uinit \
  --pubkey=$(initiad tendermint show-validator) \
  --moniker=nodeismi \
  --chain-id=initiation-1 \
  --commission-rate=0.05 \
  --commission-max-rate=0.10 \
  --commission-max-change-rate=0.01 \
  --from=cüzdanismi \
  --details="" \
  --gas=2000000 --fees=300000uinit \
  -y
```








