# ETH Transaction callback handler validation

The API Co-Signer automates approving and signing transactions and approving workspace changes. The API Co-Signer replaces using a mobile device for manual approvals. This is ideal for any workspace that expects a high volume of transactions, frequent workspace activity, or 24-hour access.

The API Co-Signer can be configured to work with a callback handler. This is a web endpoint (HTTPS) that can approve or reject transactions and workspace changes based on custom logic that you can implement.

Fireblocks provides with the ability to verify ETH or BTC transactions before these are getting signed by the API Co-Signer. 
It is possible to configure your workspace to receive the RAW data of the ETH or the BTC transaction as a part of the payload sent to the callback handler.

In this article we are going to cover how to verify Ethereum raw transactions.



## ETH - Callback Handler Payload structure

First, let’s take a look on the payload that is sent from the Co-Signer to the Callback handler (a detailed spec can be found in [here](https://developers.fireblocks.com/reference/transaction-signing-request-approval)):

```json
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


Note that the RLP encoded payload  (```rawTx.rawTx```) is the actual payload that you are signing on, or to be precise, the signature is done over the keccak256 hash of the RLP encoded payload, which is exactly the hash provided in the ```rawTx.payload``` property


## Creating our callback application:
Before diving into the verification of the ETH transaction process, let’s start with spinning up our callback server. In this example I am going to use Express.js framework:

Install express:\
```npm i express```

We will also need some additional packages to be installed:\
```npm i jsonwebtoken fs body-parser @ethereumjs/tx```

Initiating the app:
```js
const port = 8080;
const app = express();
```
Let’s set our middleware (we are using body-parser here in order to access the raw body sent over to our POST endpoint that we will add shortly):
```js
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
```js
app.post("/v2/tx_sign_request", (req, res) => { 
	console.log("The raw body of the HTTP request:", req.rawBody)
}
```

Now, let’s understand how exactly we get this rawBody of the HTTP request.
In fact, Fireblocks API Co-Signer does not send this payload in clear text, there is some authentication process in place. This authentication process is done by performing mutual authentication using signed Json Web Tokens.
You can find more information about the auth process here: <link_to_callback_jwt_auth>
But just to cover it in a few words - each request from the Co-Signer to the callback carries a signed JWT in the request body signed with the private key of the Co-Signer application. Same goes for the response done by the callback to the Co-Signer - it should be signed by the callback private key. 
We assume that you have gone through the authentication setup article and you have the callback private + public keys and also the Co-Signer public key.

Loading the callback’s private key + the cosigner public key:
```js
const fs = require("fs");

// Read the callback handler private key
const privateKey = fs.readFileSync("private.pem");

// Read the cosigner public key (you can get it by running: ./cosigner print-public-key on the cosigner machine)
const cosignerPubKey = fs.readFileSync("cosigner_public.pem");
```

In our example the raw body of the HTTP request will actually look like this:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJ0eElkIjoiNDhmNjg3MWMtMDQ2Ny00YjU2LThjNDQtNDI2ZjNkNjQ4N2Y1Iiwib3BlcmF0aW9uIjoiVFJBTlNGRVIiLCJzb3VyY2VUeXBlIjoiVkFVTFQiLCJzb3VyY2VJZCI6IjAiLCJkZXN0VHlwZSI6IlZBVUxUIiwiZGVzdElkIjoiMSIsImFzc2V0IjoiRVRIIiwiYW1vdW50IjowLjAxMDAwMDAwLCJhbW91bnRTdHIiOiIwLjAxMDAwMDAwMDAwMDAwMDAwMCIsInJlcXVlc3RlZEFtb3VudCI6MC4wMTAwMDAwMCwicmVxdWVzdGVkQW1vdW50U3RyIjoiMC4wMSIsImZlZSI6IjAuMDAwNTg0MjYxNjA3NjU3MDAwIiwiZGVzdEFkZHJlc3NUeXBlIjoiV0hJVEVMSVNURUQiLCJkZXN0QWRkcmVzcyI6IjB4NWRDNjlCMUZiYjEzQmFmZDA5YWY4OGE3ODJGMEYyODU3NzJBZDVmOCIsImRlc3RpbmF0aW9ucyI6W3siYW1vdW50TmF0aXZlIjowLjAxMDAwMDAwLCJhbW91bnROYXRpdmVTdHIiOiIwLjAxIiwiYW1vdW50VVNEIjoxOC43NDUwMTkzNiwiZHN0QWRkcmVzc1R5cGUiOiJXSElURUxJU1RFRCIsImRzdElkIjoiMSIsImRzdFdhbGxldElkIjoiIiwiZHN0TmFtZSI6Ik5ldHdvcmsgRGVwb3NpdHMiLCJkc3RTdWJUeXBlIjoiIiwiZHN0VHlwZSI6IlZBVUxUIiwiZGlzcGxheURzdEFkZHJlc3MiOiIweDVkQzY5QjFGYmIxM0JhZmQwOWFmODhhNzgyRjBGMjg1NzcyQWQ1ZjgiLCJhY3Rpb24iOiJBTExPVyIsImFjdGlvbkluZm8iOnsiY2FwdHVyZWRSdWxlTnVtIjo1LCJydWxlc1NuYXBzaG90SWQiOjgxNjQsImJ5R2xvYmFsUG9saWN5IjpmYWxzZSwiYnlSdWxlIjp0cnVlLCJjYXB0dXJlZFJ1bGUiOiJ7XCJ0eXBlXCI6XCJUUkFOU0ZFUlwiLFwidHJhbnNhY3Rpb25UeXBlXCI6XCJUUkFOU0ZFUlwiLFwiYXNzZXRcIjpcIipcIixcImFtb3VudFwiOjAsXCJvcGVyYXRvcnNcIjp7XCJ3aWxkY2FyZFwiOlwiKlwifSxcImFwcGx5Rm9yQXBwcm92ZVwiOnRydWUsXCJhY3Rpb25cIjpcIkFMTE9XXCIsXCJzcmNcIjp7XCJpZHNcIjpbW1wiKlwiXV19LFwiZHN0XCI6e1wiaWRzXCI6W1tcIipcIl1dfSxcImRzdEFkZHJlc3NUeXBlXCI6XCIqXCIsXCJhbW91bnRDdXJyZW5jeVwiOlwiVVNEXCIsXCJhbW91bnRTY29wZVwiOlwiU0lOR0xFX1RYXCIsXCJwZXJpb2RTZWNcIjowfSJ9fV0sInJhd1R4IjpbeyJrZXlEZXJpdmF0aW9uUGF0aCI6IlsgNDQsIDYwLCAwLCAwLCAwIF0iLCJyYXdUeCI6IjAyZWYwMTA0ODQzYjlhY2EwMDg1MDY3YTUxYmU4NTgyNTIwODk0NWRjNjliMWZiYjEzYmFmZDA5YWY4OGE3ODJmMGYyODU3NzJhZDVmODg3MjM4NmYyNmZjMTAwMDA4MGMwIiwicGF5bG9hZCI6IjZjYTE4YzQ3Y2NkOGRkNDdiYjBiMGIwYzczY2M1MjM3YmJmZjlkMGVhYWU0NjljYTY1YTY3MjMyOTY0M2JmZGEifV0sInBsYXllcnMiOlsiMjE5MjZlY2MtNGE4YS00NjE0LWJiYWMtN2M1OTFhYTdlZmRkIiwiMjc5MDA3MzctNDZmNi00MDk3LWExNjktZDBmZjQ1NjQ5ZWQ1IiwiZjg5Y2FjNTAtYzY1Ni00ZTc0LTg3OWYtMDQxYWZmOGQwMWI1Il0sInJlcXVlc3RJZCI6IjQ4ZjY4NzFjLTA0NjctNGI1Ni04YzQ0LTQyNmYzZDY0ODdmNSJ9.00000000000000000000000000000000000000000000000
```

* Kindly note that the signature part of the JWT is obfuscated as a security best practice.

If you take this JWT and parse it in jwt.io - you’ll get the same clear text json payload as at the beginning of this guide.

## JWT Verification

So what should we do with this JWT? We need to verify it’s signature by using the Co-Signer public key and we need to decode it to JSON:
```js
const jwt = require("jsonwebtoken");
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

app.post("/v2/tx_sign_request", (req, res) => { 

	try {
		// jwt.verify returns the decoded payload on successful verification 
        const tx = jwt.verify(req.rawBody, cosignerPubKey);
    } catch(e) {
	    console.error(e)
        res.sendStatus(401).send();
    }
}
```

## Building the ETH transaction object

So now we have set the authentication and if the token is verified we can actually move on to the transaction validation part. Let define a new function validateETHTransaction. It will get the plain text payload and will build an ETH transaction object from the RLP encoded hex:

```js
const { FeeMarketEIP1559Transaction } = require('@ethereumjs/tx');

const validateETHTransaction = (payload) => {
   
    // Decode RLP
    const unsignedTx = FeeMarketEIP1559Transaction.fromSerializedTx(
        Buffer.from(
            payload.rawTx[0].rawTx, 
            "hex"
        )
    )
}
```

Basically, we are taking the RLP encoded hex and by using the 'FeeMarketEIP1559Transaction class from @ethereumjs/tx, we are building the transaction object.

If we will print the unsignedTx object, we’ll get:
```json
{
  "chainId": "0x1",
  "nonce": "0x4",
  "maxPriorityFeePerGas": "0x4d4ea640",
  "maxFeePerGas": "0x7c4b19ac5",
  "gasLimit": "0x5208",
  "to": "0x5dc69b1fbb13bafd09af88a782f0f285772ad5f8",
  "value": "0x2386f26fc10000",
  "data": "0x",
  "accessList": []
}
```

## Verifying the transaction parameters

So now we can actually check whether the destination and the amount in the decoded transaction match the JSON values of the payload:
```js
const { FeeMarketEIP1559Transaction } = require('@ethereumjs/tx');

const validateETHTransaction = (payload) => {
    // Decode RLP
    const unsignedTx = FeeMarketEIP1559Transaction.fromSerializedTx(
        Buffer.from(
            payload.rawTx[0].rawTx, 
            "hex"
        )
    )

    return (
        parseFloat(unsignedTx.value, 16) / 1e18 === payload.destinations[0].amountNative 
        && 
        unsignedTx.to.toString("hex") === payload.destinations[0].displayDstAddress.toLowerCase() 
    )
}
```
*Note that the address in the payload is in checksum format hence we need to lower case it


We can actually add one more check - verify that the hash of the unsignedTx matches the provided hash in the payload:
```js
const { FeeMarketEIP1559Transaction } = require('@ethereumjs/tx');

const validateETHTransaction = (payload) => {
    const providedHash = payload.rawTx[0].payload;
    // Decode RLP
    const unsignedTx = FeeMarketEIP1559Transaction.fromSerializedTx(
        Buffer.from(
            payload.rawTx[0].rawTx, 
            "hex"
        )
    )
    
    return (
        parseFloat(unsignedTx.value, 16) / 1e18 === payload.destinations[0].amountNative 
        && 
        unsignedTx.to.toString("hex") === payload.destinations[0].displayDstAddress.toLowerCase() 
        && 
        unsignedTx.getMessageToSign(true).toString("hex") == providedHash
    )
}

```
What do we do here? We are taking the hash from the payload provided by the Co-Signer and comparing it with the result of unsignedTx.getMessageToSign(true) which returns the hash of the unsigned transaction that should be signed.


## Callback Handler response

The response from the callback should be in the following format (signed with RS256 algorithm by using the callback private key):
```
{
	action: 'APPROVE' OR 'REJECT',
	requestId: 'The unique identifier of the call that was received in the approval request',
	rejectionReason: (Optional) 'Free text of the rejection reason for logging purposes'
}
```

Let’s just add one more function (my personal preference) that will generate the signed response that should be sent from the callback handler:

```js
const generateSignedResponse = (action) => {
    
    const signedRes = jwt.sign(
        action,
        privateKey,
        { algorithm: "RS256" }
    );

    return signedRes
}
```

## All parts together

```js
const fs = require("fs");
const express = require("express");
const jwt = require("jsonwebtoken");
const bodyParser = require("body-parser");
const { FeeMarketEIP1559Transaction } = require('@ethereumjs/tx');

// Read the callback handler private key
const privateKey = fs.readFileSync("private.pem");

// Read the cosigner public key (you can get it by running: ./cosigner print-public-key on the cosigner machine)
const cosignerPubKey = fs.readFileSync("cosigner_public.pem");

// Start express app and set middleware 
const port = 8080; 
const app = express();

app.use(bodyParser.urlencoded({ extended: true }));
app.use(bodyParser.json())
app.use( (req) => {
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

// Generate JWT response
const generateSignedResponse = (action) => {
    
    const signedRes = jwt.sign(
        action,
        privateKey,
        { algorithm: "RS256" }
    );
    
    return signedRes
}

// Verify that the params of the rawTx match the payload
const validateETHTransaction = (payload) => {    
    
    const providedHash = payload.rawTx[0].payload
    // Decode RLP
    const unsignedTx = FeeMarketEIP1559Transaction.fromSerializedTx(
        Buffer.from(payload.rawTx[0].rawTx, "hex")) 
    return (
        parseFloat(unsignedTx.value, 16) / 1e18 === payload.destinations[0].amountNative 
        && 
        unsignedTx.to.toString("hex") === payload.destinations[0].displayDstAddress.toLowerCase() 
        && 
        unsignedTx.getMessageToSign(true).toString("hex") == providedHash
    )
}

// Tx Sign Request endpoint 
app.post("/v2/tx_sign_request", (req, res) => { 
    
    try {
        let response; 
        const tx = jwt.verify(req.rawBody, cosignerPubKey);
        if (validateETHTransaction(tx)) {
            response = generateSignedResponse({
                action: "APPROVE",
                requestId: tx.requestId
            })
       } else {
            response = generateSignedResponse({
                action: "REJECT",
                requestId: tx.requestId,
                rejectionReason: `Failed to validate ETH transaction`
           	})
       }
        
        res.status(200).send(response);
    
    } catch(e) {
        console.error(e)
        res.sendStatus(401).send();
    }
});

console.log(`Callback is running on http://localhost:${port}`)
app.listen(port);
```
