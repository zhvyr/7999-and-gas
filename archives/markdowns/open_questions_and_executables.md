---
title: which legacy gas introspection patterns must actua
date: 2026-09-01
---

**which legacy gas introspection patterns must actually be preserved?**

specifically, applications on Ethereum may make use of gas-related data to provide certain functionalities for users; establish a scale for these functionalities based on their current usage in order to power-scale tradeoffs of each scalar interface preservation mechanism.

this would involve establishing gas introspection (and allotment) patterns, checking the prevalence of the patterns on mainnet contracts, and checking how often those contracts are used. sample questions that should have been answered by the completion of this objective include:

1. how often are contracts designed to pass less than 63/64 of their remaining gas during subcalls? (pattern implementations/unique implementations)
    
    1. establish opcode-based motifs of how CALLER contracts may reserve >1/64 gas during sub-calls,
    2. search for these patterns on mainnet, and
    3. assess the usage of each unique instance, per reservation pattern (i.e., contract transaction count checks).
2. provide an estimate for the number of contracts that are designed to use each gas\_left() pattern on mainnet
    
    1. establish opcode-based motifs of each gas\_left() pattern
    2. search for those patterns on mainnet, and
    3. assess the usage of each unique instance per introspection pattern.

eventually present arguments for/against each of:

* the aggregate evm gas model
* universal overflow
* multidimensional subfee market model

based on the prevalence of the behaviours supported by each model on mainnet today and their apparent usage.

an alternate approach would be full-scale differential testing, in which each model outlined above is implemented in a test environment and transactions are then replayed in the environment to see if (and how) they fail.

* * *

**how much universal overflow is needed in practice?**

universal overflow’s main inspiration is to keep the buffer preservation pattern for caller contracts that pass a specific amount of gas to the callee; hence the overflow amount is to be calibrated based on the typical buffer retained by contracts on mainnet, and what it is used for.

1. this would involve establishing the prevalence and incidence of the buffer preservation pattern and establishing a floor (maybe buckets will be preferred? since if buckets are used it becomes easier to assess how the buffer is used) on buffer sizes. potentially useful questions:
    
    1. how are buffers reserved (ceiling on passed gas vs minimum post-call reserve) and what are they used for (limiting the callee’s abilities, some post-call operation?)
2. since GAS also returns the overflow, loop operations would be limited by an excessively conservative overflow; what is the average quantity of gas\_left these contracts require for their operations?

* * *

**can aggregate EVM gas safely relax the funding check?**