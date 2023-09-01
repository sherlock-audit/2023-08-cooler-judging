Steep Bamboo Elk

high

# Malicious user can get free gOHM keeper rewards by Taking their own loan, defaulting and calling claimDefaulted on themselves
## Summary

If a user takes their own loan and defaults themselves (this can be done on 2 separate addresses), they don't lose any money apart from gas fees because you're claiming all the collateral yourself. You could take a loan from a cooler created by a clearingHouse contract with duration of 1 second and let it default.

There are 2 core issues here:

1. The claimDefault does not check that the defaulted loans were craeted by the cooler. So if the cooler originated from the clearinghouse, the loan could be created by any user.
2. Users don't lose any money by loaning and defaulting themselves, so doing this purposefully can be used to farm free gOHM

Then you can call `claimDefaulted()` on your own loan to get free `keeperRewards`

## Vulnerability Detail

ClearingHouse gives free gOHM to anybody who helps to claim defaulted loans 

```solidity
function claimDefaulted(address[] calldata coolers_, uint256[] calldata loans_)
```

This free gOHM is given even if the loan was not created by the cooling house. Therefore, a user could create 2 accounts, request a loan with duration of 1 second and large collateral from account 1 and clear the loan on account 2. Then they allow the loan to default.

They call `claimDefaulted`
This:
- Gives free gOHM tokens proportional to time and collateral:

```solidity
uint256 maxAuctionReward = collateral * 5e16 / 1e18;
            // Cap rewards to avoid exorbitant amounts.
            uint256 maxReward = (maxAuctionReward < MAX_REWARD)
                ? maxAuctionReward
                : MAX_REWARD;
            // Calculate rewards based on the elapsed time since default.
            keeperRewards = (elapsed < 7 days)
                ? keeperRewards + maxReward * elapsed / 7 days
                : keeperRewards + maxReward;
```

- We can increase the rewards by setting large collateral and waiting sometime before defaulting. If somebody else tries to default our loan we can frontrun that transaction with our own `defaultClaim()` transaction
- The collateral is seized by the lender (account 1), and the loan tokens are held by the borrower (account 2). In the end account 1 and account 2 in total still get out the same collateral tokens and loan tokens they put in. 
- Therefore the keeper rewards were taken for free

## Impact

Malicious user can get free keeper rewards, draining the gOHM reward pool.

## Code Snippet

https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L191-L245

## Tool used

Manual Review

## Recommendation

Only incentivise loans made by ClearingHouse with keeper rewards
