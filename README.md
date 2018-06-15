# Wings ABI

Manual how to use Wings contracts ABI to create and manage your project.

## Introduction

In this manual we will be using `Node.js`, `web3` (^0.20.6) and `truffle-contract` to operate with contracts.

In order to use Wings contracts you need to have contracts ABI.  
In the `./abi` folder you can find the following contracts artifacts:
 - `Wings.json`
 - `DAO.json`
 - `CrowdsaleController.json`

Here is an example of how to initiate `Wings` contract:

```js
const contract = require('truffle-contract')
const Web3 = require('web3')

const wingsArtifact = require('./abi/Wings.json')

const Wings = contract.at(wingsArtifact)

Wings.setProvider(new Web3.providers.HttpProvider(web3Provider))

const wingsAddress = '0x7ea8dc2b2b00b596d077b68f5c891e03797a5eb2'

const wings = Wings.at(wingsAddress)
```

# Step by step

### 1. Create DAO

First step in project creation process is creating a DAO. DAO is a main contract in your project hierarchy.

```js
await wings.createDAO(name, tokenName, tokenSymbol, infoHash, customCrowdsale, { from: creator })
```

**Parameters:**
 - `name` - string - name of your project
 - `tokenName` - string - name of project token
 - `tokenSymbol` - string - symbol of project token
 - `infoHash` - bytes32 - decoded ipfs hash of project description
 - `customCrowdsale` - address - address of custom crowdsale (`"0"` in case of standard crowdsale)

#### Uploading your project description and media to ipfs

Head to [Media file format](https://github.com/WingsDao/ABI#media-file-format) paragraph in the Appendix section.

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
 - `ethRewardPart` - uint256 - reward percent of total collected Ether
 - `tokenRewardPart` - uint256 - reward percent of total sold tokens

*NOTE: reward percent must be multiplied by 10000.*

*Example: reward is 1.5% the argument must be passed as 15000.*

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

## In case you are using Custom Crowdsale, skip step 7 and head to the step 8.1.

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
 - `prices1to4` - uint256 -
 - `prices5to8` - uint256 -

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

## Additional functions

### update

This DAO method allows you to change project description during forecasting period.

```js
await dao.update(infoHash, { from: creator })
```

**Parameters:**
 - `infoHash` - bytes32 - decoded ipfs hash of updated project description

## Appendix

### Media file format

File that contains project media has to be in JSON format.

**Example:**
```json
{
  "version": "1.0.0",
  "shortBlurb": "Decentralized blockchain dedicated to auctions in real-time",
  "story": "{\"ops\": [{\"insert\":\"test\"},{\"insert\":{\"video\":\"https://www.youtube.com/embed/fy2XDBbDrAs?showinfo=0\"}}]}",
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
        "videoId": "Ztn3-qhSpxo",
        "videoType": "youtube"
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
```json
{
  "$id": "http://example.com/example.json",
  "type": "object",
  "definitions": {},
  "$schema": "http://json-schema.org/draft-07/schema#",
  "properties": {
    "version": {
      "type": "string"
    },
    "shortBlurb": {
      "type": "string"
    },
    "story": {
      "type": "string"
    },
    "category": {
      "type": "integer"
    },
    "gallery": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "type": {
            "type": "string"
          },
          "content": {
            "type": "object",
            "properties": {
              "videoType": {
                "type": "string"
              },
              "videoId": {
                "type": "string"
              }
            }
          },
          "src": {
            "type": "string"
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
- *story* - description of project in [Delta](https://github.com/quilljs/delta) format.
- *category* - id of category (see the list below).
- *gallery* - gallery description of project.
- *gallery/type* - type of content. Options: video, image, logo, terms.
- *gallery/video/content/videoId* -  id of video from video hosting.
- *gallery/video/content/videoType* - type of video hosting. Options: youtube, vimeo, youku.
- *gallery/image/content/hash* - ipfs hash of image uploaded to ipfs.
- *gallery/logo/content/contentType* - type of image format. Options: `image/png`, `image/jpg`.
- *gallery/logo/content/hash* - ipfs hash of logo uploaded to ipfs. Only jpg/png allowed; size < 1 mb.
- *gallery/terms/content/hash* - ipfs hash of terms uploaded to ipfs. Only pdf allowed; size < 1 mb.

#### Categories:
0. Other
1. Supply chain
2. Security
3. Service
4. Transportation
5. Entertainment
6. Education
7. Media
8. Retail
9. Finance
10. Insurance
11. Hardware
12. Science
13. Utilities
14. Charity

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
