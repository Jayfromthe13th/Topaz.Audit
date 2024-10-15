# **High Severity: Borrow Index Manipulation via Manual Snapshots**

## **Description**
The `DepositorVault` contract allows **any user to trigger snapshots manually** using the `takeSnapshot()` function. These snapshots store critical data such as the **borrow index**, total assets, and liabilities, which are used by the `Market` contract to **calculate interest rates and liabilities**. This design introduces a **vulnerability** where users can manipulate the **borrow index** to reduce their interest payments by strategically altering market utilization.

---

## **Vulnerability Impact**
- **Manipulated Borrow Index**: By **temporarily increasing deposits**, users can lower the **utilization rate**, leading to a **reduced borrowing rate** and slower growth in the borrow index.
- **Reduced Interest Payments**: If a **snapshot is taken** after a large deposit, the **borrow index remains artificially low**. This allows borrowers to **repay loans at lower interest rates**.
- **Exploits Interest Rate Calculations**: Since the **DynamicInterestRateModel** relies on snapshots, **manually timed snapshots** give users the ability to **skew the effective interest rate**, impacting protocol earnings.

---

## **Exploit Scenario**
1. **Deposit a large amount** into the vault, increasing total assets and lowering utilization.
2. **Trigger a snapshot** using `takeSnapshot()`, locking in the **reduced borrow index**.
3. **Repay the loan** at the **lower interest rate**, benefiting from the manipulated state.
4. **Withdraw the deposit**, restoring the vault’s state but **escaping with reduced interest payments**.

---

## **Affected Code**
### 1. **Manual Snapshot Creation (`DepositorVault.sol`)**
```solidity
function takeSnapshot() public {
    IDepositorVault.Snapshot memory snapshot = IDepositorVault.Snapshot({
        timestamp: block.timestamp.toUint32(),
        totalLiabilities: getTotalLiabilities().toUint128(),
        totalAssets: getTotalAssets(),
        totalShares: getTotalShares()
    });
    _observer.write(snapshot);
}

```

## Mitigation Recommendations
1. Restrict Snapshot Access: Only allow authorized users to trigger snapshots or automate snapshots at regular intervals.
2. Use Multiple Snapshots for Averaging: Calculate the interest rate using a moving average of several snapshots to reduce short-term manipulation.
3. Disable Snapshots When Paused: Add the whenNotPaused modifier to the takeSnapshot() function.
4. Monitor Large Temporary Deposits: Implement penalties or fees for deposits withdrawn soon after snapshots to discourage manipulation.
