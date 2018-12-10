# Wings ABI

Manual how to use Wings contracts ABI.

## Table of contents

 - [Introduction](#introduction)
 - [Creating a project for valuation forecast](#creating-a-project-for-valuation-forecast)
 - [Making a forecast](#making-a-forecast)

---

# Introduction

In this manual we will be using `Node.js`, `web3` (^0.20.6) and `truffle-contract` to operate with contracts.

In order to use Wings contracts you need to have contracts ABI.  
You can find all ABI files in the `./abi` folder of this repository.

Here is an example how to initiate contract interface:

```js
const contract = require('truffle-contract')
const Web3 = require('web3')

const wingsArtifact = require('./abi/Wings.json')

const Wings = contract(wingsArtifact)

Wings.setProvider(new Web3.providers.HttpProvider(web3Provider))
```

Below is an example of initiating contract instance (and addresses of main Wings contracts):

```js
const token = Token.at('0x667088b212ce3d06a1b553a7221E1fD19000d9aF') // mainnet wings Token contract address
// https://etherscan.io/token/0x667088b212ce3d06a1b553a7221E1fD19000d9aF

const userStorage = UserStorage.at('0x94B2F026A75BE2556C78A6D1f573bD79Fdfb1962') // mainnet wings User Storage contract address
// https://etherscan.io/address/0x94b2f026a75be2556c78a6d1f573bd79fdfb1962

const wings = Wings.at('0x7ea8dc2b2b00b596d077b68f5c891e03797a5eb2') // mainnet Wings contract address
// https://etherscan.io/address/0x7ea8dc2b2b00b596d077b68f5c891e03797a5eb2

const dao = DAO.at('0xd6635f49a306b015c55bd1ff878e2c2c8413f247') // this is an example, not the real address
// To get DAO address head to the project on wings.ai which you would like to forecast and get the address from the url:
// https://www.wings.ai/project/0xd6635f49a306b015c55bd1ff878e2c2c8413f247
// 0xd6635f49a306b015c55bd1ff878e2c2c8413f247 <-- is DAO address

const forecastingAddress = (await dao.forecasting.call()).toString()

const forecasting = Forecasting.at(forecastingAddress)
```

---

# Creating a project for valuation forecast

First of all you have to have enough wings on your account for wings deposit (we will refer to your account as `creator` in this step by step tutorial).

### 1. Create DAO

First step in project creation process is creating a DAO. DAO is a main contract in your project hierarchy.

#### Wings Integration with custom crowdsale contract

In case you want to create and use custom crowdsale contract you will need to perform integration with wings.

Currently there are two supported types of integration:
 - [Wings Light Bridge](https://github.com/wingsdao/wings-light-bridge) *(from the latest update you can create Bridge directly from DAO)*
 - [Wings Full Integration](https://github.com/wingsdao/wings-integration)

*NOTE: In case you've chosen Wings Full Integration you should complete integration guide and deploy your crowdsale before DAO creation.*

---

**Important:** Integration is responsible step and in case of having any doubts or questions please contact Wings team for help.

---

#### approve

When you have enough wings on your account balance, call method `approve` on wings `Token` contract to give the `Wings` contract ability to transfer deposit.

```js
const wingsTokenAddress = '0x667088b212ce3d06a1b553a7221E1fD19000d9aF' // mainnet wings Token contract address
// https://etherscan.io/token/0x667088b212ce3d06a1b553a7221E1fD19000d9aF

const wingsToken = Token.at(wingsTokenAddress)

await wingsToken.approve(wingsAddress, wingsDeposit, { from: creator })
```

**Parameters:**
 - `wingsAddress` - address of `Wings` contract (can be found above)
 - `wingsDeposit` - amount of wings to be locked in order to create project. Currently is 200 Wings.

#### createDAO

After successful approval, call the method `createDAO` on `Wings` contract instance.

```js
await wings.createDAO(name, tokenName, tokenSymbol, infoHash, customCrowdsale, { from: creator })
```

**Parameters:**
 - `name` - string - name of your project
 - `tokenName` - string - name of project token. Optional, default: `''`
 - `tokenSymbol` - string - symbol of project token. Optional, default: `''`
 - `infoHash` - bytes32 - decoded ipfs hash of project description.
 - `customCrowdsale` - address - address of custom crowdsale (`0` in case of *standard crowdsale/wings light bridge*)

#### Uploading your project description and media to ipfs

Head to [Media file format](#media-file-format) paragraph in the Appendix section.

---

**Warning:** Ipfs file can contain malicious code which means that everyone developing and using this data must sanitize it.

---

#### Generating infoHash

Ipfs hash is using the same Base58 encoding that Bitcoin uses.
To fit ipfs hash into `infoHash` we first will need to decode it.

**Example:**
```js
const bs58 = require('bs58')

const ipfsAddress = 'QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG'

let bytes = bs58.decode(ipfsAddress).toString('hex')

// As for now that's the only format that ipfs uses, so we can just cut the first two bytes

const infoHash = '0x' + bytes.substring(4)
```

### 2. Get DAO address

When DAO is created you can find it's address by calling Wings contract. As an argument you will have to pass keccak256 encrypted name of the project (same as the one you used during DAO creation).

```js
const daoId = web3.sha3(name)

const daoAddress = (await wings.getDAOById.call(daoId)).toString()
```

**Parameters:**
 - `name` - string - name of your project

### 2.1 Create Wings Light Bridge

*Required stage: initial*

In case you are going to use simple integration for your custom crowdsale, during this step you need to create `Bridge`.

```js
await dao.createBridge(projectToken, minimalGoal, hardCap, startTimestamp, endTimestamp, { from: creator })
```

**Parameters:**
 - `projectToken` - address - project token contract address. Optional.
 - `minimalGoal` - uint256 - minimal goal of crowdsale. Optional.
 - `hardCap` - uint256 - hard cap of crowdsale. Optional.
 - `startTimestamp` - uint256 - unix timestamp of the start of crowdsale. Optional.
 - `endTimestamp` - uint256 - unix timestamp of the end of crowdsale. Optional.

### 2.1.1 Communication with Wings Light Bridge

Get address of `Bridge`.

```js
const bridgeAddress = await dao.crowdsale.call()
```

More thorough guide for Wings Light Bridge can be found [here](https://github.com/wingsdao/wings-light-bridge).

### 3. Create Rewards Model

*Required stage: initial*

When you have DAO address, you can initiate a contract instance by address and create rewards model.

```js
const dao = DAO.at(daoAddress)

await dao.createModel({ from: creator })
```

### 4. Create Forecasting

*Required stage: model created*

When rewards model is created you can create a forecasting.

```js
await dao.createForecasting(forecastingDurationInHours, ethRewardPart, tokenRewardPart, { from: creator })
```

**Parameters:**
 - `forecastingDurationInHours` - uint256 - duration of forecasting in hours (from 120 to 360 hours)

**Notes:**
 - reward percent must be multiplied by 10000 (*Example: reward is 1.5% the argument must be passed as 15000*)
 - reward part must be less then 1000000 (100% * 10000)
 - `tokenRewardPart` must be greater than zero

### 5. Start Forecasting

*Required stage: forecasting created*

When the model and forecasting are created you can start forecasting.

```js
await dao.startForecasting(bucketMin, bucketMax, bucketStep, { from: creator })
```

**Parameters:**
 - `bucketMin` - uint256 - minimal bucket
 - `bucketMax` - uint256 - maximal bucket
 - `bucketStep` - uint256 - bucket step

#### Calculate buckets

The following code demonstrates the algorithm of buckets calculation.

```js
const ONE_ETH = new BigNumber('1000000000000000000')
const STEPS_IN_GOAL = 100
const STEPS_IN_MAX_AMOUNT = 150

const weiGoal = new BigNumber(web3.toWei(goal, 'ether'))

let bucketMin, bucketStep, bucketMax

let ethGoal = Number(weiGoal.div(ONE_ETH).floor())

if (ethGoal < STEPS_IN_GOAL) {
  ethGoal = STEPS_IN_GOAL
}

if (ethGoal < STEPS_IN_GOAL*1.1) {
  bucketMin = ONE_ETH
  bucketStep = ONE_ETH
  bucketMax = ONE_ETH.mul(STEPS_IN_MAX_AMOUNT)
} else {
  let e = Math.floor((ethGoal / (STEPS_IN_GOAL + 1)));
  let p = Math.floor(Math.log10(e));
  let c = Math.floor(Math.pow(10, p));
  let d = Math.floor((e / c));


  if (d < 2) d = 2;
  else if (d < 5) d = 5;
  else d = 10;

  bucketStep = ONE_ETH.mul(d).mul(c)
  bucketMin = bucketStep
  bucketMax = bucketStep.mul(STEPS_IN_MAX_AMOUNT)
}
```

### 6. Finish Forecasting

*Required stage: forecasting started*

After forecasting period you can close forecasting. This will automatically check forecasting for spam.

```js
await dao.closeForecasting({ from: creator })
```

## Note that after forecasting end you will have 45 days to start your crowdsale.

## In case you are using Custom Crowdsale or Wings Light Bridge, skip step 7 and head to the step 8.1.

### 7. Create token

*Required stage: forecasting closed*

When forecasting is closed you can create your project token. It will have the `tokenName` and a `tokenSymbol` which you used during DAO creation process.

```js
await dao.createToken(decimals, { from: creator })
```

**Parameters:**
 - `decimals` - uint8 - project token decimals

### 8. Create Crowdsale

*Required stage: token created*

When token is created you can create crowdsale.

```js
await dao.createCrowdsale(minimalGoal, hardCap, prices1to4, prices5to8, { from: creator })
```

**Parameters:**
 - `minimalGoal` - uint256 - soft cap of crowdsale (in Wei)
 - `hardCap` - uint256 - hard cap of crowdsale (in Wei)
 - `prices1to4` - uint256 - packed prices
 - `prices5to8` - uint256 - packed prices

#### Calculate and pack prices

You can create up to 7 price periods, which will be packed into two uint256 values.

`prices` is an array of price bonus objects.

```ts
interface PriceBonus {
  price: number
  durationDays: string
  durationHours: string
  durationMinutes: string
  duration: any
  isEdit: boolean
  errorText: string,
  durationErrorText: string
}
```

You will need to install and include module `bignumber.js@4.0.4`.

Here is a script with simple example of `prices` array:

*NOTE: PriceBonus objects start from the start of the array, hence PriceBonus.price must be decreasing while array index increases.*

```js
/*
  Here are variables which you need to prepare:
*/

// PriceBonus[]
const prices = [{
  price: '50', // means 50% bonus
  durationDays: '5',
  durationHours: '0', // 0-23
  durationMinutes: '0', // 0-59
  duration: '',
  isEdit: false,
  errorText: '',
  durationErrorText: ''
}]

// Amount of integer tokens per ETH
const tokenPrice = 100

/*----------------------------------- script -----------------------------------*/
const convertPrices = async (prices, tokenPrice) => {
  let result

  const priceChanges = [] // PriceChange[]
  for(const price of prices) {
    const value = new BigNumber(new BigNumber(price.price).mul(tokenPrice).div(100).plus(tokenPrice).toFixed(0))
    priceChanges.push({
      price: value,
      duration: bonusDurationToSeconds(price)
    })
  }

  const lastPrice = { // PriceChange
    price: new BigNumber(tokenPrice),
    duration: 1
  }

  priceChanges.push(lastPrice)

  result = await packPrices(priceChanges)
  return result
}

const bonusDurationToSeconds = (priceBonus) => {
  return parseInt(priceBonus.durationDays) * 86400 + parseInt(priceBonus.durationHours) * 3600 + parseInt(priceBonus.durationMinutes) * 60
}

const packPrices = async (changes) => {
  const len = changes.length

  if (len > 8) {
    throw new Error('Price changes cant contain more then 8 items')
  }

  const results = [new BigNumber(0), new BigNumber(0)]
  let i

  if (len > 0) {
    for (i = 3; i >= 0; --i) {
      if (i >= len) continue
      results[0] = results[0].shift(14).add(packSingle(changes[i]))
    }
  }

  if (len > 4) {
    for (i = 7; i >= 4; --i) {
      if (i >= len) continue
      results[1] = results[1].shift(14).add(packSingle(changes[i]))
    }
  }

  return results
}

const packSingle = (change) => {
  const PACK_MAX = 10000000

  if (change.price.lessThanOrEqualTo(0) || change.price.greaterThanOrEqualTo(PACK_MAX)) {
    throw new Error(`Price out of range: ${change.price.toString(10)}`)
  }

  if (change.duration <= 0 || change.duration >= PACK_MAX) {
    throw new Error(`Duration out of range: ${change.duration}`)
  }

  return new BigNumber(change.duration).shift(7).add(change.price)
}
/*----------------------------------- script -----------------------------------*/

// We launch our script here
convertPrices(prices, tokenPrice).then((result) => {
  // Here are two variables which you'll need to pass as arguments to `createCrowdale` method (either BigNumber or string)
  let prices1to4 = result[0].toString(10)
  let prices5to8 = result[1].toString(10)
})
```

### 8.1. Create Custom Crowdsale

*Required stage: forecasting closed*

When forecasting is closed you need to call method `createCustomCrowdsale`.  
This and following steps are required in order to finalise forecasting and reward wings community.

```js
await dao.createCustomCrowdsale({ from: creator })
```

*NOTE: During this step the manager of the Crowdsale will be transferred to a newly created Crowdsale Controller.*

### 9. Get address of Crowdsale Controller

In order to start crowdsale you'll need to find the address of Crowdsale Controller, which is the contract created during the previous step.

```js
const ccAddress = (await dao.crowdsaleController.call()).toString()
```

### 10. Start Crowdsale

When you have Crowdsale Controller address, initiate a contract instance and call the method `start`.

```js
const cc = CC.at(ccAddress)

await cc.start(startTimestamp, endTimestamp, fundingAddress, { from: creator })
```

**Parameters:**
 - `startTimestamp` - uint256 - unix timestamp of the start of crowdsale period
 - `endTimestamp` - uint256 - unix timestamp of the end of crowdsale period
 - `fundingAddress` - address - address of account, which will receive funds, collected during crowdsale period

### 11. Finish crowdsale (prepare to give rewards)

To finish crowdsale and prepare to give rewards make the following call.

```js
await cc.closeForecasting({ from: creator })
```

**Notes:**
 - requires crowdsale to be either in failed or successful state
 - can be called by any user

## Additional functions

### setRewardParts

This DAO method allows you to set/change rewards percent.

```js
await dao.setRewardsParts(ethRewardPart, tokenRewardPart, { from: creator })
```

**Parameters:**
 - `ethRewardPart` - uint256 - reward percent of total collected Ether
 - `tokenRewardPart` - uint256 - reward percent of total sold tokens

**Notes:**
 - reward percent must be multiplied by 10000 (*Example: reward is 1.5% the argument must be passed as 15000*)
 - reward part must be less then 1000000 (100% * 10000)
 - `tokenRewardPart` must be greater than zero
 - if you are changing reward parts the new parts must be greater or equal to the old parts

### update

This DAO method allows you to change project description during forecasting period.

```js
await dao.update(infoHash, { from: creator })
```

**Parameters:**
 - `infoHash` - bytes32 - decoded ipfs hash of updated project description

### stop

This DAO method stops the DAO in any state.

```js
await dao.stop({ from: creator })
```

In case of the stop the `wingsDeposit` will be returned only if forecasting hasn't started.

## Appendix

### Media file format

File that contains project media has to be in JSON format.

**Example:**
```json
{
  "version": "1.0.0",
  "shortBlurb": "My short project description",
  "story": "{\"ops\": [{\"insert\":\"test\"},{\"insert\":{\"video\":\"https://www.youtube.com/embed/CXo7i_gdNR0?showinfo=0\"}}]}",
  "category": 1,
  "gallery": [
    {
      "type": "logo",
      "content": {
        "contentType": "image/png",
        "hash": "QmWWQSuPMS6aXCbZKpEjPHPUZN2NjB3YrhJTHsV4X3vb2t"
      }
    },
    {
      "type": "video",
      "content": {
        "videoId": "CXo7i_gdNR0",
        "videoType": "youtube"
      }
    },
    {
      "type": "image",
      "content": {
        "contentType": "image/jpeg",
        "hash": "0x38134191b4b59736d5174cdd0846bb8bafdf01fbfe013ead08ca133f13e23e59"
      }
    },
    {
      "type": "terms",
      "content": {
        "hash": "QmWWQSuPMS6aXCbZKpEjPHPUZN2NjB3YrhJTHsV4X3vb2t"
      }
    }
  ]
}
```

**Schema:**
```js
{
  "$id": "http://example.com/example.json",
  "type": "object",
  "definitions": {},
  "$schema": "http://json-schema.org/draft-07/schema#",
  "properties": {
    "version": {
      "$id": "/properties/version",
      "type": "string"
    },
    "shortBlurb": {
      "$id": "/properties/shortBlurb",
      "type": "string"
    },
    "story": {
      "$id": "/properties/story",
      "type": "string"
    },
    "category": {
      "$id": "/properties/category",
      "type": "integer"
    },
    "forecastingCurrency": {
        "id": "/properties/forecastingCurrency",
        "type": "integer"
    },
    "customContract": {
        "id": "/properties/customContract",
        "type": "bool"
    },
    "gallery": {
      "$id": "/properties/gallery",
      "type": "array",
      "items": {
        "$id": "/properties/gallery/items",
        "type": "object",
        "properties": {
          "type": {
            "$id": "/properties/gallery/items/properties/type",
            "type": "string"
          },
          // "type:" "logo"
          "content": {
            "$id": "/properties/gallery/items/properties/content",
            "type": "object",
            "properties": {
              "contentType": {
                "$id": "/properties/gallery/items/properties/content/properties/contentType",
                "type": "string"
              },
              "hash": {
                "$id": "/properties/gallery/items/properties/content/properties/hash",
                "type": "string"
              }
            }
          }
          // "type:" "video"
          "content": {
            "$id": "/properties/gallery/items/properties/content",
            "type": "object",
            "properties": {
              "videoId": {
                "$id": "/properties/gallery/items/properties/content/properties/videoId",
                "type": "string"
              },
              "videoType": {
                "$id": "/properties/gallery/items/properties/content/properties/videoType",
                "type": "string"
              }
            }
          }
          // "type:" "image"
          "content": {
            "$id": "/properties/gallery/items/properties/content",
            "type": "object",
            "properties": {
              "contentType": {
                "$id": "/properties/gallery/items/properties/content/properties/contentType",
                "type": "string"
              },
              "hash": {
                "$id": "/properties/gallery/items/properties/content/properties/hash",
                "type": "string"
              }
            }
          }
          // "type:" "terms"
          "content": {
            "$id": "/properties/gallery/items/properties/content",
            "type": "object",
            "properties": {
              "hash": {
                "$id": "/properties/gallery/items/properties/content/properties/hash",
                "type": "string"
              }
            }
          }
        }
      }
    }
  }
}
```

To better understand parameters let's prepare full list:

- *version* - version of media file (means version of media file schema).
- *shortBlurb* - short description of project, string. Maximum 140 characters.
- *story* - description of project in [Delta](https://github.com/quilljs/delta) format (see the example below).
- *category* - id of category (see Categories below).
- *forecastingCurrency* - id of forecasting currency (0: ETH, 1: USD, 2: EUR).
- *customContract* - whether project will use custom contract or not.
- *gallery* - gallery description of project.
- _gallery/*/contentType_ - type of content. Options: video, image, logo, terms.
- *gallery/video/content/videoId* -  id of video from video hosting.
- *gallery/video/content/videoType* - type of video hosting. Options: youtube, vimeo, youku.
- *gallery/image/content/hash* - ipfs hash of image uploaded to ipfs.
- *gallery/logo/content/contentType* - type of image format. Options: `image/png`, `image/jpg`.
- *gallery/logo/content/hash* - ipfs hash of logo uploaded to ipfs. Only jpg/png allowed; size < 1 mb.
- *gallery/terms/content/hash* - ipfs hash of terms uploaded to ipfs. Only pdf allowed; size < 1 mb.

#### How to use Delta module to generate story:

```js
const Delta = require('quill-delta')

const delta = new Delta([
  { insert: 'test' },
  { insert: { video: 'https://www.youtube.com/embed/fy2XDBbDrAs?showinfo=0' } }
])

console.log(JSON.stringify(JSON.stringify(delta)))
// "{\"ops\":[{\"insert\":\"test\"},{\"insert\":{\"video\":\"https://www.youtube.com/embed/fy2XDBbDrAs?showinfo=0\"}}]}"
```

#### Categories:

To get up-to-date list of categories use our [public API](https://apidocs.wings.ai/#/categories/get_dictionary_categories).

---

# Making a forecast

This step by step tutorial will walkthrough from the beginning, assuming you already have wings tokens, but didn't reserve or lock any of them.
We will refer to forecaster address as `forecaster`.

#### 1. Approve wings amount to reserve

In order to reserve wings give User Storage permission to transfer wings.

```js
await token.approve(userStorage.address, amount, {
  from: forecaster
})
```

**Parameters:**
 - `amount` - uint256 - amount of wings tokens to reserve


#### 2. Reserve wings

After the approval, you can reserve wings. Reserved wings will become locked wings after you'll make a forecast.

```js
await userStorage.reserveWings(
  amount,
  {
    from: forecaster
  }
)
```

#### 3. Add account address to DAO

When making new forecast the first step is to add your account address to DAO.

```js
const daoId = (await dao.id.call()).toString()

await wings.addForecasterToDAO(daoId, {
  from: forecaster
})
```

#### 4. Place forecast

After your account address was added to DAO you can place your forecast.

```js
await forecasting.addForecast(
  forecast,
  messageHash,
  {
    from: forecaster
  }
)
```

**Parameters:**
 - `forecast` - uint256 - forecasted amount
 - `messageHash` - bytes32 - ipfs hash of message (message is a buffered string)

**Important:** If you want to reject the project (flag it as spam/scam) you need to set forecast amount to 0.

---

## Additional methods

#### Change forecast

When your forecast is already placed you can change/update it.

```js
await forecasting.changeForecast(
  forecast,
  messageHash,
  {
    from: forecaster
  }
)
```

**Parameters:**
 - `forecast` - uint256 - forecasted amount
 - `messageHash` - bytes32 - ipfs hash of message (message is a buffered string)

#### Close/Cancel forecast

Depending on the stage of the forecasting you can either close or cancel forecast.

You can **close** forecast in following scenarios:
 - project was stopped by owner;
 - project was rejected by community;
 - crowdsale deadline missed;
 - after crowdsale period;

You can **cancel** forecast from the beginning of the forecasting and to the end of crowdsale period.

*NOTE: `closeForecast` is capable of performing both actions, hence it will automatically identify which one to perform depending on the current state of forecasting*

```js
await forecasting.closeForecast({ from: forecaster })
```
