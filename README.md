# Deploy your ERC20 Smart Contract to Juno

![c11](https://user-images.githubusercontent.com/79812965/131373443-5ff0d9f6-2e2a-41bd-8347-22ac4983e625.jpg)

This repo is a collection of simple contracts built with the
[cosmwasm](https://github.com/CosmWasm/cosmwasm) framework.
Smart contracts here are for only demonstration purposes, **not production ready**.
Production grade smart contracts are collected under [cw-plus](https://github.com/CosmWasm/cw-plus).

This repo's organization is relatively simple. The top-level directory is just a placeholder
and has no real code.

<br/>

## Junod Installation and setup

### Install pre-requisites

```bash
# update the local package list and install any available upgrades
sudo apt-get update && sudo apt upgrade -y

# install toolchain and ensure accurate time synchronization
sudo apt-get install make build-essential gcc git jq chrony -y
```

<br/>

### Install [Go](https://go.dev/doc/install)


<br/>

### Install [Docker](https://docs.docker.com/get-docker/)

<br/>

### Install Rust
First, [install rustup (opens new window)](https://rustup.rs/). Once installed, make sure you have the wasm32 target:

```bash
rustup default stable
cargo version
# If this is lower than 1.49.0+, update
rustup update stable

rustup target list --installed
rustup target add wasm32-unknown-unknown
```

<br/>

### Build Juno from source

In another directory

```bash
# from $HOME dir
git clone https://github.com/CosmosContracts/juno
cd juno
git fetch
git checkout <version-tag>
```
The <version-tag> will need to be set to either a [testnet chain-id](https://docs.junonetwork.io/validators/joining-the-testnets#current-testnets)

Once you're on the correct tag, you can build:

```bash
# from juno dir
make install
```

To confirm that the installation has succeeded, you can run:

```bash
junod version
```

<br/>

## Junod Local Dev Setup

### Using the Seed User

Juno ships with an unsafe seed user in dev mode when you run the prebuilt docker container below, or one of the options that uses docker-compose. You can import this user into the CLI by using the mnemonic from the Juno repo, i.e.:

```bash
junod keys add <unsafe-test-key-name> --recover
```

When prompted, add the mnemonic:

```bash
clip hire initial neck maid actor venue client foam budget lock catalog sweet steak waste crater broccoli pipe steak sister coyote moment obvious choose
```

You will then be returned an address to use:
```
juno16g2rahf5846rxzp3fwlswy08fz8ccuwk03k57y
```

## Run Juno

There is a prebuilt docker image for you to use. This will start a container with a seeded user. When you're done, you can use ctrl+c to stop the container running.

```bash
docker run -it \
  --name juno_node_1 \
  -p 1317:1317 \
  -p 26656:26656 \
  -p 26657:26657 \
  -e STAKE_TOKEN=ujunox \
  -e UNSAFE_CORS=true \
  ghcr.io/cosmoscontracts/juno:v10.0.0 \
  ./setup_and_run.sh juno16g2rahf5846rxzp3fwlswy08fz8ccuwk03k57y
```

## Quick(est) start dev build

The quickest way to get up-and-running for development purposes is to run:

```bash
STAKE_TOKEN=ujunox UNSAFE_CORS=true docker-compose up
```

This builds and runs the node and:

- Creates and initialises a validator
- Adds a default user with a known address (`juno16g2rahf5846rxzp3fwlswy08fz8ccuwk03k57y`)

<br/>
<br/>

## Development

Before starting, we create some command lines to facilitate then later on:

```bash
TXFLAG="--gas-prices 0.1ujunox --gas auto --gas-adjustment 1.3 -y -b block --chain-id testing"
BINARY="docker exec -i juno_node_1 junod"
```

<br/>

### Compile

Let's go to contracts/erc20

```bash
cd contracts/erc20
```

We can compile our contract like so:

```bash
sudo docker run --rm -v "$(pwd)":/code \
    --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
    --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
    cosmwasm/rust-optimizer:0.12.8
```

This will result in an artifact called `cw_erc20.wasm` being created in the artifacts directory.

Then copy to the docker:

```bash
docker cp artifacts/cw_erc20.wasm juno_node_1:/cw_erc20.wasm
```

<br/>

### Store

You can now upload, or 'store' this to the chain via your local node.

```bash
$BINARY tx wasm store "/cw_erc20.wasm" --from <your-key> $TXFLAG

# Without the created command line
docker exec -i juno_node_1 junod tx wasm store "/cw_erc20.wasm" --from <your-key> --gas-prices 0.1ujunox --gas auto --gas-adjustment 1.3 -y -b block --chain-id testing
```

For `<your-key>` we have already a key in Juno called `validator` but later we need a second key, so we can create a new key named `test-user` using this command:

```bash
echo "clip hire initial neck maid actor venue client foam budget lock catalog sweet steak waste crater broccoli pipe steak sister coyote moment obvious choose" | $BINARY keys add test-user --recover --keyring-backend test
```

To see the keys list the command line is:

```bash
$BINARY keys list
```

You can see:

```bash
- name: test-user
  type: local
  address: juno16g2rahf5846rxzp3fwlswy08fz8ccuwk03k57y
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"A9C5c0m1DeTjKyKsba9mvXm/QwYACC+aLs+8q7RA7YCq"}'
  mnemonic: ""
- name: validator
  type: local
  address: juno1cqlhv64wla35ct5vzgfm7h0z25rmyysjry0kxs
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"ArO9toAFYL/owhgwQh9AbcS24c72o9eGK6itQ4HpahqP"}'
  mnemonic: ""
```

Now upload, or 'store' this to the chain via your local node with the key validator.

```bash
$BINARY tx wasm store "/cw_erc20.wasm" --from validator $TXFLAG

# Without the created command line
docker exec -i juno_node_1 junod tx wasm store "/cw_erc20.wasm" --from validator --gas-prices 0.1ujunox --gas auto --gas-adjustment 1.3 -y -b block --chain-id testing
```

You will need to look in the output for this command for the code ID of the contract. In the JSON, it will look like `{"key":"code_id","value":"6"}` in the output.

Alternatively, you can capture the output of the command run above, by doing these steps instead, and use the jq tool installed earlier to get the `code_id` value:

```bash
TX=$($BINARY tx wasm store "/cw_erc20.wasm"  --from validator $TXFLAG --output  json | jq -r '.txhash')
CODE_ID=$($BINARY query tx $TX --output json | jq -r '.logs[0].events[-1].attributes[0].value')
```

You can now see this value with:

```bash
echo $CODE_ID
```

<br/>

## Initialise the Contract

Now we've uploaded the contract, now we need to initialise it.

We're using the Poodle Coin example here - $POOD was the first meme coin deployed to a Juno testnet.

Choose another name rather than Poodle Coin/POOD, as this is likely already taken on the testnet.

### Instantiate the contract

Note that if you use rich types like CosmWasm's *`Uint128`* then they will be strings from the point of view of JSONSchema. If you have an int, you do not need quotes, e.g. 1 - but for a *`Uint128`* you will need them, e.g. `"1"`.

Note also that the --amount is used to initialise the new account associated with the contract.

```bash
$BINARY tx wasm instantiate $CODE_ID \
    '{"name":"Poodle Coin","symbol":"POOD","decimals":6,"initial_balances":[{"address":"<validator-self-delegate-address>","amount":"12345678000"}]}' \
    --amount 50000ujunox  --label "Poodlecoin erc20" --from <your-key> $TXFLAG --no-admin

# In our case
$BINARY tx wasm instantiate $CODE_ID \
    '{"name":"Poodle Coin","symbol":"POOD","decimals":6,"initial_balances":[{"address":"juno1cqlhv64wla35ct5vzgfm7h0z25rmyysjry0kxs","amount":"12345678000"}]}' \
    --amount 50000ujunox  --label "Poodlecoin erc20" --from validator $TXFLAG --no-admin
```

If this succeeds, look in the output and get contract address from output e.g `juno1a2b....` or run:

```bash
CONTRACT_ADDR=$($BINARY query wasm list-contract-by-code $CODE_ID --output json | jq -r '.contracts[0]')
```

This will allow you to query using the value of `$CONTRACT_ADDR`

```bash
$BINARY query wasm contract $CONTRACT_ADDR
```

<br/>

## Query and run commands

Now you can check that the contract has assigned the right amount to the self-delegate address:

```bash
$BINARY query wasm contract-state smart $CONTRACT_ADDR '{"balance":{"address":"juno1cqlhv64wla35ct5vzgfm7h0z25rmyysjry0kxs"}}' --chain-id testing --output json
```

From the example above, it will return:

```bash
{"data":{"balance":"12345678000"}}
```

Using the commands supported by `execute` work the same way. The incantation for executing commands on a contract via the CLI is:


```bash
$BINARY tx wasm execute [contract_addr_bech32] [json_encoded_send_args] --amount [coins,optional] [flags]
```

You can omit `--amount` if not needed for `execute` calls.

In this case ( this is where `test-user` address `juno16g2rahf5846rxzp3fwlswy08fz8ccuwk03k57y` come in ), your command will look something like:

```bash
$BINARY tx wasm execute $CONTRACT_ADDR '{"transfer":{"amount":"200","owner":"juno1cqlhv64wla35ct5vzgfm7h0z25rmyysjry0kxs","recipient":"juno16g2rahf5846rxzp3fwlswy08fz8ccuwk03k57y"}}' --from juno1cqlhv64wla35ct5vzgfm7h0z25rmyysjry0kxs $TXFLAG
```

Now if you use the command:

```bash
$BINARY query wasm contract-state smart $CONTRACT_ADDR '{"balance":{"address":"juno16g2rahf5846rxzp3fwlswy08fz8ccuwk03k57y"}}' --chain-id testing --output json
```

You can see the result is:

```bash
{"data":{"balance":"200"}}
```

Which means that Validator ( `juno1cqlhv64wla35ct5vzgfm7h0z25rmyysjry0kxs` ) transfer Poodle Coin to test-user ( `juno16g2rahf5846rxzp3fwlswy08fz8ccuwk03k57y` ) which means the `transfer` command was successful.
