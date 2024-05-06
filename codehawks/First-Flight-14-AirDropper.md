# First Flight #14: AirDropper - Findings Report

Dates: Apr 25th, 2024 - May 2nd, 2024

[See the contest details here](https://www.codehawks.com/contests/clvb821kr0001jzdbi6ggixb0)

## Table of contents
- [Summary](#summary)
- [High Risk Findings](#high-risk-findings)
  - [H-01. Address discrepancy leading to disruption in protocol functionality](#h-01-address-discrepancy-leading-to-disruption-in-protocol-functionality)
  - [H-02. Lack of a claim verification mechanism in the function `MerkleAirdrop::claim` results in the USDC protocol balance draining](#h-02-lack-of-a-claim-verification-mechanism-in-the-function-merkleairdropclaim-results-in-the-usdc-protocol-balance-draining)


## Summary

| Scope |
| --- |
| MerkleAirdrop.sol |
| Deploy.s.sol |

**Issues found**
| Category | Number of issues found |
| --- | --- |
| High Risk Findings | 2 |


## High Risk Findings

### H-01. Address discrepancy leading to disruption in protocol functionality

#### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/main/script/Deploy.s.sol#L8

#### Summary

The variable `s_zkSyncUSDC` is used in the `Deploy.s.sol` script to hold the address of the USDC contract on the zkSync chain. However, an erroneous character in the address is causing misdirection to an incorrect address.

This error affects the functionality of the `MerkleAirdrop` contract, as it does not allow the distribution of the airdrop intended for eligible users upon calling the `claim` function.

#### Vulnerability Details

In the `Deploy.s.sol` script, the variable `_zkSyncUSDC` is assigned the value `0x1D17CbCf0D6d143135be902365d2e5E2a16538d4`, which is incorrect.

<details>

<summary>Code</summary>

```solidity
    address public s_zkSyncUSDC = 0x1D17CbCf0D6d143135be902365d2e5E2a16538d4;
```

</details>

The accurate address for the USDC contract can be verified from the [zkSync Era Block Explorer](https://explorer.zksync.io/tokens), which is `0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4`.

#### Impact

Calls made to the address stored in the variable `s_zkSyncUSDC` are likely to fail. Consequently, the `safeTransfer` function within the `claim` function of the `MerkleAirdrop` contract will be unsuccessful in transferring USDC tokens from the contract to the user's address.

#### Tools Used

- [zkSync Era Block Explorer](https://explorer.zksync.io/)
- Manual code review

#### Recommendations

Replace the value of `s_zkSyncUSDC` in the `Deploy.s.sol` script with the correct address `0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4`.

<details>

<summary>Code</summary>

```diff
-    address public s_zkSyncUSDC = 0x1D17CbCf0D6d143135be902365d2e5E2a16538d4;
+    address public s_zkSyncUSDC = 0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4;
```

</details>


### H-02. Lack of a claim verification mechanism in the function `MerkleAirdrop::claim` results in the USDC protocol balance draining

#### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-airdropper/blob/main/src/MerkleAirdrop.sol#L30-L40

#### Summary

The `claim` function in the MerkleAirdrop contract enables eligible users to claim their 25 USDC airdrop. However, the current implementation of the `MerkleAirdrop.sol` contract lacks a mechanism to prevent users from claiming the airdrop multiple times, which could lead to draining the contract's USDC balance.

#### Vulnerability Details

The vulnerability arises due to the absence of restrictions on how many times an eligible user can call the `MerkleAirdrop::claim` function to collect the airdrop. This allows an attacker to call the function multiple times and claim the USDC airdrop intended for other users as well.

<details>

<summary>Proof of Code</summary>

Add the following test to the `MerkleAirdropTest.t.sol` test suite.

```Solidity
    function testUsersCanClaimMultipleTimes() public {
        uint256 startingBalance = token.balanceOf(collectorOne);
        vm.deal(collectorOne, airdrop.getFee() * 4);

        vm.startPrank(collectorOne);
        for (uint i = 0; i < 4; i++) {
            airdrop.claim{value: airdrop.getFee()}(
                collectorOne,
                amountToCollect,
                proof
            );
        }
        vm.stopPrank();

        uint256 endingBalance = token.balanceOf(collectorOne);
        assertEq(endingBalance - startingBalance, amountToSend);
    }
```

</details>

#### Impact

This vulnerability allows an eligible user to claim all the USDC tokens present in the protocol, potentially draining the protocol's USDC balance.

#### Tools Used

- Manual code review
- Foundry

#### Recommendations

Add a verification mechanism to make sure that an user can claim only its airdrop. An example is the following:


<details>

<summary>Code</summary>

```diff
+    error MerkleAirdrop__AirdropAlreadyClaimed();
+    mapping(address => bool) private claimed; // Track claimed status
...

    function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable {
        if (msg.value != FEE) {
            revert MerkleAirdrop__InvalidFeeAmount();
        }
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
        if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
            revert MerkleAirdrop__InvalidProof();
        }
+      if (claimed[account]) { // Check if user already claimed
+          revert MerkleAirdrop__AirdropAlreadyClaimed();
+      }
+      claimed[account] = true; // Mark user as claimed
        emit Claimed(account, amount);
        i_airdropToken.safeTransfer(account, amount);
    }

```

</details>
