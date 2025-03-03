## 1. Init and Config

### 1.0. Configure Environment Variables

Before you begin, you must set some key environment variables. In particular, configure your home directory for wasmd (the location where node configuration and data are stored). For example, add the following lines to your shell profile (e.g. `~/.bash_profile`, `~/.bashrc`, or `~/.zshrc`):

```bash
# Base directory for Aizel node configurations
export HERMESHOME=$HOME/.cosmos/hermes

```

Then reload your shell configuration:

```bash
source ~/.bash_profile
```

### 1.1. create hermes binary alias
Add the following content to ~/.zshrc

```shell
hermes () {
	$HOME/data/project/hermes/target/release/hermes --config "${HERMESHOME}/config.toml" "$@"
}
```

Validate config file 

```shell
hermes config validate
```

### 1.2. config hermes under path HERMESHOME

The config file will looks like that :

```toml
[[chains]]
id = "evmos_9002-20151225"
rpc_addr = "http://localhost:26657"
grpc_addr = "http://localhost:9090"
event_source = { mode = 'push', url = 'ws://localhost:26657/websocket', batch_delay = '200ms' }
gas_price = { price = 0.001, denom = 'aevmos' }
account_prefix = 'evmos'
key_name = 'wallet'
store_prefix = 'ibc'

[[chains]]
id = "wasmd-20151225"
rpc_addr = "http://localhost:36657"
grpc_addr = "http://localhost:10090"
event_source = { mode = 'push', url = 'ws://localhost:36657/websocket', batch_delay = '200ms' }
gas_price = { price = 0.001, denom = 'stake' }
account_prefix = 'wasm'
key_name = 'wallet'
store_prefix = 'ibc'
```

## 2. add keys to a chain

### 2.0 export evmosd key file 

```shell
evmosd keys export validator1 \
  --unsafe \
  --unarmored-hex \
  --keyring-backend file \
  --home $EVMOSHOME/node1 \
  --output $HERMESHOME/wallet.json

```

### 2.1. add keys to chains from mnemonic

```shell

hermes keys add --key-name <KEY_NAME> --chain <CHAIN_ID> --mnemonic-file <MNEMONIC_FILE>

hermes keys add --hd-path "m/44'/60'/0'/0/0" --key-name wallet --chain evmos_9002-20151225 --mnemonic-file $HERMESHOME/alice.key --overwrite

hermes keys add --key-name wallet --chain wasmd-20151225 --mnemonic-file $HERMESHOME/alice.key --overwrite

```
The keys location is $HOME/.hermes/keys

## 3. Create the relay path

### 3.1. create clients

#### 3.1.1. create a client on wasmd tracking the state of evmos.

```shell
hermes create client --host-chain wasmd-20151225 --reference-chain evmos_9002-20151225
```

#### 3.1.2. create a client on evmos tracking the state of wasmd.

```shell
hermes create client --host-chain evmos_9002-20151225 --reference-chain  wasmd-20151225

```

### 3.2. create connections

After creating clients on both chains, you have to establish a connection between them. 
Both chains will assign connection-0 as the identifier of their first connection:

```shell
hermes create connection --a-chain evmos_9002-20151225 --a-client 07-tendermint-0 --b-client 07-tendermint-0

```

### 3.3. channel identifiers

Finally, after the connection has been established, you can now open a new channel on top of it. 
Both chains will assign channel-0 as the identifier of their first channel:

```shell
hermes create channel --a-chain evmos_9002-20151225 --a-connection connection-0 --a-port transfer --b-port transfer

```

### 3.4. Visualize the current network

You can visualize the topology of the current network with:

```shell
hermes query channels --show-counterparty --chain evmos_9002-20151225

```
## 4. Start relaying

In the previous section, you created clients, established a connection between them, and opened a channel on top of it. 
Now you can start relaying on this path.

### 4.1. Query balances

Use the following commands to query balances on your local chains:

* Balances on evmos_9002-20151225:

```shell
evmosd --home $EVMOSHOME/node1 query bank balances $(evmosd --home $EVMOSHOME/node1 keys --keyring-backend="file" show validator1 -a) --node tcp://localhost:26657

```

* Balances on wasmd-20151225:

```shell
wasmd --home $WASMHOME/node1 query bank balances $(wasmd --home $WASMHOME/node1 keys --keyring-backend="file" show alice -a) --node tcp://localhost:36657

```

### 4.2. Exchange packets
Now, let's exchange samoleans between two chains.

* Open a new terminal and start Hermes using the start command :
```shell
hermes start

```
Hermes will first relay the pending packets that have not been relayed and then start passively relaying by listening for and acting on packet events.

* In a separate terminal, use the ft-transfer command to send 100000 aevmos from evmosd to wasmd over channel-0:
```shell
hermes tx ft-transfer \
  --timeout-seconds 1000 \
  --dst-chain wasmd-20151225 \
  --src-chain evmos_9002-20151225 \
  --src-port transfer \
  --src-channel channel-0 \
  --denom aevmos \
  --amount 100000

```

* Wait a few seconds, then query balances on ibc-1 and ibc-0. You should observe something similar to:

Balances at evmos_9002-20151225:
```shell
balances:
- amount: "99998999999799999999698300"
  denom: aevmos
pagination:
  total: "1"

```
Balances at wasmd-20151225:
```shell
balances:
- amount: "300000"
  denom: ibc/8EAC8061F4499F03D2D1419A3E73D346289AE9DB89CAB1486B72539572B1915E
- amount: "99998999999799999999960878"
  denom: stake
pagination:
  total: "2"
```
The samoleans were transferred to wasmd-20151225 and are visible under the denomination ibc/8EAC8.... 

* Transfer back these tokens to evmos_9002-20151225:
```shell
hermes tx ft-transfer \
  --timeout-seconds 1000 \
  --dst-chain evmos_9002-20151225 \
  --src-chain wasmd-20151225 \
  --src-port transfer \
  --src-channel channel-0 \
  --denom ibc/8EAC8061F4499F03D2D1419A3E73D346289AE9DB89CAB1486B72539572B1915E \
  --amount 300000

```
