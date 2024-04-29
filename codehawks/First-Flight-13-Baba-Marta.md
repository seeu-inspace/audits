# First Flight #13: Baba Marta - Findings Report

Dates: Apr 11th, 2024 - Apr 18th, 2024

[See the contest details here](https://www.codehawks.com/contests/cluseb1bf0001s4tjl2rzajup)

## Table of contents
- [Summary](#summary)
- [High Risk Findings](#high-risk-findings)
  - [H-01. No restriction implemented in `MartenitsaToken::updateCountMartenitsaTokensOwner` allows any user to update any MartenitsaToken balance breaking the operativity and purpose of the protocol](#h-01-no-restriction-implemented-in-martenitsatokenupdatecountmartenitsatokensowner-allows-any-user-to-update-any-martenitsatoken-balance-breaking-the-operativity-and-purpose-of-the-protocol)
- [Medium Risk Findings](#medium-risk-findings)
  - [M-01. `MartenitsaEvent::stopEvent` does not remove the list of partecipants not allowing recurring users to join new events](#m-01-martenitsaeventstopevent-does-not-remove-the-list-of-partecipants-not-allowing-recurring-users-to-join-new-events)


## Summary

**Scope**
- 

**Issues found**
| Category | Number of issues found |
| --- | --- |
| High Risk Findings | 1 |
| Medium Risk Findings | 1 |


## High Risk Findings

### H-01. No restriction implemented in `MartenitsaToken::updateCountMartenitsaTokensOwner` allows any user to update any MartenitsaToken balance breaking the operativity and purpose of the protocol

#### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/main/src/MartenitsaToken.sol#L57-L70

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/main/src/MartenitsaToken.sol#L62

#### Summary

If an user wants to buy a `MartenitsaToken`, it's supposed to call `MartenitsaMarketplace::buyMartenitsa` to purchase it, where there are the necessary checks to verify that the user has the requirements to do so. The balance of both the buyer and seller is updated by calling the function `updateCountMartenitsaTokensOwner` from the contract `MartenitsaToken`.

However, an user can directly call `MartenitsaToken::updateCountMartenitsaTokensOwner`, bypassing any previous restriction, to update its own balance or that of any other user as there is no control over who is calling the function. This means that an attacker can negatively or positively influence not only its own balance, but also that of other users.

#### Vulnerability Details

If you look at `MartenitsaToken::updateCountMartenitsaTokensOwner`, you can see that the there is no restriction implemented for the function. This means that any user can call this function acting on the balance of every other user partecipating in the protocol.

<details>
<summary>Code</summary>

```Solidity
@>    function updateCountMartenitsaTokensOwner(address owner, string memory operation) external {
@>      if (keccak256(abi.encodePacked(operation)) == keccak256(abi.encodePacked("add"))) {
            countMartenitsaTokensOwner[owner] += 1;
        } else if (keccak256(abi.encodePacked(operation)) == keccak256(abi.encodePacked("sub"))) {
            countMartenitsaTokensOwner[owner] -= 1;
        } else {
            revert("Wrong operation");
        }
    }
```

</details>

You can test this by adding `testUnrestricted_updateCountMartenitsaTokensOwner()` to `MartenitsaToken.t.sol` test suite. A possible solution, is to make 

<details>
<summary>Proof of Code</summary>

```Solidity
    function testUnrestricted_updateCountMartenitsaTokensOwner() public createMartenitsa {
        address newUser = makeAddr("newUser");
        address evilUser = makeAddr("evilUser");

        vm.startPrank(newUser);
        for (uint256 i = 0; i < 100; i++) {
            martenitsaToken.updateCountMartenitsaTokensOwner(newUser, "add");
        }
        vm.stopPrank();
        assert(martenitsaToken.getCountMartenitsaTokensOwner(newUser) == 100);

        vm.startPrank(evilUser);
        for (uint256 i = 0; i < 100; i++) {
            martenitsaToken.updateCountMartenitsaTokensOwner(newUser, "sub");
        }
        vm.stopPrank();
        assert(martenitsaToken.getCountMartenitsaTokensOwner(newUser) == 0);
    }
```

</details>

#### Impact

This enables anyone to reduce or increase any balance of `MartenitsaToken` of any user, breaking the purpose of `MartenitsaMarketplace::buyMartenitsa` and the whole purpose of the protocol in general.

#### Tools Used

Manual Review, Foundry

#### Recommendations

You should implement some checks on the function `MartenitsaToken::updateCountMartenitsaTokensOwner` to see who is calling it. One possible solution is the following.

<details>
<summary>Code</summary>

```diff
+import {MartenitsaMarketplace} from "./MartenitsaMarketplace.sol";

...

+   MartenitsaMarketplace private _martenitsaMarketplace;

...

+   function setMarketAddress(address martenitsaMarketplace) public onlyOwner {
+       _martenitsaMarketplace = MartenitsaMarketplace(martenitsaMarketplace);
+   }

...

    function updateCountMartenitsaTokensOwner(address owner, string memory operation) external {
+       require(msg.sender == address(_martenitsaMarketplace), "Unable to call this function");
        if (keccak256(abi.encodePacked(operation)) == keccak256(abi.encodePacked("add"))) {
            countMartenitsaTokensOwner[owner] += 1;
        } else if (keccak256(abi.encodePacked(operation)) == keccak256(abi.encodePacked("sub"))) {
            countMartenitsaTokensOwner[owner] -= 1;
        } else {
            revert("Wrong operation");
        }
    }
```

</details>
		
## Medium Risk Findings

### M-01. `MartenitsaEvent::stopEvent` does not remove the list of partecipants not allowing recurring users to join new events

#### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/main/src/MartenitsaEvent.sol

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/main/src/MartenitsaEvent.sol#L57-L65

#### Summary

The `stopEvent` function in the `MartenitsaEvent` contract fails to remove participants from the list of participants after the event ends. This oversight prevents recurring users from joining new events as their addresses remain stored in the `_participants` mapping.

#### Vulnerability Details

The `stopEvent` function is designed to end the event and remove the producer role from participants. However, it lacks the functionality to remove participants from the list entirely. As a result, addresses of previous participants persist in the `_participants` mapping, which may inadvertently block them from joining future events.

<details>
<summary>Proof of Code</summary>

Add this test to the `MartenitsaEvent.t.sol` test suite.

```Solidity
    function testJoinNewEvent() public eligibleForReward {
        martenitsaEvent.startEvent(1 days);

        vm.startPrank(bob);
        marketplace.collectReward();
        healthToken.approve(address(martenitsaEvent), 10 ** 18);
        martenitsaEvent.joinEvent();
        vm.stopPrank();

        vm.warp(block.timestamp + 1 days + 1);
        martenitsaEvent.stopEvent();

        //start a new event
        martenitsaEvent.startEvent(1 days);

        vm.startPrank(bob);
        marketplace.collectReward();
        healthToken.approve(address(martenitsaEvent), 10 ** 18);
        vm.expectRevert(bytes("You have already joined the event"));
        martenitsaEvent.joinEvent();
        vm.stopPrank();
    }
```

</details>

#### Impact

Users who have participated in previous events remain listed as participants even after the event has ended. This prevents them from joining new events since the contract mistakenly believes they are still active participants.

#### Tools Used

Manual review, Foundry

#### Recommendations

You can implement the following changes to the `stopEvent` function.

<details>

<summary>Code</summary>

```diff
    /**
     * @notice Function to remove the producer role of the participants after the event is ended.
     */
    function stopEvent() external onlyOwner {
        require(block.timestamp >= eventEndTime, "Event is not ended");
        for (uint256 i = 0; i < participants.length; i++) {
            isProducer[participants[i]] = false;
+          _participants[participants[i]] = false;
        }
    }
```

</details>
