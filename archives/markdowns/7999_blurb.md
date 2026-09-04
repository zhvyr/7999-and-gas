---
title: proposal 7999:
date: 2026-09-04
---

[proposal 7999](https://github.com/ethereum/EIPs/blob/556e7170681be3401774b207ab3470d25bda63b5/EIPS/eip-7999.md):

1. introduces multidimensional resource pricing, so that ‘gas’ is no longer the single measure of every processing work done by execution clients;
2. unifies the network’s base fee update mechanism to the preferred 4844 mechanism; and
3. introduces reserve pricing for all resulting fee markets

the second and third goals are somewhat straightforward: the 4844 mechanism already exists and so simply needs to be implemented to calculate new blocks’ `base_fees` ; while the reserve pricing mechanism is a first-price auction.

meanwhile, the first goal introduces four new resources that transactions can pay clients for:

* execution `0` which is expended by every processed transaction;
* blobs `1` which are expended by transactions from data-posting L2s;
* data `2` which is expended by transaction call data, access list entries, `authorisations` in `set_code_tx_type` transactions, blob hashes in `blob_tx_type` transactions, and block access lists; and
* state `3` which is expended by transactions that contain state writes.

users specify only an aggregate `max_fee` for a new transaction type, `MULTIDIM_TX_TYPE`, with payload: `chain_id, nonce, gas_limit, to, value, data, access_list, blob_versioned_hashes, max_fee, max_priority_fee_per_gas, y_parity, r, s`.

block processing now proceeds as follows:

1. initialisation of a `gas_used_so_far` vector to `0, 0, 0, 0`
2. derivation of the block’s `base_fees` via a `get_block_base_fees (block.parent)` function
3. transaction processing then proceeds thus:
    
    1. `max_fee` derivation via a `get_max_fee` function
    2. `tx_gas_limits` derivation via the `get_gas_limits` function
    3. `max_base` derivation via the `get_required_max_fee` function
    4. block capacity checks for indices `0, 1, 2` and fee checks (`max_base <= max_fee`)
    5. `fee_to_deduct` calculation and deduction
    6. post transaction processing, a four-item `tx_gas_consumed` vector is calculated, where
        
        1. the item at `0` is the gas consumed by EVM execution,
        2. `1` is the gas consumed by blobs, which must be equal to blob\_gas,
        3. `2` is the consumed portion of `data_gas` (based on the transaction’s calldata) plus execution data gas metered according to used BAL bytes, and
        4. `3` is the consumed portion of `state_gas`
    7. calculation and burning of the transaction’s base fees, transfer of the priority fees to the block producer, and refunding excess payments to the sender
    8. update the block’s `gas_used_so_far` vector by adding the `tx_gas_consumed` vector
4. after processing a block, require that elements of `block.gas_used` correspond to elements of `gas_used_so_far`

* * *

**get\_gas\_limits**

the `get_gas_limits` function takes in a transaction hash and calculates `data_gas` and `blob_gas`, where:

* `data_gas` is defined by `get_intrinsic_data_gas(tx)` , which returns the “computational power” consumed by user-controlled content bytes
* `blob_gas = len (getattr (tx, blob_versioned_hashes, [])) x GAS_PER_BLOB`

if the transaction is a `MULTIDIM_TX_TYPE` , the `get_gas_limits` function returns `[tx.gas_limit [0], blob_gas, data_gas, 0]` ; for any other txn type, the function checks that `tx.gas_limit >= data_gas` and returns `tx.gas_limit [0] - data_gas, blob_gas, data_gas, 0` . where `tx.gas_limit [0]` is the gas limit of execution, which is declared by the user, and 0 is for the state resource which has no enforced ceiling per-transaction.

for other transaction types, the `get_gas_limits` function checks that `tx.gas_limit` is greater than or equal to the computed `data_gas`, and if so returns `tx.gas_limit - data_gas, blob_gas, data_gas, 0`.

**get\_required\_max\_fee**

the `get_required_max_fee` function first checks if `len(base_fees) == len(tx_gas_limits)`, and if so it returns the sum of the products of each resource’s base fee and gas limit: `sum(vector_mul(base_fees, tx.gas_limits))`.

for other transaction types that continue providing a single gas limit post-7999, the function attempts to establish a tenable cost ceiling by:

1. deducing the intrinsic execution gas (via a `get_intrinsic_exec_gas` function) and the intrinsic state gas (via a `get_intrinsic_state_gas` function)
2. calculating the transaction’s intrinsic cost as: `base_fees [0] x intrinsic_exec_gas + base_fees [3] x instrinsic_state_gas`
3. metering declared blob and data gas limits at their corresponding base fees: `base_fees [2] x tx_gas_limits [2] + base_fees [1] x tx_gas_limits [1]`
4. any leftover gas is `remaining_gas`, which is given as `tx_gas_limits [0] - intrinsic_exec_gas - intrinsic_state_gas` and capped as `gas_left = min (TX_MAX_GAS_LIMIT - intrinsic_exec_gas, remaining_gas)`
5. this `gas_left` budget is then priced at the highest base fee out of the base fees for execution, data, and state, and can be spent by any of those three; so that it is essentially a transaction’s budget for its resource expenditure across execution, data, and state
6. the difference between `remaining_gas` and `gas_left` (referred to as `state_gas_reservoir`) is costed at state’s base fee, since state is the only resource that has no explicit limit on its consumption in a block.

another way to interpret the spiel above is: transactions that continue providing a single `gas_limit` post-7999 would still have this limit divided into budgets for each distinct resource, which are then consumed at the base fee of that resource during execution. however, as it is not possible to know which resources transactions would actually expend *before execution*, the intrinsic costs of single limit transactions are tracked via `intrinsic_exec_gas` and `intrinsic_state_gas`, which are deducted from the provided `tx.gas_limit` and costed at the execution and state base fees respectively. the leftover gas is then split into a pool (i.e., `gas_left`) for resources in `SHARED_GAS_INDICES` that is metered at the highest base fee of a given block height, and a state-only reservoir (i.e., `state_gas_reservoir`) that is metered at the state base fee.

altogether, this current design incorporates features of the **updated EVM** and **aggregate EVM gas** models discussed in [Designs for EVM gas accounting in EIP-7999](https://notes.ethereum.org/@anderselowsson/gasAccountingIn7999); the former introduces a new transaction and sub-call format that allows transactors to specify separate budgets for each resource their transactions and call frames might consume, while the latter is a compatibility shim for legacy contracts that use the `GAS` opcode and the gas argument of `CALL*` opcodes.

* * *

the **aggregate gas** model is most backwards-compatible with mainnet as app-layer gas semantics are minimally affected. specifically: since the budget for the transaction's major processing costs is aggregated into a singular `gas_left` budget, the `gas` argument of `CALL*` opcodes will push any specified amount to a callee from `gas_left`; and `GAS` and `GASPRICE` will be associated with `gas_left`.

the major downside is that intrinsic costs typically make up only a small fraction of a transaction’s entire cost. hence, most of the resources reserved for the transaction in this model is metered at the highest base fee to calculate the amount (`max_base`) that must be made available by a user for their transaction to be considered valid. this results to fee quotes that do not reflect the true cost of processing transactions that specify a single gas limit but don’t actually consume the most expensive resource at runtime; and this ostensibly worsens ux as it would cause otherwise well-paying transactions to fail. after all, users will most likely specify a single gas limit for the perceived convenience, rather than because they actually intend to use the most expensive resource.

the aggregate gas model also weakens `GAS` semantics to an extent, as a call frame may consume gas from the state reservoir which is not exposed externally and so is explicitly invisible to a caller. this poses a problem for contracts that utilise solidity’s gasleft() function for accounting, and also limits the model’s extensibility as expanding the resources priced by the protocol would increase the the number of operations whose costs may not reflect properly.

furthermore, block producers would be adversely impacted by this design to some extent: blocks will continue to have protocol-enforced ceilings per resource, but since there is no way for a block producer to know which resource is consumed by a transaction pre-execution, they’d have to reserve enough capacity for a transaction’s `gas_left` across the block’s resource vector (`gas_used_so_far`). this reduces block packing efficiency depending on the quantity of `gas_left` possessed by transactions in its mempool.

**the primary deliverable for this design is evaluating the possibility of relaxing the funding check. stretch goals can include fully laying out the variants mentioned in the original essay, and proposing their possible failure conditions and fixes.**

* * *

the **universal overflow** model requires that transactors specify resource-specific budgets plus a separate budget referred to as the overflow, which may be utilised by any operation at runtime provided its own budget has been exhausted. then, only the overflow budget has to be priced defensively pre-execution while other resource-based limits are priced at their associated resources’ base fees. the behaviours of `GAS` and the `CALL*` opcodes are modified in this model, so that the former returns the remaining amount of overflow at any time, and the latter’s gas argument is drawn from the overflow budget during sub-calls.

this model is primarily designed to preserve the `CALL*(g)` reservation pattern but has the additional benefit of relaxing the funding requirement for transactions, as only the overflow budget would have to be funded at a block’s highest base fee. thus, transactors who don’t need to reserve gas for themselves across call frames, who should be the majority, can take full advantage of the model by setting their transactions’ overflow budget to zero.

universal overflow is also greatly extensible: new resources are supported by adding their dedicated limit and base fees, while a scalar overflow remains available for when fungibility is necessary.

for block producers, they’d have to reserve enough blockspace for a transaction’s overflow across the block’s resource vector, as the transaction’s target resource is still ambiguous. however, this could be less of a problem if the overflow amount is capped explicitly so that transactions use it for ‘backup’ as intended.

the primary drawback of this model is that it breaks contracts that sandbox call frames with the gas argument during sub-calls (e.g., the 2300 send() stipend), as limiting the budget passed via the gas argument of sub-calls will only limit the amount of overflow a call frame receives—every other budget is passed according to current semantics (63/64).

also, contracts that need to know the instantaneous values of their distinct budgets will need multidimensional introspection.

**the primary deliverable for this design is evaluating the amount of gas typically reserved by contracts during sub-calls, and what the reservation is typically used for. stretch goals include comparing the efficiency gains of the universal overflow model against the overflow vector variant.**

* * *

the **multidimensional subfee market** model proposes that transactions set a gas limit for the resource whose base fee is chosen as the denominator for calculating a scalar gas cost from the ratios of resources base fees, and set byte limits for every other resource. hence, transactions will specify a regular limit for the execution resource, and byte limits for state and data resources (blobs have a separate consumption path and so aren't included).

under this model, the protocol computes a scalar cost per resource byte at the start of each block height, which is fixed for the block’s duration. using the current resources for example:

* at the start of a block, the protocol calculates the
    
    * state cost per byte rate, C<sub>s</sub>, using the base fees for state and execution, and a baseline state gas per byte, c<sub>s</sub>: C<sub>s</sub> = c<sub>s</sub> \* b<sub>s</sub>/b<sub>e</sub>
    * data cost per byte rate, C<sub>d</sub>, using the base fees for data and execution, and a baseline data gas per byte, c<sub>d</sub>: C<sub>d</sub> = c<sub>d</sub> \* b<sub>d</sub>/b<sub>e</sub>
* transactions that specify state and data byte limits, L<sub>s</sub> and L<sub>d</sub>, will then purchase C<sub>s</sub>L<sub>s</sub> and C<sub>d</sub>L<sub>d</sub> in scalar gas, along with their specified limit for execution L<sub>e</sub>. these are summed together as ‘G’ and charged at the execution base fee.

their scalar sum, G, is then reported in `GAS` and is forwardable via `CALL*(g)`; and contracts that need to interact with the value would need to read each resource’s cost per byte at a specific block in order to obtain the concrete quantities of each resource it represents.

the primary disadvantage of the subfee model is that opcode pricing becomes nondeterministic, as execution’s demand volatility will affect the subfee rates for other resources, which in turn affects the costs of data and state operations. of course this is easily solved by requiring that contracts read the per byte rates before performing any related operations, but it isn’t easily implemented.

**not really sure what deliverable to target for this model as it seems ‘complete’ as a standalone; comparing its efficiency gains vs implementation complexity and overhead to other models seems to be the best approach atm. although, one question is whether the 7918 reserve mechanism should also be implemented for the subfees to prevent rate collapse if the execution base fee decreases rapidly within some time (alternately, including the execution resource in** `GAS_RESERVE_INDEX` **might be sufficient)**

* * *

the three models above each provide a scalar budget that preserves different gas introspection patterns to varying extents, but they come with different tradeoffs. each can be implemented as the standalone budgeting mechanism of 7999, but that isn’t necessarily a good idea as it leaves the protocol with a higher extent of technical debt that'd have to be dealt with eventually and limits the benefits of resource multidimensionality that’d otherwise be immediately available to developers and users.

so the more fitting approach longterm would be updating the EVM to support distinct budgeting per protocol resource; so that a vector of budget values is carried by every call frame’s gas argument `CALL*(g)` and returned by the `GAS` opcode, or their equivalent replacements. since there would be some lag between 7999’s implementation and its total adoption by applications, there should be a compatibility shim that allows old contracts keep transacting using legacy patterns. the compatibility shim could implement any of the scalar preserving mechanisms mentioned above.

**the most important deliverable for this model would be evaluating how calls between new and legacy contracts will be handled in the interim.**