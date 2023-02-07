In this learning module, we will follow a tutorial to connect a React app to the Algorand testnet and deploy and interact with a marketplace smart contract using the Algorand JavaScript SDK.

### Prerequisites

- You should have created an Algorand TEAL smart contract for the marketplace as described in our [Smart Contract Development](/CQV5jnJgT_2QfOK8sPjjpw) learning module.
-  Please make sure you have [Node JS](https://nodejs.org/en/download/) v16 or higher installed.
- You should have a basic understanding of [React](https://reactjs.org/): know how to use JSX, props, state, lifecycle methods, and hooks.

### Tech Stack

We will use the following tech stack:

- [React](https://reactjs.org/) - A JavaScript library for building user interfaces.
- [AlgoSDK](https://developer.algorand.org/docs/sdks/javascript/) - The official JavaScript library for communicating with the Algorand network.
- [MyAlgoConnect](https://connect.myalgo.com/) - A Javascript library to securely sign transactions with My Algo Wallet

## 1. Algorand testnet wallet
In this first section of the tutorial, we will set up a wallet on the Algorand testnet, which we will use later on to test the marketplace.
 
### 1.1 Create MyAlgo Wallet account
1. Go to https://wallet.myalgo.com/new-account, mark the checkbox that you read the terms and services and click continue. Please enter a password for your new account and make sure to remember it or to store it somewhere safe. Then click continue.
2. On the next page, make sure to select `TESTNET` from the drop-down menu in the top right corner. Then click the "New Account" button.
3. If you are not familiar with mnemonic phrases you can read about them using the link provided in the pop-up. Then click "continue".
4. Write down your mnemonic phrase in a safe location, mark the "I have written down and backed up my 25-word mnemonic phrase in the correct order" checkbox and click "continue".
5. Next, you must complete a small quiz where you must correctly name four words at specific positions of your mnemonic phrase. The idea of the quiz is to validate whether you correctly wrote down the mnemonic phrase.
6. After completing the quiz, you can enter a name for your account and click "Create Account".
7. Your testnet account is now created, and you should now see your wallet.

![](https://i.imgur.com/tyhvSyx.gif)


### 1.2 Add funding to testnet wallet
As you will need funding in the form of Algo tokens for interactions with the marketplace, we will need to add tokens to the newly created account. Algorand provides a testnet faucet to do so.
1. First, copy your account address to the clipboard by clicking the clipboard icon next to the account name and address on the wallet page.
2. Then open https://bank.testnet.algorand.network/, enter your account address as "target address", mark the "I am not a robot" checkbox and click "Dispense".
3. 10 Algo tokens should have been added to your account. Go back to the wallet and check if your balance shows 10 Algos. It can take up to 20 seconds for the tokens to show up. If you want, you can repeat step 2 to add additional funding.

If the dispensed funding doesn't show up after 20 seconds, make sure that you have selected the `TESTNET` environment in MyAlgo Wallet in the upper right corner and reload the page.


![](https://i.imgur.com/bw3IUva.gif)
    
That's it! You have successfully created your testnet wallet. Next, we are going to set up the React project. 
    
## 2. Project Setup

In the second section of this tutorial, we will set up the project and install the necessary dependencies.

Make sure that you have `nodejs` v16 or higher installed:

```bash
node -v
```

We will use the `create-react-app` utility to create a new React project:

```bash
npx create-react-app algorand-marketplace
```

Let's cd into the newly created project:

```bash
cd algorand-marketplace
```

Unfortunately, `react-scripts` of version 5 is not compatible with the Algorand JavaScript SDK yet, so we should use `react-scripts` of version 4.0.3:

```bash
npm install react-scripts@4.0.3
```

Also, we will downgrade `react` to version 17, as not all of the frontend libraries we will install in a later learning module are compatible with version 18 yet:

```bash
npm install react@17.0.2 react-dom@17.0.2 @testing-library/react@12.1.4 react-error-overlay@6.0.9
```

After downgrading to React 17 you have to change the content `src/index.js` file to the following:
```js
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';
import reportWebVitals from './reportWebVitals';

ReactDOM.render(
    <React.StrictMode>
        <App/>
    </React.StrictMode>,
    document.getElementById('root'),
);

// If you want to start measuring performance in your app, pass a function
// to log results (for example: reportWebVitals(console.log))
// or send to an analytics endpoint. Learn more: https://bit.ly/CRA-vitals
reportWebVitals();
```
Next, we need to install the `algosdk` library:

```bash
npm install algosdk
```

Also, we will install the `@randlabs/myalgo-connect` library, which we will use to connect to the algorand wallet and sign transactions:

```bash
npm install @randlabs/myalgo-connect
```

Finally, we will install `raw-loader`, which we will use to import the smart contract TEAL files which we created in the [Smart Contract Development](/CQV5jnJgT_2QfOK8sPjjpw) learning module:

```bash
npm install raw-loader --save-dev
```

Thats it! Now we can start the project and see if everything is working:

```bash
npm start
```

## 3. Configuring connection to Algorand

Now that we have a project, we can set up our connection to the Algroand testnet and test our smart contract using the JavaScript SDK.

We create a `utils` folder in the `src` directory with the `src/utils/constants.js` file to define the configuration for our connection to the Algorand network. For accessing the testnet we use public nodes provided by https://algoexplorer.io.

```js
import algosdk from "algosdk";
import MyAlgoConnect from "@randlabs/myalgo-connect";

const config = {
    algodToken: "",
    algodServer: "https://testnet-api.algonode.network",
    algodPort: "",
    indexerToken: "",
    indexerServer: "https://testnet-idx.algonode.network",
    indexerPort: "",
}

export const algodClient = new algosdk.Algodv2(config.algodToken, config.algodServer, config.algodPort)

export const indexerClient = new algosdk.Indexer(config.indexerToken, config.indexerServer, config.indexerPort);

export const myAlgoConnect = new MyAlgoConnect({
    timeout: 100000000,
});
```
First, we define a config, which specifies the token, server and port for the algod API and the indexer API.
Then, we create the `algodClient`, `indexerClient` and `myAlgoConnect`.
The `algodClient` will be used to perform transactions and retrieve account information.
The `indexerClient` allows searching the blockchain for certain transactions or applications.
Finally, `myAlgoConnect` allows us to connect the wallet we created in section 1 and sign transactions. We set a high timeout to prevent the Algorand request from timing out due to slow Algorand testnet nodes.


## 4. Implementing the Marketplace
Now that we have the configuration for communicating with the Algorand APIs, we can implement the interactions with the marketplace.

### 4.1 Smart contracts
First we create a `contracts` folder in the `src` directory and copy our `marketplace_approval.teal` and `marketplace_clear.teal` created in the [Smart Contract Development](/CQV5jnJgT_2QfOK8sPjjpw) learning module into the `src/contracts` directory.

### 4.2 Additional constants
For the implementation of the marketplace, we need some additional constants. Add the following to `src/utils/constants.js`

```js
// ...
export const minRound = 25556983;

// https://github.com/algorandfoundation/ARCs/blob/main/ARCs/arc-0002.md
export const marketplaceNote = "tutorial-marketplace:uv1"

// Maximum local storage allocation, immutable
export const numLocalInts = 0;
export const numLocalBytes = 0;
// Maximum global storage allocation, immutable
export const numGlobalInts = 2; // Global variables stored as Int: count, sold
export const numGlobalBytes = 3; // Global variables stored as Bytes: name, description, image
```

We define a `minRound` variable, which we will use later on to limit the search for transactions up to a specific round.
Next, we define the a`marketplaceNote`, which is required to find products of the marketplace later on.
Finally, we define the values `numLocalInts`, `numLocalBytes`, `numGlobalInts` `numGlobalBytes`, which will specify the storage allocation of a smart contract.

### 4.3 Conversions
Next, we create a `src/utils/conversions.js` file, that will contain functions which we will need for for converting strings to interact with the smart contract.
```js
export const base64ToUTF8String = (base64String) => {
    return Buffer.from(base64String, 'base64').toString("utf-8")
}

export const utf8ToBase64String = (utf8String) => {
    return Buffer.from(utf8String, 'utf8').toString('base64')
}
```

### 4.4 Marketplace functionality
Now, we create a `src/utils/marketplace.js` file, that will contain the functions that we will use to interact our smart contract and start with the imports and the definition of a `Product` class:
```js
import algosdk from "algosdk";
import {
    algodClient,
    indexerClient,
    marketplaceNote,
    minRound,
    myAlgoConnect,
    numGlobalBytes,
    numGlobalInts,
    numLocalBytes,
    numLocalInts
} from "./constants";
/* eslint import/no-webpack-loader-syntax: off */
import approvalProgram from "!!raw-loader!../contracts/marketplace_approval.teal";
import clearProgram from "!!raw-loader!../contracts/marketplace_clear.teal";
import {base64ToUTF8String, utf8ToBase64String} from "./conversions";

class Product {
    constructor(name, image, description, price, sold, appId, owner) {
        this.name = name;
        this.image = image;
        this.description = description;
        this.price = price;
        this.sold = sold;
        this.appId = appId;
        this.owner = owner;
    }
}
```
We import `algosdk`, as well as the constants and conversion functions we defined in the previous steps. Also, we import our `.teal` files as `approvalProgram` and `clearProgram`.

Then, we define a `Product` class, with the variables that define a product in the marketplace.

#### 4.4.1 Create Product
Next we implement a `createProductAction` function, which will be used to deploy a smart contract to the blockchain. We also add the helper function `compileProgram`. Please add the following to `src/utils/marketplace.js`: 
```js
//...
// Compile smart contract in .teal format to program
const compileProgram = async (programSource) => {
    let encoder = new TextEncoder();
    let programBytes = encoder.encode(programSource);
    let compileResponse = await algodClient.compile(programBytes).do();
    return new Uint8Array(Buffer.from(compileResponse.result, "base64"));
}

// CREATE PRODUCT: ApplicationCreateTxn
export const createProductAction = async (senderAddress, product) => {
    console.log("Adding product...")

    let params = await algodClient.getTransactionParams().do();
    params.fee = algosdk.ALGORAND_MIN_TX_FEE;
    params.flatFee = true;

    // Compile programs
    const compiledApprovalProgram = await compileProgram(approvalProgram)
    const compiledClearProgram = await compileProgram(clearProgram)

    // Build note to identify transaction later and required app args as Uint8Arrays
    let note = new TextEncoder().encode(marketplaceNote);
    let name = new TextEncoder().encode(product.name);
    let image = new TextEncoder().encode(product.image);
    let description = new TextEncoder().encode(product.description);
    let price = algosdk.encodeUint64(product.price);

    let appArgs = [name, image, description, price]

    // Create ApplicationCreateTxn
    let txn = algosdk.makeApplicationCreateTxnFromObject({
        from: senderAddress,
        suggestedParams: params,
        onComplete: algosdk.OnApplicationComplete.NoOpOC,
        approvalProgram: compiledApprovalProgram,
        clearProgram: compiledClearProgram,
        numLocalInts: numLocalInts,
        numLocalByteSlices: numLocalBytes,
        numGlobalInts: numGlobalInts,
        numGlobalByteSlices: numGlobalBytes,
        note: note,
        appArgs: appArgs
    });

    // Get transaction ID
    let txId = txn.txID().toString();

    // Sign & submit the transaction
    let signedTxn = await myAlgoConnect.signTransaction(txn.toByte());
    console.log("Signed transaction with txID: %s", txId);
    await algodClient.sendRawTransaction(signedTxn.blob).do();

    // Wait for transaction to be confirmed
    let confirmedTxn = await algosdk.waitForConfirmation(algodClient, txId, 4);

    // Get the completed Transaction
    console.log("Transaction " + txId + " confirmed in round " + confirmedTxn["confirmed-round"]);

    // Get created application id and notify about completion
    let transactionResponse = await algodClient.pendingTransactionInformation(txId).do();
    let appId = transactionResponse['application-index'];
    console.log("Created new app-id: ", appId);
    return appId;
}
```
The `createProductAction` function takes the parameters `senderAdress`, which is the address of the creator of the product and an object `product`, which contains the data for the new product. The function looks like a lot of code but is quite straightforward.
    
First, transaction parameters, including the transaction fee, are configured.
Then, the teal files are compiled, as the algod API requires them as `Uint8Array`. Next, the note and the app args (which contain the variables for the newly created product) are also converted to `Uint8Array`.
    
Next, the `ApplicationCreateTxn` is built. Here, the sender, params, compiled approval and clear program, arguments, note and the storage allocation settings declared in `constants.js` are passed.

Afterwards, the transaction is signed using myAlgoConnect. The `signTransaction` function will open a pop-up, requiring the user to enter their wallet password and approve the transaction.

Then, the transaction is sent, and its confirmation is awaited.

Finally, the `appId` of the newly created app is returned.

#### 4.4.2 Buy product
As a next step, we will implement a `buyProductAction` function which is used to buy a product that was added to the marketplace using the `createProductAction` function. In our marketplace smart contract developed in [Smart Contract Development](/CQV5jnJgT_2QfOK8sPjjpw) learning module buying a product is done by grouping two different transactions together in a group transaction. A `ApplicationCallTx` and a `PaymentTxn`. Please add the following to `src/utils/marketplace.js`:

```js
//...
// BUY PRODUCT: Group transaction consisting of ApplicationCallTxn and PaymentTxn
export const buyProductAction = async (senderAddress, product, count) => {
    console.log("Buying product...");

    let params = await algodClient.getTransactionParams().do();
    params.fee = algosdk.ALGORAND_MIN_TX_FEE;
    params.flatFee = true;

    // Build required app args as Uint8Array
    let buyArg = new TextEncoder().encode("buy")
    let countArg = algosdk.encodeUint64(count);
    let appArgs = [buyArg, countArg]

    // Create ApplicationCallTxn
    let appCallTxn = algosdk.makeApplicationCallTxnFromObject({
        from: senderAddress,
        appIndex: product.appId,
        onComplete: algosdk.OnApplicationComplete.NoOpOC,
        suggestedParams: params,
        appArgs: appArgs
    })

    // Create PaymentTxn
    let paymentTxn = algosdk.makePaymentTxnWithSuggestedParamsFromObject({
        from: senderAddress,
        to: product.owner,
        amount: product.price * count,
        suggestedParams: params
    })

    let txnArray = [appCallTxn, paymentTxn]

    // Create group transaction out of previously build transactions
    let groupID = algosdk.computeGroupID(txnArray)
    for (let i = 0; i < 2; i++) txnArray[i].group = groupID;

    // Sign & submit the group transaction
    let signedTxn = await myAlgoConnect.signTransaction(txnArray.map(txn => txn.toByte()));
    console.log("Signed group transaction");
    let tx = await algodClient.sendRawTransaction(signedTxn.map(txn => txn.blob)).do();

    // Wait for group transaction to be confirmed
    let confirmedTxn = await algosdk.waitForConfirmation(algodClient, tx.txId, 4);

    // Notify about completion
    console.log("Group transaction " + tx.txId + " confirmed in round " + confirmedTxn["confirmed-round"]);
}
```
The `buyProductAction` function takes the parameters `senderAdress`, which is the address of the buyer of the product, an object `product`, which is the product that will be bought, and an integer variable `count`, specifying how many products will be bought. 

Then, again, first, the parameters and arguments for the transactions are built. Next, the two transactions required for the group transaction are created. We send the payment to the product owner and use the price multiplied by the count as the amount. Next, the transactions are grouped, and a `groupID` is computed. 

Finally, the transaction is sent, and its confirmation is awaited.

#### 4.4.3 Delete product
We go on with implementing a `deleteProductAction` function. Its goal is to delete a product represented by an application by sending a `ApplicationDeleteTxn`. Please add the following to `src/utils/marketplace.js`:
    
```js
//...
// DELETE PRODUCT: ApplicationDeleteTxn
export const deleteProductAction = async (senderAddress, index) => {
    console.log("Deleting application...");

    let params = await algodClient.getTransactionParams().do();
    params.fee = algosdk.ALGORAND_MIN_TX_FEE;
    params.flatFee = true;

    // Create ApplicationDeleteTxn
    let txn = algosdk.makeApplicationDeleteTxnFromObject({
        from: senderAddress, suggestedParams: params, appIndex: index,
    });

    // Get transaction ID
    let txId = txn.txID().toString();

    // Sign & submit the transaction
    let signedTxn = await myAlgoConnect.signTransaction(txn.toByte());
    console.log("Signed transaction with txID: %s", txId);
    await algodClient.sendRawTransaction(signedTxn.blob).do();

    // Wait for transaction to be confirmed
    const confirmedTxn = await algosdk.waitForConfirmation(algodClient, txId, 4);

    // Get the completed Transaction
    console.log("Transaction " + txId + " confirmed in round " + confirmedTxn["confirmed-round"]);

    // Get application id of deleted application and notify about completion
    let transactionResponse = await algodClient.pendingTransactionInformation(txId).do();
    let appId = transactionResponse['txn']['txn'].apid;
    console.log("Deleted app-id: ", appId);
}
```

The `deleteProductAction` function takes the parameters `senderAddress`, which is the address of the account that wants to delete the product and an `index` which is the appId of the product that should be deleted.
The logic is similar to the `createProductAction` and `buyProductAction`. After building the transaction params, the `ApplicationDeleteTxn` is created, signed, submitted, and its confirmation awaited.

#### 4.4.4 Get products
As a final function for our marketplace, we need to implement a `getProductsAction` function, searching for available products on the blockchain, fetching them, and returning them. For this, we will utilise Algorand's [Indexer API](https://developer.algorand.org/docs/get-details/indexer/), which can be accessed using the `indexerClient` created in our `constants.js` file.
Please add the following to `src/utils/marketplace.js`:
```js
//...
// GET PRODUCTS: Use indexer
export const getProductsAction = async () => {
    console.log("Fetching products...")
    let note = new TextEncoder().encode(marketplaceNote);
    let encodedNote = Buffer.from(note).toString("base64");

    // Step 1: Get all transactions by notePrefix (+ minRound filter for performance)
    let transactionInfo = await indexerClient.searchForTransactions()
        .notePrefix(encodedNote)
        .txType("appl")
        .minRound(minRound)
        .do();
    let products = []
    for (const transaction of transactionInfo.transactions) {
        let appId = transaction["created-application-index"]
        if (appId) {
            // Step 2: Get each application by application id
            let product = await getApplication(appId)
            if (product) {
                products.push(product)
            }
        }
    }
    console.log("Products fetched.")
    return products
}

const getApplication = async (appId) => {
    try {
        // 1. Get application by appId
        let response = await indexerClient.lookupApplications(appId).includeAll(true).do();
        if (response.application.deleted) {
            return null;
        }
        let globalState = response.application.params["global-state"]

        // 2. Parse fields of response and return product
        let owner = response.application.params.creator
        let name = ""
        let image = ""
        let description = ""
        let price = 0
        let sold = 0

        const getField = (fieldName, globalState) => {
            return globalState.find(state => {
                return state.key === utf8ToBase64String(fieldName);
            })
        }

        if (getField("NAME", globalState) !== undefined) {
            let field = getField("NAME", globalState).value.bytes
            name = base64ToUTF8String(field)
        }

        if (getField("IMAGE", globalState) !== undefined) {
            let field = getField("IMAGE", globalState).value.bytes
            image = base64ToUTF8String(field)
        }

        if (getField("DESCRIPTION", globalState) !== undefined) {
            let field = getField("DESCRIPTION", globalState).value.bytes
            description = base64ToUTF8String(field)
        }

        if (getField("PRICE", globalState) !== undefined) {
            price = getField("PRICE", globalState).value.uint
        }

        if (getField("SOLD", globalState) !== undefined) {
            sold = getField("SOLD", globalState).value.uint
        }

        return new Product(name, image, description, price, sold, appId, owner)
    } catch (err) {
        return null;
    }
}
```

First, we will look for transactions with which products got created. We are doing this by performing a [note field search](https://developer.algorand.org/docs/get-details/indexer/?from_query=note%20field%20searc#note-field-searching) on all transactions using `indexerClient.searchForTransactions().notePrefix(encodedNote)`.
As an additional filter we only include application related transactions with `.txType("appl")`. As searching all transactions would take too long, we also add a `minRound` filter to limit the search up to a certain point in the chain's history. Once we have all transactions containing the `appId`s of the created applications, we fetch each application using `indexerClient.lookupApplications(appId)`. If the application is already deleted, we return null. Otherwhise, we use the response to extract the application's global state and associate the base64 encoded state keys to our product variables. The values of the keys are used to build our product object. Values, which were stored as bytes, are encoded as base64 strings and have to be converted to utf8. Integer values can be accessed directly.

Finally, we build a product object, add it to the products array and return all products.

### 4.5 Adding the Marketplace to the app

Lastly, we are going to add the functionality of the marketplace to our app. Open the `src/App.js` file and change the code to the following:

```js
import React, {useEffect, useState} from "react";
import './App.css';
import MyAlgoConnect from "@randlabs/myalgo-connect";
import {getProductsAction} from "./utils/marketplace";

const App = function AppWrapper() {

    const [address, setAddress] = useState(null);
    const [products, setProducts] = useState([]);

    const connectWallet = async () => {
        new MyAlgoConnect().connect()
            .then(accounts => {
                const _account = accounts[0];
                setAddress(_account.address);
            }).catch(error => {
            console.log('Could not connect to MyAlgo wallet');
            console.error(error);
        })
    };

    useEffect(() => {
        getProductsAction().then(products => {
            setProducts(products)
        });
    }, []);

    return (
        <>
            {address ? (
                products.forEach((product) => console.log(product))
            ) : (
                <button onClick={connectWallet}>CONNECT WALLET</button>
            )}
        </>
    );
}

export default App;
```

This code is just for testing purposes. We try to connect to the wallet, and if we are connected, we fetch the products. Then we display the products in the console, making use of the `getProductsAction` function that we just created.
If we are not connected, we display a button that will connect to the wallet by calling the `connectWallet` function defined above.

Now you can start the app:

```bash
npm start
```

And you should see something like this:

![](https://i.imgur.com/jKdyJLU.gif)

Great! Now you can see the products in the console. Continue the next learning module to learn how to build the fronted components for your marketplace dapp.
