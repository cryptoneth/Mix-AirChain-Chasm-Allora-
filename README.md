# Mix-AirChain-Chasm-Allora

AirChain Source = https://paragraph.xyz/@sarox.eth/airchain-rollup?referrer=0xbefEf0FE13B9bD398A88DAB74CCd62099C51333C

Update Packages
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```
Install Go
```
VERSION="1.21.6"
ARCH="amd64"
curl -O -L "https://golang.org/dl/go${VERSION}.linux-${ARCH}.tar.gz"
tar -xf "go${VERSION}.linux-${ARCH}.tar.gz"
sudo rm -rf /usr/local/go
sudo mv -v go /usr/local
export GOPATH=$HOME/go
export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin
source ~/.bash_profile
go version
```
Setup EVM station service:

Clone GitHub Repositories
```
git clone https://github.com/airchains-network/evm-station.git
git clone https://github.com/airchains-network/tracks.git
```
Setup the station
```
rm -r ~/.evmosd
cd ~/evm-station
git checkout --detac v1.0.2
go mod tidy
/bin/bash ./scripts/local-setup.sh
```
Create env file
```
cd ~
```
```
echo 'MONIKER="localtestnet"
KEYRING="test"
KEYALGO="eth_secp256k1"
LOGLEVEL="info"
HOMEDIR="$HOME/.evmosd"
TRACE=""
BASEFEE=1000000000
CONFIG=$HOMEDIR/config/config.toml
APP_TOML=$HOMEDIR/config/app.toml
GENESIS=$HOMEDIR/config/genesis.json
TMP_GENESIS=$HOMEDIR/config/tmp_genesis.json
VAL_KEY="mykey"' > .rollup-env
```
Create evmosd service file
```
sudo tee /etc/systemd/system/evmosd.service > /dev/null << EOF
[Unit]
Description=ZK
After=network.target

[Service]
User=root
EnvironmentFile=/root/.rollup-env
ExecStart=/root/evm-station/build/station-evm start --metrics "" --log_level info --json-rpc.api eth,txpool,personal,net,debug,web3 --chain-id "stationevm_1234-1"
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
```

Start evmosd service
```
sudo systemctl enable evmosd
sudo systemctl start evmosd
sudo journalctl -u evmosd -f --no-hostname -o cat
```
کنترل + سی + بزنید

the service logs should be like this

Get Your Private Key of EVM Station
```
cd evm-station
/bin/bash ./scripts/local-keys.sh
```

Change evmosd ports
```
#stop evmosd
systemctl stop evmosd
```
```
#change ports
echo "export G_PORT="17"" >> $HOME/.bash_profile
source $HOME/.bash_profile

sed -i.bak -e "s%:1317%:${G_PORT}317%g;
s%:8080%:${G_PORT}080%g;
s%:9090%:${G_PORT}090%g;
s%:9091%:${G_PORT}091%g;
s%:8545%:${G_PORT}545%g;
s%:8546%:${G_PORT}546%g;
s%:6065%:${G_PORT}065%g" $HOME/.evmosd/config/app.toml

sed -i.bak -e "s%:26658%:${G_PORT}658%g;
s%:26657%:${G_PORT}657%g;
s%:6060%:${G_PORT}060%g;
s%:26656%:${G_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${G_PORT}656\"%;
s%:26660%:${G_PORT}660%g" $HOME/.evmosd/config/config.toml

sed -i -e 's/address = "127.0.0.1:17545"/address = "0.0.0.0:17545"/' -e 's/ws-address = "127.0.0.1:17546"/ws-address = "0.0.0.0:17546"/' $HOME/.evmosd/config/app.toml
```
```
sudo ufw allow 17545
sudo ufw allow 17546
```
```
Restart evmosd service

sudo systemctl restart evmosd


sudo journalctl -u evmosd -f --no-hostname -o cat
```
کنترل + سی + بزنید


Setup EigenDA Keys

Download the Binary to Generate Keys
```
cd ~
wget https://github.com/airchains-network/tracks/releases/download/v0.0.2/eigenlayer
```

Create and List Keys
```
chmod +x ./eigenlayer
./eigenlayer operator keys create --key-type ecdsa myEigenDAKey
```
this is the result you will see and you should save.

Set Up and Run Tracker
```
sudo rm -rf ~/.tracks
cd tracks
go mod tidy
```
Initiate Sequencer

just replace <eigen-wallet-address> and execute it
```
go run cmd/main.go init --daRpc "disperser-holesky.eigenda.xyz" --daKey "<eigen-wallet-address>" --daType "eigen" --moniker "mySequencer" --stationRpc "http://127.0.0.1:17545" --stationAPI "http://127.0.0.1:17545" --stationType "evm"
```

Setup Tracker Component


برای ساخت کیف دو کامند وجود داره . یا اگر از قبل ولتی دارید . یا اگر میخواید جدید بسازید

اگر از قبل ولت دارید

```

go run cmd/main.go keys import  --accountName mySequencerAccount --accountPath $HOME/.tracks/junction-accounts/keys --mnemonic 'your-mnemonic-phrase'
```

otherwise, generate a new key with this command and save its result (Mnemonic & Address)

ساخت ولت جدید

```

go run cmd/main.go keys junction --accountName mySequencerAccount --accountPath $HOME/.tracks/junction-accounts/keys
```

سایت پایین کم فاست میده . از دیسکورد بگیرید تو چنل Switchyard Faucet
get facuet for airchain address, on this site https://airchains.faucetme.pro/

Initiate Prover

```
go run cmd/main.go prover v1EVM
```
Get your Node ID
```
cat ~/.tracks/config/sequencer.toml | grep node_id
```
Create station on Junction

replace <tracker-wallet-address> (address with air...)

replace <node_id>

```
go run cmd/main.go create-station --accountName mySequencerAccount --accountPath $HOME/.tracks/junction-accounts/keys --jsonRPC "https://junction-testnet-rpc.synergynodes.com/" --info "EVM Track" --tracks <tracker-wallet-address> --bootstrapNode "/ip4/127.0.0.1/tcp/2300/p2p/<node_id>"
```


Create staiond service file

```
sudo tee /etc/systemd/system/stationd.service > /dev/null << EOF
[Unit]
Description=station track service
After=network-online.target
[Service]
User=root
WorkingDirectory=/root/tracks/
ExecStart=/usr/local/go/bin/go run cmd/main.go start
Restart=always
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

Run stationd service
```
sudo systemctl enable stationd
sudo systemctl restart stationd
sudo journalctl -u stationd -f --no-hostname -o cat
```

حالا اسکریپت ران میکنید که تراکنش بزنه
```
git clone https://github.com/sarox0987/evmos-farmer.git
cd evmos-farmer
```

حالا یه اسکرین باز کنید اونجا تراکنش بخوره

```
screen -S farm
```

```
go mod tidy
go run main.go
```
اینجا کلید خصوصی که اول اول بهتون داد یعنی اولین کلید خصوصی که داد رو وارد کنید
بعدش تو قسمت RPC 

http://127.0.0.1:17545

این رو وارد کنید



حالا کنترل A + D بزنید از اسکرین خارج شید

یه اسکرین دیگه باز کنید
```
screen -S fix
```

```
nano fix.sh
```

همه کد زیر رو تو فایل کپی کنید

```
service_name="stationd"
error_strings=(
  "ERROR"
  "with gas used"
  "Failed to Init VRF"
  "Client connection error: error while requesting node"
  "Error in getting sender balance : http post error: Post"
  "rpc error: code = ResourceExhausted desc = request ratelimited"
  "rpc error: code = ResourceExhausted desc = request ratelimited: System blob rate limit for quorum 0"
  "ERR"
  "Retrying the transaction after 10 seconds..."
  "Error in VerifyPod transaction Error"
  "Error in ValidateVRF transaction Error"
  "Failed to get transaction by hash: not found"
  "json_rpc_error_string: error while requesting node"
  "can not get junctionDetails.json data"
  "JsonRPC should not be empty at config file"
  "Error in getting address"
  "Failed to load conf info"
  "error unmarshalling config"
  "Error in initiating sequencer nodes due to the above error"
  "Failed to Transact Verify pod"
  " VRF record is nil"
)
restart_delay=120
config_file="$HOME/.tracks/config/sequencer.toml"

unique_urls=(
  "https://airchains-testnet-rpc.crouton.digital/"
  "https://rpc-testnet-airchains.nodeist.net/"
  "https://airchains-rpc.sbgid.com/"
)

function select_random_url {
  local array=("$@")
  local rand_index=$(( RANDOM % ${#array[@]} ))
  echo "${array[$rand_index]}"
}

function update_rpc_and_restart {
  local random_url=$(select_random_url "${unique_urls[@]}")
  sed -i -e "s|JunctionRPC = \"[^\"]*\"|JunctionRPC = \"$random_url\"|" "$config_file"
  systemctl restart "$service_name"
  echo "Service $service_name restarted"
  echo -e "\e[32mRemoved RPC URL: $random_url\e[0m"
  sleep "$restart_delay"
}

function display_waiting_message {
  echo -e "\e[35mI am waiting for you AIRCHAIN\e[0m"
}

echo "Script started to monitor errors in PC logs..."
echo -e "\e[32mby onixia\e[0m"
echo "Timestamp: $(date)"

while true; do
  logs=$(systemctl status "$service_name" --no-pager | tail -n 10)

  for error_string in "${error_strings[@]}"; do
    if echo "$logs" | grep -q "$error_string"; then
      echo "Found error ('$error_string') in logs, updating $config_file and restarting $service_name..."

      update_rpc_and_restart

      systemctl stop "$service_name"
      cd ~/tracks

      echo "Starting rollback after changing RPC..."
      go run cmd/main.go rollback
      go run cmd/main.go rollback
      go run cmd/main.go rollback
      echo "Rollback completed, restarting $service_name..."

      systemctl start "$service_name"
      display_waiting_message
      break
    fi
  done

  sleep "$restart_delay"
done
```

کنترل + ایکس بزنید . y بزنید . اینتر بزنید فایل سیو شه

```
chmod +x fix.sh

bash fix.sh
```

کنترل + آ + دی بزنید از اسکرین خارج شید



airChain Finished
========================================================================================================================================================

Chasm Source : https://github.com/0xmoei/Chasm-Network?tab=readme-ov-file

سایت ها 
https://scout.chasm.net/private-mint
https://console.groq.com/keys
https://openrouter.ai/

```

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

```
cd ~

nano .env
```

کل کامند پایین رو بعد جایگذاری سیو کنید . ************ جایگذاری ها تو ویدیو توضیح داده شده یا از داکیومنت معی میتونید ببینید https://github.com/0xmoei/Chasm-Network?tab=readme-ov-file

```
PORT=3001
LOGGER_LEVEL=debug

# Chasm
ORCHESTRATOR_URL=https://orchestrator.chasm.net
SCOUT_NAME=myscout
SCOUT_UID=
WEBHOOK_API_KEY=
WEBHOOK_URL=http://x.x.x.x:3001/

# Chosen Provider (groq, openai)
PROVIDERS=groq
MODEL=gemma2-9b-it
GROQ_API_KEY=

# Optional
OPENROUTER_API_KEY=
OPENAI_API_KEY=

NODE_ENV=production
```

کنترل + ایکس بزنید . y بزنید . اینتر بزنید فایل سیو شه

```
cp .env 2.env

nano 2.env
```

هر چی هست رو پاک کنید اول

کل کد پایین رو کپی کنید

```
PORT=3002
LOGGER_LEVEL=debug

# Chasm
ORCHESTRATOR_URL=https://orchestrator.chasm.net
SCOUT_NAME=myscout
SCOUT_UID=
WEBHOOK_API_KEY=
WEBHOOK_URL=http://x.x.x.x:3002/

# Chosen Provider (groq, openai)
PROVIDERS=groq
MODEL=gemma2-9b-it
GROQ_API_KEY=

# Optional
OPENROUTER_API_KEY=
OPENAI_API_KEY=

NODE_ENV=production
```

نترل + ایکس بزنید . y بزنید . اینتر بزنید فایل سیو شه

```
# Open Port
sudo ufw allow 3001
sudo ufw allow 3002


docker pull johnsonchasm/chasm-scout

docker run -d --restart=always --env-file ./.env -p 3001:3001 --name scout johnsonchasm/chasm-scout

docker run -d --restart=always --env-file ./2.env -p 3002:3002 --name scout2 johnsonchasm/chasm-scout
```

حالا لاگ هر دو Scout رو ببینید . 
```
docker logs scout
```
```
docker logs scout2
```

اگر سالم بود هیچی . اگر نه چند بار کامند پایین رو تا ****** بزنید
```
docker stop scout
docker rm scout
docker stop scout2
docker rm scout2

sudo ufw allow 3001
sudo ufw allow 3002


docker pull johnsonchasm/chasm-scout

docker run -d --restart=always --env-file ./.env -p 3001:3001 --name scout johnsonchasm/chasm-scout

docker run -d --restart=always --env-file ./2.env -p 3002:3002 --name scout2 johnsonchasm/chasm-scout
```

حالا لاگ هر دو Scout رو ببینید . 
```
docker logs scout
```
```
docker logs scout2
```
****** تا اینجا

اگر سالم بود هیچی . اگر نه چای بالای میتونیید پایینی هارو امتحان کنید


+++++++++++++++ تست 1
```
docker stop scout
docker rm scout
docker stop scout2
docker rm scout2

docker pull chasmtech/chasm-scout

docker run -d --restart=always --env-file ./.env -p 3001:3001 --name scout chasmtech/chasm-scout

docker run -d --restart=always --env-file ./2.env -p 3002:3002 --name scout2 chasmtech/chasm-scout
```

حالا باز لاگ رو ببینید 

```
docker logs scout
```
```
docker logs scout2
```
اگر سالم بود که هیچ اگر نه


++++++++++++++ تست 2

```
docker stop scout
docker rm scout
docker stop scout2
docker rm scout2

docker pull chasmtech/chasm-scout:0.0.4


docker run -d --restart=always --env-file ./.env -p 3001:3001 --name scout chasmtech/chasm-scout:0.0.4
docker run -d --restart=always --env-file ./2.env -p 3002:3002 --name scout2 chasmtech/chasm-scout:0.0.4
```

حالا باز لاگ رو ببینید 

```
docker logs scout
```
```
docker logs scout2
```
اگر سالم بود که هیچ اگر نه چندین بار بالایی هارو بزنید لاگ چک کنید

========================================================================================================================================================

Allora Worker Node , Source : 
