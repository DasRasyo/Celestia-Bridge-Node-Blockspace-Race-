# Celestia Bridge Node (Blockspace-Race) Kurulum Rehberi

![Ekran görüntüsü 2023-03-28 040506](https://user-images.githubusercontent.com/94050636/228100509-865911b0-e167-40e5-9049-7538e0017276.png)


## Minimum Önerilen Sistem Gereksinimleri

Memory: 8 GB RAM <br/>
CPU: 6 cores <br/>
Disk: 500 GB SSD Storage <br/>
Bandwidth: 1 Gbps for Download/100 Mbps for Upload <br/>
Ubuntu Linux 20.04 (LTS) <br/>
NOT: Bridge node celestia-app ile birlikte çalışacağı için daha iyi özelliklerde bir sistem daha faydalı olacaktır. <br/>


## Sistem güncellemerini yaparak başlıyoruz.

```
sudo apt update && sudo apt upgrade -y && sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential \
git make ncdu -y
```

## Go kurulumu ile devam ediyoruz.

```
ver="1.19.4"
cd $HOME
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```

go versiyon komutunun çıktısı şu şekilde olmalıdır:

```
go version go1.19.4 linux/amd64
```

## Öncelikle celestia-app kuruyoruz

```
cd $HOME 
rm -rf celestia-app 
git clone https://github.com/celestiaorg/celestia-app.git 
cd celestia-app/ 
APP_VERSION=v0.13.0
git checkout tags/$APP_VERSION -b $APP_VERSION
make install
```

celestia-app versiyon kontrolü:

```
celestia-appd version
```

Çıktı şu şekilde olmalı:

```
0.12.1
```

## P2P network indirerek devam ediyoruz

```
cd $HOME
rm -rf networks
git clone https://github.com/celestiaorg/networks.git
```

## Init işlemini yapalım

```
celestia-appd init "node-adı" --chain-id blockspacerace-0
```

## Genesis&&Seed

```
cp $HOME/networks/blockspacerace/genesis.json $HOME/.celestia-app/config
sed -i -e "s|^seeds *=.*|seeds = \"0293f2cf7184da95bc6ea6ff31c7e97578b9c7ff@65.109.106.95:26656\"|" $HOME/.celestia-app/config/config.toml
```

## Pruning ayarını güncelleyelim

```
PRUNING="nothing"
sed -i -e "s/^pruning *=.*/pruning = \"$PRUNING\"/" $HOME/.celestia-app/config/app.toml
```

```
celestia-appd tendermint unsafe-reset-all --home $HOME/.celestia-app
```

## Servis oluşturma

```
sudo tee <<EOF >/dev/null /etc/systemd/system/celestia-appd.service
[Unit]
Description=celestia-appd Cosmos daemon
After=network-online.target

[Service]
User=$USER
ExecStart=$HOME/go/bin/celestia-appd start
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```

## Node'u Başlatalım

```
sudo systemctl enable celestia-appd
sudo systemctl start celestia-appd
```

## Log kontrolü

```
sudo journalctl -u celestia-appd.service -f
```

## Senkronize olduktan sonra bridge node kurulumu ile devam ediyoruz

```
cd $HOME
rm -rf celestia-node
git clone https://github.com/celestiaorg/celestia-node.git
cd celestia-node/
git checkout tags/v0.9.5
make build
make install
make cel-key
```

## Version kontrolü

```
celestia version
```

Çıktı şu şekilde olmalı:

```
Semantic version: v0.9.5 
Commit: 2718b1dfb7ee4fbcc8614601dc7d58019bfb1437
```

## Init işlemi ile devam ediyoruz. Bu işlemin ardından cüzdanımız oluşacak, cüzdan adresimizi ve kelimlerimizi alacağız. Bunları yedeklemeyi unutmayın!!!

```
celestia bridge init --core.ip localhost --core.rpc.port 26657 --core.grpc.port 9090 --p2p.network blockspacerace
```

## Cüzdan adresi ve adını şu kod ile görüntüleyebilirsiniz:

```
./cel-key list --node.type bridge --p2p.network blockspacerace
```

Cüzdan adresinize Celestia Discord sunucusundaki Blockspace Race başlığı altında bulunan #faucet kanalından test coini almayı unutmayın. Cüzdanda coin olması gereklidir.

## Bridge Node için servis oluşturalım

```
sudo tee <<EOF >/dev/null /etc/systemd/system/celestia-bridge.service
[Unit]
Description=celestia-bridge Cosmos daemon
After=network-online.target

[Service]
User=$USER
ExecStart=/usr/local/bin/celestia bridge start --core.ip localhost --core.rpc.port 26657 --core.grpc.port 9090 --keyring.accname my_celes_key --metrics.tls=false --metrics --metrics.endpoint otel.celestia.tools:4318 --p2p.network blockspacerace
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```

## Bridge Node'umuzu başlatalım

```
sudo systemctl enable celestia-bridge
sudo systemctl start celestia-bridge
```

## Log kontrolü

```
sudo journalctl -u celestia-bridge.service -f
```

## Node ID öğrenme

Not: Aşağıdaki iki komutu girerek node ID'mizi öğreneceğiz. Bu node ID ile Celestia Bridge Node'umuzu [Tiascan](https://tiascan.com/bridge-nodes) üzerinden uptime gibi çeşitli verilerle birlikte takip edebileceğiz. Ayrıca Knack Portal'da tasklarda node ID'mizi girmemiz gerekecek. Bu ID ile node'umuz takip edilecek, çalışma süresi izlenecek.

```
AUTH_TOKEN=$(celestia bridge auth admin --p2p.network blockspacerace)

curl -X POST \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H 'Content-Type: application/json' \
     -d '{"jsonrpc":"2.0","id":0,"method":"p2p.Info","params":[]}' \
     http://localhost:26658
```

Bu iki komutu girdiğinizde şuna benzer bir çıktı almalısınız;

```
{"jsonrpc":"2.0","result":{"ID":"12D3KooWJXXXXXXXXXXXXXXXXXXXXXXXX"
```

12D3 ile başlayan node'umuzun ID'sidir. [Tiascan](https://tiascan.com/bridge-nodes) üzerinden aratabilirsiniz.

## Yedeklenmesi Gereken Dosyalar

WinSCP veya aynı işleve sahip bir araç ile .celestia-bridge-blockspacerace-0 klasörünü altında bulunan keys klasörünü yedeklemeniz gerekmektedir. Key bilgilerimiz burada. Knack portala node bilgilerinizi kaydetmeden önce bu klasörü mutlaka yedekleyin!

## BAŞARILAR!
