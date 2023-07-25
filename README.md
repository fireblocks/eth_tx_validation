# ETH Transaction callback handler validation

The API Co-Signer automates approving and signing transactions and approving workspace changes. The API Co-Signer replaces using a mobile device for manual approvals. This is ideal for any workspace that expects a high volume of transactions, frequent workspace activity, or 24-hour access.

The API Co-Signer can be configured to work with a callback handler. This is a web endpoint (HTTPS) that can approve or reject transactions and workspace changes based on custom logic that you can implement.

Fireblocks provides with the ability to verify ETH or BTC transactions before these are getting signed by the API Co-Signer. 
It is possible to configure your workspace to receive the RAW data of the ETH or the BTC transaction as a part of the payload sent to the callback handler.

In this article we are going to cover how to verify Ethereum and Bitcoin raw transactions.



## ETH - Callback Handler Payload structure

First, let’s take a look on the payload that is sent from the Co-Signer to the Callback handler (a detailed spec can be found in here):

```
{
  "txId": "9c794cee-7e27-46c9-9e9a-ed68295ff06b",
  "operation": "TRANSFER",
  "sourceType": "VAULT",
  "sourceId": "0",
  "destType": "VAULT",
  "destId": "1",
  "asset": "ETH",
  "amount": 0.01,
  "amountStr": "0.010000000000000000",
  "requestedAmount": 0.01,
  "requestedAmountStr": "0.01",
  "fee": "0.000597803762241000",
  "destAddressType": "WHITELISTED",
  "destAddress": "0x5dC69B1Fbb13Bafd09af88a782F0F285772Ad5f8",
  "destinations": [
    {
      "amountNative": 0.01,
      "amountNativeStr": "0.01",
      "amountUSD": 18.74292937,
      "dstAddressType": "WHITELISTED",
      "dstId": "1",
      "dstWalletId": "",
      "dstName": "Network Deposits",
      "dstSubType": "",
      "dstType": "VAULT",
      "displayDstAddress": "0x5dC69B1Fbb13Bafd09af88a782F0F285772Ad5f8",
      "action": "ALLOW",
      "actionInfo": {
        "capturedRuleNum": 5,
        "rulesSnapshotId": 8164,
        "byGlobalPolicy": false,
        "byRule": true,
        "capturedRule": "{\"type\":\"TRANSFER\",\"transactionType\":\"TRANSFER\",\"asset\":\"*\",\"amount\":0,\"operators\":{\"wildcard\":\"*\"},\"applyForApprove\":true,\"action\":\"ALLOW\",\"src\":{\"ids\":[[\"*\"]]},\"dst\":{\"ids\":[[\"*\"]]},\"dstAddressType\":\"*\",\"amountCurrency\":\"USD\",\"amountScope\":\"SINGLE_TX\",\"periodSec\":0}"
      }
    }
  ],
  "rawTx": [
    {
      "keyDerivationPath": "[ 44, 60, 0, 0, 0 ]",
      "rawTx": "02ef0104843b9aca008506a0c1987d825208945dc69b1fbb13bafd09af88a782f0f285772ad5f8872386f26fc1000080c0",
      "payload": "77b4e74099ce90c08503c0e0bb6e672dbe1c5e3e127ce333bf22eb581cd3f6ce"
    }
  ],
  "players": [
    "21926ecc-4a8a-4614-bbac-7c591aa7efdd",
    "27900737-46f6-4097-a169-d0ff45649ed5",
    "f89cac50-c656-4e74-879f-041aff8d01b5"
  ],
  "requestId": "9c794cee-7e27-46c9-9e9a-ed68295ff06b"
}
```

The payload above contains a lot of information but we will be focusing only on some parts of it:

1. Amount (```destinations[0].amountNative```)
2. Destination Address (```destinations[0].displayDstAddress```)
3. Raw Transaction array (```rawTx```)
RLP encoded payload (```rawTx.rawTx```)
The hash of the raw transaction (```rawTx.payload```)


Note that the RLP encoded payload  (```rawTx.rawTx````) is the actual payload that you are signing on, or to be precise, the signature is done over the keccak256 hash of the RLP encoded payload, which is exactly the hash provided in the ```rawTx.payload``` property


## Creating our callback application:

### Before diving into the verification of the ETH transaction process, let’s start with spinning up our callback server. In this example I am going to use Express.js framework:

Install express:
```npm i express```

We will also need some additional packages to be installed:
```npm i jsonwebtoken fs body-parser @ethereumjs/tx```

Initiating the app:
```
const port = 8080;
const app = express();
```
Let’s set our middleware (we are using body-parser here in order to access the raw body sent over to our POST endpoint that we will add shortly):
```
const bodyParser = require("body-parser");

app.use(bodyParser.urlencoded({ extended: true }));
app.use(bodyParser.json())
app.use(function (req) {
    req.rawBody = "";
    req.setEncoding("utf8");
    req.on("data", function(chunk) {
      req.rawBody += chunk;
    });
    req.on("end", function () {
      req.next();
    });
  }
);
```

Defining our route for the /v2/tx_sign_request endpoint:
```
app.post("/v2/tx_sign_request", (req, res) => { 
	console.log("The raw body of the HTTP request:", req.rawBody)
}
```


