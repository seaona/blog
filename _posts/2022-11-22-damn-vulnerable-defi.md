# Damn Vulnerable DeFi
In this post, we are going to review the Damn Vulnerable DeFi challenges one by one. We are going to analyze the contracts and explore and hack their vulnerabilities.

On [damnvulnerabledefi](https://www.damnvulnerabledefi.xyz/) page you can check the instructions for getting started. Let's go!

## Preliminares
### Tools and Environment
**IDE: Visual Studio Code + Solidity Visual Auditor**

We are going to use Visual Studio Code for taking advantage of one particular VSC extension: the [Solidity Visual Auditor](https://marketplace.visualstudio.com/items?itemName=tintinweb.solidity-visual-auditor), developed by Consensys Diligence.

This extension will assist us during the challenges, by providing code insights, syntax and semantic highlighting, and visual graphs which represent contracts and interactions.

Here is an example of a warning in VSC, highlighting an external call:

![Solidity Visual Developer](https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/solidity-visual-dev.png)

We can also generate visual graphs by utilizing Surya capabilities, embedded in the pluggin. We'll do this for every challenge, so we get a clear overview on the contracts we are hacking and the calls between them.

![Solidity Visual Developer](https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/surya-call-graph.png)

**Hardhat**

The development environment we'll use is [Hardhat](https://hardhat.org/). It provides us with a rich testing environment which we'll be using for our hacks. It uses [Ethers.js](https://docs.ethers.io/v5/) library for smart contract interaction.

**Codebase**

We'll have to clone the repository, checkout the latest version and install the dependencies:
```
git clone https://github.com/tinchoabbate/damn-vulnerable-defi.git
cd damn-vulnerable-defi
git checkout v2.2.0
yarn
```
There is one folder called `contracts` where we can find all the challenges, each on a separate folder. We'll also find 3 contracts without a specific folder, which are utilized along the different challenges. These are:
- `DamnValuableNFT.sol`
- `DamnValuableToken.sol`
- `DamnValuableTokenSnapshot.sol`

The other relevant folder is called `test`. Inside we'll find again separate folders for each challenge. Here in the `js` files is where we'll code the solutions and we'll verify them by running the command:

```
yarn run *challenge-name*
```

This command basically uses the hardhat command for running tests. I recommend running the command below, so you start getting more familiar with hardhat commands (if you are not):
```
yarn hardhat test test/*challenge-folder*/*challenge-file*
```

### Setup

For completing the challenges we'll just need to open the challenge scripts on the test folder and include our hack. For validating we simply need to call the script and, voilà!

1. Open a terminal inside the project folder

2. Run the first challenge script
```
yarn run unstoppable
```
Now you'll see that the script will fail - as we haven't yet started to work on the challenge.

## Challenge #1: Unstoppable
### The Goal
The Unstoppable challenge states the following:
> There's a lending pool with a million DVT tokens in balance, offering flash loans for free. If only there was a way to attack and stop the pool from offering flash loans... You start with 100 DVT tokens in balance.

Our goal is to break the Flash Loan functionality from the pool, so it is no longer possible to perform another Flash Loan. Let's start by analizing the `flashLoan`function, to see what we get from there.

### Contract Call Graphs
There are 2 contracts:
- UnstoppableLender.sol
- UnstoppableReceiver.sol

The contract diagrams generated with the Solidity Visual Auditor extension are the following:

![Unstoppable Lender](https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/unstoppable-lender.png)

![Unstoppable Receiver](https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/unstoppable-receiver.png)


### Contracts Highlights
On the `flashLoan(uint256 borrowAmount)` function we can see that it is required that the `poolBalance` equals the `balanceBefore` value. However, where do these values come from?

```
 assert(poolBalance == balanceBefore);
```

A couple of lines above, we can see that the `balanceBefore` value is taken using the `balanceOf` function from the token contract.

```
uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
```

On the other hand, the `poolBalance` value is a contract variable updated every time we call the `depositTokens(uint256 amount)` function.

Does this ring a bell?

### The Hack
As you might be realizing, there is a way in which we can make the assertion of balances `false`, which is, sending some tokens to the pool directly by using the token function `transfer` instead of `depositTokens`.

If we do this, the assertion won't be no longer true, and it won't be possible to execute any other flashloans, as the variable `poolBalance` won't be updated.

So the exploit looks like this. On the `test/unstoppable/unstoppable.challange.js`, include the following code, which performs a transfer to the pool address
```
it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE */
    const poolAddr = await this.pool.address
    await this.token.transfer(poolAddr,1) 
});
```
Now, run the script with `yarn run unstoppable` and see how we pass the challenge.

![Unstoppable Challenge](https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/unstoppable-challenge.png)


Congrats!!

## Challenge #2: Naive Receiver
### The Goal
The Naive Receiver challenge states the following:
> There's a lending pool offering quite expensive flash loans of Ether, which has 1000 ETH in balance. (...) A user has deployed a contract with 10 ETH in balance, capable of interacting with the lending pool and receiveing flash loans of ETH. 

Our goal is to hack the user's contract and drain all the funds. If we can do it with a single transaction is a bonus.

### Contract Call Graphs
There are 2 contracts:
- NaiveReceiverLenderPool.sol
- FlashLoanReceiver.sol

The contract diagrams generated with the Solidity Visual Auditor extension are the following:

![Naive Receiver Lender Pool](https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/naive-receiver.png)

![FlashLoan Receiver](https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/flashloan-receiver.png)

### Contracts Highlights
The flashLoan function accepts 2 arguments, the borrower address and the borrowerAmount. 
`function flashLoan(address borrower, uint256 borrowAmount)`.
This means that we can "impersonate" the borrower, by passing its contract address.

Another important aspect is that on the last require we ensure that the pool balance after the flashloan is equal or **greatest** than the initial balance. The greatest part is key, as this will allow the pool to receive more ETH.

### The Hack
We are going to impersonate the FlashLoanReceiver contract, by passing its address to the flashLoan function, like this:
`await this.pool.flashLoan(this.receiver.address, ethers.utils.parseEther('1'))`

The pool can perform this action 10 times, and this will drain all the receiver's funds (10ETH).

There is a bonus for performing this in a single transaction. For that, we can setup a simple smart contract, that executes the 10x calls above.