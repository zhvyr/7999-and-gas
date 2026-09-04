---
title: Research questions adapted from Designs for EVM ga
date: 2026-09-04
---

Research questions adapted from [Designs for EVM gas accounting in EIP-7999](https://notes.ethereum.org/@anderselowsson/gasAccountingIn7999#Open-questions)

**Which legacy gas introspection patterns must actually be preserved?**

Applications on Ethereum can make use of gas-related data to provide certain functionalities for users; establish a scale for these functionalities based on their current usage as a means to power-scale each scalar interface preservation mechanism. This would involve establishing gas introspection (and allotment) patterns, checking the prevalence of the patterns on mainnet contracts, and checking how often those contracts are used.

Sample questions that should have been answered by the completion of this objective include:

1. How often are contracts designed to pass less than 63/64 of their remaining gas during sub-calls?
    
    1. Establish opcode-based motifs of how contracts pass gas to their sub-calls;
    2. search for these patterns on mainnet; and
    3. assess and report the usage of each unique instance, especially for contracts that reserve >1/64 of their gas during sub-calls.
2. Provide an estimate for the number of contracts that are designed to use each `gas_left()` pattern on mainnet:
    
    1. establish opcode-based motifs of each gas\_left() pattern,
    2. search for those patterns on mainnet, and
    3. assess the usage of each unique instance per introspection pattern.

Eventually present arguments for/against each of:

* the aggregate EVM gas model
* universal overflow
* multidimensional subfee market model

based on the prevalence of the behaviours supported by each model on mainnet today and their apparent usage.

* * *

**How much universal overflow is needed in practice?**

Universal overflow’s main inspiration is to keep the buffer preservation pattern for caller contracts that pass a specific amount of gas to the callee; hence the overflow amount is to be calibrated based on the typical buffer retained by contracts on mainnet, and what it is used for. This necessitates establishing the prevalence of the buffer preservation pattern and establishing a floor on buffer sizes per unique usage path.

1. How are buffers reserved (ceiling on passed gas vs minimum post-call reserve) and what are they used for (limiting the callee’s abilities, some post-call operation?)
2. Since `GAS` also returns the overflow, loop operations would be limited by an excessively conservative overflow; what is the average quantity of `gas_left` these contracts require for their operations?