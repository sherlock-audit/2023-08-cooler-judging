Salty Flaxen Reindeer

medium

# ERC4626 vaults vulnerable large asset deposits breaking share calculations, especially by early depositors
## Summary
Vault shares can be manipulated by making a large deposit early on affecting value shares, redemptions etc 

## Vulnerability Detail
[sdai_ in contacts is wrapped into Vault ](https://github.com/sherlock-audit/2023-08-cooler/blob/main/Cooler/src/Clearinghouse.sol#L86C8-L86C31) that uses Solmate => import {ERC4626} from "solmate/mixins/ERC4626.sol";

This is a consequence and affects code related to the _sweepIntoDSR() and defund() functions which make use of ERC4626 vault functions convertToAssets(uint256 shares)

Vaults that use ERC4626 are susceptible to share price manipulation that affects subsequent deposits if the earlier deposits are accompanied by a large vault deposit exploit that inflates the ERC4626.totalAssets

This  inflated  base share price that is high will force all subsequence deposit to use this share price as a base and result in low shares or even zero shares due to rounding down since shares calculated using mulDivDown below:
```solidity 
return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
```
In case of a very high share price, due to totalAssets() > assets * supply, shares will be 0.

See Past Audit Reports as Reference Below:
[See Other Audit Examples M-13 Astaria - First ERC4626 deposit can break share calculation](https://solodit.xyz/issues/m-13-first-erc4626-deposit-can-break-share-calculation-sherlock-astaria-astaria-git)

[First xERC4626 deposit exploit can break share calculation](https://solodit.xyz/issues/m-02-first-xerc4626-deposit-exploit-can-break-share-calculation-code4rena-tribe-xtribe-contest-git)

## Impact
ERC4626 vault share price can be maliciously inflated leading to later deposits getting very small shares or even zero shares for subsequent deposits into vault. Therfore the dai that is deposited or swept into the vault can be lost as zero rounded value results in no funds being returned to treasury etc. Additionaly the values at redemption are disadvantaged as they are made smaller due to proportion with the large attack deposit into vault. 

## Code Snippet

```solidity
    function sweepIntoDSR() public {
        uint256 daiBalance = dai.balanceOf(address(this));
        _sweepIntoDSR(daiBalance);
    }

    /// @notice Sweep excess DAI into vault.
    function _sweepIntoDSR(uint256 amount_) internal {
        dai.approve(address(sdai), amount_);
        sdai.deposit(amount_, address(this));
    }

// defund() redemptions impacted
            uint256 daiAmount = (token_ == sdai)
                ? sdai.previewRedeem(amount_)
                : amount_;
    
            TRSRY.setDebt({
                debtor_: address(this),
                token_: dai,
                amount_: (outstandingDebt > daiAmount) ? outstandingDebt - daiAmount : 0
            });

// ERC4626 vault function convertShares
function convertToAssets(uint256 shares) public view virtual returns (uint256) {
        uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

        return supply == 0 ? shares : shares.mulDivDown(totalAssets(), supply);
    }


```

## Tool used
Manual Review

## Recommendation
Consider an initial large deposit into vault when contracts are deployed so that shares can be proportioned more fairly 

