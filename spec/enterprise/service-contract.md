# Contract Service API

Service definition for smart contract interaction.

## API Schema

```
namespace enterprise.service.contract
requires {Address} from ledger
requires {NetworkSetting} from ledger.config
requires {Account, TransactionSigner} from consensusnode.client

ParamSupplier<$$SolidityType> {
    $$SolidityType getNativeContractValue()
}

Param<$$LangType, $$SolidityType> {
    @@immutable value:$$LangType
    @@immutable nativeType:string
    @@immutable supplier:ParamSupplier<$$SolidityType>
}

ContractCallResult {
    @@immutable uint8 size;
    
    type getType(index:uint8)
    
    any get(index:uint8)
}

SmartContractService {

    @@throws(service-error) Address createContract(fileId:Address, constructorParams:Param<ANY, ANY>...)
    
    @@throws(service-error) Address createContract(contents:bytes, constructorParams:Param<ANY, ANY>...)
    
    @@throws(service-error) ContractCallResult callContractFunction(contractId:Address, functionName:string, params:Param<ANY, ANY>...)
}

// Factory methods for params to wrap native types in solidity types
Param<string> ofString(value:string)
Param<string> ofBytes(value:string)
Param<string> ofBytes23(value:string)
Param<bytes> ofBytes(value:bytes)
Param<bytes> ofBytes23(value:bytes)
Param<string> ofAddress(value:string)
Param<Address> ofAddress(value:Address)
Param<boolean> ofBool(value:boolean)
Param<uint8> uint8(value:uint8)
Param<int8> int8(value:int8)
Param<uint256> uint256(value:uint256)
Param<int256> int256(value:int256)

//Factory method to create Service (not needed for real framework integration where injection is used)
@@static
SmartContractService createService(networkSettings: NetworkSetting, operatorAccount: Account)

@@static
SmartContractService createService(networkSettings: NetworkSetting, operatorAccount: Account, transactionSigner: TransactionSigner)

```