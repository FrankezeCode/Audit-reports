| **title**                  | **author**            | **date**           |
|----------------------------|-----------------------|--------------------|
| Thunder Loan Audit Report  | Franklyn Ezeugonna    | May 2, 2024        | 

# Thunder Loan Audit Report
Prepared by: Franklyn Ezeugonna
Lead Auditors: 

- [Franklyn Ezeugonna](https://github.com/FrankezeCode)

Assisting Auditors:

- None

# Table of Contents
- [Thunder Loan Audit Report](#thunder-loan-audit-report)
- [Table of Contents](#table-of-contents)
- [About me](#about-me)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
- [Protocol Summary](#protocol-summary)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Erroneous `ThunderLoan::UpdateExchangeRate` in the `deposit` function causes protocol to think it has more fees than it really does, which blocks redemption and incorrectly sets the exchange rate.](#h-1-erroneous-thunderloanupdateexchangerate-in-the-deposit-function-causes-protocol-to-think-it-has-more-fees-than-it-really-does-which-blocks-redemption-and-incorrectly-sets-the-exchange-rate)
    - [\[H-2\] Mixing up variable location causes storage collision in `ThunderLoan::s_flashLoanfee` and `ThunderLoan::s_currentlyFlashloaning` , freezing protocol](#h-2-mixing-up-variable-location-causes-storage-collision-in-thunderloans_flashloanfee-and-thunderloans_currentlyflashloaning--freezing-protocol)
  - [Medium](#medium)
    - [\[M-1\] Centralization risk for trusted owners](#m-1-centralization-risk-for-trusted-owners)
      - [Impact:](#impact)
      - [Contralized owners can brick redemptions by disapproving of a specific token](#contralized-owners-can-brick-redemptions-by-disapproving-of-a-specific-token)
    - [\[M-2\] Using TSwap as price oracle leads to price and oracle manipulation attacks](#m-2-using-tswap-as-price-oracle-leads-to-price-and-oracle-manipulation-attacks)
  - [Low](#low)
    - [\[L-1\] Missing critial event emissions](#l-1-missing-critial-event-emissions)


# About me
I'm passionate about uncovering vulnerabilities in systems and smart contract , always curious and eager to learn . Most importantly, I love making new friends . Feel free to reach out.

# Disclaimer

I Franklyn Ezeugonna makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the solidity implementation of the contracts.

# Risk Classification

|                |        | Impact |        |     |
| -------------- | ------ | ------ | ------ | --- |
|                |        | High   | Medium | Low |
|                | High   | H      | H/M    | M   |
| **Likelihood** | Medium | H/M    | M      | M/L |
|                | Low    | M      | M/L    | L   |

# Audit Details

**The findings described in this document correspond the following commit hash:**
```
026da6e73fde0dd0a650d623d0411547e3188909
```

## Scope 

```
#-- interfaces
|   #-- IFlashLoanReceiver.sol
|   #-- IPoolFactory.sol
|   #-- ITSwapPool.sol
|   #-- IThunderLoan.sol
#-- protocol
|   #-- AssetToken.sol
|   #-- OracleUpgradeable.sol
|   #-- ThunderLoan.sol
#-- upgradedProtocol
    #-- ThunderLoanUpgraded.sol
```
Solc Version: 0.8.20
Chain(s) to deploy contract to: Ethereum
ERC20s: USDC , DAI , LINK , WETH

# Protocol Summary 

The ⚡️ThunderLoan⚡️ protocol is meant to do the following:

- Give users a way to create flash loans
- Give liquidity providers a way to earn money off their capital 

## Roles

- Owner: The owner of the protocol who has the power to upgrade the implementation.
- Liquidity Provider: A user who deposits assets into the protocol to earn interest.
- User: A user who takes out flash loans from the protocol.

# Executive Summary

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 2                      |
| Medium   | 2                      |
| Low      | 1                      |
| Info     | 0                      |
| Gas      | 0                      |
| Total    | 5                      |

# Findings

## High 

### [H-1] Erroneous `ThunderLoan::UpdateExchangeRate` in the `deposit` function causes protocol to think it has more fees than it really does, which blocks redemption and incorrectly sets the exchange rate.

**Description:** In the ThunderLoan system , the `exchangeRate` is responsible for calculating the exchange rate betwween assetTokens and underlying tokens. In a way , it's rseponsible for keeping track of how many fees to give to liquidity providers.

However, the `deposit` function, updates this rate, without collecting any fees!

```javascript
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token]; 
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
        // @audit-high
@>      uint256 calculatedFee = getCalculatedFee(token, amount);
@>      assetToken.updateExchangeRate(calculatedFee);
        token.safeTransferFrom(msg.sender, address(assetToken), amount); 
    }
```
**Impact:** There are several impacts to this bug.
1. The `redeem` function is blocked, because the protocol thinks the owed token is more than it has
2. Rewards are incorrectly calculated, leading to liquidity providers potentially getting way more or less than desrved.

**Proof of Concept:**
1. LP deposits
2. Users takes out a flash loan
3. It is now impossible for LP to redeem.

<details>
<summary>Proof of Code<summary>

Place the following into `ThunderloanTest.t.sol`

```javascript
  function testRedeemAfterLoan() public setAllowedToken hasDeposits {
        uint256 amountToBorrow = AMOUNT * 10;
        uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);
        vm.startPrank(user);
        tokenA.mint(address(mockFlashLoanReceiver),calculatedFee);// fee
        thunderLoan.flashloan(address(mockFlashLoanReceiver), tokenA, amountToBorrow, "");
        vm.stopPrank();

        uint256 amountToRedeem = type(uint256).max;
        vm.startPrank(liquidityProvider);
        thunderLoan.redeem(tokenA, amountToRedeem);
    }
```
<details>

**Recommended Mitigation:** Removed the incorrect updated exchange rate lines from `deposit`

```diff
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token]; 
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
        // @audit-high
-       uint256 calculatedFee = getCalculatedFee(token, amount);
-       assetToken.updateExchangeRate(calculatedFee);
        token.safeTransferFrom(msg.sender, address(assetToken), amount); 
    }
```

### [H-2] Mixing up variable location causes storage collision in `ThunderLoan::s_flashLoanfee` and `ThunderLoan::s_currentlyFlashloaning` , freezing protocol

**Description:**  `ThunderLoan.sol` has two variables in the following order:

```javascript
    uint256 private s_feePrecision; 
    uint256 private s_flashLoanFee; // 0.3% ETH fee
```

However, the upgraded contract `ThunderLoanUpgraded.sol` has them in a different order:

```javascript
    uint256 private s_flashLoanFee; // 0.3% ETH fee
    uint256 public constant FEE_PRECISION = 1e18;
```
Due to how solidity storage works, after the upgrade the `s_flashLoanFee` will have the value of `s_feePrecision` . You cannot adjust the position of storage variables, and removing storage variable for constant variables, breaks the storage locations as well.

**Impact:** After  the upgrade, the `s_flashLoanFee` will have the value of `s_feePrecision`, This means that users who take out flash loans right after an upgrade will be charged the wrong fee.

More importantly, the `s_currentlyFlashLoaning` mapping with storage in the wrong storage slot.

**Proof of Concept:**

<details>
<summary>PoC</summary>

Place the following into `ThunderLoanTest.t.sol`.

```javascript
import {ThunderLoanUpgraded} from "src/upgradedProtocol/ThunderLoanUpgraded.sol";
.
.
.
 function testUpgradeBreaks() public {
        uint256 feeBeforeUpgrade = thunderLoan.getFee();
        vm.startPrank(thunderLoan.owner());
        ThunderLoanUpgraded upgraded = new ThunderLoanUpgraded();
        thunderLoan.upgradeToAndCall(address(upgraded) , "");
        uint256 feeAfterUpgrade = thunderLoan.getFee();
        vm.stopPrank();

        console2.log("Fee Before:", feeBeforeUpgrade);
        console2.log("Fee After:", feeAfterUpgrade);
        assert(feeBeforeUpgrade != feeAfterUpgrade);
    }
```
You can also see the storage layout difference by running `forge inspect ThunderLoan storage` and `forge inspect ThunderLoanUpgraded storage`

</details>

**Recommended Mitigation:** If you must remove the storage variable, leave it as blank as to not mess up the storage slots.

```diff
-  uint256 private s_feePrecision;
-  uint256 private s_flashLoanFee; // 0.3% ETH fee
+  uint256 private s_blank;
+  uint256 private s_flashLoanFee; // 0.3% ETH fee
+  uint256 public constant FEE_PRECISION = 1e18
```
## Medium 

### [M-1] Centralization risk for trusted owners

#### Impact:
Contracts have owners with privileged rights to perform admin tasks and need to be trusted to not perform malicious updates or drain funds.

*Instances (2)*:
```solidity
File: src/protocol/ThunderLoan.sol

223:     function setAllowedToken(IERC20 token, bool allowed) external onlyOwner returns (AssetToken) {

261:     function _authorizeUpgrade(address newImplementation) internal override onlyOwner { }
```

#### Contralized owners can brick redemptions by disapproving of a specific token


### [M-2] Using TSwap as price oracle leads to price and oracle manipulation attacks

**Description:** The TSwap protocol is a constant product formula based AMM (automated market maker). The price of a token is determined by how many reserves are on either side of the pool. Because of this, it is easy for malicious users to manipulate the price of a token by buying or selling a large amount of the token in the same transaction, essentially ignoring protocol fees. 

**Impact:** Liquidity providers will drastically reduced fees for providing liquidity. 

**Proof of Concept:** 

The following all happens in 1 transaction. 

1. User takes a flash loan from `ThunderLoan` for 1000 `tokenA`. They are charged the original fee `fee1`. During the flash loan, they do the following:
   1. User sells 1000 `tokenA`, tanking the price. 
   2. Instead of repaying right away, the user takes out another flash loan for another 1000 `tokenA`. 
      1. Due to the fact that the way `ThunderLoan` calculates price based on the `TSwapPool` this second flash loan is substantially cheaper. 
```javascript
    function getPriceInWeth(address token) public view returns (uint256) {
        address swapPoolOfToken = IPoolFactory(s_poolFactory).getPool(token);
@>      return ITSwapPool(swapPoolOfToken).getPriceOfOnePoolTokenInWeth();
    }
```
    3. The user then repays the first flash loan, and then repays the second flash loan.


**Recommended Mitigation:** Consider using a different price oracle mechanism, like a Chainlink price feed with a Uniswap TWAP fallback oracle. 

## Low

### [L-1] Missing critial event emissions

**Description:** When the `ThunderLoan::s_flashLoanFee` is updated, there is no event emitted. 

**Recommended Mitigation:** Emit an event when the `ThunderLoan::s_flashLoanFee` is updated.

```diff 
+    event FlashLoanFeeUpdated(uint256 newFee);
.
.
.
    function updateFlashLoanFee(uint256 newFee) external onlyOwner {
        if (newFee > s_feePrecision) {
            revert ThunderLoan__BadNewFee();
        }
        s_flashLoanFee = newFee;
+       emit FlashLoanFeeUpdated(newFee); 
    }
```

