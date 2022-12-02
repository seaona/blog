# Ethernaut
In this post, we are going to review the Ethernaut challenges one by one. We are going to analyze the contracts and explore and hack their vulnerabilities.

On the [Ethernaut Challenge](https://ethernaut.openzeppelin.com/) page you can check the instructions for getting started. Let's go!

## Challenge #1: Fallback
The goal of this challenge is to claim ownership of the contract and drain its funds.

The key piece of code that will help us to achieve the result is the `receive` fallback function. As we see below, if we send some ETH to the contract, and we have at least 1 contribution, we will then be set as the owner of the contract.

```
 receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
```

To solve this challenge, we need to first call the function `contribute`, so our address has at least one contribution:

`await contract.contribute({value: toWei("0.0001")})`

Then we can claim ownership by sending some ETH directly to the contract address:

`await web3.eth.sendTransaction({from: player, to:contract.address, value: toWei("0.000000001")});`

Finally we can withdraw the funds using the contract function:

`await contract.withdraw()`

## Challenge #2: Fallout
The goal of this challenge is to claim ownership of the contract.

The first thing that calls our attention is that the contract is using the old implementation of the `constructor`function. Prior to `0.4.22`version, the way to define a `constructor` function was to write a function with the same name as the contract.
This lead to a lot of problems, as only one typo could mean that the contract was not initialized at deployment time. The `constructor` keyword was later introduced and it is the current way for defining `constructor` functions.

Now let's check who is the owner of the contract:

`await contract.owner()`

To our surprise, it is not the contract deployer but the zero address.

This should make us suspicious on the `constructor` function. Something might be wrong there, as it is definetly not called when the contract is deployed.

If we look closely, we see that the function name is actually not the same as the contract name. Instead if `Fallout` we see it's written `Fal1out`, with a `1`.
This explains why it was not called on the contract deployement.

So what we can do to solve the challenge is to call the intended `constructor` function with some value, and we we'll be assigned as the contract owners.

`await contract.Fal1out({value: toWei("0.0001")})`


## Challenge #3: Coin Flip
In 0.8.0 or better, math overflows revert by default.

block.number refers to the current block number (the one the transaction is oart of). So the blockvalue is already guessable, by going to the etherscan.

Then we can calculate the coinFlip ourselves with the function:

`uint256 coinFlip = blockValue / FACTOR;`

web3.eth.getBlock(blockNumber).hash;