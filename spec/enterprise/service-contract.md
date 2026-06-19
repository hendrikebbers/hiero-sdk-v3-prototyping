# Contract Service API

Service definition for smart contract interaction.

## API Schema

```
namespace enterprise.service.contract
requires {Page} from common
requires {Address, ContractId, EvmAddress} from ledger
requires {Session} from enterprise.service

ParamSupplier<$$SolidityType> {
    $$SolidityType getNativeContractValue()
}

Param<$$LangType, $$SolidityType> {
    @@immutable value:$$LangType
    @@immutable nativeType:string
    @@immutable supplier:ParamSupplier<$$SolidityType>
}

ContractCallResult {
    @@immutable size: uint8
    
    type getType(index:uint8)
    
    ANY get(index:uint8)
}

@@finalType
Contract {
    @@immutable contractId: ContractId
}

SmartContractService {

    @@throws(service-error) Contract createContract(fileId:Address, constructorParams:Param<ANY, ANY>...)

    @@throws(service-error) Contract createContract(contents:bytes, constructorParams:Param<ANY, ANY>...)

    @@throws(service-error) ContractCallResult callContractFunction(contract:Contract, functionName:string, params:Param<ANY, ANY>...)

    @@throws(service-error) ContractCallResult callContractFunction(contractId:ContractId, functionName:string, params:Param<ANY, ANY>...)

    // Return the contract information for the given contract id
    @@throws(service-error) @@nullable Contract findById(contractId: ContractId)

    // Return all known contracts
    @@throws(service-error) Page<Contract> findAll()
}

// Factory methods for params to wrap native types in solidity types.
// Solidity's `address` type is a 20-byte EVM address — the EvmAddress overload is the
// canonical typed entry point. The string overload accepts hex form ("0x..."), and the
// long-form (shard.realm.num) Hedera id overloads (AccountId / ContractId) belong here as
// follow-ups once a HAPI long-form-encoding helper is decided.
@@static Param<string, ANY> ofString(value:string)
@@static Param<string, ANY> ofBytes(value:string)
@@static Param<string, ANY> ofBytes23(value:string)
@@static Param<bytes, ANY> ofBytes(value:bytes)
@@static Param<bytes, ANY> ofBytes23(value:bytes)
@@static Param<string, ANY> ofAddress(value:string)
@@static Param<EvmAddress, ANY> ofAddress(value:EvmAddress)
@@static Param<bool, ANY> ofBool(value:bool)
@@static Param<uint8, ANY> uint8(value:uint8)
@@static Param<int8, ANY> int8(value:int8)
@@static Param<uint256, ANY> uint256(value:uint256)
@@static Param<int256, ANY> int256(value:int256)

//Factory method to create Service (not needed for real framework integration where injection is used)
@@static
SmartContractService createService(session: Session)

```