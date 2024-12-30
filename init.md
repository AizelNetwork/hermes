## 1. Init and Config

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

## 3. create clients

### 3.1. create a client on wasmd tracking the state of evmos.

```shell
hermes create client --host-chain wasmd-20151225 --reference-chain evmos_9002-20151225
```

### 3.2. create a client on evmos tracking the state of wasmd.

```shell
hermes create client --host-chain evmos_9002-20151225 --reference-chain  wasmd-20151225

```