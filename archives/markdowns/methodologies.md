---
title: for the first question:
date: 2026-09-01
---

for the first question:

1. data sourcing from bigquery and processing
    
    1. from the ‘traces’ dataset: to\_address, from\_address, transaction\_hash, and call\_type values for every trace\_id that appears over the observation window; download table. from the ‘contracts’ dataset: extract address, bytecode, and funtion\_sighashes columns; download table.
    2. query locally for from\_address and to\_address presence in local contracts table
    3. detect and resolve proxies: query locally for where the values of the traces’ dataset ‘call\_type’ column is ‘delegatecall’ and ‘callcode’, then collect from\_address/to\_address pair as potential proxy/implementation contract pair.
    4. query locally and record each contract’s activity counts (unique transaction/call/caller counts)
    5. deduplicate corpus based on runtime bytecode values, record the address counts, bytecode-to-address mapping, and sum up their individual address activity counts.
2. candidate selection
    
    1. run each unique bytecode through a simple disassembler (likely heimdall-rs), perform a walk over of their instruction list, and flag occurrences of `GAS` `CALL` `CALLCODE` `DELEGATECALL` and `STATICCALL` separately.
        
        1. for each opcode, record the number of unique bytecode it occurs in, and for each occurrence, record the bytecode offset (PC) and associated function selector. also record the number of unique opcode occurrences per-bytecode.
        2. for every DELGATECALL- and CALLCODE-flag, check the corresponding address’ proxy status (resolve to obtain implementation runtime bytecode if not in identified proxies) (also collect sample of bytecode that contain each opcode, and check the correctness rate of 1c’s output )
        3. generate CFGs for each flagged bytecode (sample evmole and ethersolve for the better option).
3. syntactic analysis
    
    1. for each GAS occurrence:
        
        1. using the collected offset position, locate and record each occurrence’s basic block and its reachable operations/blocks
        2. flag occurrences according to the contents of their basic block, based on candidate opcode motifs that may arise from the known functionalities
            
            1. there should be an ‘unresolved’ bucket for unrecognised occurrences and missed motifs; if it is large enough then full-scale data-flow analysis is considered for the bucket.
        3. data flow analysis of each flag’s bucket to confirm structural relevancy
    2. for each CALL\* occurrence:
        
        1. using the collected offset position, locate and record each occurrence’s basic block and its preceding blocks
        2. flag contracts that contain basic blocks with GAS and an arithmetic operation whose result is pushed to stack as candidates for trace-level analysis or symbolic execution.
        3. flag contracts whose gas argument ingest a PUSH value as candidates for the hardcoded gas stipend format
        4. <s>bucket any other contracts as unresolved. since this might miss contracts that push the gas argument to stack as a constant value; bucket them as ‘unresolved’, then check only the traces of this bucket as outlined:</s>
            
            1. <s>the big query traces dataset provides _gas_ and _gas\_used_ columns, where the former is the ‘gas\_sent\_with\_call’ term</s>
                
                1. <s>check the traces of transactions of contracts in the unresolved bucket to check for the reentrancy defence model (i.e., when GSWC ≤ 2300</s><s>)</s>
4. motif validation using historical traces/dynamic replay (extend into differential testing (get their pre-state and pass it through per-model semantics modified evm)): for each of 2b’s candidate contracts; replay their transactions, collect the gas-spend details and

* * *