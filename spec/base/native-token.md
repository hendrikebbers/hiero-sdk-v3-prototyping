# Native Token API

## Description



## API Schema — Abstraction

```
namespace nativeToken

// A unit of a native token, either the smallest indivisible unit or a named multiple of it.
abstraction NativeTokenUnit {
    @@immutable symbol: string         // display symbol of the unit (e.g. "ℏ")
    @@immutable baseUnitFactor: int64  // number of base (smallest) units contained in one of this unit
}

// An amount of a network's native token, expressed in a given unit.
// $$Self is the concrete token type (e.g. Hbar), so `to(...)` returns that exact type rather than the abstraction.
abstraction NativeToken<$$Self extends NativeToken<$$Self, $$Unit>, $$Unit extends NativeTokenUnit> {
    @@immutable amount: int64          // amount expressed in `unit`
    @@immutable unit: $$Unit  // the unit `amount` is expressed in

    // Convert this amount to a different unit of the same token
    $$Self to(targetUnit: $$Unit)

    // Total amount expressed in base (smallest) units
    int64 toBaseUnits()
}

// Represents the exchange rate of a native token in USD cents.
abstraction ExchangeRate {
    @@immutable expirationTime: zonedDateTime    // expiration time of the exchange rate
    @@immutable exchangeRateInUsdCents: double   // exchange rate of the native token in USD cents

    // Check if this exchange rate has expired
    // returns true if current time is past expirationTime
    bool isExpired()
}
```

## Questions & Comments

- [@hendrikebbers](https://github.com/hendrikebbers): Should `ExchangeRate.exchangeRateInUsdCents` really be `double`
  instead of `decimal`?