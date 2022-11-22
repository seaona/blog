# Damn Vulnerable DeFi
In this post, we are going to review the Damn Vulnerable DeFi challenges one by one. As I go through the process of solving them, the notes I take and the steps I perform for reaching the final solution.

On [damnvulnerabledefi](https://www.damnvulnerabledefi.xyz/) page you can check the instructions for getting started. Let's go!

## Preliminares
### Tools and Environment
**IDE: Visual Studio Code + Solidity Visual Auditor**

I'm going to use Visual Studio Code for taking advantage of one particular VSC extension: the [Solidity Visual Auditor](https://marketplace.visualstudio.com/items?itemName=tintinweb.solidity-visual-auditor), developed by Consensys Diligence.

This extension will assist us during the challenges, by providing code insights, syntax and semantic highlighting,etc.

Here is an example of a warning in VSC:

![Solidity Visual Developer](https://raw.githubusercontent.com/seaona/blog/main/_media/solidity-visual-dev.png)

**Hardhat**

This repository is built on top of [Hardhat](https://hardhat.org/). For that reason, we'll need to use the most relevants commands for running a local node, compiling and deploying contracts, as well as running test.
This last part is specially important, as it is where we are going to include our exploits.

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

For completing the challenges we don't necessarily need to deploy any contract directly. Using the test scripts is sufficient. However, I highly recommend doing so, in parallel to our investigations. Making all our thinking process step by step manually, sometimes can help to realize things we have missed by simply writting code. In addition to that, you'll also be learning the commands for the deployment process - which is an essential step for any auditor.

1. Open a terminal inside the project folder and run a local node with hardhat:
```
npx hardhat node
```

2. Create a deploy folder and script
```
touch scripts/deploy.js
```

3. Deploy a contract on the local network
```
npx hardhat run --network localhost scripts/deploy.js
```

## Challenge #1: Unstoppable

