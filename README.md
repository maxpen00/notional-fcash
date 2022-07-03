# Notional Wrapped fCash

Wrapped fCash (wfCash) is a compatibility layer for developers who want to integrate with Notional lending but unable to use fCash's native ERC1155 specification. wfCash is compatible with ERC20 and ERC4626.

Technical Walkthrough: https://www.youtube.com/watch?v=RvCYFR2Yjls

Audits: TBD

## Deployment

Each wfCash contract is deployed from the `WrappedfCashFactory` using a `CREATE2` opcode and parameterized by a currency id and a maturity. Currency IDs are autoincrementing IDs generated by the Notional contract and correspond to a currency pair (i.e. USDC/cUSDC, DAI/cDAI, etc) that can be lent and borrowed on Notional.

The maturity the unix timestamp of the block time when the fCash will mature (and convert to it accruing money market interest).

One canonical wfCash contract can be deployed permissionlessly for every currencyId and maturity combination through the WrappedfCashFactory. Each wfCash contract is a OpenZeppelin `BeaconProxy` that refers to a single `BeaconImplementation` of the `wfCashERC4626` contract.

**ERC4626**: the ERC4626 standard stipulates a single asset token for the contract. In this case we use the underlying token (i.e. DAI, USDC) rather than the money market token (i.e. cDAI, cUSDC). This choice was made in case the money market token changes in the future, the underlying token will not change in the future. As a consequence, ERC4626 interactions will be less gas efficient.

## Lending

Lending fCash can be done by calling `mintViaAsset`, `mintViaUnderlying`, `mint` or `deposit`. ERC4626 is provided as a compatibility layer but due to the computation required it is not the most gas efficient implementation. Using the non-ERC4626 `mintViaAsset` or `mintViaUnderlying` function is significantly more gas efficient. (See table below).

Creating wfCash tokens can also be done by using an ERC1155 `safeTransferFrom` to the corresponding wfCash contract from the main Notional contract. This allows accounts to seamlessly enter and exit wfCash positions from Notional ERC1155.

**Caveat**: Notional ERC1155 fCash can be used as collateral for borrowing. However, wfCash cannot be used in this way since it is held in escrow by the wfCash contract. This should not be a limitation for lend-only accounts.

## Redemption

wfCash can be redeemed anytime before or after maturity. Redeeming fCash prior to maturity means selling it on Notional for cash (i.e. underlying or money market tokens). Selling fCash is subject to prevailing market interest rates and therefore does not guarantee that the holder will receive their promised fixed interest.

fCash (and wfCash is no different) settles to money market tokens (i.e. cTokens or aTokens) at maturity and will accrue the variable money market interest rate from that point forward. Accounts that redeem their wfCash after maturity will receive their fixed interest as well as any accrued variable rate interest on top.

Before or after maturity, redeeming wfCash can be done by calling `redeem`, `redeemToUnderlying`, `redeemToAsset`, or `withdraw`. Accounts are also able to call `redeem` and specify the `transferfCash` boolean in `RedeemOpts` to have the wfCash transfer the ERC1155 fCash token to their Notional account.

## Gas Costs

Gas costs are estimated in `scripts/gas_costs.py` and the current output can be seen in `gas_costs.json`. ERC4626 adds significant gas costs and also does not allow the user to specify slippage thresholds, however, it does provide a very simple access point into wrapped fCash.

Using asset tokens is the most gas efficient and capital efficient way to access wrapped fCash. Using ERC1155 transfer and batchLend calls saves 1 or 2 ERC20 token transfers and therefore will save about 100k gas (about 20%).

## Code Stats

| Module    | File                    | Code | Comments | Total Lines | Complexity / Line |
| :-------- | :---------------------- | ---: | -------: | ----------: | ----------------: |
| Contracts | wfCashBase.sol          |   96 |       30 |         150 |              12.5 |
| Contracts | wfCashERC4626.sol       |  180 |       33 |         247 |               8.9 |
| Contracts | wfCashLogic.sol         |  214 |       69 |         319 |               8.4 |
| Lib       | Constants.sol           |   26 |       11 |          46 |               0.0 |
| Lib       | DateTime.sol            |   93 |       28 |         140 |              34.4 |
| Lib       | EncodeDecode.sol        |   92 |        3 |         103 |               0.0 |
| Lib       | Types.sol               |   91 |       69 |         172 |               0.0 |
| Proxy     | WrappedfCashFactory.sol |   28 |        5 |          42 |               7.1 |
| Proxy     | nBeaconProxy.sol        |    7 |        2 |          12 |               0.0 |
| Proxy     | nUpgradeableBeacon.sol  |    5 |        3 |          10 |               0.0 |
