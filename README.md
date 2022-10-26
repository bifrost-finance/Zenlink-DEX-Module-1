# Integration

##  Supported dependencies

It should at least work until the following commits

-   rustc 1.62.0-nightly (ecd44958e 2022-05-10)
-   Polkadot release-v0.9.29
-   Cumulus polkadot-v0.9.29 
-   Substrate polkadot-v0.9.29 


## Precondition
* Implement MultiAssetsHandler with `Native Currency`, 
`Local assets` and `Other assets` which defined on your parachain.
* Establish HRMP channels between zenlink registered parachains.

## Zenlink Protocol v0.4.0 Overview
![zenlink protocol](./images/zenlink-protocol-v0.4.0.png)

`Zenlink Protocol` mainly consists of two parts: assets and asset operations, 
namely `Zenlink Assets` and `Zenlink Actions`.
Through the `UAI`(unified asset identifier) and `MultiAssetsHandler` interfaces, 
`Zenlink` can perform `Swap` and `Transfer By XCM` any asset on the chain.

## Zenlink Protocol v0.4.0 Features
- Based on the `newest` XCM design
- Based on `FRAMEv2`
- `Compatible` Polkadot XCMP cross-chain asset processing
- Zenlink Protocol registration whitelist-XCM `trust collection management`
- Through Transfer-By-XCM, `assets can flow freely` between parachains
- ZenlinkMultiAssets adapter comes with `LIQUIDITY` and `Foreign` asset processing, which can `adapt to various native assets such as NATIVE/LOCAL/RESERVE`
- Provide LocalAssetHandler, OtherAssetHandler and other traits to `support locally defined assets such as LOCAL and RESERVE`
- Under the control of MultiAssetsHandler, `assets can swap freely`

##  UAI v0.1.0

- UAI v0.1.0 definition

```rust
pub struct AssetId {
    pub chain_id: u32, // ParaId
    pub asset_type: u8, // Asset types supported by Zenlink Assets
    pub asset_index: u32, // the index number of the asset
}
```

- UAI v0.1.0 asset type

`asset_type` currently only supports the following values:

```rust
/// Native currency, the most basic asset on each parachain
pub const NATIVE: u8 = 0;
/// Swap module asset, LP token generated by Zenlink DEX
pub const LIQUIDITY: u8 = 1;
/// Other asset type on this chain, other asset modules on this chain
pub const LOCAL: u8 = 2;
/// Reserved for future, reserved asset type
pub const RESERVED: u8 = 3;
```

In the current version of UAI, `Zenlink Assets` supports up to four locally defined assets of `Native/LIQUIDITY/LOCAL/RESERVED`.
When these four assets are transferred across chains, a cross-chain mapping asset category `Foreign` will be generated inside `ZenlinkProtocol`.

- `Before and after the cross-chain asset transfer, the UAI of the asset has not changed`.

## MultiAssetsHandler

The trait is defined as follows
```rust
pub trait MultiAssetsHandler<AccountId> {
    fn balance_of(asset_id: AssetId, who: &AccountId) -> AssetBalance;
    fn total_supply(asset_id: AssetId) -> AssetBalance;
    fn is_exists(asset_id: AssetId) -> bool;
    fn transfer(
        asset_id: AssetId,
        origin: &AccountId,
        target: &AccountId,
        amount: AssetBalance,
    ) -> DispatchResult {
        let withdrawn = Self::withdraw(asset_id, origin, amount)?;
        let _ = Self::deposit(asset_id, target, withdrawn)?;
        Ok(())
    }
    fn deposit(
        asset_id: AssetId,
        target: &AccountId,
        amount: AssetBalance,
    ) -> Result<AssetBalance, DispatchError>;

    fn withdraw(
        asset_id: AssetId,
        origin: &AccountId,
        amount: AssetBalance,
    ) -> Result<AssetBalance, DispatchError>;
}

```

- `is_exists` is a unified access interface for `Zenlink Actions`
- `balance_of`, `total_supply`, `transfer` are `Zenlink Swap Action` access interface
- `deposit`, `withdraw` are `Zenlink Transfer-By-XCM Action` access interface
- `MultiAssetsHandler` can operate in addition to the four native defined assets identified by `UAI`, 
as well as the `cross-chain mapping asset Foreign` inside `ZenlinkProtocol`.


## Swap and Transfer-By-XCM
- Four native defined assets (`NATIVE/LIQUIDITY/LOCAL/RESERVED`) and cross-chain mapping assets (`Foreign`) can be exchanged through `MultiAssetsHandler`.
- Four native defined assets (`NATIVE/LIQUIDITY/LOCAL/RESERVED`) Cross-chain asset transfer through `Transfer-By-XCM`, and a mapped asset `Foreign` is generated 
on the target parachain, Also through the `Transfer-By-XCM` cross-chain transfer back to the original definition chain of the asset.

## Example

[see runtime example](./example)

### Compile Relaychain
```
git clone https://github.com/paritytech/polkadot.git
git checkout release-v0.9.1
cd polkadot & cargo build --release 
```
### Run testnet
In this case, the test network consists of 5 Relaychain nodes and 2 Parachain Collectors. The Parachain IDs are 200 and 300, respectively.
Execute the following shell script to generate the required files, including the Relaychain configuration file, the Parachain Genesis file, and the WASM file.

```
#!/bin/bash

# polkadot
cp ../polkadot/target/release/polkadot .
mkdir config

./polkadot build-spec --chain rococo-local --disable-default-bootnode > ./config/rococo-custom-local.json
sed -i 's/"validation_upgrade_frequency": 600/"validation_upgrade_frequency": 10/g' ./config/rococo-custom-local.json
sed -I ’s/“validation_upgrade_delay”: 300/“validation_upgrade_delay”: 5/g’ ./config/rococo-custom-local.json
./polkadot build-spec —chain ./config/rococo-custom-local.json —disable-default-bootnode —raw > ./config/rococo-local.json


# dev-parachain
cp ../dev-parachain/target/release/dev-parachain .

./dev-parachain build-spec —disable-default-bootnode > ./config/200para.json
sed -I ’s/“para_id”: 500,/“para_id”: 200,/g’ ./config/200para.json
sed -I ’s/“parachainId”: 500/“parachainId”: 200/g’ ./config/200para.json
./dev-parachain export-genesis-state —parachain-id 200 > ./config/200.genesis

./dev-parachain build-spec —disable-default-bootnode > ./config/300para.json
sed -I ’s/“para_id”: 500,/“para_id”: 300,/g’ ./config/300para.json
sed -I ’s/“parachainId”: 500/“parachainId”: 300/g’ ./config/300para.json
./dev-parachain export-genesis-state —parachain-id 300 > ./config/300.genesis

./dev-parachain export-genesis-wasm > ./config/200-300.wasm
```

Execute the following shell script to start the testnet.

```
#!/bin/bash

cp config/rococo-custom-local.json config/rococo-custom-local-5.json
sed ‘/palletSession/,/palletGrandpa/{/palletSession/!{/palletGrandpa/!d}}’ -I config/rococo-custom-local-5.json
sed ‘/palletSession/ r palletSession.txt’ -I config/rococo-custom-local-5.json

./polkadot build-spec —chain config/rococo-custom-local-5.json —disable-default-bootnode —raw > config/rococo-local-5.json

mkdir data

./polkadot  —ws-port 9944 —rpc-port 10025 —port 30033  -d ./data/data-a —validator —chain config/rococo-local-5.json —Alice   > ./data/log.a 2>&1 &
./polkadot  —ws-port 9946 —rpc-port 10026 —port 30034  -d ./data/data-b —validator —chain config/rococo-local-5.json —bob     > ./data/log.b 2>&1 &
./polkadot  —ws-port 9947 —rpc-port 10027 —port 30035  -d ./data/data-c —validator —chain config/rococo-local-5.json —Charlie > ./data/log.c 2>&1 &
./polkadot  —ws-port 9948 —rpc-port 10028 —port 30036  -d ./data/data-d —validator —chain config/rococo-local-5.json —Dave    > ./data/log.d 2>&1 &
./polkadot  —ws-port 9949 —rpc-port 10029 —port 30037  -d ./data/data-e —validator —chain config/rococo-local-5.json —eve     > ./data/log.e 2>&1 &


./dev-parachain -d ./data/data-200  —ws-port 9977 —port 30336 —rpc-port 10002 —rpc-cors all —unsafe-ws-external  —unsafe-rpc-external  —parachain-id 200  —validator —execution=native -lruntime=debug,cumulus-collator=trace,cumulus-network=trace,cumulus-consensus=trace —chain ./config/200para.json  —   —chain ./config/rococo-local-5.json  —discover-local —execution=wasm > ./data/log.200 2>&1 &

./dev-parachain -d ./data/data-300  —ws-port 9988 —port 30337 —rpc-port 10003 —rpc-cors all —unsafe-ws-external  —unsafe-rpc-external  —parachain-id 300  —validator —execution=native -lruntime=debug,cumulus-collator=trace,cumulus-network=trace,cumulus-consensus=trace —chain ./config/300para.json  —   —chain ./config/rococo-local-5.json  —discover-local —execution=wasm > ./data/log.300 2>&1 &
```

### Register Parachain & Establish HRMP channel
Connect Relaychain with endpoint 'ws://127.0.0.1:9944' on https://polkadot.js.org/apps
- Register Parachain200

![Register Parachain200](./images/register_parachain200.png)

- Register Parachain300

![Register Parachain300](./images/register_parachain300.png)

- Establish HRMP channel, Parachain200 -> Parachain300

![Establish HRMP 200->300](./images/establish_200_300.png)

- Establish HRMP channel, Parachain300 -> Parachain200

![Establish HRMP 300->200](./images/establish_300_200.png)

Transaction type

```
{
   "AccountData": {
      "free": "Balance",
      "reserved": "Balance",
      "frozen": "Balance"
   },
  "Currency": "CurrencyIdOf",
  "CurrencyIdOf": "CurrencyId",
  "AmountOf": "Balance",
  "Amount": "AmountOf",
  "TransferOriginType": {
    "_enum": {
      "FromSelf": 0,
      "FromRelayChain": 1,
      "FromSiblingParaChain": 2
    }
  },
  "TokenSymbol": {
    "_enum": {
      "ASG": 0,
      "BNC": 1,
      "KUSD": 2,
      "DOT": 3,
      "KSM": 4,
      "ETH": 5
    }
  },
  "CurrencyId": {
    "_enum": {
      "Native": "TokenSymbol",
      "VToken": "TokenSymbol",
      "Token": "TokenSymbol",
      "Stable": "TokenSymbol",
      "VSToken": "TokenSymbol",
      "VSBond": "(TokenSymbol, ParaId, LeasePeriod, LeasePeriod)",
      "LPToken": "(TokenSymbol, u8, TokenSymbol, u8)"
    }
  },
  "TAssetBalance": "Balance",
  "PalletBalanceOf": "Balance",
  "BlockNumberFor": "BlockNumber",
  "NumberOrHex": {
    "_enum": {
      "Number": "u64",
      "Hex": "U256"
    }
  },
  "IsExtended": "bool",
  "SystemPalletId": "PalletId",
  "RewardRecord": {
    "account_id": "AccountId",
    "record_amount": "Balance"
  },
  "MaxLocksOf": "u32",
  "VestingInfo": {
    "locked": "Balance",
    "per_block": "Balance",
    "starting_block": "BlockNumber"
  },
  "OrderId": "u64",
  "OrderInfo": {
    "owner": "AccountIdOf",
    "currency_sold": "CurrencyIdOf",
    "amount_sold": "BalanceOf",
    "currency_expected": "CurrencyIdOf",
    "amount_expected": "BalanceOf",
    "order_id": "OrderId",
    "order_state": "OrderState"
  },
  "OrderState": {
    "_enum": [
      "InTrade",
      "Revoked",
      "Clinchd"
    ]
  },
  "AssetId": {
    "chain_id": "u32",
    "asset_type": "u8",
    "asset_index": "u64"
  },
  "ZenlinkAssetBalance": "u128",
  "PairInfo": {
    "asset0": "AssetId",
    "asset1": "AssetId",
    "account": "AccountId",
    "totalLiquidity": "ZenlinkAssetBalance",
    "holdingLiquidity": "ZenlinkAssetBalance",
    "reserve0": "ZenlinkAssetBalance",
    "reserve1": "ZenlinkAssetBalance",
    "lpAssetId": "AssetId"
  },
  "PairMetadata": {
    "pair_account": "AccountId",
    "target_supply": "AssetBalance"
  },
  "AssetBalance": "u128",
  "BootstrapParamter": {
    "min_contribution": "(AssetBalance, AssetBalance)",
    "target_supply": "(AssetBalance, AssetBalance)",
    "accumulated_supply": "(AssetBalance, AssetBalance)",
    "end_block_number": "BlockNumber",
    "pair_account": "AccountId"
  },
  "PairStatus": {
    "_enum": {
      "Trading": "PairMetadata",
      "Bootstrap": "BootstrapParamter",
      "Disable": null
    }
  }
}`
```

Switch to 'Settings' -> 'Developer', input the above json, then save.

![Setting Type](./images/setting.png)

### Cross chain transfer
   In this example, the Balances module index equal 2 and it just has one asset.
   So the AssetId of native currency is (200,2,0). 
   It means the first asset of the second module on parachain 200.
   Now we transfer it to parachain 300.

   ![Cross_transfer](./images/cross_chain_transfer.png)
   We can see the event in parachain 300.
   It means that the zenlink protocol first issued an asset, and then minted 1,000,000.

   ![Cross_transfer](./images/cross_transfer_event.png)

   We can see the result in zenlink protocol storage.

   ![Cross_transfer](./images/cross_transfer_result.png)

### Create Pair
   Now we have two assets in parachain 300. 
   So we can create a pair.
   ![Create_Pair](./images/create_pair.png)

### Add liquidity
   ![Create_Pair](./images/add_liquidity.png)

### Swap
   ![Swap](./images/swap.png)
