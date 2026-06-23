# Mirror Node Contract Query API

## Description

## API Schema

```
namespace mirrornode.contract

requires {AccountId, ContractId, EvmAddress, MirrorNode} from ledger
requires {Authority} from authority
requires {Page} from common

@@finalType
Contract {
    @@immutable contractId: ContractId
    @@immutable @@nullable adminAuthority: Authority
    @@immutable @@nullable autoRenewAccount: AccountId
    @@immutable autoRenewPeriod: seconds
    @@immutable createdTimestamp: zonedDateTime
    @@immutable deleted: bool
    @@immutable @@nullable expirationTimestamp: zonedDateTime
    @@immutable @@nullable fileId: string                              // TODO: should be typed `Address` once the string-typed entity-id fields in this file are cleaned up (separate work)
    @@immutable @@nullable evmAddress: EvmAddress
    @@immutable @@nullable memo: string
    @@immutable @@nullable maxAutomaticTokenAssociations: int32
    @@immutable @@nullable nonce: int64
    @@immutable @@nullable obtainerId: string                          // TODO: should be typed `AccountId` once the string-typed entity-id fields are cleaned up
    @@immutable permanentRemoval: bool
    @@immutable @@nullable proxyAccountId: string                      // TODO: should be typed `AccountId` once the string-typed entity-id fields are cleaned up
    @@immutable fromTimestamp: zonedDateTime
    @@immutable toTimestamp: zonedDateTime
    @@immutable @@nullable bytecode: string
    @@immutable @@nullable runtimeBytecode: string
}

ContractRepository {
    @@async @@throws(mirror-node-error)
    Page<Contract> findAll()

    @@async @@throws(mirror-node-error)
    @@nullable Contract findById(contractId: ContractId)
}

@@static ContractRepository createRepository(mirrorNode: MirrorNode)

```