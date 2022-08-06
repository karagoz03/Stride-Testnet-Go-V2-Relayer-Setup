# Setup GO Relayer v2 between Stride and GAIA
An example of using GO Relayer v2 for Stride Testnet 

## Preparation before you start
Before setting up relayer you need to make sure you already have. (In this example, i'll use relayer between Stride and Gaia chains)
1. Fully synchronized RPC nodes for each Cosmos project you want to connect
2. RPC endpoints should be exposed and available from relayer instance
#### RPC configuration is located in `config.toml` file
# STRIDE
`nano $HOME/.stride/config/config.toml` , find laadr and replace with laddr = `"tcp://0.0.0.0:16657"` 
# GAIA
`nano $HOME/.gaia/config/config.toml` , find laadr and replace with laddr = `"tcp://0.0.0.0:23657"`   

3. Indexing is set to `kv` and is enabled on each node
```
# STRIDE
sed -i -e "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.stride/config/config.toml
# GAIA
sed -i -e "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.gaia/config/config.toml  
```

4. Dont forget to restart fullnode service after making changes
5. For each chain you will need to have wallets that are funded with tokens. This wallets will be used to do all relayer stuff
# Prepare Rpc adresses for each chain. To learn your rpc, run:

`echo "$(curl -s ifconfig.me)$(grep -A 3 "[rpc]" ~/.stride/config/config.toml | egrep -o ":[0-9]+")"` in stride,

`echo "$(curl -s ifconfig.me)$(grep -A 3 "[rpc]" ~/.gaia/config/config.toml | egrep -o ":[0-9]+")"` in gaia

and take notes of outputs. It should be in `ip:port` format. They are your rpc addresses.

## Update system
```
sudo apt update && sudo apt upgrade -y
```

## Install GO
```
if ! [ -x "$(command -v go)" ]; then
  ver="1.18.3"
  cd $HOME
  wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
  sudo rm -rf /usr/local/go
  sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
  rm "go$ver.linux-amd64.tar.gz"
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
  source ~/.bash_profile
fi
```

## Install and initialize GO Relayer v2
```
git clone https://github.com/cosmos/relayer.git
cd relayer && git checkout v2.0.0-rc4
make install
rly config init
```

## Create relayer configuration files
### 1. Generate CHAIN_A config file using your own rpc.
```
sudo tee $HOME/.relayer/stride.json > /dev/null <<EOF
{
  "type": "cosmos",
  "value": {
    "key": "wallet",
    "chain-id": "STRIDE-TESTNET-2",
    "rpc-addr": "http://149.102.147.205:16657",
    "account-prefix": "stride",
    "keyring-backend": "test",
    "gas-adjustment": 1.2,
    "gas-prices": "0.001ustrd",
    "debug": true,
    "timeout": "20s",
    "output-format": "json",
    "sign-mode": "direct"
  }
}
EOF
```

### 2. Generate CHAIN_B config file using your own rpc.
```
sudo tee $HOME/.relayer/gaia.json > /dev/null <<EOF
{
  "type": "cosmos",
  "value": {
    "key": "wallet",
    "chain-id": "GAIA",
    "rpc-addr": "http://217.160.207.56:26657",
    "account-prefix": "cosmos",
    "keyring-backend": "test",
    "gas-adjustment": 1.2,
    "gas-prices": "0.001uatom",
    "debug": true,
    "timeout": "20s",
    "output-format": "json",
    "sign-mode": "direct"
  }
}
EOF
```

## Load chain configuration into the relayer
```
rly chains add --file=$HOME/.relayer/stride.json stride
rly chains add --file=$HOME/.relayer/gaia.json gaia
```

Check chains added to relayer
```
rly chains list
```

Successful output:
```
1: GAIA             -> type(cosmos) key(✘) bal(✘) path(✘)
2: STRIDE-TESTNET-2 -> type(cosmos) key(✘) bal(✘) path(✘)

```

## Load wallets into the relayer
```
rly keys restore stride wallet "write your stride mnemonic"
rly keys restore gaia wallet "write your gaia mnemonic"
```

Check wallet balance
```
rly q balance stride
rly q balance gaia
```

## Add path for STRIDE-TESTNET-2 and GAIA to relayer configuration file
Open `nano $HOME/.relayer/config/config.yaml` file, write your discord id to memo: and replace `paths: {}` with:
```
paths:
    stride-gaia:
        src:
            chain-id: STRIDE-TESTNET-2
            client-id: 07-tendermint-0
            connection-id: connection-0
        dst:
            chain-id: GAIA
            client-id: 07-tendermint-0
            connection-id: connection-0
        src-channel-filter:
            rule: allowlist
            channel-list: [channel-0, channel-1, channel-2, channel-3, channel-4]
```
to close ctrl+x and save changes with y, then pres enter.


Check path is correct
```
rly paths list
```

Successful output:
```
0: stride-gaia -> chns(✔) clnts(✔) conn(✔) (STRIDE-TESTNET-2<>GAIA)
```
If everything is correct, we can proceed with relayer service creation

## Create service
```
sudo tee /etc/systemd/system/relayerd.service > /dev/null <<EOF
[Unit]
Description=GO Relayer v2 Service
After=network.target
[Service]
Type=simple
User=$USER
ExecStart=$(which rly) start stride-gaia -p events
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

## Start the service
```
sudo systemctl daemon-reload
systemctl enable relayerd
systemctl start relayerd
```

## Check logs
```
journalctl -u relayerd -f -o cat
```
![image](https://user-images.githubusercontent.com/38834586/183264898-1d43ed07-aac9-4635-823a-50aa0ffbd536.png)



Check your wallet transaction in explorer to find the GO v2 Relayer Update Client(IBC) message:
![image](https://user-images.githubusercontent.com/38834586/183264811-037c8285-a321-4aa0-a2e2-b2343fa1024a.png)
https://poolparty.stride.zone/STRIDE/tx/C9890158D20229766E0C1BF473865DF93033A379933BF27AA0AEE7552C0F4ED5

## Special thanks to kjnodes#8455 for his awesome guides and helpfulness

