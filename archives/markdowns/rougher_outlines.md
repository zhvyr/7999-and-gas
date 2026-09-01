---
title: general impact of semantic changes on mainnet
date: 2026-09-01
---

general impact of semantic changes on mainnet

1. what percentage of contracts on mainnet today reserve gas during sub-calls by forwarding an explicit amount via a CALL’s gas argument ((would it be easier to answer “how many contracts reserve >1/64 of their gas? are the questions even equivalent?)),
2. how is gas reserved during a message call typically used by the CALLER later?
3. what percentage of contracts typically call GAS mid-execution, and
4. what do the contracts do after calling GAS? (i.e., why is the check necessary for their functionality?)

answering these will potentially enhance app-dev outreach efforts. would probably be even better if there was a recommended path to get around the impacts of the overflow model and 7999’s semantic changes in existing contracts (where possible).

### executables

executables for the first question:

* identify transactions that perform sub-calls,
* filter the ones where there is gas left behind in the CALLER, and
* classify CALLERs based on how the leftover gas was used.

executables for the second question:

* identify contracts that call GAS mid-execution,
* document their functionality and whether the changes of universal overflow will break them,
* maybe explore mitigations.

big-query’s crypto\_ethereum database archives contracts, transactions, and traces. i think the data from these should be enough for this study, which is great for costs.

### methodology

* * *

big-query’s traces.gas and traces.gas\_used only show block-level data, so that there is no way to consistently reconstruct remaining\_gas correctly from the rest of available data. but when gas\_used is very close to gas, it can be inferred that the caller didn’t send all their available gas.

the results from 2ai. and 2aii. contribute to the answer of q1, as they show which patterns are actively in use; estimated ‘demand’ for each pattern can be established by checking the transaction counts of contracts that follow each pattern from the record of 1a.

1. data sourcing would be from bigquery’s ‘traces’ dataset
    
    1. the transactions of the flagged contracts that contain a CALL\* will serve as the corpus we check; so their transactions within the cut-off range should be extracted from the transactions dataset we already have first.
    2. 1. <s>if we can reconstruct the remaining\_gas term for a given gswc, then we can be sure if a CALL\*’s gswc is all\_but\_one\_64th or requested\_gas</s>
        2. <s>in the latter case, check how much gas was left behind and what the reserved gas was used for</s>

* * *

<s>decisions decisions</s>

<s>for 1b/c/d; bytecode-based patterns may not be obvious if evmole is used since it only does static disassembly and so may not correctly identify a jumpdest. ‘gigahorse’ may be better suited for dataflow confirmation at that level.</s>

<s>a cheaper alternative for the pattern matching exercise is to apply the patterns as a triaging mechanism anyway; i.e., flag bytecodes that _likely_ satisfy some pattern. proper confirmation can then be obtained using the downstream reth-replay</s>

<s>an alternative and probably better approach is to carry out the exercise as described (bytecode proximity-based cfg); then add a fourth bucket for ambiguous contracts that perform a jump before the pattern is completed. dataflow confirmation is then performed for the fourth bucket alone</s>.

Check CFGs of CALL\*-flagged contracts for basic blocks that contain a GAS and an arithmetic operation and pushes the result to stack and flag them; then use data-flow to confirm. this might miss contracts that push the gas argument to stack as a constant value; bucket them as ‘unresolved’, then check only the traces of this bucket.

* * *

1. 1. in each flagged contract, collect the opcode offset (position of the CALL\*/GAS in the runtime bytecode), and the surrounding block (what other opcodes are present at that ‘execution site’?)
        
        1. compare the collected basic block of each CALL\*-flagged contract to opcode-based motifs of how a CALLER may reserve gas and flag candidate contracts that exhibit each reservation pattern.
            
            1. note the next basic block in each flagged candidate.
        2. classify GAS-flagged contracts according to their immediate consumer (e.g., GAS → LT, GAS → MSTORE, etc.) and check where their CFG
    2. track stack values of GAS and CALL\* (g), per flagged candidate from 2ai and 2aii, to their sinks and sources respectively for motif confirmation and bucketing

* * *

**collect address, bytecode, and function selectors from contracts dataset; make this one the database for contracts identification and bytecode/function selector checks.**

**collect from\_address, to\_address, trace\_id, transaction\_hash and call\_type from traces dataset; make the result the database for usage assessment by joining on addresses and counting traces and also use for proxy resolution by limiting to call\_type = delegatecall/callcode and collecting the addresses**

record their unique transaction/call/caller counts

**what are the selectors collected for again?** in the reth replay stage (confirmatory), we can search the traces corpus for transaction hashes whose corresponding input contains a relevant selector of the candidate bytecode. collecting the selectors a target opcode is located in would reduce replay costs since the checks are highly targeted.