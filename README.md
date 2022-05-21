# Orion Protocol SDK

[![npm version](https://img.shields.io/npm/v/@orionprotocol/sdk.svg)](https://www.npmjs.com/package/@orionprotocol/sdk)
[![Downloads](https://img.shields.io/npm/dm/@orionprotocol/sdk.svg)](https://www.npmjs.com/package/@orionprotocol/sdk)

# Install

```console
npm i @orionprotocol/sdk
```

# Usage

## High level methods

### Wallet and chain initialization

```ts
// Node.js
import "dotenv/config";
import { OrionUnit } from "@orionprotocol/sdk";
import { Wallet } from "ethers";

const chain = process.env.CHAIN; // "56" or "bsc"
const env = process.env.ENV; // production
const privateKey = process.env.PRIVATE_KEY; // 0x...

const wallet = new Wallet(privateKey);
// OrionUnit is chain-in-environment abstraction
//Create an instance of OrionUnit, which has access to terminal functions.
const orionUnit = new OrionUnit(chain, env);
```

```ts
//If using Metamask UI

import { OrionUnit } from "@orionprotocol/sdk";
import detectEthereumProvider from "@metamask/detect-provider";
import { BaseProvider } from "@metamask/providers";
import { providers } from "ethers";

const chain = "97"; // or "bsc" (in testing environment point to bsc-testnet)
const env = "testing";

const startApp = async (provider: BaseProvider) => {
  const web3Provider = new providers.Web3Provider(provider);
  await web3Provider.ready;
  const signer = web3Provider.getSigner(); // ready to go
  const orionUnit = new OrionUnit(chain, env); // ready to go
};

detectEthereumProvider().then((provider) => {
  if (provider) {
    startApp(provider as BaseProvider);
  } else {
    console.log("Please install MetaMask!");
  }
});
```

#### Withdraw

```ts
orionUnit.exchange.withdraw({
  amount: 435.275,
  asset: "USDT",
  signer: wallet, // or 'signer' when using Metamask module.
});
```

#### Deposit

```ts
orionUnit.exchange.deposit({
  amount: 2.5,
  asset: "ORN",
  signer: wallet, // or signer when UI
});
```

#### Swap Function

//Example of a swap

```ts
orionUnit.exchange
  .swapMarket({
    type: "exactSpend",
    assetIn: "ORN",
    assetOut: "USDT",
    feeAsset: "ORN",
    amount: 23.89045345,
    slippagePercent: 1,
    signer: wallet, // or 'signer' when using Metamask module.
    options: {
      logger: console.log,
      // Set it to true if you want the issues associated with
      // the lack of allowance to be automatically corrected
      autoApprove: true,
    },
  })
  .then(console.log); //Output the status and transaction details after a swap.
```

#### Add liquidity

```ts
orionUnit.farmingManager.addLiquidity({
  poolName: "ORN-USDT",
  amountAsset: "ORN", // ORN or USDT for this pool
  amount: 23.352345,
  signer: wallet, // or signer when UI
});
```

#### Remove all liquidity

```ts
orionUnit.farmingManager.removeAllLiquidity({
  poolName: "ORN-USDT",
  signer: wallet, // or signer when UI
});
```

## Low level methods

### Get historical price

```ts
import { simpleFetch } from "@orionprotocol/sdk";

const candles = await simpleFetch(orionUnit.priceFeed.getCandles)(
  "ORN-USDT",
  1650287678, // interval start
  1650374078, // interval end
  "5m", // interval
  "all" // exchange
);
```

### Using contracts

```ts
import { contracts } from "@orionprotocol/sdk";

const exchangeContract = contracts.Exchange__factory.connect(
  exchangeContractAddress,
  orionUnit.provider
);
const erc20Contract = contracts.ERC20__factory.connect(
  tokenAddress,
  orionUnit.provider
);
const governanceContract = contracts.OrionGovernance__factory.connect(
  governanceAddress,
  orionUnit.provider
);
const orionVoting = contracts.OrionVoting__factory.connect(
  votingContractAddress,
  orionUnit.provider
);
```

### Get tradable pairs

```ts
import { simpleFetch } from "@orionprotocol/sdk";
const pairsList = await simpleFetch(orionUnit.orionAggregator.getPairsList)();
```

### Get swap info

//Example of getting swap data (dummy swap)
```ts
import { simpleFetch } from '@orionprotocol/sdk';

const swapInfo = await simpleFetch(orionUnit.orionAggregator.getSwapInfo)(
  // Use 'exactSpend' when 'amount' is how much you want spend. Use 'exactReceive' otherwise
  type: 'exactSpend',
  assetIn: 'ORN',
  assetOut: 'USDT',
  amount: 6.23453457,
);
```

### Place order in Orion Aggregator

```ts
import { simpleFetch } from "@orionprotocol/sdk";

// You can use simpleFetch or "default" (verbose) fetch
// Simple fetch

const { orderId } = await simpleFetch(orionUnit.orionAggregator.placeOrder)(
  {
    senderAddress: "0x61eed69c0d112c690fd6f44bb621357b89fbe67f",
    matcherAddress: "0xfbcad2c3a90fbd94c335fbdf8e22573456da7f68",
    baseAsset: "0xf223eca06261145b3287a0fefd8cfad371c7eb34",
    quoteAsset: "0xcb2951e90d8dcf16e1fa84ac0c83f48906d6a744",
    matcherFeeAsset: "0xf223eca06261145b3287a0fefd8cfad371c7eb34",
    amount: 500000000,
    price: 334600000,
    matcherFee: 29296395, // Orion Fee + Network Fee
    nonce: 1650345051276,
    expiration: 1652850651276,
    buySide: 0,
    isPersonalSign: false, // https://docs.metamask.io/guide/signing-data.html#a-brief-history
  },
  false // Place in internal orderbook
);

// Default ("verbose") fetch

const placeOrderFetchResult = await orionUnit.orionAggregator
  .placeOrder
  // Same params as above
  ();

if (placeOrderFetchResult.isErr()) {
  // You can handle fetching errors here
  // You can access error text, statuses
  const { error } = placeOrderFetchResult;
  switch (error.type) {
    case "fetchError": // (no network, connection refused, connection break)
      console.error(error.message);
      break;
    case "unknownFetchError": // Instance of Error
      console.error(`URL: ${error.url}, Error: ${error.message}`);
      break;
    case "unknownFetchThrow":
      console.error("Something wrong happened furing fetching", error.error);
      break;
    // ... and 8 errors types more
    // see src/fetchWithValidation.ts for details
  }
} else {
  // Success result
  const { orderId } = placeOrderFetchResult.value;
}
```

### Orion Aggregator WebSocket

Available subscriptions:

```ts
ADDRESS_UPDATES_SUBSCRIBE = 'aus', // Orders history, balances info
SWAP_SUBSCRIBE = 'ss', // Swap info updates
AGGREGATED_ORDER_BOOK_UPDATES_SUBSCRIBE = 'aobus', // Bids and asks
ASSET_PAIRS_CONFIG_UPDATES_SUBSCRIBE = 'apcus',
BROKER_TRADABLE_ATOMIC_SWAP_ASSETS_BALANCE_UPDATES_SUBSCRIBE = 'btasabus', // Need for Orion Bridge
```

### Swap Info

```ts
import { v4 as uuidv4 } from "uuid";

const swapRequestId = uuidv4();
orionUnit.orionAggregator.ws.subscribe(
  "ss", // SWAP_SUBSCRIBE
  {
    payload: {
      d: swapRequestId, // generated by client
      i: assetIn, // asset in
      o: assetOut, // asset out
      e: true, // true when type of swap is exactSpend, can be omitted (true by default)
      a: 5.62345343, // amount
    },
    // Handle data update in your way
    callback: (swapInfo) => {
      switch (swapInfo.kind) {
        case "exactSpend":
          console.log(swapInfo.availableAmountOut);
          break;
        case "exactReceive":
          console.log(swapInfo.availableAmountOut);
          break;
      }
    },
  }
);
```

### Balances and order history stream

```ts
orionUnit.orionAggregator.ws.subscribe(
  "aus", // ADDRESS_UPDATES_SUBSCRIBE — orders, balances
  {
    payload: "0x0000000000000000000000000000000000000000", // Some wallet address
    callback: ({ fullOrders, orderUpdate, balances }) => {
      // Each field is optional
      if (fullOrders) console.log(fullOrders); // Completed orders

      if (orderUpdate) {
        switch (orderUpdate.kind) {
          case "full":
            console.log("Order completed", orderUpdate);
            break;
          case "update":
            console.log("Order in the process of execution", orderUpdate);
            break;
          default:
            break;
        }
      }

      if (balances) console.log("Balance update", balances);
    },
  }
);
```

### Orderbook stream

```ts
orionUnit.orionAggregator.ws.subscribe("aobus", {
  payload: "ORN-USDT", // Some trading pair
  callback: (asks, bids, pairName) => {
    console.log(`${pairName} orderbook asks`, asks);
    console.log(`${pairName} orderbook bids`, bids);
  },
});
```

### Orion Aggregator WS Stream Unsubscribing

```ts
// Swap request unsubscribe
orionAggregator.ws.unsubscribe(swapRequestId); // Pass here id that you generate when subscribe

// Address update (balances / order history) unsubscribe
orionUnit.orionAggregator.ws.unsubscribe(
  "0x0000000000000000000000000000000000000000"
);

// Pair orderbook unsubscribe
orionUnit.orionAggregator.ws.unsubscribe("ORN-USDT");

// Asset pairs config updates unsubscribe
orionUnit.orionAggregator.ws.unsubscribe("apcu");

// Broker tradable atomic swap assets balance unsubscribe
orionUnit.orionAggregator.ws.unsubscribe("btasabu");
```

## Price Feed Websocket Stream

> :warning: **Currently supported only one subscription per subscription type**

```ts
orionUnit.priceFeed.ws.subscribe(
  "allTickers",
  (tickers) => {
    console.log(tickers);
  },
  undefined
);
orionUnit.priceFeed.ws.unsubscribe("allTickers");

orionUnit.priceFeed.ws.subscribe(
  "ticker",
  (ticker) => {
    console.log(tricker);
  },
  "ORN-USDT"
);
orionUnit.priceFeed.ws.unsubscribe("ticker");

orionUnit.priceFeed.ws.subscribe(
  "lastPrice",
  ({ pair, price }) => {
    console.log(`Price: ${price}`);
  },
  "ORN-USDT"
);
orionUnit.priceFeed.ws.unsubscribe("lastPrice");
```
