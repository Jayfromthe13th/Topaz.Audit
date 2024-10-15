# Topaz.Audit

# TopazFinance

## What is Topaz Finance?

Topaz Finance is a borrow/lending protocol which allows user's to borrow USDT against a basket of collateral. User's can also experience great yield on their USDT if they choose to deposit it into the depositorVault and earn interest from user's borrowing it.

Planned deployment network: ETH-Mainnet.

## What are the user flows?

![System Diagram](./TopazDiagram.png)

Above is a diagram of the entire Topaz Finance system.

Depicted are the following user flows:

### Deposit USDT to be lent out

Users can deposit their USDT and earn a yield on it from the protocol lending it out by calling the `deposit` function on the `depositorVault` directly.

### Withdraw deposited USDT

Users can withdraw their deposited USDT from the depositorVault with the `withdraw` or `redeem` function.

### Deposit supported assets as collateral

Users can deposit supported assets as collateral using the `supply` function on the `Portfolio` contract. Users can then borrow USDT against this collateral amount.

When users deposit assets using the Portfolio contract, the Portfolio sends these assets to dedicated custodian contracts. There is one custodian per supported asset, users are then issued "shares" of the custodian's total holdings. There are two tyeps of custodians, non-yield-bearing and yield-bearing.

Yield-bearing custodians will earn yield over time, and thus user's custodian shares will earn yield. Non-yield-bearing custodians will not earn yield, and thus user's custodian shares will remain at a 1:1 exchange rate with the underlying asset.

When user's deposit collateral the `beforeSupply` hook will be called if there is a configured callback for the asset being supplied. The only use-case for this hook at the moment is the Loyalty points, if user's are depositing TPAZ as collateral. There is more information on the Loyalty system below.

Supported collateral tokens:

- WETH
- USDC
- VaultToken

Note: The vault token that user's receive for depositing into the `depositorVault` is a valid collateral token. This can yield some interesting edge cases.

### Withdraw collateral

Users can withdraw their collateral using the `withdraw` function on the `Portfolio` contract. Users may not withdraw collateral that is necessary to keep their position healthy.

### Borrow USDT

Users can borrow USDT from the system using the `borrow` function on the `Market` contract. Users may not borrow more USDT than their collateral allows, factoring in the collateral's LTV ratio.

The `Market` contract then uses the `borrow` function on the `depositorVault` to borrow the USDT from depositors. The `depositorVault` tracks the debt on a `Market` basis (different markets may have different interest rate models for different use-cases in the future), and the `Market` tracks the debt on a user basis.

### Repay USDT

User's can repay their USDT debt with the `repay` function on the `Market` contract. When user's repay their debt they will be charged the accumulated interest amount.

The `Market` contract then uses the `repay` function on the `depositorVault` to repay the USDT to depositors.

## What is the liquidation flow?

The protocol is expected to keep a liquidation fund of USDT in the `Liquidator` contract that can be used to repay debts of accounts that are liquidated. Funds from the liquidation of user's collateral go towards the treasury, repaying the liquidation fund, and a small reward for the caller of the `liquidate` function.

## Snapshots

Snapshots of the vault USDT deposit balance are recorded every 5 minutes so that the dynamic interest rate for the amount of interest charged to a user cannot be manipulated by depositing or withdrawing from the vault in a single transaction.

Snapshots are recorded with the Observer, the implementation of which can be seen in the `src/protocol/vault/SnapshotLib.sol` file.

## Upgradeability Through Migrations

For legal reasons, the Topaz Finance team will not use directly upgradeable contracts. Instead, the codebase employs an "Eternal Storage" pattern, where there are dedicated "Storage" contracts which hold the relevant storage for each facet of the protocol.

If an implementation for a contract that holds funds should need to be upgraded, a migration can take place with one of the Migrator contracts. These are the `CustodianMigrator` and `DepositorVaultMigrator` contracts.

The Migrator contracts will sweep funds to the treasury and revoke access roles from the old implementation contract before sending funds and granting access to the new implementation.

## Scope of the security review

The following files are in scope for the security review:

```
src/protocol
├── core
│   ├── AddressRegistry.sol
│   └── AssetStorage.sol
├── governance
│   ├── Executable.sol
│   ├── Governor.sol
│   └── Treasury.sol
├── lens
│   ├── AccountLens.sol
│   └── ProtocolLens.sol
├── loyalty
│   ├── BoostModule.sol
│   ├── FirstLoanBoostModule.sol
│   ├── Loyalty.sol
│   ├── LoyaltyHooks.sol
│   ├── LoyaltyLib.sol
│   └── LoyaltyStorage.sol
├── market
│   ├── DiscountModel.sol
│   ├── DynamicInterestRateModel.sol
│   ├── FixedInterestRateModel.sol
│   ├── Liquidator.sol
│   ├── Market.sol
│   ├── MarketLiquidation.sol
│   └── MarketStorage.sol
├── marketplace
│   ├── DepositorVaultMarketplaceAdapter.sol
│   └── SpotMarketMarketplaceAdapter.sol
├── oracle
│   ├── ChainlinkAggregatorPriceOracle.sol
│   ├── DepositorVaultTokenPriceOracle.sol
│   └── FallbackPriceOracle.sol
├── portfolio
│   ├── BaseCustodian.sol
│   ├── Custodian.sol
│   ├── CustodianMigrator.sol
│   ├── Portfolio.sol
│   ├── PortfolioStorage.sol
│   └── YieldBearingCustodian.sol
├── security
│   ├── AccessControlList.sol
│   ├── AdminAccessControl.sol
│   └── AuthorizedAccessControl.sol
├── utils
│   ├── Migratable.sol
│   ├── Pausable.sol
│   └── Sweepable.sol
└── vault
    ├── DepositorVault.sol
    ├── DepositorVaultMigrator.sol
    ├── DepositorVaultStorage.sol
    ├── DepositorVaultToken.sol
    ├── LinearDistributedYieldVault.sol
    ├── SnapshotLib.sol
    ├── Vault.sol
    └── YieldVault.sol
```

## Loyalty Points

In order to drive value capture back to the TPAZ token, there needs to be desirable utility for token holders.
The initial utility for the TPAZ token will enable:

Governance voting power
Protocol revenue sharing
Discounted borrowing rates

In order to reduce the liquidity of a token, a common practice is DeFi is to implement a veToken model which then increases utility power of a token by encouraging users to lock the token away for an extended period of time. This is an effective model, but provides a cumberson UX and is an unfamiliar concept for retail users.

There is an alternative solution that can achieve a similar outcome but with less friction and that is through the use of Loyalty Points. Loyalty Points are what drive the users utility power and can be earnt the longer you hold then token, effecitvely acting as a retrospective form of the veToken model.

Loyalty Points can have several advantages:

Most people are familiar with loyalty points as they are a scheme that is offered by most businesses to encourage consumers to remain in their ecosystem.

Loyalty points would encourage users to hold the token without the cumberson UX of a veToken model.

Loyalty points can be boosted through other means within the protocol, for example, taking out a loan can earn loyalty points.

Loyalty point vesting can be used to limit MEV seekers from front running revenue sharing in the protocol.

### Staking

In lieu of the traditional staking of tokens, Topaz has a more natural mechanism which is for users to add TPAZ tokens to their portfolio. By adding TPAZ tokens to their portfolio, the portfolio value will increase and decrease in line with the price of the token, however, the registered TPAZ asset will have a 0% loan-to-value ratio which means that it cant be used to increase a users borrowing limit, effectively, we dont allow borrowing against the TPAZ token.

### Vesting

Loyalty points are tied directly to TPAZ tokens held in a users portfolio whereby 1 TPAZ token will be worth 10 loyalty points, or 0.1 TPAZ will equal 1 point.

To encourage stickiness and avoid situations where whale manipulation could take place, loyalty points are vested over an epoch which is defined as Sunday to Sunday. TPAZ tokens must held for a complete epoch before they become points.

Internally, the points are tracked in four states, pending, vesting, accrued, and claimed. When tokens are first added they are considered pending. When the next epoch occurs, those pending points are considered vesting. In the subsequent epoch, those vesting points are now considered accrued.

When the user withdraws their TPAZ tokens, the equivilant amount of loyalty points are reduced, starting from pending, and moving through to claimed points.

### Claiming

Once points have accrued through the epoch they will be available to be claimed by the account. The claiming process does add some additional friction to the process, but it is necessary to ensure that relevant state is updated accordingly.

### Boosting

Actions within the protocol can be enhanced by providing a boost to a users loyalty points. A boost is a range between 1 and 5 whereby 5 would mean an accounts claimed points are effecitvely multiplied by 5.

The first boost that will be available will allow the user to claim a 1.1x boost on their points if they take out a loan. A specific boost will only be allowed to be claimed once per account.

### Burn to Boost

Points will only exist for an account whilst they are held in the accounts portfolio. When they are withdrawn, the equivelant points will be removed.

As an incentive to users to burn their own TPAZ tokens (or a portion of), they can opt to burn their TPAZ tokens in exchange for a lifetime 5x boost on the amount of points burnt. Given those tokens can no longer be withdrawn, those points will always exist for the account.

### Revenue Sharing and Rewards

A portion of protocol revenue (in the form of USDT) generated from borrowing fees, and liquidations (and other methods in the future) will be provided to points holders via the Loyalty contract. Protocol revenue will be donated on periodic basis, likely, per epoch assuming a minimum amount has been reached.

Additionally, other rewards can also be donated to points holders in the same manner. This can be useful if the protocol is deployed to an ecosystem where there is an ecosystem grant available.
