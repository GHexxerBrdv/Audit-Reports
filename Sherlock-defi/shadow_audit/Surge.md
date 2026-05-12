# Findings Overview

| ID | Severity | Title |
|----|----------|-------|
| H-01 | High | [Lack of Decimal Normalization in Collateral Ratio Calculation Allows Undercollateralized Borrowing](#lack-of-decimal-normalization-in-collateral-ratio-calculation-allows-undercollateralized-borrowing) |
| H-02(missed) | High | [First depositor can inflate share price and steal funds from other users](#first-depositor-can-inflate-share-price-and-steal-funds-from-other-users) |
| M-01(missed) | Medium | [Pool.sol::approve can be frontrun](#poolsolapprove-can-be-frontrun) |
| M-02(missed) | Medium | [Operator can cause fee share to transfer to zero address](#operator-can-cause-fee-share-to-transfer-to-zero-address) |
| M-03(missed) | Medium | [Borrower can skip the interest accrual by interacting with the pool each block](#borrower-can-skip-the-interest-accrual-by-interacting-with-the-pool-each-block) |
| M-04(missed) | Medium | [Incorrect _accruedFeeShares Calculation Causes Excessive Fee Dilution](#incorrect-_accruedfeeshares-calculation-causes-excessive-fee-dilution) |
| M-05(missed) | Medium | [Attacker can freeze the collateral ratio](#attacker-can-freeze-the-collateral-ratio) |
| M-06(missed) | Medium | [Liquidator can distribute user's debt to remaining borrowers partially](#liquidator-can-distribute-users-debt-to-remaining-borrowers-partially) |

# High

---

## > Lack of Decimal Normalization in Collateral Ratio Calculation Allows Undercollateralized Borrowing

## Severity
**High**

## Summary
The Surge protocol fails to account for decimal differences between collateral and loan tokens when calculating the user's collateral ratio. This allows attackers to borrow the entire pool with near-zero collateral if the loan token has fewer decimals than the collateral token (e.g., USDC/WETH), or bricks the pool if the loan token has more decimals.

## Vulnerability Detail
In `Pool.sol`, the user's collateral ratio is calculated as:

```solidity
uint userCollateralRatioMantissa = userDebt * 1e18 / collateralBalanceOf[msg.sender];
```

This calculation is used in `borrow`, `removeCollateral`, and `liquidate`. However, it directly divides `userDebt` (denominated in loan token units) by `collateralBalanceOf` (denominated in collateral token units) without normalizing for their respective decimals.

Consider a pool with:
- **Loan Token**: USDC (6 decimals)
- **Collateral Token**: WETH (18 decimals)
- **Max Collateral Ratio**: 1e18 (100%)

An attacker wants to borrow 3,200 USDC ($3,200).
- `userDebt` = 3,200 * 10^6 = 3.2e9
- To pass the check `userCollateralRatioMantissa <= maxCollateralRatioMantissa`:
  `3.2e9 * 1e18 / collateralBalance <= 1e18`
  `3.2e9 <= collateralBalance`

The attacker only needs **3.2e9 units of WETH**, which is **0.0000000032 WETH** (approx. $0.00001). This allows the attacker to drain the pool's loan tokens with negligible collateral.

Conversely, if the loan token has more decimals than the collateral token (e.g., WETH/USDC), the calculated ratio will be so high that borrowing becomes impossible, bricking the pool.

## Impact
Permissionless pool creators are likely to use standard 1e18 values for `MAX_COLLATERAL_RATIO_MANTISSA` without realizing they must manually adjust for decimal differences. This leads to:
1. **Lender Loss**: Attacker drains the pool with mass-less collateral.
2. **Broken Pools**: Legitimate users cannot borrow in pools where loan decimals > collateral decimals.

Since the pool creation is permissionless and misleadingly uses the term `MANTISSA` (conventionally 1e18), this is a high-probability, high-impact issue for lenders.

## Code Snippet

```solidity
uint userCollateralRatioMantissa = userDebt * 1e18 / collateralBalanceOf[msg.sender];
```


## Proof of Concept
Add the following test to `test/Pool.t.sol` and run `forge test --mt test_drain_pool`:

```solidity
function test_drain_pool() public {
    // Setup: WETH (18 decimals) as collateral, USDC (6 decimals) as loan
    uint256 loanAmount = 2000e6;  // 2000 USDC
    MockERC20 weth = new MockERC20(100e18, 18);
    MockERC20 usdc = new MockERC20(2 * loanAmount, 6);
    
    Pool pool = factory.deploySurgePool(
        IERC20(address(weth)), usdc, 1e18, 0.8e18, 1e15, 1e15, 0.1e18, 0.4e18, 0.6e18
    );

    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address attacker = makeAddr("attacker");

    // Lenders deposit 4000 USDC total
    usdc.transfer(alice, loanAmount);
    usdc.transfer(bob, loanAmount);
    vm.prank(alice); usdc.approve(address(pool), loanAmount);
    vm.prank(bob); usdc.approve(address(pool), loanAmount);
    vm.prank(alice); pool.deposit(loanAmount);
    vm.prank(bob); pool.deposit(loanAmount);

    // Attacker deposits tiny WETH collateral (~$0)
    uint256 tinyCollateral = 3200e6; 
    weth.transfer(attacker, tinyCollateral);
    
    vm.startPrank(attacker);
    weth.approve(address(pool), tinyCollateral);
    pool.addCollateral(attacker, tinyCollateral);
    pool.borrow(3200e6); // Borrow 3200 USDC
    vm.stopPrank();

    assertEq(usdc.balanceOf(attacker), 3200e6);
}
```

## Recommendation
Normalize the collateral balance to the loan token's decimals before performing the ratio calculation.

```diff
- uint userCollateralRatioMantissa = userDebt * 1e18 / collateralBalanceOf[msg.sender];
+ uint8 loanDecimals = LOAN_TOKEN.decimals();
+ uint8 collateralDecimals = COLLATERAL_TOKEN.decimals();
+ uint normalizedCollateral = collateralBalanceOf[msg.sender];
+ if (loanDecimals > collateralDecimals) {
+     normalizedCollateral = normalizedCollateral * (10 ** (loanDecimals - collateralDecimals));
+ } else if (collateralDecimals > loanDecimals) {
+     normalizedCollateral = normalizedCollateral / (10 ** (collateralDecimals - loanDecimals));
+ }
+ uint userCollateralRatioMantissa = userDebt * 1e18 / normalizedCollateral;
```

---

# Missed

---

# High

---

## > First depositor can inflate share price and steal funds from other users

## Summary
Attacker can first deposit small amount of loan token to get pool tokens, and front-run other depositors' transactions and inflate pool token price by large "donation", thus attacker can withdraw more loan tokens than he initially owned.

# Vulnerability Detail
User can get pool token by depositing loan tokens to Pool, the amount of minted pool token is calculated as:

```solidity
function tokenToShares (uint _tokenAmount, uint _supplied, uint _sharesTotalSupply, bool roundUpCheck) internal pure returns (uint) {
    if(_supplied == 0) return _tokenAmount;
    uint shares = _tokenAmount * _sharesTotalSupply / _supplied;
    if(roundUpCheck && shares * _supplied < _tokenAmount * _sharesTotalSupply) shares++;
    return shares;
}
```
Let's assume:

- Alice deployes a pool and submits transaction to deposit 2 ether;
- Bob sees Alice's transaction in mempool and front-runs by first depositing 1 wei to the pool and then get 1 pool token;
- Bob then transfers 1 ether loan token directly to the pool, inflates pool token price to (1 ether + 1);
- Alice's deposit transaction gets confirmed and Alice get 1 pool token;
- Bob withdraw from pool and get 1.5 ether loan tokens back, making 0.5 ether profit.

Test Code for PoC:
```solidity
function testFirstDeposit() public {
    address alice = address(1);
    address bob = address(2);

    deal(address(loanToken), bob, 1 ether + 1);
    deal(address(loanToken), alice, 2 ether);

    vm.startPrank(bob);
    loanToken.approve(address(pool), 1);
    pool.deposit(1);
    loanToken.transfer(address(pool), 1 ether);
    vm.stopPrank();

    vm.startPrank(alice);
    loanToken.approve(address(pool), 2 ether);
    pool.deposit(2 ether);
    vm.stopPrank();

    vm.startPrank(bob);
    pool.withdraw(type(uint256).max);
    assertEq(loanToken.balanceOf(bob), 1500000000000000000);
    vm.stopPrank();
}
```

## Impact
User's deposited loan tokens may be stolen by attacker.

---

# Medium 

---

## > `Pool.sol::approve` can be frontrun

## Severity
**Medium**

## Summary
The `Pool.sol::approve` function can be frontrun.

## Vulnerability Detail
In `Pool.sol`, User can approve an address using the `approve` function. However if user again approves the same address with a different amount, and the spender frontrun the transaction and calls `transferFrom`. the old approval is used complately and the new approval can be used again by the spender.

## Impact
- user loses the funds

## Code Snippet
```solidity
    function approve(address spender, uint amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
```

```solidity
    function transferFrom(address from, address to, uint amount) external returns (bool) {
        require(to != address(0), "Pool: to cannot be address 0");
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        unchecked {
            balanceOf[to] += amount;
        }
        emit Transfer(from, to, amount);
        return true;
    }
```


## Proof of Concept
1. msg.sender calls Pool.approve(spender, 50)
2. msg.sender then changes its mind and instead want to change the previous approve to 100 by calling again Pool.approve
3. Now if the spender calls Pool.transferFrom before the event (2), s/he will gain the first 50 tokens, and then again 100 tokens, resulting in the first actor losing 50 tokens

## Recommendation
Before approving an account with new amount, the user must set approve to 0 first. if spender act maliciously, then the future funds can be prevented from being spent incorrectly.

---

## > Operator can cause fee share to transfer to zero address

## Severity
**Medium**

## Summary
The `Factory::setFeeMantissa` function sets the fee mantissa for the pool. However fee mantissa can be be set to positive number if the fee recipient is zero address, 

```solidity
    function setFeeMantissa(uint _feeMantissa) external {
        require(msg.sender == operator, "Factory: not operator");
        require(_feeMantissa <= MAX_FEE_MANTISSA, "Factory: fee too high");
        if(_feeMantissa > 0) require(feeRecipient != address(0), "Factory: fee recipient is zero address");
        feeMantissa = _feeMantissa;
    }
```

this is not the main root cause.

the `Factory::setFeeRecipient` function allows the operator to set the fee recipient to a zero address withought cheking if there is any positive fee mantissa value already set.

## Vulnerability Detail
If operator incorrectly set the fee recipient to a zero address with a positive fee mantissa present in pool. the accrued fees can be directed to the zero address, eventually it result into loss of fee shares.

## Impact
- fees can be lost forever

## Recommendation
add check to `Factory::setFeeRecipient` to ensure fee mantissa is zero when setting to zero address.

---

## > Borrower can skip the interest accrual by interacting with the pool each block

## Severity
**Medium**

## Summary
The interest on total debt is calculated by `Pool::getCurrentState` function everytime when user interact with the pool. the interest is calculated as follows:

```solidity
uint _interest = _totalDebt * _borrowRate * _timeDelta / (365 days * 1e18); // does the optimizer optimize this? or should it be a constant?
```

when `_totalDebt * _borrowRate * _timeDelta` < `(365 days * 1e18)`, the interest truncate to zero. However the `lastAccrueInterestTime` is updated to the current `block.timestamp` even if the `interest = 0`.

this can be exploited by the borrower by calling `Pool::removeCollateral(0)` every block. While the collateral balance remain unchanged. However calling `Pool::removeCollateral(0)` every block will be cheaper then the interest accumulated if the loan token has less decimals.

## Impact
- The interest can be bypassed by the borrower

## Code Snippet

```solidity
uint _interest = _totalDebt * _borrowRate * _timeDelta / (365 days * 1e18); // does the optimizer optimize this? or should it be a constant?
```

## Recommendation
Update `lastAccrueInterestTime` only if `interest != 0`.

---

## > Incorrect _accruedFeeShares Calculation Causes Excessive Fee Dilution

## Summary

The protocol incorrectly calculates _accruedFeeShares during interest accrual by using the pre-interest pool valuation when minting fee shares to the protocol fee recipient.

As a result, the fee recipient receives more shares than intended, causing excessive dilution of liquidity providers and transferring a larger portion of accrued interest to the protocol than configured.

## Vulnerability Details

During interest accrual in Pool.getCurrentState(), the protocol computes accrued protocol fees and mints LP shares to the fee recipient:
```solidity
uint _interest = _totalDebt * _borrowRate * _timeDelta / (365 days * 1e18);

_currentTotalDebt += _interest;

uint fee = _interest * _feeMantissa / 1e18;

_accruedFeeShares = fee * _totalSupply / _supplied;

_currentTotalSupply += _accruedFeeShares;
```

The issue is that `_accruedFeeShares` is calculated using `_supplied`, which represents the pool assets before interest accrual.

However, after interest accrual:

new pool assets = _supplied + _interest

and existing LP shares should already reflect the lender-owned portion of the newly accrued interest.

Because the protocol uses the stale pre-interest valuation, fee shares are minted at an artificially cheap price, resulting in excessive protocol ownership dilution.

## Root Cause

The protocol mints fee shares using the old share price instead of the updated post-interest share price.

Current implementation:
```solidity
_accruedFeeShares = fee * _totalSupply / _supplied;
```

implicitly assumes:
```solidity
share price = _supplied / _totalSupply
```
which is no longer true after `_interest` has been added to the pool debt.

The correct lender-owned asset base after interest accrual is:

```solidity
_supplied + _interest - fee
```

Therefore, fee shares should be minted relative to the updated share price.

## Impact

The fee recipient systematically receives more value than intended.

Consequences:

- LPs receive less interest than expected
- Protocol fee extraction exceeds configured fee rate
- Dilution compounds over time
- Protocol accounting becomes economically incorrect

This issue directly transfers value from liquidity providers to the fee recipient.

---

## > Attacker can freeze the collateral ratio

## Summary
Attacker can freeze the collateral ratio at their intended level. The pool calculates the change in collateral ratio either as follows:
```solidity
uint change = timeDelta * _maxCollateralRatioMantissa / _collateralRatioRecoveryDuration;

or 

uint change = timeDelta * _maxCollateralRatioMantissa / _collateralRatioFallDuration;
```
according to current utilization. However the attacker can interact with protocol such that very less time has passed since the last accrual. This results in `change = 0`, which means the collateral ratio will not change.

## Vulnerability Details
The collateral ratio can only changed by the `Pool::getCollateralRatioMantissa` function. where the `change` in collateral ratio is calculated as shown above. However if `timeDelta * _maxCollateralRatioMantissa` is less then the duration time, then due to solidity trucation `change` will be rounded down to `0`, leaving the collateral ratio unchanged.

attacker can interact with the protocol in very small time durations (e.g. each block). and take advantage of their intended favorable collateral ratio to supply and borrow.

## Root Cause
User interaction with the protocol in very small time durations allows the attacker to freeze the collateral ratio by taking advantage of solidity truncation.

## Impact

- The collateral ratio become frozen and do not change over a time

---

## > Liquidator can distribute user's debt to remaining borrowers partially

## Summary
The liquidator can liquidate any user by paying their debt and get their collateral in change. However user can pay the debt of user withough reducing their debt shares and walks away with their collateral. That means now liquidated user has no collateral but left with debt shares which means user has debt in the protocol. by liquidating the global debt account is decreased and the remaining debt will be distributed to the remaining borrowers. 

the liquidator can partially do this also, if they are the borrower himself.

## Vulnerability Details
In `Pool::liquidate` function the global debt and the debt share of the user is decreased and the user collateral is transfered to the liquidator as follows:
```solidity
    function liquidate(address borrower, uint amount) external {
        uint _loanTokenBalance = LOAN_TOKEN.balanceOf(address(this));
        (address _feeRecipient, uint _feeMantissa) = FACTORY.getFee();
        (  
            uint _currentTotalSupply,
            uint _accruedFeeShares,
            uint _currentCollateralRatioMantissa,
            uint _currentTotalDebt
        ) = getCurrentState(
            _loanTokenBalance,
            _feeMantissa,
            lastCollateralRatioMantissa,
            totalSupply,
            lastAccrueInterestTime,
            lastTotalDebt
        );

        uint collateralBalance = collateralBalanceOf[borrower];
        uint _debtSharesSupply = debtSharesSupply;
        uint userDebt = getDebtOf(debtSharesBalanceOf[borrower], _debtSharesSupply, _currentTotalDebt);
        uint userCollateralRatioMantissa = userDebt * 1e18 / collateralBalance;
        require(userCollateralRatioMantissa > _currentCollateralRatioMantissa, "Pool: borrower not liquidatable");

        address _borrower = borrower; // avoid stack too deep
        uint _amount = amount; // avoid stack too deep
        uint _shares;
        uint collateralReward;
        if(_amount == type(uint).max || _amount == userDebt) {
            collateralReward = collateralBalance;
            _shares = debtSharesBalanceOf[_borrower];
            _amount = userDebt;
        } else {
            uint userInvertedCollateralRatioMantissa = collateralBalance * 1e18 / userDebt;
            collateralReward = _amount * userInvertedCollateralRatioMantissa / 1e18; // rounds down
            _shares = tokenToShares(_amount, _currentTotalDebt, _debtSharesSupply, false);
        }
@>      _currentTotalDebt -= _amount;

        // commit current state
@>      debtSharesBalanceOf[_borrower] -= _shares;
@>      debtSharesSupply = _debtSharesSupply - _shares;
        collateralBalanceOf[_borrower] = collateralBalance - collateralReward;
        totalSupply = _currentTotalSupply;
@>      lastTotalDebt = _currentTotalDebt;
        lastAccrueInterestTime = block.timestamp;
        lastCollateralRatioMantissa = _currentCollateralRatioMantissa;
        emit Liquidate(_borrower, _amount, collateralReward);
        if(_accruedFeeShares > 0) {
            address __feeRecipient = _feeRecipient; // avoid stack too deep
            balanceOf[__feeRecipient] += _accruedFeeShares;
            emit Transfer(address(0), __feeRecipient, _accruedFeeShares);
        }

        // interactions
        safeTransferFrom(LOAN_TOKEN, msg.sender, address(this), _amount);
@>      safeTransfer(COLLATERAL_TOKEN, msg.sender, collateralReward);
    }
```

If liquidator pay very little amount of loan token for a user, such that the calculated `share = 0`. That means by paying small amount of debt of user the global `lastTotalDebt` is reduced by the `amount` (even if 1 wei), but the share accounting will not be changed since `share = 0` for the user and at the end the user's collateral has been sent to the liquidator.

That means liquidator has just get the user's collateral by paying very little amount of loan token repitatively such that global debt reduced but the user is holding the debt yet. in this way liquidator can distribute their debt (if any) to the remaining borrowers and walks away wil collateral

## Root Cause
The `Pool::liquidate` function allows very tiny amount of loan token to be paid result in share = 0.

## Impact
- liquidator can distribute their debt to the other borrower
- the liquidated borrower will lose their collateral but holding protocol debt yet

---