In this learning module, we will follow a tutorial to build the frontend for a marketplace contract.
This tutorial assumes that you have already completed the [Connect a React Dapp to Algorand](/zrXXmmJpS_Sa-4x6c7bV4w)  learning module and continue in the same project.

### Prerequisites

- [Node JS](https://nodejs.org/en/download/) - Please ensure you have Node.js v16 or higher installed.
- You should have a basic understanding of [React](https://reactjs.org/): know how to use JSX, props, state, lifecycle methods, and hooks.
- You should have followed the [Connect a React Dapp to Algorand](/zrXXmmJpS_Sa-4x6c7bV4w) learning module and have `react` v17.0.2, `react-scripts` v4.0.3, `algosdk` and `@randlabs/myalgo-connect` installed.

### Tech Stack

We will use the following tech stack:

- [React](https://reactjs.org/) - A JavaScript library for building user interfaces.
- [Bootstrap](https://getbootstrap.com/) - A CSS framework.
- [AlgoSDK](https://developer.algorand.org/docs/sdks/javascript/) - The official JavaScript library for communicating with the Algorand network.
- [MyAlgoConnect](https://connect.myalgo.com/) - A Javascript library to securely sign transactions with My Algo Wallet

## 1. Project Setup

Since we already have the important dependencies installed, we now only need to add our dependencies for the frontend, styling and formatting numbers:

```bash
npm install react-bootstrap bootstrap bootstrap-icons react-toastify prop-types react-jazzicon
```

We will use `react-bootstrap` to handle the `Bootstrap` styling of our react components. We will use `react-toastify` to display notifications to the user, so we don't have to handle that ourselves. We will use `prop-types` to define component props as required. We will use `react-jazzicon` to display a wallet address as an identicon. Finally, we will use `bignumber.js` to format numbers.


### 1.1 index.js

Let's open the `src/index.js` file and start adding our bootstrap component and styles:

```js
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';
import reportWebVitals from './reportWebVitals';
import 'bootstrap-icons/font/bootstrap-icons.css';
import 'bootstrap/dist/css/bootstrap.min.css';
import "react-toastify/dist/ReactToastify.min.css";

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

### 1.2 utils

The `src/utils/` directory should look like this if you followed the [Connect a React Dapp to Algorand](/zrXXmmJpS_Sa-4x6c7bV4w) learning module:

```
├── utils
│   ├── constants.js
│   ├── conversions.js
│   └── marketplace.js
```

#### 1.2.2 constants.js
We are going to expand the `util/constants.js` file with one more variable, which we will need for the next step:
```json
//...
export const ALGORAND_DECIMALS = 6;
```
The Algorand blockchain handels amounts in microAlgos. In order to display amounts in the frontend as Algos, we have to shift the comma by 6 decimals.

#### 1.2.2 conversions.js
We are going to expand `utils/conversions.js` with three more functions, which allow us to format a wallet address and convert amounts from Algos to microAlgos and vice versa. We will need these functions for the next steps.
```js
import {ALGORAND_DECIMALS} from "./constants";
import BigNumber from "bignumber.js";

//...

// Truncate is done in the middle to allow for checking of first and last chars simply to ensure correct address
export const truncateAddress = (address) => {
    if (!address) return
    return address.slice(0, 5) + "..." + address.slice(address.length - 5, address.length);
}

// Amounts in microAlgos (e.g. 10500) are shown as algos (e.g. 10.5) in the frontend
export const microAlgosToString = (num) => {
    if (!num) return
    let bigNumber = new BigNumber(num)
    return bigNumber.shiftedBy(-ALGORAND_DECIMALS).toFixed(3);
}

// Convert an amount entered as algos (e.g. 10.5) to microAlgos (e.g. 10500)
export const stringToMicroAlgos = (str) => {
    if (!str) return
    let bigNumber = new BigNumber(str)
    return bigNumber.shiftedBy(ALGORAND_DECIMALS).toNumber();
}
```
The `truncateAddress` function truncates an address to allow for checking of first and last chars simply to ensure the correct address.

The `microAlgosToString` and `stringToMicroAlgos` functions utilise `bignumber.js` to shift the comma of microAlgos or Algos value.

Now that we have our helper functions, we can start implementing our UI components.

### 1.3 App.js

We will set up the `App.js` file to render our UI. You can clear the content of the `App.js` created in the previous learning module, as we will fresh. Let's start with the imports:

```js
import React, {useState} from "react";
import Cover from "./components/Cover";
import './App.css';
import Wallet from "./components/Wallet";
import {Container, Nav} from "react-bootstrap";
// import Products from "./components/marketplace/Products";
// import {Notification} from "./components/utils/Notifications";
import {indexerClient, myAlgoConnect} from "./utils/constants";
import coverImg from "./assets/img/sandwich.jpg"
//..
```

We import the `Wallet` and `Cover` components and the `coverImg` image file. All of which have not been created yet. We also import our `algodClient` and `myAlgoConnect` from `src/utils/constants.js`

For now, the `Notification` and `Products` components will remain uncommented as we will implement them later.

Let's create our `App` component now:

```js
//..
const App = function AppWrapper() {

    const [address, setAddress] = useState(null);
    const [name, setName] = useState(null);
    const [balance, setBalance] = useState(0);

    const fetchBalance = async (accountAddress) => {
        indexerClient.lookupAccountByID(accountAddress).do()
            .then(response => {
                const _balance = response.account.amount;
                setBalance(_balance);
            })
            .catch(error => {
                console.log(error);
            });
    };

    const connectWallet = async () => {
        myAlgoConnect.connect()
            .then(accounts => {
                const _account = accounts[0];
                setAddress(_account.address);
                setName(_account.name);
                fetchBalance(_account.address);
            }).catch(error => {
            console.log('Could not connect to MyAlgo wallet');
            console.error(error);
        })
    };

    const disconnect = () => {
        setAddress(null);
        setName(null);
        setBalance(null);
    };
//..
```

The `connectWallet` function uses `myAlgoConnect` to connect to the wallet and then sets the `account`. It also calls `fetchBalance`, which utilises the `algodClient` to retrieve the account information and set the `balance`. Finally, the disconnect function disconnects our app from the wallet by setting address, name and balance to `null`.

Let's return the JSX for our `App` component:

```js
//..
   return (
        <>
            {/* <Notification /> */}
            {address ? (
                <Container fluid="md">
                    <Nav className="justify-content-end pt-3 pb-5">
                        <Nav.Item>
                            <Wallet
                                address={address}
                                name={name}
                                amount={balance}
                                disconnect={disconnect}
                                symbol={"ALGO"}
                            />
                        </Nav.Item>
                    </Nav>
                    <main>
                        {/* <Products address={address} fetchBalance={fetchBalance}/> */}
                    </main>
                </Container>
            ) : (
                <Cover name={"Street Food"} coverImg={coverImg} connect={connectWallet}/>
            )}
        </>
    );
}

export default App;
```

If the user is connected to the wallet, we display our dapp. If they aren't, we render the `Cover` component.

We pass a `name` for our dapp and a `coverImg` as props to the `Cover` component. We also pass a `connect` function to connect to the wallet.

The dapp consists of the `Wallet` component, which displays the user's account address and balance. We also show the `Products` component, which we will implement later.
The `Wallet` needs the account address, the user's balance, a symbol for the currency we display, and a `disconnect` function to log out of the wallet as props.

Now let's create the components we already used.

## 2. Components

In this tutorial section, we will create the custom components we will use in our dapp.

The components directory will look like this:

```
├── components
│   ├── marketplace
│   ├── utils
│   ├── Cover.jsx
│   └── Wallet.jsx
```

We will start with the `Cover` component.

### 2.1 Cover.jsx

Create a `components` folder in the `src` directory and create a `src/components/Cover.jsx` file with the following code:

```js
import React from 'react';
import {Button} from "react-bootstrap";
import PropTypes from 'prop-types';

const Cover = ({name, coverImg, connect}) => {
    return (
        <div className="d-flex justify-content-center flex-column text-center bg-black min-vh-100">
            <div className="mt-auto text-light mb-5">
                <div
                    className=" ratio ratio-1x1 mx-auto mb-2"
                    style={{maxWidth: "320px"}}
                >
                    <img src={coverImg} alt=""/>
                </div>
                <h1>{name}</h1>
                <p>Please connect your wallet to continue.</p>
                <Button
                    onClick={() => connect()}
                    variant="outline-light"
                    className="rounded-pill px-3 mt-3"
                >
                    Connect Wallet
                </Button>
            </div>
            <p className="mt-auto text-secondary">Powered by Algorand</p>
        </div>
    );
};

Cover.propTypes = {
    name: PropTypes.string,
    coverImg: PropTypes.string,
    connect: PropTypes.func
};

export default Cover;
```

This component is pretty simple. If it receives the `name`, `connect` and `coverImg` as props, we render the `coverImg` and the `name` of the dapp. We also display a `Connect Wallet` button that calls the `connect` function when clicked.

#### 2.1.1 Cover Image

Since we are using a cover image, we need to import the `coverImg` image file.
For this tutorial, we chose a `sandwich.jpg` image that you can find [here](https://github.com/dacadeorg/algorand-react-marketplace/blob/master/src/assets/img/sandwich.jpg). We create a `src/assets/img` directory and store the image there `src/assets/img/sandwich.jpg`.

Now let's continue with the `Identicon` component.

### 2.2 Indenticon.jsx
The indenticon component will be used to display a visual representation of a wallet address to identify an account's address quickly.

Create a `utils` folder in the `components` directory and create a `src/components/utils/Indenticon.jsx` file with the following code:

```js
import Jazzicon from "react-jazzicon";
import PropTypes from "prop-types";

const Identicon = ({size, address, ...rest}) => (
    <div {...rest} style={{width: `${size}px`, height: `${size}px`}}>
        <Jazzicon diameter={size} seed={parseInt(address.slice(2, 10), 16)}/>
    </div>
);

Identicon.propTypes = {
    size: PropTypes.number.isRequired,
    address: PropTypes.string.isRequired
};

export default Identicon;
```

We receive a `size` and account `address` which will be used to create the identicon. To modify the components' classes and style from where the component is used, we also take additional props as `...rest`.

Next, we will create the `Wallet` component.

### 2.3 Wallet.jsx

The wallet component will display the user's account address, name, identicon, balance, and logout button. Create a `components/Wallet.jsx` file with the following code:

```js
import React from 'react';
import {Dropdown, Spinner, Stack} from 'react-bootstrap';
import {microAlgosToString, truncateAddress} from '../utils/conversions';
import Identicon from './utils/Identicon'
import PropTypes from "prop-types";

const Wallet = ({address, name, amount, symbol, disconnect}) => {
    if (!address) {
        return null;
    }
    return (
        <>
            <Dropdown>
                <Dropdown.Toggle variant="light" align="end" id="dropdown-basic"
                                 className="d-flex align-items-center border rounded-pill py-1">
                    {amount ? (
                        <>
                            {microAlgosToString(amount)}
                            <span className="ms-1"> {symbol}</span>
                        </>
                    ) : (
                        <Spinner animation="border" size="sm" className="opacity-25"/>
                    )}
                    <Identicon address={address} size={28} className="ms-2 me-1"/>
                </Dropdown.Toggle>

                <Dropdown.Menu className="shadow-lg border-0">
                    <Dropdown.Item href={`https://testnet.algoexplorer.io/address/${address}`}
                                   target="_blank">
                        <Stack direction="horizontal" gap={2}>
                            <i className="bi bi-person-circle fs-4"/>
                            <div className="d-flex flex-column">
                                {name && (<span className="font-monospace">{name}</span>)}
                                <span className="font-monospace">{truncateAddress(address)}</span>
                            </div>
                        </Stack>
                    </Dropdown.Item>
                    <Dropdown.Divider/>
                    <Dropdown.Item as="button" className="d-flex align-items-center" onClick={() => {
                        disconnect();
                    }}>
                        <i className="bi bi-box-arrow-right me-2 fs-4"/>
                        Disconnect
                    </Dropdown.Item>
                </Dropdown.Menu>
            </Dropdown>
        </>
    )
};

Wallet.propTypes = {
    address: PropTypes.string,
    name: PropTypes.string,
    amount: PropTypes.number,
    symbol: PropTypes.string,
    disconnect: PropTypes.func
};

export default Wallet;

```

We receive the `address`, the account `name`, the user balance (`amount`), and the `symbol` of the currency we display as props. We also receive a `disconnect` function to log out of the wallet. As described earlier, these are passed from the `App` component.

Now we should be ready to run our dapp and see if we can log in, log out, and see our balance and address. This only works if you have a created a MyAlgo Wallet account with funding on it, as descriped in our [Connect a React Dapp to Algorand](/zrXXmmJpS_Sa-4x6c7bV4w) learning module.

Run the dapp:

```bash
npm start
```

The app should now look and behave like this:

![](https://i.imgur.com/b1bCxk0.gif)

### 2.4 utils

Let's next work on some more utility components. The `utils` directory will look like this:

```
├── components
│   ├── utils
│   │   ├── Identicon.jsx
│   │   ├── Loader.jsx
│   │   └── Notifications.jsx
```

We already have the `Identicon` component. Let's create the `Loader` component next.

#### 2.4.1 utils/Loader.jsx

The `Loader` component will display a loading animation. Create a new `components/utils/Loader.jsx` file with the following code:

```js
import React from "react";
import {Spinner} from "react-bootstrap";

const Loader = () => (
    <div className="d-flex justify-content-center">
        <Spinner animation="border" role="status" className="opacity-25">
            <span className="visually-hidden">Loading...</span>
        </Spinner>
    </div>
);

export default Loader;
```

This component is pretty simple. It just displays the bootstrap `Spinner` component.

#### 2.4.2 utils/Notifications.jsx

The `Notifications` component will be used to display notifications to the user.
Create a new `components/utils/Notifications.jsx` file with the following code:

```js
import React from "react";
import {ToastContainer} from "react-toastify";
import PropTypes from "prop-types";

const Notification = () => (
    <ToastContainer
        position="bottom-center"
        autoClose={5000}
        hideProgressBar
        newestOnTop
        closeOnClick
        rtl={false}
        pauseOnFocusLoss
        draggable={false}
        pauseOnHover
    />
);

const NotificationSuccess = ({text}) => (
    <div>
        <i className="bi bi-check-circle-fill text-success mx-2"/>
        <span className="text-secondary mx-1">{text}</span>
    </div>
);

const NotificationError = ({text}) => (
    <div>
        <i className="bi bi-x-circle-fill text-danger mx-2"/>
        <span className="text-secondary mx-1">{text}</span>
    </div>
);

const Props = {
    text: PropTypes.string,
};

const DefaultProps = {
    text: "",
};

NotificationSuccess.propTypes = Props;
NotificationSuccess.defaultProps = DefaultProps;

NotificationError.propTypes = Props;
NotificationError.defaultProps = DefaultProps;

export {Notification, NotificationSuccess, NotificationError};
```

This component uses the `react-toastify` library to display notifications.
We distinguish between success and error notifications and otherwise display the `text` as a string.
The `Notification` component is implemented in the `App` component.

### 2.5 marketplace

Now we create our final component, where we will create the UI for the marketplace. The `components/marketplace` directory will look like this:

```
├── components
│   ├── marketplace
│   │   ├── AddProduct.jsx
│   │   ├── Product.jsx
│   │   └── Products.jsx
│   ├── utils
│   ├── Cover.jsx
│   └── Wallet.jsx
```

Let's start with the `Products` component.

#### 2.5.1 Products.jsx

The `Products` component will display a list of products.
It will be the main component of the marketplace that will contain the `AddProducts` and `Product` components.
Create a new `marketplace` folder in the `components` directory and create a `components/marketplace/Products.jsx` file.

Let's start with the imports:

```js
import React, {useEffect, useState} from "react";
import {toast} from "react-toastify";
import AddProduct from "./AddProduct";
import Product from "./Product";
import Loader from "../utils/Loader";
import {NotificationError, NotificationSuccess} from "../utils/Notifications";
import {buyProductAction, createProductAction, deleteProductAction, getProductsAction,} from "../../utils/marketplace";
import PropTypes from "prop-types";
import {Row} from "react-bootstrap";
//...
```

We import the `AddProduct` and `Product` components that we will create later.
We also import the `Loader` and `NotificationSuccess` and `NotificationError` components from the `utils` directory.
Finally, we import the `buyProductAction`, `createProductAction`,  `deleteProductAction` and `getProductsAction` utility functions from the `src/utils/marketplace.js` file we created in the [Connect a React Dapp to Algorand](/zrXXmmJpS_Sa-4x6c7bV4w) learning module.

Let's create our main `Products` component and a `getProducts` function, which we use to fetch the list of products:

```js
//...
const Products = ({address, fetchBalance}) => {
    const [products, setProducts] = useState([]);
    const [loading, setLoading] = useState(false);
    
    const getProducts = async () => {
        setLoading(true);
        getProductsAction()
            .then(products => {
                if (products) {
                    setProducts(products);
                }
            })
            .catch(error => {
                console.log(error);
            })
            .finally(_ => {
                setLoading(false);
            });
    };

    useEffect(() => {
        getProducts();
    }, []);
//...
```
As props, the products component takes the `address` of the connected wallet and a `fetchBalance` function to refresh the balance after performing interactions with the blockchain.


We set the `loading` state to `true` when we fetch the products and set the `products` state to the list of products. We use the `getProductsAction` utility that we imported earlier to fetch the products.
Once we have the products, we set the `loading` state to `false`.

Next we create the `createProduct` function:

```js
//...
	const createProduct = async (data) => {
	    setLoading(true);
	    createProductAction(address, data)
	        .then(() => {
	            toast(<NotificationSuccess text="Product added successfully."/>);
	            getProducts();
	            fetchBalance(address);
	        })
	        .catch(error => {
	            console.log(error);
	            toast(<NotificationError text="Failed to create a product."/>);
	            setLoading(false);
	        })
	};
//...
```

We receive the product data as a parameter and use the `createProductAction` utility function to create the product. We then fetch the products again and display a success message or an error message if the product could not be created.

Also, we create a `buyProduct` function:

```js
//...
	const buyProduct = async (product, count) => {
	    setLoading(true);
	    buyProductAction(address, product, count)
	        .then(() => {
	            toast(<NotificationSuccess text="Product bought successfully"/>);
	            getProducts();
	            fetchBalance(address);
	        })
	        .catch(error => {
	            console.log(error)
	            toast(<NotificationError text="Failed to purchase product."/>);
	            setLoading(false);
	        })
	};
//...
```

We need the `product` and `count` (how many products will be bought) and call the `buyProductAction` utility function to buy the product. We the product was bought successfully, we fetch the products again and display a success message or an error message if the product could not be bought.

Finally, we create a `deleteProduct` function:
```js
//...
    const deleteProduct = async (product) => {
        setLoading(true);
        deleteProductAction(address, product.appId)
            .then(() => {
                toast(<NotificationSuccess text="Product deleted successfully"/>);
                getProducts();
                fetchBalance(address);
            })
            .catch(error => {
                console.log(error)
                toast(<NotificationError text="Failed to delete product."/>);
                setLoading(false);
            })
    };
//...
```
It takes the product that should be deleted and calls the `deleteProductAction` utility function with the products `appId`. Again, we fetch the products on success and display an error message otherwise.

Now can return the `Products` component and write the JSX code:

```js
//...
	if (loading) {
	    return <Loader/>;
	}
	return (
	    <>
	        <div className="d-flex justify-content-between align-items-center mb-4">
	            <h1 className="fs-4 fw-bold mb-0">Street Food</h1>
	            <AddProduct createProduct={createProduct}/>
	        </div>
	        <Row xs={1} sm={2} lg={3} className="g-3 mb-5 g-xl-4 g-xxl-5">
	            <>
	                {products.map((product, index) => (
	                    <Product
	                        address={address}
	                        product={product}
	                        buyProduct={buyProduct}
	                        deleteProduct={deleteProduct}
	                        key={index}
	                    />
	                ))}
	            </>
	        </Row>
	    </>
	);
};

Products.propTypes = {
    address: PropTypes.string.isRequired,
    fetchBalance: PropTypes.func.isRequired
};

export default Products;
```

In the not yet created `AddProduct` component, we pass the `createProduct` function that we created earlier as a prop.

Then we map the list of `products` to the not yet created `Product` component, which is a component that will display the individual product as a card. We pass the `product`'s data as props to the `Product` component to render the product. We also pass the `buyProduct` function as a prop.

Finally, we export the `Products` component.

Let's now create the `Product` component that we just discussed.

#### 2.3.2 Product.jsx

This file will contain the `Product` component, which will display the individual product as a card. In the `marketplace` directory, create a `components/marketplace/Product.jsx` file with the following code:

```js
import React, {useState} from "react";
import PropTypes from "prop-types";
import {Badge, Button, Card, Col, FloatingLabel, Form, Stack} from "react-bootstrap";
import {microAlgosToString, truncateAddress} from "../../utils/conversions";
import Identicon from "../utils/Identicon";

const Product = ({address, product, buyProduct, deleteProduct}) => {
    const {name, image, description, price, sold, appId, owner} =
        product;

    const [count, setCount] = useState(1)

    return (
        <Col key={appId}>
            <Card className="h-100">
                <Card.Header>
                    <Stack direction="horizontal" gap={2}>
                        <span className="font-monospace text-secondary">{truncateAddress(owner)}</span>
                        <Identicon size={28} address={owner}/>
                        <Badge bg="secondary" className="ms-auto">
                            {sold} Sold
                        </Badge>
                    </Stack>
                </Card.Header>
                <div className="ratio ratio-4x3">
                    <img src={image} alt={name} style={{objectFit: "cover"}}/>
                </div>
                <Card.Body className="d-flex flex-column text-center">
                    <Card.Title>{name}</Card.Title>
                    <Card.Text className="flex-grow-1">{description}</Card.Text>
                    <Form className="d-flex align-content-stretch flex-row gap-2">
                        <FloatingLabel
                            controlId="inputCount"
                            label="Count"
                            className="w-25"
                        >
                            <Form.Control
                                type="number"
                                value={count}
                                min="1"
                                max="10"
                                onChange={(e) => {
                                    setCount(Number(e.target.value));
                                }}
                            />
                        </FloatingLabel>
                        <Button
                            variant="outline-dark"
                            onClick={() => buyProduct(product, count)}
                            className="w-75 py-3"
                        >
                            Buy for {microAlgosToString(price) * count} ALGO
                        </Button>
                        {product.owner === address &&
                            <Button
                                variant="outline-danger"
                                onClick={() => deleteProduct(product)}
                                className="btn"
                            >
                                <i className="bi bi-trash"></i>
                            </Button>
                        }
                    </Form>
                </Card.Body>
            </Card>
        </Col>
    );
};

Product.propTypes = {
    address: PropTypes.string.isRequired,
    product: PropTypes.instanceOf(Object).isRequired,
    buyProduct: PropTypes.func.isRequired,
    deleteProduct: PropTypes.func.isRequired
};

export default Product;

```
This component is pretty simple. We get the product data: the `name`, `image`, `description`, `price`, how many times it has been `sold`, `appId`, and `owner` from the `product` prop. Also, we get a `buyProduct` and a `deleteProduct` function.

We display the product data as a bootstrap `Card` component. We add an input field to select how many products should be bought (`count`) and call the `buyProduct` function when the "Buy" button is clicked, passing our `product` and the `count`. We also have a "Delete" button, which is only shown when the current account is the owner of the product. It calls the `deleteProduct` function. 

In the next section, we will create the `AddProduct` component.

#### 2.3.3 AddProduct.jsx

This file will contain the `AddProduct` component, which will display a form to add a new product. In the `marketplace` directory, create a `components/marketplace/AddProduct.jsx` file. We will start with the import statements and the `AddProduct` component:

```js
import React, {useCallback, useState} from "react";
import PropTypes from "prop-types";
import {Button, FloatingLabel, Form, Modal} from "react-bootstrap";
import {stringToMicroAlgos} from "../../utils/conversions";

const AddProduct = ({createProduct}) => {
    const [name, setName] = useState("");
    const [image, setImage] = useState("");
    const [description, setDescription] = useState("");
    const [price, setPrice] = useState(0);

    const isFormFilled = useCallback(() => {
        return name && image && description && price > 0
    }, [name, image, description, price]);

    const [show, setShow] = useState(false);

    const handleClose = () => setShow(false);
    const handleShow = () => setShow(true);
//...
```

We create state variables for the product data and a boolean state variable that we use to store the state of the modal with the form for the product data.

In the next step, we will create the button and form that will be displayed in the popup modal:

```js
//...
    return (
        <>
            <Button
                onClick={handleShow}
                variant="dark"
                className="rounded-pill px-0"
                style={{width: "38px"}}
            >
                <i className="bi bi-plus"></i>
            </Button>
            <Modal show={show} onHide={handleClose} centered>
                <Modal.Header closeButton>
                    <Modal.Title>New Product</Modal.Title>
                </Modal.Header>
                <Form>
                    <Modal.Body>
                        <FloatingLabel
                            controlId="inputName"
                            label="Product name"
                            className="mb-3"
                        >
                            <Form.Control
                                type="text"
                                onChange={(e) => {
                                    setName(e.target.value);
                                }}
                                placeholder="Enter name of product"
                            />
                        </FloatingLabel>
                        <FloatingLabel
                            controlId="inputUrl"
                            label="Image URL"
                            className="mb-3"
                        >
                            <Form.Control
                                type="text"
                                placeholder="Image URL"
                                value={image}
                                onChange={(e) => {
                                    setImage(e.target.value);
                                }}
                            />
                        </FloatingLabel>
                        <FloatingLabel
                            controlId="inputDescription"
                            label="Description"
                            className="mb-3"
                        >
                            <Form.Control
                                as="textarea"
                                placeholder="description"
                                style={{ height: "80px" }}
                                onChange={(e) => {
                                    setDescription(e.target.value);
                                }}
                            />
                        </FloatingLabel>
                        <FloatingLabel
                            controlId="inputPrice"
                            label="Price in ALGO"
                            className="mb-3"
                        >
                            <Form.Control
                                type="text"
                                placeholder="Price"
                                onChange={(e) => {
                                    setPrice(stringToMicroAlgos(e.target.value));
                                }}
                            />
                        </FloatingLabel>
                    </Modal.Body>
                </Form>
                <Modal.Footer>
                    <Button variant="outline-secondary" onClick={handleClose}>
                        Close
                    </Button>
                    <Button
                        variant="dark"
                        disabled={!isFormFilled()}
                        onClick={() => {
                            createProduct({
                                name,
                                image,
                                description,
                                price
                            });
                            handleClose();
                        }}
                    >
                        Save product
                    </Button>
                </Modal.Footer>
            </Modal>
        </>
    );
};

AddProduct.propTypes = {
    createProduct: PropTypes.func.isRequired,
};

export default AddProduct;
```

This file has a lot of code, but it's pretty straightforward. We create a modal that will be displayed when the user clicks on the `New Product` button.
Inside the modal, we display a form with the product data fields.

If the user clicks on the "Save product" button, we call the `createProduct` function that we passed as a prop with the product data as parameters.

We are done with the components, now we need to do a bit of cleanup in our `App.js` file, and we are ready to go!

## 3. Finishing the dapp

In this last section, we will put the final touches on the dapp.

### 3.1. Update `App.js`

Let's finish the dapp by adding our `Products` and `Notification` components to our `App.js` file.

Let's start by uncommenting the import of the `Products` and `Notification` components, so it looks like this:

```js
//...
import {Container, Nav} from "react-bootstrap";
import Products from "./components/marketplace/Products";
import {Notification} from "./components/utils/Notifications";
import {indexerClient, myAlgoConnect} from "./utils/constants";
//...
```

Now we need to uncomment the `<Notification />` component in the JSX:

```js
//...
    return (
        <>
            <Notification />
            {address ? (
//...
```

And we also need to uncomment the `<Products />` component:

```js
//...
<main>
    <Products address={address} fetchBalance={fetchBalance}/>
</main>
//...
```

Do a final test. See if your products are displayed and if you can create a new product and buy it.

The dapp should now behave like this:

![](https://i.imgur.com/SDuaSzq.gif)


### 3.2. Cleanup

We can do a little bit of housekeeping in our project. In our `src` directory, we can remove the `logo.svg` and `setupTests.js` files.

In the `public` directory, we can add our `favicon.ico`, `logo192.png`, `logo512.png`, and `manifest.json` files that fit our project.

We should also change the `title` and `description` of our dapp in the `index.html` file in the `public` directory.

### 3.3. Deploy to GitHub Pages

In the last section, we will briefly look at how to deploy our dapp to GitHub Pages.

1. First, add your project to GitHub. If you don't know how to do this, check out this [GitHub guide](https://docs.github.com/en/get-started/importing-your-projects-to-github/importing-source-code-to-github/adding-an-existing-project-to-github-using-the-command-line).

2. Install the [gh-pages](https://www.npmjs.com/package/gh-pages) package. This will allow us to deploy our dapp to GitHub Pages.

```bash
npm install gh-pages
```

3. In the package.json file, add the following lines:

- At the top of the file, add the following line:
  ```
  "homepage": "https://${GithubUsername}.github.io/${RepositoryName}",
  ```
  Replace `${GithubUsername}` and `${RepositoryName}` with your GitHub username and the repository name.
- add the following lines at the bottom of the `scripts` section:
  ```
    "predeploy": "npm run build",
    "deploy": "gh-pages -d build"
  ```

4. Push your changes to GitHub.
5. Run `npm run deploy` to deploy your dapp to a new GitHub Pages branch.
6. Go to your repository on github.com and follow the instructions: ![](https://i.imgur.com/DacKkj4.png)

- Click on settings.
- Navigate to the "Pages" section.
- Select the gh-pages branch as the default branch for your new page.
- Click "Save".

Now you are done, and you should be able to see your dapp on `https://${GithubUsername}.github.io/${RepositoryName}`. It can take up to a minute for the page to deploy.

Awesome! You have successfully created your first Algorand dapp. Now you can go ahead and create your own dapp in our challenge and receive feedback and earn some Algorand!
