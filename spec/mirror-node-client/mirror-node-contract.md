# Mirror Node Contract Query API

## Description

## API Schema

```
namespace mirrornode.contract

requires ledger, keys

@@finalType
Contract {
    @@immutable contractId: ledger.Address
    @@immutable @@nullable adminKey: keys.PublicKey
    @@immutable @@nullable autoRenewAccount: ledger.Address
    @@immutable autoRenewPeriod: int32
    @@immutable createdTimestamp: zonedDateTime
    @@immutable deleted: bool
    @@immutable @@nullable expirationTimestamp: zonedDateTime
    @@immutable @@nullable fileId: string
    @@immutable @@nullable evmAddress: string
    @@immutable @@nullable memo: string
    @@immutable @@nullable maxAutomaticTokenAssociations: int32
    @@immutable @@nullable nonce: int64
    @@immutable @@nullable obtainerId: string
    @@immutable permanentRemoval: bool
    @@immutable @@nullable proxyAccountId: string
    @@immutable fromTimestamp: zonedDateTime
    @@immutable toTimestamp: zonedDateTime
    @@immutable @@nullable bytecode: string
    @@immutable @@nullable runtimeBytecode: string
}

ContractRepository {
    @@async @@throws(mirror-node-error)
    Page<Contract> findAll()

    @@async @@throws(mirror-node-error)
    @@nullable Contract findById(contractId: ledger.Address)
}

```