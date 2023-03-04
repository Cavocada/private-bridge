# Coinbase Goerli <=> Findora Forge Deploy

## Prerequisites
- Go 1.15+ installation or later
- nodejs 14.17.x
- npm 6.14.x
- Docker
- docker-compose

## Tooling
### cb-sol-cli
We will be using the ChainBridge contract CLI to deploy and interact with the contracts. Grab and install the CLI by running:
```
git clone -b v1.0.2 --depth 1 https://github.com/Cavocada/chainbridge-tools \
&& cd chainbridge-deploy/cb-sol-cli \
&& npm install \
&& make install \
&& cd ../cfgBuilder \
&& make install
```

### chainbridge
The ChainBridge binary will be required for inserting relay keys into the keystore. It can be built from here:
[chainbridge](https://github.com/ChainSafe/chainbridge-deploy/tree/main/cfgBuilder).

### architecture and roles
Client: User initiates cross-chain transfer — The beginning of a transaction

Relayers: The message routing system to pass information from the source chain to the destination chain

Destination Chain — The destination chain of the asset to be moved to

<img width="970" alt="architecture" src="https://user-images.githubusercontent.com/109700451/222875352-a741e830-2445-4ce6-953b-0c270f12384d.png">


## Getting started

### Generate ethereum account key-pairs
You can generate it in a way you think is safe, and then save the private key and mnemonic by yourself.
Testing accounts are provided in the `wallet`.

### Set chainbridge vars
To avoid duplication in the subsequent commands set the following env vars in your shell.
```
SRC_GATEWAY=https://prod-forge.prod.findora.org:8545/
DST_GATEWAY=https://goerli.base.org:8545/

SRC_ADDR="<relayers public key on Findora Forge>"
SRC_PK="<deployer private key on Findora Forge>"
DST_ADDR="<relayers public key on Base Goerli>"
DST_PK="<deployer private key on Base Goerli>"

# the zkUSDT token
SRC_TOKEN="0x5b15Cdff7Fe65161C377eDeDc34A4E4E31ffb00B"
```
You could also write the above to a file (e.g. `vars-base-gorlie.sh`) and load it into your shell by running `source vars-base-gorlie.sh`.


#### Steps
1. Deploy contracts on Findora Forge testnet
The following command will deploy the bridge contract and ERC20 handler contract in Forge testnet.

*Note: Findora network min gas price is 100 Gwei*
```
cb-sol-cli --url $SRC_GATEWAY --privateKey $SRC_PK --gasPrice 10000000000 deploy \
    --bridge --erc20Handler \
    --relayers $SRC_ADDR \
    --relayerThreshold 1 \
    --expiry 100 \
    --chainId 0
```
output:
```
Deploying contracts...
✓ Bridge contract deployed
✓ ERC20Handler contract deployed

================================================================
Url:        https://prod-forge.prod.findora.org:8545/
Deployer:   0x3E5D560d2Ab3006fBDA617663B28ABFc0382086e
Gas Limit:   8000000
Gas Price:   10000000000
Deploy Cost: 0.05879326

Options
=======
Chain Id:    0
Threshold:   1
Relayers:    0x169504A3F5Ea27252b371ff4Db7C5a33dfF2FD16,0xE2E2a99Dbc0E004D013d934c2756cb6C90B00cF2
Bridge Fee:  0
Expiry:      100

Contract Addresses
================================================================
Bridge:             0xD520c4eaEe94AEa1C74b2bdfD0A60F7F1662EFEc
----------------------------------------------------------------
Erc20 Handler:      0x656DF8FA08e32563184C8f1f0a3fF5808C4F1BBb
================================================================
```
Take note of the output of the above command and assign the following variables to `vars-base-gorlie.sh`. Run `source vars-base-gorlie.sh` to update environment variables. To confirm the updated environment variables, run `./show-vars.sh`
```
SRC_BRIDGE="0xD520c4eaEe94AEa1C74b2bdfD0A60F7F1662EFEc"
SRC_HANDLER="0x656DF8FA08e32563184C8f1f0a3fF5808C4F1BBb"
```

2. Configure contracts in Findora Forge
The following registers the `zkUSDT` token as a resource with a bridge contract and configures which handler to use.
```
cb-sol-cli --url $SRC_GATEWAY --privateKey $SRC_PK --gasPrice 10000000000 bridge register-resource \
    --bridge $SRC_BRIDGE \
    --handler $SRC_HANDLER \
    --resourceId $RESOURCE_ID_ZK_USDT \
    --targetContract $SRC_TOKEN
```
output:
```
[bridge/register-resource] Registering contract 0x5b15Cdff7Fe65161C377eDeDc34A4E4E31ffb00B with resource ID 0x000000000000000000000000000000a16ebe4a02bbccc786776a9388ff000000 on handler 0x656DF8FA08e32563184C8f1f0a3fF5808C4F1BBb
Waiting for tx: 0xc098f9ae85ff063bb4a5ba366d1064da0718a7c97ee7b368a44d905daaa291b1...
```

3. Deploy contracts on Base Gorlie testnet

The following command deploys the bridge contract, handler and a new ERC20 contract (`zkUSDT.f`) on the destination chain.
```
cb-sol-cli --url $DST_GATEWAY --privateKey $DST_PK --gasPrice 2000000000 deploy \
    --erc20Decimals 6 \
    --erc20Name "Zero Knowledge USDT" \
    --erc20Symbol zkUSDT.f \
    --bridge --erc20 --erc20Handler \
    --relayers $DST_ADDR \
    --relayerThreshold 1 \
    --expiry 800 \
    --chainId 1
```
output:
```
Deploying contracts...
✓ Bridge contract deployed
✓ ERC20Handler contract deployed
✓ ERC20 contract deployed

================================================================
Url:        https://goerli.base.org
Deployer:   0x3E5D560d2Ab3006fBDA617663B28ABFc0382086e
Gas Limit:   8000000
Gas Price:   2000000000
Deploy Cost: 0.022394793147517584

Options
=======
Chain Id:    1
Threshold:   1
Relayers:    0x169504A3F5Ea27252b371ff4Db7C5a33dfF2FD16,0xE2E2a99Dbc0E004D013d934c2756cb6C90B00cF2
Bridge Fee:  0
Expiry:      800

Contract Addresses
================================================================
Bridge:             0xD520c4eaEe94AEa1C74b2bdfD0A60F7F1662EFEc
----------------------------------------------------------------
Erc20 Handler:      0x656DF8FA08e32563184C8f1f0a3fF5808C4F1BBb
----------------------------------------------------------------
Erc20:              0x74e918F18b1260728d92A2606a46521D7Db490d0
================================================================
```
Again, assign the following env variables to `vars-base-gorlie.sh` and run
```
source vars-base-gorlie.sh
```
to update environment variables. To confirm the updated environment variables, run `./show-vars.sh`.
```
DST_BRIDGE="0xD520c4eaEe94AEa1C74b2bdfD0A60F7F1662EFEc"
DST_HANDLER="0x656DF8FA08e32563184C8f1f0a3fF5808C4F1BBb"
DST_TOKEN="0x74e918F18b1260728d92A2606a46521D7Db490d0"
```

Final vars-bsc-testnet.sh
```
SRC_GATEWAY=https://prod-forge.prod.findora.org:8545/
DST_GATEWAY=https://goerli.base.org

SRC_ADDR="0x169504A3F5Ea27252b371ff4Db7C5a33dfF2FD16","0xE2E2a99Dbc0E004D013d934c2756cb6C90B00cF2"
DST_ADDR="0x169504A3F5Ea27252b371ff4Db7C5a33dfF2FD16","0xE2E2a99Dbc0E004D013d934c2756cb6C90B00cF2"

SRC_BRIDGE="0xD520c4eaEe94AEa1C74b2bdfD0A60F7F1662EFEc"
SRC_HANDLER="0x656DF8FA08e32563184C8f1f0a3fF5808C4F1BBb"
SRC_TOKEN="0x5b15Cdff7Fe65161C377eDeDc34A4E4E31ffb00B"

DST_BRIDGE="0xD520c4eaEe94AEa1C74b2bdfD0A60F7F1662EFEc"
DST_HANDLER="0x656DF8FA08e32563184C8f1f0a3fF5808C4F1BBb"
DST_TOKEN="0x74e918F18b1260728d92A2606a46521D7Db490d0"
```

4. Configure contracts on Base Gorlie testnet
The following registers the new token (`zkUSDT.f`) as a resource on the bridge similar to the above.
```
cb-sol-cli --url $DST_GATEWAY --privateKey $DST_PK --gasPrice 2000000000 bridge register-resource \
    --bridge $DST_BRIDGE \
    --handler $DST_HANDLER \
    --resourceId $RESOURCE_ID_ZK_USDT \
    --targetContract $DST_TOKEN
```
The following registers the token as burnable on the bridge.
```
cb-sol-cli --url $DST_GATEWAY --privateKey $DST_PK --gasPrice 2000000000 bridge set-burn \
    --bridge $DST_BRIDGE \
    --handler $DST_HANDLER \
    --tokenContract $DST_TOKEN
```
The following gives permission for the handler to mint new `zkUSDT.f` (ERC-20) tokens.
```
cb-sol-cli --url $DST_GATEWAY --privateKey $DST_PK --gasPrice 2000000000 erc20 add-minter \
    --minter $DST_HANDLER \
    --erc20Address $DST_TOKEN
```

## Set up keys
The relayer maintains its own keystore. Run following command to import relayer's key and enter a password when prompted.
Make sure to run this command for each relayer.
```
chainbridge accounts import --privateKey e93d230575cf6d080b6ed2ae34f27f6f6751b0b08dd34d3083ae37637ffa7bda
```

## Lets test the bridge!
### Start relayers
Before starting your relayers, make sure your docker-compose file is updated with the correct key files along with their passwords.
You will also need to allocate a volume for the blockstore in order to reduce memory usage in the container.

For Ex. 

```
environment:
      - KEYSTORE_PASSWORD=123
volumes:
      - ./keys/0x169504A3F5Ea27252b371ff4Db7C5a33dfF2FD16.key:/keys/0x169504A3F5Ea27252b371ff4Db7C5a33dfF2FD16.key
      - ./config/config0.json:/config/config.json
      - ./blockstore/relayer0:/root/.chainbridge
```

Once you've set up your docker-compose file, run the following command to start your relayers. 

```
docker-compose up -d 
```

### Deposit token
#### Findora Forge => Base Gorlie

##### Privacy routing comes first
TODO in UI

##### Approve zkUSDT token
Approve the handler to spend tokens on our behalf (to transfer them to the token safe).

```
cb-sol-cli --url $SRC_GATEWAY --privateKey $USER_PK --gasPrice 100000000000 erc20 approve \
    --amount 1 \
    --erc20Address $SRC_TOKEN \
    --recipient $SRC_HANDLER
```

##### Execute a deposit.
```
cb-sol-cli --url $SRC_GATEWAY --privateKey $USER_PK --gasPrice 100000000000 erc20 deposit \
    --amount 1 \
    --dest 1 \
    --bridge $SRC_BRIDGE \
    --recipient $USER_ADDR \
    --resourceId $RESOURCE_ID_ZK_USDT
```
The relayer will wait 1 block confirmations before submitting a request on Base Gorlie testnet.

#### Base Gorlie => Findora Forge

##### Approve zkUSDT.f token
Approve the handler on the destination chain to move tokens on our behalf (to burn them).
```
cb-sol-cli --url $DST_GATEWAY --privateKey $USER_PK --gasPrice 10000000000 erc20 approve \
    --amount 1 \
    --erc20Address $DST_TOKEN \
    --recipient $DST_HANDLER
```

##### Execute a deposit.
Transfer the wrapped tokens back to the bridge. This should result in the locked tokens being freed on the source chain and returned to your account.
```
cb-sol-cli --url $DST_GATEWAY --privateKey $USER_PK --gasPrice 10000000000 erc20 deposit \
    --amount 1 \
    --dest 0 \
    --bridge $DST_BRIDGE \
    --recipient $USER_ADDR \
    --resourceId $RESOURCE_ID_ZK_USDT
```
  
##### Privacy routing comes last
TODO in UI
