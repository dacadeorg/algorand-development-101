This learning module consists of a tutorial, where you will learn how to create a smart contract for a decentralized marketplace on the Algorand blockchain.

We will use PyTeal to write our smart contracts.

### Prerequisites

- Have some basic knowledge of Blockchain technology and smart contracts.
- Have some basic Python knowledge.
- Be comfortable using a terminal.

### Tech Stack

We will use the following tech stack:

- [PyTeal](https://pypi.org/project/pyteal/) - A Python language binding for Algorand Smart Contracts.
- [Algorand Sandbox](https://github.com/algorand/sandbox) - A fast way to run a Algorand developmment environment locally

## 1. Setup
In this first section of the tutorial, we will set up our development environment and project.

You will need to have a version of Python 3 (at least 3.10.x) installed. You can check the currently installed version using
```bash
python3 --version
```
or on Windows
```bash
python --version
```
We recommend using a code editor that supports code completion and syntax highlighting for Python to help you write PyTeal code, like Visual Studio Code or Atom. We will use Visual Studio Code for this learning module.

### 1.1 Project Setup
First, we will set up our project. Please create a new root directory for our project and call it something like `algorand-marketplace-contract`.

### 1.2 Create venv and install PyTeal
Next we use the Python module [venv](https://docs.python.org/3/library/venv.html) to set up a Python virtual environment. A virtual environment  can have its own independent set of installed Python packages in its site directories, so packages required for a project do not have to be installed globally.

Open a terminal and go into your root directory. Then run 
```bash
python3 -m venv venv
```
or on Windows:
```bash
python -m venv venv
```
to create a virtual environment in the subdirectory `venv`.
Next, run
```bash
source venv/bin/activate
```
or on Windows
```bash
venv\Scripts\activate.bat
```
to activate the virtual environment.

Finally, install PyTeal inside the virtual environment using 
```bash
pip install pyteal
```

## 2. Define smart contract
For this Algorand tutorial, we want to create a decentraliced marketplace, were products can be offered, bought and deleted. To build this marketplace using the Algorand blockchain we are going to utilise Algorand's [smart contract feature](https://developer.algorand.org/docs/get-details/dapps/smart-contracts/apps/). These smart contracts are also referred to as applications. The marketplace and its products is going to be represented by multiple applications, were each of the applications stands for one product. Buying one of these products will be done by calling the application and performing a payment to the creator of the application.

In this section, we are going to write the smart contract that will represent a product. For the Algorand blockchain, smart contracts are written in the smart contract language [TEAL](https://developer.algorand.org/docs/get-details/dapps/avm/teal/). To simplify the creation of TEAL code, the Python library PyTeal allows to write contracts in Python and offers a `compileProgram` method to produce TEAL code. TEAL source can then be compiled into bytecode and deployed to the blockchain as an application.

We will write the smart contract with PyTeal in `marketplace_contract.py`. Please create the file in the project's root directory and open the file in an editor.

First we will import PyTeal using
```python=
from pyteal import *
```

In PyTeal, two datatypes are available: Byte slices (`Byte`) and integers (`Int`).
Multiple variables define a product in our marketplace. It has a name, an image URL and a description, stored as byte slices. Also, it has a price and a variable "sold", which counts how often a product has already got sold. These variables are stored as integers.
The variables are stored in the application's global state, consisting of key-value pairs, where keys are byte slices, and values can be integers or byte slices.

In PyTeal we first create a class `Product`. While classes are not required for the smart contract to work, it helps in understanding the code. We also define two subclasses. The class `Variables` defines the keys for our Global state variable. The class `AppMethods` defines methods available for the product. Here, only "buy" is added for now.
```python=
from pyteal import *


class Product:
    class Variables:
        name = Bytes("NAME")
        image = Bytes("IMAGE")
        description = Bytes("DESCRIPTION")
        price = Bytes("PRICE")
        sold = Bytes("SOLD")

    class AppMethods:
        buy = Bytes("buy")
```

### 2.1 Create a product
Next, we are going to define what should happen when creating the application.

Therefore, we add the following method to our `Product` class:

```python=15
    def application_creation(self):
        return Seq([
            Assert(Txn.application_args.length() == Int(4)),
            Assert(Txn.note() == Bytes("tutorial-marketplace:uv1")),
            Assert(Btoi(Txn.application_args[3]) > Int(0)),
            App.globalPut(self.Variables.name, Txn.application_args[0]),
            App.globalPut(self.Variables.image, Txn.application_args[1]),
            App.globalPut(self.Variables.description, Txn.application_args[2]),
            App.globalPut(self.Variables.price, Btoi(Txn.application_args[3])),
            App.globalPut(self.Variables.sold, Int(0)),
            Approve()
        ])
```
Multiple things happen here. First, we use the PyTeal expression `Assert` to perform validity checks:
- The number of arguments attached to the transaction should be exactly 4. 
- The note attached to the transaction must be `"tutorial-marketplace:uv1"`, which we define to be the note that marks a product within our marketplace. Our goal here was to create a unique note that helps us finding the products later on. Conventions on how to build a note can be found [here](https://github.com/algorandfoundation/ARCs/blob/main/ARCs/arc-0002.md).
The fourth argument, which is used to represent the product's price, has to be greater than zero.

If all the checks succeed, the next lines will be executed. Here we store the transaction arguments into the application's global state using `App.globalPut`. As transaction arguments are transmitted as byte slices, we can store the first three arguments as name, image and description without conversion. The fourth argument representing the price has to be converted to `Int` using the PyTeal method `Btoi()`. Finally, we set the sold variable of the product to `0`.

### 2.2 Buy a product
Next, we are going to write a handler for the buy interaction.
Buying a product will be represented as a group transaction where a smart contract call transaction and a payment transaction to the product owner are grouped together.

We add the following method to our `Product` class
```python=28
    def buy(self):
        count = Txn.application_args[1]
        valid_number_of_transactions = Global.group_size() == Int(2)

        valid_payment_to_seller = And(
            Gtxn[1].type_enum() == TxnType.Payment,
            Gtxn[1].receiver() == Global.creator_address(),
            Gtxn[1].amount() == App.globalGet(self.Variables.price) * Btoi(count),
            Gtxn[1].sender() == Gtxn[0].sender(),
        )

        can_buy = And(valid_number_of_transactions,
                      valid_payment_to_seller)

        update_state = Seq([
            App.globalPut(self.Variables.sold, App.globalGet(self.Variables.sold) + Btoi(count)),
            Approve()
        ])

        return If(can_buy).Then(update_state).Else(Reject())

```
Again, validity checks are performed:
- The number of transactions within the group transaction must be exactly `2.` 
- The second transaction of the group must be the payment transaction. 
- The receiver of the payment should be the creator of the app
The payment amount should match the product's price multiplied by the number of products bought. 
- The sender of the payment transaction should match the sender of the smart contract call transaction.
The global state is updated using `App.globalPut()` if all checks succeed. Specifically, the sold variable of the product is incremented by the number of products bought.
If the checks do not succeed, the transaction is rejected.

### 2.3 Delete a product
Lastly, we want to be able to delete a product.
Herefore we add the following method:
```python=49
    def app_deletion(self):
        return Return(Txn.sender() == Global.creator_address())
```
Here we check if the sender of the delete transaction matches the app's creator, as only the creator should be able to delete an application.

### 2.4 Check transaction conditions

To allow for the different types of calls for the smart contract, the PyTeal Expression `Cond` is used. It chains a series of tests to select a result expression.
We define the `Cond` expression within the Python function `application_start` that is added to our `Product` class.

```python=52
    def application_start(self):
        return Cond(
            [Txn.application_id() == Int(0), self.application_creation()],
            [Txn.on_completion() == OnComplete.DeleteApplication, self.application_deletion()],
            [Txn.application_args[0] == self.AppMethods.buy, self.buy()]
        )
```

- The first condition checks if the `application_id` field of a transaction matches `0`. If this is the case, the application does not exist yet, and the `application_creation()` method is called.
- As a second, if the the `OnComplete` action of the transaction is `DeleteApplication`, the `application_deletion()` method is called.
- Finally, if none of the other conditions match and the first argument of the transaction matches the `AppMethods.buy` value, the `buy()` method is called.
If none of the conditions match, the transaction is rejected.

### 2.5 Approval and clear program
A smart contract for the Algorand blockchain consists of two different programs called approval program and clear program. In PyTeal both of these programs are generally created in the same Python file.

The approval program is responsible for processing all application calls to the contract, except for the clear call. The approval program is responsible for implementing most of the logic of an application. Like smart signatures, this program will succeed only if one nonzero value is left on the stack upon program completion or the return opcode is called with a positive value on the top of the stack.

The clear program is used to handle accounts using the clear call to remove the smart contract from their balance record. This program will pass or fail the same way the approval program does.

Therefore, we define the two programs required for a smart contract.
```python=59
    def approval_program(self):
        return self.application_start()

    def clear_program(self):
        return Return(Int(1))
```
The `approval_program()` returns `application_start()`, which we defined in the previous step. As we do not use the local state for our application, we can use a basic `clear_program()` which returns `Int(1)`.


The final `marketplace_contract.py` looks like this:
```python=
from pyteal import *


class Product:
    class Variables:
        name = Bytes("NAME")
        image = Bytes("IMAGE")
        description = Bytes("DESCRIPTION")
        price = Bytes("PRICE")
        sold = Bytes("SOLD")

    class AppMethods:
        buy = Bytes("buy")

    def application_creation(self):
        return Seq([
            Assert(Txn.application_args.length() == Int(4)),
            Assert(Txn.note() == Bytes("tutorial-marketplace:uv1")),
            Assert(Btoi(Txn.application_args[3]) > Int(0)),
            App.globalPut(self.Variables.name, Txn.application_args[0]),
            App.globalPut(self.Variables.image, Txn.application_args[1]),
            App.globalPut(self.Variables.description, Txn.application_args[2]),
            App.globalPut(self.Variables.price, Btoi(Txn.application_args[3])),
            App.globalPut(self.Variables.sold, Int(0)),
            Approve()
        ])

    def buy(self):
        count = Txn.application_args[1]
        valid_number_of_transactions = Global.group_size() == Int(2)

        valid_payment_to_seller = And(
            Gtxn[1].type_enum() == TxnType.Payment,
            Gtxn[1].receiver() == Global.creator_address(),
            Gtxn[1].amount() == App.globalGet(self.Variables.price) * Btoi(count),
            Gtxn[1].sender() == Gtxn[0].sender(),
        )

        can_buy = And(valid_number_of_transactions,
                      valid_payment_to_seller)

        update_state = Seq([
            App.globalPut(self.Variables.sold, App.globalGet(self.Variables.sold) + Btoi(count)),
            Approve()
        ])

        return If(can_buy).Then(update_state).Else(Reject())

    def application_deletion(self):
        return Return(Txn.sender() == Global.creator_address())

    def application_start(self):
        return Cond(
            [Txn.application_id() == Int(0), self.application_creation()],
            [Txn.on_completion() == OnComplete.DeleteApplication, self.application_deletion()],
            [Txn.application_args[0] == self.AppMethods.buy, self.buy()]
        )

    def approval_program(self):
        return self.application_start()

    def clear_program(self):
        return Return(Int(1))
```

That is it! Now we only have to compile the smart contract to TEAL and are then ready to deploy the contract.

## 3. Compile contract
In order to deploy a smart contract we need to compile it first to Algorand's Smart Contract Language TEAL. 
Please create a `compile_contract.py` in the root directory of the project.
Then, we use the following python code to compile the approval and clear program written in PyTeal Code to TEAL:

`compile_contract.py`
```python=
from pyteal import *

from marketplace_contract import Product

if __name__ == "__main__":
    approval_program = Product().approval_program()
    clear_program = Product().clear_program()

    # Mode.Application specifies that this is a smart contract
    compiled_approval = compileTeal(approval_program, Mode.Application, version=6)
    print(compiled_approval)
    with open("marketplace_approval.teal", "w") as teal:
        teal.write(compiled_approval)
        teal.close()

    # Mode.Application specifies that this is a smart contract
    compiled_clear = compileTeal(clear_program, Mode.Application, version=6)
    print(compiled_clear)
    with open("marketplace_clear.teal", "w") as teal:
        teal.write(compiled_clear)
        teal.close()
```

We import `pyteal` and the `Product` class from our marketplace_contract.py. Then we compile the `approval_program` and `clear_program` using PyTeals `compileTeal` method and write the compiled programs to `marketplace_approval.teal` and `marketplace_clear.teal` respectively.

Now, while still being in the virtual environment, compile the contracts by running:
```
python3 compile_contract.py
```

This should create `marketplace_approval.teal` and `marketplace_clear.teal` in the root directory of the project.

The resulting TEAL files can also be found [here](https://github.com/dacadeorg/algorand-react-marketplace/tree/master/src/contracts).


In the next section you will learn how to test the smart contract in a local test environment.

## 4. Test smart contract using Algorand sandbox


In order to test the smart contract written in the previous step, we are going to run a local development environment of the Algorand blockchain using the [Algorand sandbox](https://github.com/algorand/sandbox).

### 4.1 Prerequisites
The Algorand sandbox requires Docker Compose to be installed on your machine. You can find instructions on how to install Docker [here](https://docs.docker.com/compose/install/).

Make sure the docker daemon is running and docker-compose is installed.

Then, open a terminal and run:
```bash
git clone https://github.com/algorand/sandbox.git
```

In whatever local directory the sandbox should reside. Then go into the sandbox directory using
```bash
cd sandbox
```
Finally run
```bash
./sandbox up -v
```
to start the sandbox as a local private network. This can take some while when first starting the sandbox.

### 4.2 List accounts
The local private network comes with precreated accounts that already have funding on them.
You can list available accounts using the [goal cli](https://developer.algorand.org/docs/clis/goal/goal/) of the sandbox
```bash
./sandbox goal account list
```

This should give you something like:
```bash
[offline]	A72PORR7DCZMIKM3YYOK3IPYZ6BYDGZI6MZDBNMVX3LLWSMUQ3NSPNUZMM	A72PORR7DCZMIKM3YYOK3IPYZ6BYDGZI6MZDBNMVX3LLWSMUQ3NSPNUZMM	4000000000000000 microAlgos
[offline]	CLQ2U2RFKSTBKFIVMLW5MGFRGF4FZHJUQRDDTV75VNOQ5IEZG74OJQFFYY	CLQ2U2RFKSTBKFIVMLW5MGFRGF4FZHJUQRDDTV75VNOQ5IEZG74OJQFFYY	1000000000000000 microAlgos
[online]	ZYKAGOZWO34XHUHE7DSQJ6Y5MENUKAV2SK3JINXLSHAB6F7SKDD4NVFZHU	ZYKAGOZWO34XHUHE7DSQJ6Y5MENUKAV2SK3JINXLSHAB6F7SKDD4NVFZHU	4000000000000000 microAlgos
```

Select one of the accounts and write down its address (e.g. `A72PORR7DCZMIKM3YYOK3IPYZ6BYDGZI6MZDBNMVX3LLWSMUQ3NSPNUZMM`). It will be used in the next steps.

### 4.3 Deploy contract 
Now we are going to deploy the smart contract to the local blockchain using [goal app create](https://developer.algorand.org/docs/clis/goal/app/create).
In preperation for that we are going to copy our teal programs to the sandbox container:
```bash
./sandbox copyTo ${PATH_TO_APPROVAL_PROGRAM}
```
```bash
./sandbox copyTo ${PATH_TO_CLEAR_PROGRAM}
```
- `${PATH_TO_APPROVAL_PROGRAM}`: Path to `marketplace_approval.teal`
- `${PATH_TO_CLEAR_PROGRAM}`: Path to `marketplace_clear.teal`

Then we can create the application like this
```bash
./sandbox goal app create --creator ${ACCOUNT_ADDRESS} --approval-prog marketplace_approval.teal --clear-prog marketplace_clear.teal --note tutorial-marketplace:uv1 --global-byteslices 3 --global-ints 2 --local-byteslices 0 --local-ints 0 --app-arg str:TestName --app-arg str:TestImage --app-arg str:TestDescription --app-arg int:1000000
```
- `${ACCOUNT_ADDRESS}`: Address of the account that will create the app. Can be one of the addresses that were listed in the previous step.

We pass our approval program and clear program using the `--approval-prog` and `--clear-prog` flags. We pass the predefined note using the `--note` flag. As the application is storing 5 variables in the global state (3 as bytes, 2 as ints) and will not be using the local state we configure this using `--global-byteslices 3 --global-ints 2 --local-byteslices 0 --local-ints 0`. Finally, we pass the required application arguments using `--app-arg` in the correct order. For the last`--app-arg`, which represents the price, we pass 1000000 microAlgos, which is equivalent to 1 Algo.

If the command was entered correctly you should see something like this:

![](https://i.imgur.com/JR2rPdQ.gif)


You have successfully deployed the contract to your local network!


### 4.4 Delete app
As a second test we can use goal app delete to delete our application
The command to delete an application looks like this:
```bash
./sandbox goal app delete --app-id ${APP_ID} --from ${ACCOUNT_ADDRESS}
```
- `${APP_ID}` App id of the application that was created in the previous step.
- `${ACCOUNT_ADDRESS}`: Address of the account that will be delete the app. Has to be the creator of the app, otherwise the call will get rejected.
    
If the command was entered correctly you should see something like this:

![](https://i.imgur.com/GQiOhbK.gif)

    

That’s it! We have successfully written and tested a smart contract for a decentralized marketplace.


The next nearning module will explain how to deploy the contract to the Algorand testnet, and interact with the smart contract in a frontend.