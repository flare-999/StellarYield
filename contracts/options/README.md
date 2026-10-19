# Options Contract

## Purpose

`options` supports minting, exercising, and expiring collateralized call/put options with on-chain settlement.

## Public API

- `initialize(admin, oracle)`
- `mint(minter, option_type, underlying_asset, quote_asset, strike_price, opening_time, expiration_time, collateral_amount)`
- `exercise(exerciser, option_id)` exercises all remaining collateral
- `exercise_partial(exerciser, option_id, amount)` exercises part of the collateral
- `expire(option_id)`
- `deposit(owner, asset, amount)` / `withdraw(owner, asset, amount)`
- `get_free_collateral(owner, asset)` / `get_reserved_collateral(owner, asset)`
- `get_premium(spot, strike, time_to_expiry_years, iv)`

## Events

- `mint`
- `exercise` (position fully exercised)
- `partial` (part of the collateral exercised)
- `expire`
- `deposit`
- `withdraw`

## Errors

`OptionsError` covers initialization, authorization, invalid option state, expiry windows, and amount/price validation.

## Collateral Accounting

Collateral held by the contract is tracked per `(owner, asset)` in two buckets:

- **Free**: deposited via `deposit`, withdrawable at any time with `withdraw`.
- **Reserved**: locked behind open option positions. It cannot be withdrawn.

`mint` backs the position with the minter's free collateral first and pulls only
the shortfall from their wallet (so minting without a deposit works as before).
The full `collateral_amount` is then reserved. The reserve is released as the
collateral leaves the contract: by `exercise` / `exercise_partial` (the exercised
amount) and by `expire` (whatever was not exercised). `withdraw` can only take
free collateral, so it fails with `InsufficientFreeCollateral` rather than
undercollateralize an open position.

Positions minted before reserve tracking existed have no recorded reserve;
releasing their collateral saturates at zero. Their collateral was never in the
free bucket, so it was never withdrawable.

## Storage Model

- Instance: admin, oracle, option counter.
- Persistent: option records by id, free and reserved collateral per `(owner, asset)`, and the amount already paid out by partial exercises per option.

## Local Test

```bash
cd contracts
cargo test -p options
```

## Integration Notes

- Uses SAC token transfers for collateral and settlement legs.
- Oracle address is stored for pricing extensions.
