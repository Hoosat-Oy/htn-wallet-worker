# Hoosat Wallet Worker

Hoosat Wallet implements Wallet functionality for the [Hoosat Network](https://github.com/hoosatnet/hoosatd)

Hoosat Wallet is implemented in TypeScript and can be used server-side (NodeJs) and client-side (browser)

**PLEASE NOTE: This project is under heavy development**

## Components

Hoosat Wallet uses the following modules:

- [hoosat-grpc](https://github.com/aspectron/hoosat-grpc) - provides gRPC bindings for `hoosatd`
- [hoosat-grpc-node](https://github.com/aspectron/hoosat-grph-node) - gRPC transport for server-side (NodeJs)
- [hoosat-grpc-web](https://github.com/aspectron/hoosat-grpc-web) - gRPC transport for client-side (browsers)
- [hoosatcore-lib](https://github.com/aspectron/hoosatcore-lib) - Hoosat UTXO and transaction data structures

Applications built on top of Hoosat Wallet Framework:

- [hoosat-faucet](https://github.com/aspectron/hoosat-faucet) - Hoosat faucet website
- [hoosat-wallet-cli](https://github.com/aspectron/hoosat-wallet-cli) - command-line wallet
- [KDX](https://github.com/aspectron/kdx) - Hoosat desktop wallet

**PLEASE NOTE:** all Hoosat applications and libraries are under heavy development

## Hoosat Wallet Framework

Before you can use Hoosat Wallet, you need to initialize the framework. Framework initialization loads various dependencies such as `secp256k1-wasm` and `blake2b-wasm` modules use in the underlying transaction cryptography.

```js
const { Wallet, initHoosatFramework } = require('@hoosat/wallet');
const { RPC } = require('@kaspa/grpc-node');

(async () => {
  await initHoosatFramework();
  ...
})();
```

## Creating a wallet

Network types are identified by address prefixes:

- `hoosat` (Mainnet)
- `hoosattest` (Testnet)
- `hoosatdev` (Devnet)
- `hoosatsim` (Simnet)

Wallet class can be created using two static functions:

```ts
static fromMnemonic(
  seedPhrase: string,
  networkOptions: NetworkOptions,
  options: WalletOptions = {}): Wallet { }

static async import(
  password: string,
  encryptedMnemonic: string,
  networkOptions: NetworkOptions,
  options: WalletOptions = {}): Promise <Wallet> { }
```

Wallet creation functions accept following configuration objects:

```ts
export interface WalletOptions {
  skipSyncBalance?: boolean; // do not perform balance sync
  addressDiscoveryExtent?: number; // address derivation scan (default 128)
  syncOnce?: boolean; // 'sync-and-exit' (true) or 'monitoring' mode
  logLevel?: string; // wallet log level
  disableAddressDerivation?: boolean; // disable address derivation and discovery
}

export interface NetworkOptions {
  network: Network; // network: hoosat, hoosattest, hoosatdev, hoosatsim
  rpc?: IRPC; // gRPC interface (must be bound to transport before use)
}
```

Following options are important:

- `addressDiscoveryExtent` - the number of HD address derivations to scan forward from the last known used address
- `syncOnce` - allows wallet to be started temporarily, without starting monitoring services
- `disableAddressDerivation` - starts wallet in a single-address mode, where receive address and change address will always be the first receive address generated from the private key.

Creating from Mnemonic:

```js
const network = "hoosattest";
const { port } = Wallet.networkTypes[hoosattest].port; // default port for testnet
const rpc = new RPC({ clientConfig: { host: "127.0.0.1:" + port } });

Wallet.fromMnemonic("user mnemonic string", { network, rpc }, { disableAddressDerivation: true });
```

Creating new wallet instance with dynamically generated mnemonic:

```js
const wallet = new Wallet(null, null, { network, rpc });
const encryptedMnemonic = await wallet.export(cmd.password);
console.log("mnemonic:", wallet.mnemonic);
console.log("encrypted mnemonic:", encryptedMnemonic);
```

Restoring from encrypted mnemonic:

```js
const password = "user password";
const encryptedMnemonic = "previously encrypted mnemonic";
let wallet = await Wallet.import(password, encryptedMnemonic, { network, rpc });
```

## Logging and debugging

Wallet class contains an integrated logger that can be set to one of the following levels: `error`, `warn`, `info`, `verbose`, `debug`.
The default log level is `info`. You can set the log level to `verbose` to see internal wallet data processing activity.

Wallet log level can be supplied as a part of `WalletOptions` (describe above) or set at runtime as follows:

```js
wallet.setLogLevel("verbose");
```

## Synchronizing a wallet

The function `Wallet::sync(once?:boolean)` can be used to perform wallet synchronization. Wallet synchronization
will connect to `hoosatd` and scan available UTXO entries for wallet addresses, update the wallet
balance and if `once` is true, exit or if `once` is false, start wallet monitoring services.

When operating with monitoring enabled, wallet will retain connection to `hoosatd` and dynamically
update wallet UTXO entries as well as balances.

- `wallet.sync()` - starts the wallet in monitoring mode
- `wallet.sync(true)` - performs a single-time synchronization

## Sending transactions

`submitTransaction()` function can be used to create transactions on the Hoosat network:

```js
async submitTransaction(txParamsArg: TxSend): Promise < TxResp | null > {
  // ...
}
```

This function accepts `TxSend` object on input and returns a `Promise<TxResp>` object:

```ts
export interface TxSend {
  toAddr: string;
  amount: number;
  fee: number;
  changeAddrOverride?: string;
  networkFeeMax?: number;
}
```

- `toAddr` - Destination address
- `amount` - Amount of HTN in base units
- `fee` - Transaction priority fee
- `changeAddrOverride` - (optional) Allows you to supply your own address for the change transaction
- `networkFeeMax` - (optional) Allows you to set an upper bound for automatic network (data storage) fee calculation. Hoosat Wallet will automatically calculate appropriate fees and add them to the transaction based on the transaction size. This feature is disabled if the property is omitted or set to zero.

```ts
export interface TxResp {
  txid: string;
  rpctx?: string; // reserved
}
```

- `txid` - Generated transaction id

```js
try {
  let response = await this.wallet.submitTransaction({
    address, // destination address
    amount, // amount in base units
    fee, // user fees
  });
  if (!response) console.log("general error"); // if hoosatd returns null (should never occur)
  else console.log("success:", txid);
} catch (ex) {
  console.log("error:", ex.toString());
}
```

On failure, `submitTransaction()` rejects with and error indicating the reason for failure.

## Wallet balance

Wallet retains 2 types of balances:

- _available_ - balance contains HTN ready to be spent, comprised of UTXO records with block maturity blue score over 100.
- _pending_ - balance contains newly received transactions with UTXO block maturity less than 100. Upon each UTXO maturity balance is relocated from pending to available.

`Wallet::balance` is an object containing the following properties that are updated during wallet operation:

```js
wallet.balance = {
  available: 5150000000000,
  pending: 247500000000,
  total: 5397500000000,
};
```

## Wallet events

`Wallet::on(subject, (data) => { ... })` allows for event handler registration.
Similarly to NodeJs `EventEmitter` you can unregister events by supplying original
callback to `Wallet::removeEventListener(subject, handler)` as follows:

```js
const balanceHandler = (balance) => {
  console.log(balance);
};
wallet.on("balance-update", balanceHandler);
wallet.removeEventListener("balance-update", balanceHandler);
```

Following events are emitted by the Wallet class:

- `api-online` - gPRC API is online
- `api-offline` - gRPC API is offline
- `sync-start` - wallet sync started (occurs each time gRPC API connects or re-connects)
- `sync-finish` - wallet sync finished
- `ready` - wallet is ready for use (sent after sync-finish, event data contains the balance object)
- `blue-score-changed` - indicates Hoosat blue score change (new block generation)
- `utxo-change` - signaled when UTXO is added or removed from the wallet UTXO set
- `balance-update` - indicates wallet balance change (event data contains the balance object)
