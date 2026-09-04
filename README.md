# Analysis of Mainnet Dependency on Legacy Gas Introspection Patterns

7999’s multidimensional fees and resource pricing disrupts legacy gas introspection patterns, since ‘gas’ becomes a vector quantity that is allocated per unique resource and priced at the resource’s base fee, rather than a single pool for any resource to draw from as in the current model. This wouldn’t be a problem if gas introspection was eliminated entirely, but most of the pushback against EOF was based on its deprecation of these functionalities; which is evidence that they are heavily used by EVM app developers to date.

This repository studies the prevalence of legacy gas introspection patterns to determine which ones must be supported, and thus inform 7999’s gas accounting design. Specifically, it attempts to answer the first two open questions posed in Designs for EVM gas accounting in EIP-7999:
1. Which legacy gas properties must actually be preserved?
2. How much universal overflow is needed in practice?


The study is currently divided into three phases:
1. Corpus building using big-query datasets.
2. Analysis of disassembled bytecode for candidate identification and classification.
3. Validation of the classification step by replaying transactions of representative candidates from each class.
