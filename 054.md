Steep Bamboo Elk

high

# Keepers incentivised to not call claimDefaulted until another user submits their claimDefault and then front run the transaction to maximize rewards
## Summary

There is a gOHM keeper reward for anybody who calls the defaultLoan function, and this increases linearly over time. This incentivises the keepers to delay the `claimDefault` until another user calls it and then front run the transaction. This could lead to claims taking too long to default and over-rewarding of gOHM

## Vulnerability Detail

The gOHM keeper rewards increase linearly over time until they reach a maximum auction reward which is proportional to collateral:

```solidity

// Cap rewards to 5% of the collateral to avoid OHM holder's dillution.
            uint256 maxAuctionReward = collateral * 5e16 / 1e18;
            // Cap rewards to avoid exorbitant amounts.
            uint256 maxReward = (maxAuctionReward < MAX_REWARD)
                ? maxAuctionReward
                : MAX_REWARD;
            // Calculate rewards based on the elapsed time since default.
            keeperRewards = (elapsed < 7 days)
                ? keeperRewards + maxReward * elapsed / 7 days
                : keeperRewards + maxReward;
        }

```

Immediately defaulting an avaliable loan is not profitable as the rewards start out extremely low. Therefore no keeper would immediately default a loan. In fact you want to wait as long as possible as long as another user submits the loan.

This creates a waiting game for keepers, who wish to wait until the reward reaches the maximum reward. Then, when one keeper submits a defaultLoan() transaction, they will all bid to frontrun each other. If there are many keepers trying to front run each other, almost all the profit will go to miners (gas fees).

Therefore the linearly increasing loans incentivises delaying loan defaulting, followed by a malicious frontrunning battle when the reward becomes large enough.

The other possibility is a limited number of keepers collude, and they can happily allow the defaulting reward accumulate (and frontrun anybody who tries to interfere).

## Impact
- Abuse of gOHM rewarded system
- Breaks the incentive assumptions of defaulting loans
- More gOHM rewards than reasonable due to keepers gaming the system or colluding

## Code Snippet

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L215C13-L225C10

## Tool used

Manual Review

## Recommendation

The ClearingHouse itself can default loans which can be defaulted, which avoids the incentive pitfalls in the issue described above.
