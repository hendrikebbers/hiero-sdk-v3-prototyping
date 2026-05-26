# Hedera API

## Description

## API Schema — Abstraction

```
namespace hedera
requires {NetworkSetting} from ledger.config
requires {NativeToken, NativeTokenUnit} from nativeToken

constant HEDERA_MAINNET_IDENTIFIER:string = "hedera-mainnet" // identifier for the Hedera mainnet
constant HEDERA_TESTNET_IDENTIFIER:string = "hedera-testnet" // identifier for the Hedera testnet

HederaNetworkSetting extends NetworkSetting {
}

// Definition of the different units of HBAR, the native token of the Hedera network.
// `symbol` and `baseUnitFactor` are inherited from NativeTokenUnit; each constant provides its own
// values (e.g. HBAR -> symbol "ℏ", baseUnitFactor 100_000_000; TINYBAR -> symbol "tℏ", baseUnitFactor 1).
enum HbarUnit extends NativeTokenUnit {
    TINYBAR  // tℏ
    MICROBAR // μℏ
    MILLIBAR // mℏ
    HBAR     // ℏ
    KILOBAR  // kℏ
    MEGABAR  // Mℏ
    GIGABAR  // Gℏ
}

// HBAR, the native token of the Hedera network. `amount`, `unit`, and `to(...)` are inherited from
// NativeToken.
Hbar extends NativeToken<Hbar, HbarUnit> {

    // Total amount in tinybars (the base unit of HBAR); equivalent to toBaseUnits().
    int64 toTinybars()
}

```

## Questions & Comments
