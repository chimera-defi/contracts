# StakeWise Smart Contracts Security Audit Report

## Executive Summary

This report documents a comprehensive security audit of the StakeWise smart contracts codebase. The audit was conducted to identify potential vulnerabilities that could be exploited, particularly in the context of potential malicious modifications.

**Overall Risk Level: MEDIUM-HIGH**

Several critical vulnerabilities and potential issues were identified that require immediate attention.

---

## Critical Vulnerabilities

### 1. **CRITICAL: Division by Zero in RewardEthToken.updateTotalRewards()**

**Location:** `contracts/tokens/RewardEthToken.sol:221`

**Issue:**
```solidity
uint256 newRewardPerToken = prevRewardPerToken.add(
    periodRewards.sub(maintainerReward).mul(1e18).div(stakedEthToken.totalDeposits())
);
```

**Vulnerability:** If `stakedEthToken.totalDeposits()` returns 0 (e.g., all users have withdrawn), this will cause a division by zero error, DoS'ing the reward update mechanism.

**Impact:** 
- HIGH: Oracle reward updates will fail, preventing all reward distributions
- Could freeze the protocol if all staked tokens are withdrawn

**Recommendation:**
```solidity
uint256 totalDeposits = stakedEthToken.totalDeposits();
require(totalDeposits > 0, "RewardEthToken: no deposits");
uint256 newRewardPerToken = prevRewardPerToken.add(
    periodRewards.sub(maintainerReward).mul(1e18).div(totalDeposits)
);
```

---

### 2. **HIGH: Potential Underflow in Pool.setActivatedValidators()**

**Location:** `contracts/collectors/Pool.sol:114`

**Issue:**
```solidity
function setActivatedValidators(uint256 newActivatedValidators) external override {
    require(msg.sender == oracles || hasRole(DEFAULT_ADMIN_ROLE, msg.sender), "Pool: access denied");
    
    // subtract activated validators from pending validators
    pendingValidators = pendingValidators.sub(newActivatedValidators.sub(activatedValidators));
    activatedValidators = newActivatedValidators;
    emit ActivatedValidatorsUpdated(newActivatedValidators, msg.sender);
}
```

**Vulnerability:** 
- If `newActivatedValidators < activatedValidators`: First subtraction will revert (SafeMath), but this is acceptable.
- If `newActivatedValidators > activatedValidators + pendingValidators`: Second subtraction will revert, causing DoS.
- No validation that `newActivatedValidators` is reasonable

**Impact:**
- HIGH: Malicious oracle or admin could set invalid validator counts, causing DoS
- Could prevent validator activation processing

**Recommendation:**
```solidity
uint256 delta = newActivatedValidators > activatedValidators 
    ? newActivatedValidators.sub(activatedValidators)
    : activatedValidators.sub(newActivatedValidators);
    
if (newActivatedValidators > activatedValidators) {
    require(delta <= pendingValidators, "Pool: invalid validator count");
    pendingValidators = pendingValidators.sub(delta);
} else {
    // Handle decrease case if needed
    pendingValidators = pendingValidators.add(delta);
}
```

---

### 3. **MEDIUM-HIGH: Missing Reentrancy Protection in Pool Contract**

**Location:** `contracts/collectors/Pool.sol`

**Issue:** The `Pool` contract does not use `ReentrancyGuard` for critical functions:
- `addDeposit()` - Calls external `stakedEthToken.mint()`
- `activate()` - Calls external `stakedEthToken.mint()`
- `activateMultiple()` - Calls external `stakedEthToken.mint()`

**Vulnerability:** While `stakedEthToken.mint()` likely doesn't have external callbacks, if StakedEthToken is upgraded maliciously or has hooks, reentrancy attacks are possible.

**Impact:**
- MEDIUM: Low immediate risk if StakedEthToken is trusted, but dangerous if contract is upgraded maliciously
- Could allow double-spending or manipulation of activation logic

**Recommendation:** Add `ReentrancyGuard` protection to `addDeposit()`, `activate()`, and `activateMultiple()`, or ensure StakedEthToken is immutable/trusted.

---

### 4. **MEDIUM: Integer Overflow Risk in Pool.addDeposit()**

**Location:** `contracts/collectors/Pool.sol:132-135`

**Issue:**
```solidity
uint256 _pendingValidators = pendingValidators.add((address(this).balance).div(VALIDATOR_DEPOSIT));
uint256 _activatedValidators = activatedValidators; // gas savings
uint256 validatorIndex = _activatedValidators.add(_pendingValidators);
```

**Vulnerability:** While SafeMath protects against overflow, extremely large values could cause:
- Gas exhaustion
- Incorrect validator index calculation
- Potential DoS if values become too large

**Impact:**
- MEDIUM: Could cause incorrect activation scheduling if contract balance is manipulated

**Recommendation:** Add reasonable bounds checking for validator counts.

---

### 5. **MEDIUM: Oracle Consensus Logic Vulnerability**

**Location:** `contracts/Oracles.sol:150`

**Issue:**
```solidity
if (candidateNewVotes.mul(3) > oraclesCount.mul(2)) {
    // Execute update
}
```

**Vulnerability:** The check `candidateNewVotes.mul(3) > oraclesCount.mul(2)` means:
- With 3 oracles: Need 3 votes (100%)
- With 6 oracles: Need 5 votes (83.3%)
- With 9 oracles: Need 7 votes (77.7%)

**Impact:**
- MEDIUM: Less than 2/3 majority can execute updates if oracle count is low
- Could allow collusion among minority of oracles

**Recommendation:** Ensure oracle count is always sufficiently high, or use stricter threshold (e.g., require exactly 2/3).

---

### 6. **MEDIUM: Missing Access Control on addAdmin/removeAdmin**

**Location:** `contracts/presets/OwnablePausable.sol:52-61`

**Issue:**
```solidity
function addAdmin(address _account) external override {
    grantRole(DEFAULT_ADMIN_ROLE, _account);
}

function removeAdmin(address _account) external override {
    revokeRole(DEFAULT_ADMIN_ROLE, _account);
}
```

**Vulnerability:** Any admin can add/remove other admins without restrictions. This is standard OpenZeppelin behavior but could be exploited if a single admin account is compromised.

**Impact:**
- MEDIUM: Single point of failure - if one admin is compromised, entire protocol can be controlled
- No multi-sig or timelock protections visible

**Recommendation:** 
- Implement multi-sig for admin operations
- Add timelock for critical role changes
- Consider using OpenZeppelin's `TimelockController`

---

### 7. **MEDIUM: Potential Front-Running in Pool.activate()**

**Location:** `contracts/collectors/Pool.sol:154-163`

**Issue:** The `activate()` function doesn't validate that `msg.sender` is authorized to activate for `_account`. Anyone can call `activate()` for any account.

**Vulnerability:** 
- Front-running: Attacker could monitor mempool for activation transactions and front-run them
- While this doesn't steal funds, it could disrupt user experience

**Impact:**
- LOW-MEDIUM: Minor UX issue, but could be annoying

**Recommendation:** Add `require(_account == msg.sender)` or remove the `_account` parameter if not needed.

---

### 8. **MEDIUM: Unchecked External Call in PoolEscrow.withdraw()**

**Location:** `contracts/collectors/PoolEscrow.sol:68`

**Issue:**
```solidity
function withdraw(address payable payee, uint256 amount) external override onlyOwner {
    require(payee != address(0), "PoolEscrow: payee is the zero address");
    emit Withdrawn(msg.sender, payee, amount);
    payee.sendValue(amount);  // Uses Address.sendValue() - returns bool
}
```

**Vulnerability:** `sendValue()` returns a boolean but the return value is not checked. If the send fails, the function will still succeed silently.

**Impact:**
- MEDIUM: Failed withdrawals will not revert, misleading owner
- Could cause incorrect accounting

**Recommendation:**
```solidity
require(payee.sendValue(amount), "PoolEscrow: transfer failed");
```

---

### 9. **LOW-MEDIUM: Missing Validation in Solos.cancelDeposit()**

**Location:** `contracts/collectors/Solos.sol:97-121`

**Issue:** The function checks that `newAmount.mod(VALIDATOR_DEPOSIT) == 0` but doesn't prevent partial cancellation that could leave invalid states.

**Impact:**
- LOW-MEDIUM: Could allow users to cancel deposits in ways that leave invalid validator states

**Note:** This is protected by `nonReentrant`, which is good.

---

### 10. **LOW: Potential DoS in MerkleDistributor.claim()**

**Location:** `contracts/merkles/MerkleDistributor.sol:110-145`

**Issue:** The `claim()` function loops through arrays without gas limit consideration:
```solidity
for (uint256 i = 0; i < tokensCount; i++) {
    // Process each token
}
```

**Vulnerability:** If arrays are very large, could cause out-of-gas errors.

**Impact:**
- LOW: Unlikely to be exploited unless malicious merkle root is set

**Recommendation:** Add reasonable bounds on array sizes.

---

## Access Control Issues

### Centralized Control Risks

1. **Admin Role:** Single admin can pause contracts, modify critical parameters
2. **Oracle Role:** Oracles have significant control over reward distribution
3. **Operator Role:** Can register validators (but validated by Validators contract)

**Recommendation:** Implement multi-sig and timelock for all admin operations.

---

## Upgradeability Risks

### Storage Layout Compatibility

All upgradeable contracts extend `OwnablePausableUpgradeable`. Any storage layout changes in upgrades could corrupt state.

**Recommendation:** 
- Maintain strict storage layout compatibility
- Use storage gaps for future extensibility
- Conduct thorough testing before upgrades

---

## Test Coverage Analysis

### Areas with Good Coverage
- Pool deposit and activation flows
- Validator registration
- Token transfers

### Areas Needing More Coverage
- Edge cases in `setActivatedValidators()` (negative delta, underflow scenarios)
- Division by zero in `updateTotalRewards()` when totalDeposits = 0
- Reentrancy attack scenarios
- Front-running scenarios

---

## Recommendations Summary

### Immediate Actions Required

1. **Fix division by zero** in `RewardEthToken.updateTotalRewards()`
2. **Add validation** to `Pool.setActivatedValidators()` to prevent underflow
3. **Add reentrancy protection** to Pool critical functions (or verify StakedEthToken is safe)
4. **Fix `sendValue()` return value check** in PoolEscrow

### High Priority

5. **Implement multi-sig** for admin operations
6. **Add timelock** for critical parameter changes
7. **Improve oracle consensus** threshold validation
8. **Add access control** to `activate()` function

### Medium Priority

9. **Add bounds checking** for validator counts
10. **Improve test coverage** for edge cases
11. **Document upgrade procedures** and storage layout requirements

---

## Conclusion

The codebase shows good use of OpenZeppelin's battle-tested libraries and SafeMath protections. However, several critical vulnerabilities were identified that could lead to DoS attacks or protocol freeze. The most critical issues are:

1. Division by zero in reward updates
2. Underflow risk in validator activation
3. Missing reentrancy protections

These should be addressed before deployment or upgrading contracts. Additionally, the centralized admin control suggests a need for multi-sig and timelock implementations to reduce single points of failure.

**Overall Assessment:** The contracts are generally well-structured but contain several vulnerabilities that could be exploited. With the recommended fixes, the security posture would improve significantly.

---

## Appendix: Code Review Checklist

- [x] Reentrancy vulnerabilities
- [x] Integer overflow/underflow
- [x] Access control issues
- [x] Division by zero
- [x] Unchecked external calls
- [x] Front-running vulnerabilities
- [x] Upgradeability risks
- [x] Oracle dependency risks
- [x] Gas optimization issues
- [x] Test coverage gaps
