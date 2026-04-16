<!DOCTYPE html>
<html>
<head>
<style>
    .full-page {
        width:  100%;
        height:  100vh; /* This will make the div take up the full viewport height */
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }
    .full-page img {
        width: 500px;
        height: 500px;
        max-width:  200;
        max-height:  200;
        margin-bottom: 5rem;
    }
    .full-page div{
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }
</style>
</head>
<body>

<div class="full-page">
    <img src="../../assets/Logo.svg" alt="Logo">
    <div>
    <h1>Iron Bank Shadow Audit Report</h1>
    <h3>Prepared by: HEXXA Protocol</h3>
    </div>
</div>

</body>
</html>

<!-- Your report starts here! -->
# Iron Bank Audit Report


- Title: Iron Bank Audit Report
- author: Gaurang Bharadava
- date: April 16, 2026


Prepared by: HEXXA Protocol
- Lead Auditors: [Gaurang Bharadava](#about-gaurang-bharadava)

Assisting Auditors:
- None

# Table of contents
<details>

<summary>See table</summary>

- [Iron Bank Audit Report](#iron-bank-audit-report)
- [Table of contents](#table-of-contents)
- [About Gaurang Bharadava](#about-gaurang-bharadava)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Protocol Summary](#protocol-summary)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [Using stale `borrowIndex` to calculate user borrow might return wrong debt value or prevent liquidation](#using-stale-borrowindex-to-calculate-user-borrow-might-return-wrong-debt-value-or-prevent-liquidation)
    - [Missed findings - High](#missed-findings---high)
        -[`supplyNativeToken` will strand ETH when used after deferred liquidity check due to `msg.value` context loss](#supplynativetoken-will-strand-eth-when-used-after-deferred-liquidity-check-due-to-msgvalue-context-loss)
  - [Medium](#medium)
    - [No checks in `PriceOracle` for price returned by `getPriceFromChainlink` function](#no-checks-in-priceoracle-for-price-returned-by-getpricefromchainlink-function)
    - [Missed findings - Medium](#missed-findings---medium)
        -[Oracle will return wrong price if underlying aggregator hits `minAnswer`](#oracle-will-return-wrong-price-if-underlying-aggregator-hits-minanswer)
        - [Price oracle will return wrong price for PToken which has WstEth as underlying asset](#price-oracle-will-return-wrong-price-for-ptoken-which-has-wsteth-as-underlying-asset)
</details>
</br>

# About Gaurang Bharadava

Gaurang Bharadava is experienced smart contract engineer and security researcher. Building HEXXA Protocol, will provide Smart contract development and security audit.

# Disclaimer

The HEXXA Protocol team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |


# Protocol Summary

Iron Bank is a decentralized lending platform focused on capital efficiency allowing protocols and individuals to supply and borrow cryptoassets. 

# Executive Summary

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 2                      |
| Medium   | 1                      |
| Low      | 0                      |
| Info     | 0                      |
| Gas      | 0                      |
| Missed   | 2                      |
| Total    | 5                      |

# Findings

## High

# $> Using stale `borrowIndex` to calculate user borrow might return wrong debt value or prevent liquidation

## Description

In `IronBank.sol`, the `_getBorrowBalance` function is used to calculate and return the user borrow balance (includes interest) for specific market. However this is not the issue here.

This function is used by `_getAccountLiquidity` and `_isLiquidatable` function, which loops thorugh all the market in which the user has entered or borrowed the asset. However these functions blindly call `_getBorrowBalance` function for the markets.

As result there could be such market that are not active since long or they are having stale `borrowIndex`.

```solidity
function _isLiquidatable(address user) internal view returns (bool) {
    uint256 liquidationCollateralValue;
    uint256 debtValue;

    address[] memory userEnteredMarkets = allEnteredMarkets[user];
    for (uint256 i = 0; i < userEnteredMarkets.length; i++) {
        DataTypes.Market storage m = markets[userEnteredMarkets[i]];
        if (!m.config.isListed) {
            continue;
        }
 @>     uint256 borrowBalance = _getBorrowBalance(m, user);
        
```

```solidity
function _getBorrowBalance(DataTypes.Market storage m, address user) internal view returns (uint256) {
    DataTypes.UserBorrow memory b = m.userBorrows[user]; //>/ get user borrow balance if there is

    if (b.borrowBalance == 0) {
        return 0;
    } //>/ return zero if no borrow has been made for perticular market

    // borrowBalanceWithInterests = borrowBalance * marketBorrowIndex / userBorrowIndex
@>  return (b.borrowBalance * m.borrowIndex) / b.borrowIndex; 
}
```

If there is any stale market then above function will calculate wrong debt value for user for that market, However the intended use of above function is to calculate and return debt value (updated with interest), is failing here. 

## Risk

**Impact**:

* `_getAccountLiquidity` function will return wrong debt value that could break the system.

* User might be unable to liquidate

## Proof of Concept

```solidity
function test_hypo_2() public {
    uint256 market1SupplyAmount = 100e18;
    uint256 market2BorrowAmount = 500e18;
    
    console2.log("=== SETUP ===");
    console2.log("Market1: $1500, Market2: $200, CF: 80%");
    
    // User2 provides liquidity
    vm.startPrank(user2);
    market2.approve(address(ib), 10000e18);
    ib.supply(user2, user2, address(market2), 10000e18);
    vm.stopPrank();

    // User1 creates position
    vm.startPrank(user1);
    market1.approve(address(ib), market1SupplyAmount);
    ib.supply(user1, user1, address(market1), market1SupplyAmount);
    ib.borrow(user1, user1, address(market2), market2BorrowAmount);
    vm.stopPrank();
    
    console2.log("\n=== INITIAL STATE ===");
    {
        (uint256 col, uint256 debt) = ib.getAccountLiquidity(user1);
        console2.log("Collateral: $", col / 1e18);
        console2.log("Debt: $", debt / 1e18);
        console2.log("Liquidatable?", ib.isUserLiquidatable(user1));
    }
    
    // Time passes
    console2.log("\n=== 365 DAYS LATER (NO MARKET2 ACTIVITY) ===");
    vm.warp(block.timestamp + 365 days);
    
    // Check STALE state
    uint256 staleBorrow = ib.getBorrowBalance(user1, address(market2));
    console2.log("Borrow (stale):", staleBorrow / 1e18);
    
    uint256 staleDebt;
    {
        (, uint256 debt) = ib.getAccountLiquidity(user1);
        staleDebt = debt;
        console2.log("Debt (stale): $", debt / 1e18);
        console2.log("Liquidatable (stale)?", ib.isUserLiquidatable(user1));
    }
    
    // Accrue interest
    console2.log("\n=== ACCRUE INTEREST ===");
    ib.accrueInterest(address(market2));
    
    // Check ACTUAL state
    uint256 actualBorrow = ib.getBorrowBalance(user1, address(market2));
    console2.log("Borrow (actual):", actualBorrow / 1e18);
    console2.log("Interest:", (actualBorrow - staleBorrow) / 1e18);
    
    uint256 actualDebt;
    {
        (, uint256 debt) = ib.getAccountLiquidity(user1);
        actualDebt = debt;
        console2.log("Debt (actual): $", debt / 1e18);
        console2.log("Hidden debt: $", (debt - staleDebt) / 1e18);
    }
    
    // Price drop
    console2.log("\n=== PRICE DROP ===");
    
    // Calculate price to make user barely liquidatable
    uint256 newPrice = (actualDebt * 1e18 * 10000 * 99) / (market1SupplyAmount * 8000 * 100);
    console2.log("New price: $", newPrice / 1e18);
    
    vm.prank(admin);
    registry.setAnswer(address(market1), Denominations.USD, int256(newPrice / 1e10));
    
    // Final check
    console2.log("\n=== FINAL STATE ===");
    {
        (uint256 col, ) = ib.getAccountLiquidity(user1);
        console2.log("Collateral: $", col / 1e18);
    }
    
    console2.log("Debt (if stale): $", staleDebt / 1e18);
    console2.log("Debt (actual): $", actualDebt / 1e18);
    
    bool liquidatable = ib.isUserLiquidatable(user1);
    console2.log("Liquidatable?", liquidatable);
    
    // Calculate what stale would show
    uint256 colValue = (market1SupplyAmount * newPrice * 8000) / (1e18 * 10000);
    bool staleWouldSayLiquidatable = staleDebt > colValue;
    bool actualSaysLiquidatable = actualDebt > colValue;
    
    console2.log("\n=== BUG PROOF ===");
    console2.log("Stale would say:", staleWouldSayLiquidatable ? "LIQUIDATABLE" : "SAFE");
    console2.log("Actual says:", actualSaysLiquidatable ? "LIQUIDATABLE" : "SAFE");
    
    if (actualSaysLiquidatable != staleWouldSayLiquidatable) {
        console2.log("!!! BUG CONFIRMED: Different results !!!");
    }
}
```

## Recommended Mitigation

Update the market before using it's `borrowIdex`

## Missed findings - High

# $> `supplyNativeToken` will strand ETH when used after deferred liquidity check due to `msg.value` context loss

## Description

The `supplyNativeToken` function relies on `msg.value` to wrap ETH into WETH and supply it to IronBank:

```solidity
function supplyNativeToken(address user) internal nonReentrant {
@>    WethInterface(weth).deposit{value: msg.value}();
    IERC20(weth).safeIncreaseAllowance(address(ironBank), msg.value);
    ironBank.supply(address(this), user, weth, msg.value);
}
```

This implementation assumes that msg.value correctly reflects the ETH sent by the user. However, this assumption breaks when ACTION_DEFER_LIQUIDITY_CHECK is used.

execution flow:

```bash
execute → executeInternal → ACTION_DEFER_LIQUIDITY_CHECK  
→ ironBank.deferLiquidityCheck(...)  
→ onDeferredLiquidityCheck(...)  // new external call context  
→ executeInternal → supplyNativeToken
```

When ironBank.deferLiquidityCheck is called, execution is paused and later resumed via the callback onDeferredLiquidityCheck. This callback is a new external call, which creates a new execution context.

In this new context:

- `msg.sender` = `iron bank`
- `msg.value` = 0

As a result, when supplyNativeToken is executed after the callback:

```solidity
WethInterface(weth).deposit{value: msg.value}(); // msg.value == 0
```

No ETH is deposited into WETH, and no collateral is supplied to IronBank. However, the original ETH sent by the user remains in the contract balance

This leads to a mismatch where:

- ETH is received by the contract
- But not accounted for or supplied on behalf of the user

## Risk

User will lose their funds and permanently locked in the contract, user can be incorreclty liquidate because their deposit is not transfered to iron bank.



## Medium

# $> No checks in `PriceOracle` for price returned by `getPriceFromChainlink` function

## Description

The `getPriceFromChainlink` function fetched the price from chainlink oracle, However as shown in code below it only reads the price

```solidity
function getPriceFromChainlink(address base, address quote) internal view returns (uint256) {
    (, int256 price,,,) = registry.latestRoundData(base, quote);
    require(price > 0, "invalid price");

    // Extend the decimals to 1e18.
    return uint256(price) * 10 ** (18 - uint256(registry.decimals(base, quote))); //>/ @audit it assumes the price feed decimal will always less then 18
}
```

There is be possiblity that chainlink price could stale, However in that case it just uses the price, there is no such mechanism such that we can check and discard stale price value.

## Risk

Wrong liquidations and wrong collateral / debt value might possible

## Recommended Mitigation

Add checks and staleness threshold to prevent using stale price.



## Missed findings - Medium

# $> Oracle will return wrong price if underlying aggregator hits `minAnswer`

## Description

The `getPriceFromChainlink` function fetched the price from chainlink oracle, However chainlink feeds have built in safety limit such as minAnswer and maxAnswer. chainlink oracle does not revert if the asset's current price gose beyond this price limit. it just return the boundry value. That means the oracle will return wrong price for an asset

## Risk

**Impact**:

There are multi ple impacts such as wrong liquidation, wrong collateral and debt value etc.


# $> Price oracle will return wrong price for PToken which has WstEth as underlying asset

## Description

The `getPrice` function returns the price of an asset, However there is multi step has been performed to derive the price of WstEth as shown below

```solidity
function getPrice(address asset) external view returns (uint256) {
@>    if (asset == wsteth) {
        uint256 stEthPrice = getPriceFromChainlink(steth, Denominations.USD); //>/ get price of staked eth in usd in 18 decimals
        uint256 stEthPerToken = WstEthInterface(wsteth).stEthPerToken(); //>/ returns the amount of stETH per wstETH token in 18 decimals precision
        uint256 wstEthPrice = (stEthPrice * stEthPerToken) / 1e18;
        return getNormalizedPrice(wstEthPrice, asset);
    }

    AggregatorInfo memory aggregatorInfo = aggregators[asset];
    uint256 price = getPriceFromChainlink(aggregatorInfo.base, aggregatorInfo.quote);
    if (aggregatorInfo.quote == Denominations.ETH) {
        // Convert the price to USD based if it's ETH based.
        uint256 ethUsdPrice = getPriceFromChainlink(Denominations.ETH, Denominations.USD);
        price = (price * ethUsdPrice) / 1e18;
    }
    return getNormalizedPrice(price, asset); //>/ @audit look deep into this
}
```

otherwise if asset is anyother normal token then there is strait forward mechanism.

However the IronBank supports PToken as market which can have underlying token. However if PToken will be having WstEth as underlying token then the price will be calculated strait forward for that asset. but the price must follow the 2 steps as shown in function above. 

There is mismatch in caulating the price for WstEth when it is in form of PToken.

## Risk

Since users holding PToken for WstETH will have wrong valuation, this potentially creates opportunities for malicious over-borrowing or unfair liquidations, putting the protocol at risk.

## Recommended Mitigation

Use underlying asset to fetch price for PToken
