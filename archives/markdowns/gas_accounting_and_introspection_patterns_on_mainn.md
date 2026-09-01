---
title: gas accounting and introspection patterns on mainn
date: 2026-09-01
---

### gas accounting and introspection patterns on mainnet

gas introspection/observability is the ability of contracts to

* **precisely allot gas to their subcalls**, via the `gas` argument of `CALL*` opcodes (`CALL, DELEGATECALL, STATICCALL, CALLCODE`); and
* **monitor gas expenditure** at any point during their transactions’ execution via the `GAS` opcode, which returns the amount of unspent gas a transaction has at any point.

7999’s multidimensional fees and resource pricing disrupts these protocol functionalities, since ‘gas’ becomes a vector quantity that is allocated per unique resource and priced at the resource’s base fee, rather than a single pool for execution to draw from as in the current model. this wouldn’t be a problem if gas introspection was eliminated entirely, but most of the pushback against EOF was based on its deprecation of these functionalities; which is evidence that they are heavily used by EVM app developers to date.

the purpose of this study is to establish to what extent each discussed introspection pattern is used, in order to determine which ones must be supported in the short to medium term, and which ones can be easily served in an alternate manner, and ultimately inform 7999’s design choices.

`CALL*` opcodes

ignoring their mechanics which aren’t very important atm, the gas cost of executing any `CALL*` opcode is given as: `gas_cost = base_gas + gas_sent_with_call`, where `base_gas` is the innate cost of the opcode and `gas_sent_with_call` (GSWC) is the amount of gas allotted to the target contract (also the “callee”) for their processing work.

the GSWC may be obtained in any of two ways:

* via the aforementioned gas argument of the opcodes, as an explicit `requested_gas`
* via the 63/64 rule of EIP-150, which effectively forwards 63/64 of a known `remaining_gas`\*\* as `all_but_one_64th = remaining_gas - (remaining_gas/64)`

\*\*`remaining_gas` is the amount of unspent gas after `base_gas` has been consumed but before GSWC is known; it is calculated as `available_gas - base_gas`. where `available_gas` is the original amount of gas that was leftover from the execution context before the `CALL*`

the gas argument of CALL\* opcodes may be used to either limit the amount of gas handed to a subcall, or to reserve some gas for the CALLER’s purposes. the former is typically implemented as a reentrancy defense pattern, which has been discouraged post-Dencun; while the latter is still supported by the protocol and so is of more importance. hence, knowing the frequency of contracts that use requested\_gas to reserve a buffer >1/64 of their remaining\_gas is important.

* figure out how to check if GSWC is requested\_gas or all\_but\_one\_64th; and how to check what callers that reserve gas do with it

GAS opcode

monitoring gas expenditure at runtime is made possible by the `GAS` opcode, which can be accessed by evm contracts via solidity’s inbuilt `gasleft()` function. this functionality helps contracts avoid the `out_of_gas_error` at runtime, since they can observe their spending and exit an operation before their available gas finishes

this functionality is mostly used across meta-transaction systems in the following ways:

* checking the amount of gas they have at the end of each step of a loop operation and exiting the operation when some minimum amount of gas is reached.
    
    * prospective contracts that follow this pattern include: exchange withdrawal contracts, bundlers, etc., who may use the pattern when queueing withdrawals and packing bundles.
* checking the amount of gas they have before and after performing a call and noting the difference (would this involve a TSTORE? i.e., GAS⇾CALL⇾GAS⇾TSTORE (with the necessary PUSH arguments ofc))
    
    * prospective contracts for this pattern include contracts that benchmark the work done in some context to enable gas delta accounting, such as: paymasters, intent-based systems, etc.
* checking that the amount of gas they were allotted by a CALLER is above some threshold that would be enough to execute some subcall, and throwing an error if not
    
    * contracts such as smart contract wallets can use this pattern to defend against the insufficient gas grieving attack described in “…the concept of gas and its implications” by ronan.

* * *