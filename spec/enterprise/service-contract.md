# Contract Service API

Service definition for smart contract interaction.

## API Schema

```
namespace enterprise.service.contract
requires ledger, consensus-node.client

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

    @@throws(service-error) ledger.Address createContract(fileId:ledger.Address, constructorParams:Param<ANY, ANY>...)
    
    @@throws(service-error) ledger.Address createContract(contents:bytes, constructorParams:Param<ANY, ANY>...)
    
    @@throws(service-error) ContractCallResult callContractFunction(contractId:ledger.Address, functionName:string, params:Param<ANY, ANY>...)
}

// Factory methods for params to wrap native types in solidity types
Param<string> ofString(value:string)
Param<string> ofBytes(value:string)
Param<string> ofBytes23(value:string)
Param<bytes> ofBytes(value:bytes)
Param<bytes> ofBytes23(value:bytes)
Param<string> ofAddress(value:string)
Param<ledger.Address> ofAddress(value:ledger.Address)
Param<boolean> ofBool(value:boolean)
Param<uint8> uint8(value:uint8)
Param<int8> int8(value:int8)
Param<uint256> uint256(value:uint256)
Param<int256> int256(value:int256)

//Factory method to create Service (not needed for real framework integration where injection is used)
@@static
SmartContractService createService(networkSettings: ledger.config.NetworkSetting, operatorAccount: consensus-node.client.Account)

@@static
SmartContractService createService(networkSettings: ledger.config.NetworkSetting, operatorAccount: consensus-node.client.Account, transactionSigner: consensus-node.client.TransactionSigner)

```