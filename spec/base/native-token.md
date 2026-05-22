# Native Token API


## Description


## API Schema

```
namespace native-token

// Definition of different units of Hbar
enum HbarUnit {
    TINYBAR  // tℏ
    MICROBAR // μℏ
    MILLIBAR // mℏ
    HBAR     // ℏ
    KILOBAR  // kℏ
    MEGABAR  // Mℏ
    GIGABAR  // Gℏ
    
    @@immutable symbol: string // symbol of the unit
    @@immutable tinybars: int64  // number of tinybars in one unit
    
    static list<HbarUnit> values()  // returns all HbarUnit values
}

// Hbar is a wrapper around int64 that represents a amount of Hbar based on a given unit.
Hbar {
    @@immutable amount: int64 // amount in the given unit
    @@immutable unit: HbarUnit // unit of the amount
    
    // Convert this Hbar to a different unit
    Hbar to(targetUnit: HbarUnit)
    
    // Get total amount in tinybars
    int64 toTinybars()
}

// Represents the exchange rate of Hbar in USD cents.
HBarExchangeRate {
    @@immutable expirationTime: zonedDateTime // expiration time of the exchange rate
    @@immutable exchangeRateInUsdCents: double // exchange rate of HBar in USD cents
    
    // Check if this exchange rate has expired
    // returns true if current time is past expirationTime
    bool isExpired()
}

```

## Questions & Comments

- [@hendrikebbers](https://github.com/hendrikebbers): Do we want an abstraction for currency? HBAR is the only one for
  now and can be seen as Hedera specific.
