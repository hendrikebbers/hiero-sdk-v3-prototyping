# Mirror Node Contract Query API

## Description

## API Schema

```
namespace mirrornode.contract

requires {Address, MirrorNode} from ledger
requires {PublicKey} from keys

@@finalType
Contract {
    @@immutable contractId: Address
    @@immutable @@nullable adminKey: PublicKey
    @@immutable @@nullable autoRenewAccount: Address
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
    @@nullable Contract findById(contractId: Address)
}

@static createRepository(mirrorNode: MirrorNode)

```