# Relayer between two cosmos networks Croeseid and Kichain
Here I will describe the process of creating a relayer between two cosmos networks Croeseid and Kichain to be exact In my case they were on one server, so Croeseid ports needed to be changed,
 i used chain-maind version 3.1.0-croeseid
First of all we need to check is this is at all possible

$ chain-maind q ibc-transfer params
receive_enabled: true
send_enabled: true
$ kid q ibc-transfer params
receive_enabled: true
send_enabled: true

Everything is in order and cross-transactions are available between these two networks

1) We need to install a relayer https://github.com/cosmos/relayer

$ git clone git@github.com:cosmos/relayer.git
$ git checkout v0.9.3
$ cd relayer && make install

2) We need to initialize it
$ rly config init

3) And create chain configurations In our particular case we will make a new directory and store those configurations there
$ mkdir rly_config
$ cd rly_config
$ nano kichain_config.json
{
 "chain-id": "kichain-t-4",
 "rpc-addr": "http://127.0.0.1:26657",
 "account-prefix": "tki",
 "gas-adjustment": 1.5,
 "gas-prices": "0.001tki",
 "trusting-period": "48h"
}
$ nano croeseid_config.json
{
 "chain-id": "testnet-croeseid-4",
 "rpc-addr": "http://127.0.0.1:26652",
 "account-prefix": "tcro",
 "gas-adjustment": 1.5,
 "gas-prices": "0.025basetcro",
 "trusting-period": "48h"
}

4) We will then add this to relayer config
$ rly chains add -f kichain_config.json
$ rly chains add croeseid_config.json

5) Next step is creating new keys and addresses
$ rly keys add kichain-t-4 testkichaiGnkey7
$ rly keys add testnet-croeseid-4 testcroGkey77

Or we can restore them
$ rly keys restore kichain-t-4 wallet_name "mnemonic"
$ rly keys restore testnet-croeseid-4 wallet_name_2 "mnemonic"

6) Then we need to add these keys to relayer config
$ rly chains edit testnet-croeseid-4 key testcroGkey77
$ rly chains edit kichain-t-4 key testkichaiGnkey7

7) I personally received addresses in kichain and Croeseid network I used a faucet in Croeseid network https://crypto.org/faucet to get some coins and transferred coins from my main kichain wallet to new one
We need to check the balances to make sure the funds are there
rly q balance kichain-t-4
rly q balance testnet-croeseid-4

8) Next step is clients initializing for both networks
$ rly light init kichain-t-4 -f
successfully created light client for kichain-t-4 by trusting endpoint http://127.0.0.1:26657...
$ rly light init testnet-croeseid-4
successfully created light client for testnet-croeseid-4 by trusting endpoint http://127.0.0.1:26652…
9) We create a channel between two networks
$ rly paths generate kichain-t-4 testnet-croeseid-4 fromki - port=transfer
$ rly paths generate testnet-croeseid-4 kichain-t-4 fromcro - port=transfer

10) After that we can confirm in a config file that a new path has been created
$ nano ~/.relayer/config/config.yaml
{
global:
 api-listen-addr: :5183
 timeout: 3m
 light-cache-size: 20
chains:
- key: testcroGkey77
 chain-id: testnet-croeseid-4
 rpc-addr: http://127.0.0.1:26652
 account-prefix: tcro
 gas-adjustment: 1.5
 gas-prices: 0.025basetcro
 trusting-period: 48h
- key: testkichaiGnkey7
 chain-id: kichain-t-4
 rpc-addr: http://127.0.0.1:26657
 account-prefix: tki
 gas-adjustment: 1.5
 gas-prices: 0.001utki
 trusting-period: 48h
paths:
 fromcro:
 src:
 chain-id: testnet-croeseid-4
 client-id: 07-tendermint-148
 connection-id: connection-116
 channel-id: channel-105
 port-id: transfer
 order: UNORDERED
 version: ics20–1
 dst:
 chain-id: kichain-t-4
 client-id: 07-tendermint-360
 connection-id: connection-365
 channel-id: channel-304
 port-id: transfer
 order: UNORDERED
 version: ics20–1
 strategy:
 type: naive
 fromki:
 src:
 chain-id: kichain-t-4
 client-id: 07-tendermint-357
 connection-id: connection-362
 channel-id: channel-303
 port-id: transfer
 order: UNORDERED
 version: ics20–1
 dst:
 chain-id: testnet-croeseid-4
 client-id: 07-tendermint-147
 connection-id: connection-115
 channel-id: channel-104
 port-id: transfer
 order: UNORDERED
 version: ics20–1
 strategy:
 type: naive
 }
Here we can also change timeout params from timeout: 10s to timeout: 3m
To be sure that our transactions will occur
10) We shall check that pass
$ rly paths list -d
 0: fromki -> chns(✔) clnts(✔) conn(✔) chan(✔) (kichain-t-4:transfer<>test net-croeseid-4:transfer)
 1: fromcro -> chns(✔) clnts(✔) conn(✔) chan(✔) (testnet-croeseid-4:transfe r<>kichain-t-4:transfer)
We can see checkmarks - everything is good at this point
11) We can also check if the chains are ready to relay over
$ rly chains list
 0: testnet-croeseid-4 -> key(✔) bal(✔) light(✔) path(✔)
 1: kichain-t-4 -> key(✔) bal(✔) light(✔) path(✔)
Good to go
12) We can transfer funds between out test wallets
$ rly tx transfer kichain-t-4 testnet-croeseid-4 1000000utki tcro142fz2uhvwhrrcrpqz6tgzg37xcdurr3uwkze2r - path fromki
I[2021–09–12|20:54:32.172] ✔ [kichain-t-4]@{302556} - msg(0:transfer) hash(4EC292F16A55982E2F87896E4B77648E1E27EA4B556BC2637E05D53A0528119B) https://api-challenge.blockchain.ki/txs/4EC292F16A55982E2F87896E4B77648E1E27EA4B556BC2637E05D53A0528119B
hash(39737EC0DA14A0CF5BB5E0E33E1348AA814C27D1D03976E3B0173C21CA803D28) https://api-challenge.blockchain.ki/txs/39737EC0DA14A0CF5BB5E0E33E1348AA814C27D1D03976E3B0173C21CA803D28
hash(752AA8E1768039DB29E016B6D5BE47111EEF4A78A2DE890FB696EE96BB230D4E) https://api-challenge.blockchain.ki/txs/752AA8E1768039DB29E016B6D5BE47111EEF4A78A2DE890FB696EE96BB230D4E
hash(3F25E69300B7F7380C549707C70067605791C085B3D4F8849D0B39888052684C) https://api-challenge.blockchain.ki/txs/3F25E69300B7F7380C549707C70067605791C085B3D4F8849D0B39888052684C
hash(77D44C9CD1C7C17851D029F6643D8E687ECE565DD99FCC285FE6235F54E74F17) https://api-challenge.blockchain.ki/txs/77D44C9CD1C7C17851D029F6643D8E687ECE565DD99FCC285FE6235F54E74F17
$ rly tx transfer testnet-croeseid-4 kichain-t-4 10000000basetcro tki1nadyayme8lcsqlmenvxl0lw9a63ctc9g2znz0r - path fromcro
hash 6F888163EA0B43A24C663BF6FCDE14712F6780F98267CA43EDCE9EDA62DB0ACF https://crypto.org/explorer/croeseid4/tx/6F888163EA0B43A24C663BF6FCDE14712F6780F98267CA43EDCE9EDA62DB0ACF
hash 7EEE3D10B1915ED7BE858A540E97ADA31DED7B399CA27BE129F1D10A86A23BD9 https://crypto.org/explorer/croeseid4/tx/7EEE3D10B1915ED7BE858A540E97ADA31DED7B399CA27BE129F1D10A86A23BD9
hash CC42B06437041B607842F20C242CD4E1375F52B03CB83E468F85250EB6339D5F https://crypto.org/explorer/croeseid4/tx/CC42B06437041B607842F20C242CD4E1375F52B03CB83E468F85250EB6339D5F
hash 92593D9C9545F85A4D5A96860D512DA37F45352B3C6D3FF9E33B64ADFF083BBE https://crypto.org/explorer/croeseid4/tx/92593D9C9545F85A4D5A96860D512DA37F45352B3C6D3FF9E33B64ADFF083BBE
hash 62F47051DB846657090B98CFB0F349086B6D1FF8B8A695C91E79C6195C3EAD6B https://crypto.org/explorer/croeseid4/tx/62F47051DB846657090B98CFB0F349086B6D1FF8B8A695C91E79C6195C3EAD6B
These are working quite nicely

13) But when i try to check my balances again i just get this output
$ rly q balance kichain-t-4
   665921utki
$ rly q balance testnet-croeseid-4
  538385381basetcro
Looks like i don`t recive tokens , i don`t know why , i try to fix it a lot of times , i spend half of week , but I can`t. I hope hash of my transaction can hepl you .
