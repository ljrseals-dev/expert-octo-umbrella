COMPLETE UTILITY PATENT APPLICATION (DRAFT)

Attorney Docket No.: BG-2026-001
Inventor: Lewis James Seals Jr
Filing type: Non-provisional utility application under 35 U.S.C. §111(a)

Draft notice. This is a filing-grade specification prepared for attorney review and inventor signature. It is not legal advice. A registered USPTO practitioner must review claims for §101/§103 exposure before filing, and the inventor must sign the declaration personally (37 CFR 1.63; 1.4(d)(2)).

---

TITLE OF THE INVENTION

System and Method for Exhaustive Combinatorial Search with Index-Arithmetic Sharding, Registry-Bound Objective Evaluation, and Pre-Execution Cost Enforcement

---

CROSS-REFERENCE TO RELATED APPLICATIONS

[0001] This application claims the benefit of U.S. Provisional Application No. [/,__], filed [date], the entire contents of which are incorporated herein by reference.

---

FIELD OF THE INVENTION

[0002] The present disclosure relates generally to distributed computing systems for combinatorial search, and more specifically to systems and methods that (i) enumerate a Cartesian product of discrete parameter axes across independent, resumable shards using mixed-radix index arithmetic; (ii) bind scoring functions to a closed registry to prevent execution of caller-supplied code; (iii) bound post-cancellation work using a two-tier abort check; and (iv) enforce compute-cost ceilings through recomputation-on-override and pre-execution balance debiting.

---

BACKGROUND OF THE INVENTION

[0003] Exhaustive search over a discrete grid of parameter values — commonly called a Cartesian sweep, grid search, or hyperparameter sweep — is a foundational technique in machine learning, hardware characterization, simulation design, and scientific computing. Given axes A₁ … Aₙ with cardinalities c₁ … cₙ, the search space has T = ∏ cᵢ combinations. The appeal of exhaustive search is its guarantee: every combination is evaluated, so the global optimum within the grid cannot be missed. The cost of that guarantee is that T grows multiplicatively, and practical deployments routinely reach 10⁶ to 10⁹ combinations.

[0004] Existing systems attack the cost problem in three ways, each with a specific defect.

[0005] Defect 1 — Redundant enumeration under naive sharding. A common sharding strategy distributes combinations across N workers by the rule global_index mod N == worker_id. When the global index is produced by a stateful iterator (e.g., itertools.product or an equivalent recursive enumerator), each worker must advance the iterator through the entire space and discard the (N−1)/N of items it does not own. Total iteration work is therefore N × T rather than T. At T = 10⁹ and N = 64, this is approximately 6.4 × 10¹⁰ iterator steps before any scoring occurs. The sharding overhead exceeds the scoring cost, and the system is slower than a single worker. Streaming iterator libraries provide no mechanism for a worker to jump to index k in O(1) or O(n) time.

[0006] Defect 2 — Unsafe objective binding. Systems that accept user-supplied scoring logic frequently accept it as source text and evaluate it at runtime via eval, exec, compile, or deserialization of pickled callables. This exposes the host to arbitrary code execution. Safe alternatives (sandboxed interpreters, restricted execution contexts) impose significant performance overhead per call and are frequently abandoned under throughput pressure.

[0007] Defect 3 — Post-hoc cost accounting. GPU and cloud compute is billed by wall-clock time at rates of $0.003 to $0.10 per node-hour. Systems that estimate cost before execution typically check the estimate against a ceiling once, at plan time, then execute with whatever parameters the runtime supplies. When a caller overrides a launch parameter (e.g., node count) after planning, the stale estimate is re-used for the balance check, and the actual cost can exceed the ceiling by the override ratio. Furthermore, most systems do not debit the balance before scoring begins; they observe balance at launch and rely on later reconciliation, permitting concurrent launches to collectively overspend an account.

[0008] There is accordingly a need for a combinatorial search system in which (i) shard enumeration cost is proportional to the shard's own output size, not to the full space; (ii) objective binding is provably free of caller-supplied code paths; (iii) cancellation latency is bounded by a single in-flight item rather than a full batch; and (iv) cost ceilings are enforced against effective (post-override) parameters and against a debited balance rather than an observed balance.

---

SUMMARY OF THE INVENTION

[0009] In one aspect, the invention provides a computer-implemented method for exhaustive combinatorial search, comprising: receiving a search space comprising a plurality of named axes, each axis having an ordered list of discrete values; computing a total count T as the product of the axis cardinalities without materializing the Cartesian product; decoding a global integer index k ∈ [0, T) into a combination by successive modular division against said cardinalities in reverse axis order; and assigning to each of N shards the subset of global indices {k : k ≡ shard_id (mod N)}, whereby each shard performs a number of decoding operations equal to its own output size and independent of N.

[0010] In another aspect, the invention provides a method for safe objective binding in which scoring functions are stored in a closed registry keyed by character string, selected by registry lookup at execution time, and never constructed from caller-supplied text. Registry lookup failure raises an error enumerating valid names.

[0011] In another aspect, the invention provides a method for bounded-latency cancellation in a batched scoring loop, in which a shared cancellation flag is inspected before appending each item to a pending batch and again after the batch reaches capacity but before batch evaluation, bounding post-cancellation work to one in-flight item.

[0012] In another aspect, the invention provides a method for compute-cost enforcement, in which an estimated cost is recomputed at launch time from effective launch parameters including any runtime overrides, validated against a stored ceiling, and debited against a provider balance before scoring begins via a charge primitive that raises on insufficient funds.

[0013] In another aspect, the invention provides for independent resumability of shards by persisting a per-shard last-completed global index and resuming from that index plus N, without coordination among shards or with an orchestrator.

[0014] The disclosed techniques provide concrete technical improvements over the prior art: shard enumeration work reduces from Θ(N·T) to Θ(T); objective evaluation becomes structurally incapable of executing caller-supplied code; worst-case post-cancellation compute is bounded by a constant; and the probability of an account overspend event is reduced to zero under a single-writer assumption by pre-execution debiting.

---

BRIEF DESCRIPTION OF THE DRAWINGS

[0015] FIG. 1 is a block diagram of an overall system architecture according to an embodiment, showing plan store, provider interface, shard decoder, and scoring engine.

[0016] FIG. 2 is a flowchart of the plan phase, including space validation, counting, ceiling check, and plan-record creation.

[0017] FIG. 3 is a detailed flowchart of the mixed-radix index decoder of FIG. 2 step 208.

[0018] FIG. 4 is a diagram contrasting the enumeration work of naive modular sharding with the index-arithmetic sharding of the present disclosure, showing Θ(N·T) versus Θ(T) iterator advancement.

[0019] FIG. 5 is a flowchart of the launch phase, including override recomputation, ceiling re-validation, provisioning, balance charge, and provisioning rollback on failure.

[0020] FIG. 6 is a timing diagram of the two-tier abort check of the batched scoring loop.

[0021] FIG. 7 is a flowchart of registry-bound objective lookup.

[0022] FIG. 8 is a flowchart of resume-from-checkpoint using a persisted per-shard last-completed index.

---

DETAILED DESCRIPTION

I. System Architecture (FIG. 1)

[0023] Referring to FIG. 1, a system 100 comprises a plan store 102, a search-space validator 104, a counting module 106, a mixed-radix decoder 108, a shard slicer 110, a provider interface 112, an objective registry 114, a scoring engine 116, a cancellation flag 118, and a cost-enforcement module 120.

[0024] The plan store 102 is a persistent keyed collection (e.g., an in-memory dictionary, a Redis instance, or a relational table) mapping a plan identifier (plan_id) 122 to a plan record 124. In an embodiment, the plan record 124 comprises: space 126; objective_name 128; chip 130; node count 132; total count 134; ceiling_usd 136; estimate_usd 138; state 140; shards 142; results 144; evaluated_count 146; best_so_far 148; device_used 150; created_at 152; started_at 154; finished_at 156; killed_at 158.

[0025] The search-space validator 104 rejects a space 126 that is empty, that names any axis whose value list is empty, or that names an objective not present in the registry 114.

[0026] The counting module 106 computes T as the product of axis cardinalities using integer arithmetic and returns early once the running product exceeds a hard cap (in an embodiment, 10⁹), so that spaces too large to enumerate are refused without materializing any combination.

[0027] The mixed-radix decoder 108 is described in detail in Section III.

[0028] The shard slicer 110 is described in detail in Section IV.

[0029] The provider interface 112 is an abstract contract exposing at least: create_instance(name, chip) 160; stop_instance(name) 162; run_command(name, command) 164; get_balance() 166; charge(amount_usd) 168; and get_payment_link(amount) 170. In an embodiment, a mock provider 172 implements the contract entirely in process memory for testing, and a cloud provider 174 implements the contract against a GPU cloud API.

[0030] The objective registry 114 is a closed mapping from character string to a callable function having signature f(combo) → float. In an embodiment, registry 114 contains at least: "sphere" → negative sum of squares; "rastrigin" → negative Rastrigin; "rosenbrock" → negative Rosenbrock; "linear_sum" → sum; "product" → product. No entry is derived from caller input.

[0031] The scoring engine 116 evaluates a shard against a selected objective function on a selected device and returns scored pairs. Its inner loop is described in Section VI.

[0032] The cancellation flag 118 is a shared mutable slot readable by the scoring engine 116 and writable by a cancellation handler. In an embodiment the flag is a field in the plan record 124 (state == "killed"); in another embodiment it is an atomic boolean in shared memory for multi-process execution.

[0033] The cost-enforcement module 120 is described in detail in Section V.

II. Plan Phase (FIG. 2)

[0034] At step 200, the system receives a request comprising a space 126, an objective name 128, a ceiling_usd 136, a chip 130, and a node count 132.

[0035] At step 202, the validator 104 confirms that the registry 114 contains objective_name 128. On miss, it raises an error enumerating valid names.

[0036] At step 204, the validator 104 confirms that the space 126 is non-empty and that each axis value list is non-empty. On failure it raises an error.

[0037] At step 206, the counting module 106 computes T. In an embodiment:

```
T ← 1
for each axis value list L in space:
    if |L| = 0: return 0
    T ← T · |L|
    if T > MAX_COMBINATIONS: return T   // early exit, refused below
return T
```

[0038] At step 208, if T = 0 or T > MAX_COMBINATIONS, the system raises and terminates the plan phase. In an embodiment MAX_COMBINATIONS = 10⁹.

[0039] At step 210, the validator 104 confirms that 1 ≤ node count 132 ≤ MAX_NODES. In an embodiment MAX_NODES = 64.

[0040] At step 212, the cost-enforcement module 120 computes the estimate:

```
E_minutes ← max( T / (nodes · R), 0.01 )       // R = throughput, e.g. 10^5/min/node
E_usd     ← (E_minutes / 60) · hourly_rate(chip) · nodes
```

[0041] At step 214, if E_usd > ceiling_usd 136, the system raises and terminates the plan phase without creating a plan record.

[0042] At step 216, the system creates plan_id 122 (in an embodiment, a UUIDv4), instantiates plan record 124, stores ceiling_usd 136, estimate_usd 138 = E_usd, state 140 = "planned", and returns the plan_id to the caller with the estimate.

III. Mixed-Radix Index Decoder (FIG. 3)

[0043] The decoder 108 solves the enumeration-jump problem by constructing the combination for a given global index k directly, without consulting any prior index. Given ordered axes A₁ … Aₙ with cardinalities c₁ … cₙ and value lists L₁ … Lₙ, the global index k ∈ [0, ∏cᵢ) maps to combination:

```
combo ← empty map
working ← k
for i ← n down to 1:
    d_i ← working mod c_i
    working ← working div c_i
    combo[A_i] ← L_i[d_i]
return combo
```

[0044] This is the standard mixed-radix positional decomposition. Its significance in the present context is that decoding time is O(n) — linear in the number of axes, which is typically 3–10 — and independent of T. Consequently, the decoder can jump to any global index without state.

[0045] In an embodiment, the decoder is implemented as a pure function combo_at(space, k) and is used in both the cartesian iteration path and the shard slicing path, so that a single decoding implementation is the sole source of combinations throughout the system. This guarantees that every combination in the space has exactly one decoding and that the mapping is consistent across shards.

[0046] In an embodiment, the decoder returns a dict keyed by axis name, preserving axis order.

[0047] In an embodiment, the mixed-radix decomposition is used in reverse to encode a combination back to its global index, enabling an O(n) membership test and enabling shard-local checkpointing (Section VIII).

IV. Index-Arithmetic Sharding (FIG. 4)

[0048] FIG. 4 illustrates the technical improvement. Under the naive modular strategy of the prior art, each of N shards must advance a stateful iterator through all T combinations to filter out its (1 − 1/N) non-owned share. Iterator advancement is Θ(T) per shard, so aggregate advancement is Θ(N·T). Under the index-arithmetic strategy of the present disclosure, shard j enumerates indices {j, j+N, j+2N, …} < T and decodes each in O(n) via decoder 108. Shard j performs ⌈(T − j)/N⌉ decodes; aggregate decoding is Θ(T·n) ≈ Θ(T) for fixed n.

[0049] In an embodiment, the shard slicer 110 is:

```
total ← count(space)
def _shard_iter(j):
    k ← j
    while k < total:
        yield combo_at(space, k)
        k ← k + N
```

[0050] In an embodiment, the shard slicer returns N independent generators, each holding only its own current index and its own plan reference. No shared iterator state exists. No two shards require synchronization. The N generators may be consumed sequentially, concurrently in threads, concurrently in processes, or on remote hosts without modification.

[0051] In an embodiment, the slicer also returns an exact per-shard output size:

```
base      ← total div N
remainder ← total mod N
size(j)   ← base + (1 if j < remainder else 0)
```

which permits the orchestrator to allocate work or budget proportionally.

V. Launch Phase and Cost Enforcement (FIG. 5)

[0052] At step 500, the system receives plan_id 122 and optional overrides for chip 130 and node count 132.

[0053] At step 502, the plan store 102 retrieves the plan record 124. If absent, the system raises. If state 140 is not "planned", the system raises (relaunch policy: refuse). In an alternative embodiment, state "killed" triggers a reset of shards 142, results 144, evaluated_count 146, and best_so_far 148, and proceeds.

[0054] At step 504, the cost-enforcement module 120 recomputes the estimate from effective parameters after applying overrides:

```
rate      ← hourly_rate(effective_chip)
nodes_eff ← effective_node_count
E_minutes ← max( total / (nodes_eff · R), 0.01 )
E_usd     ← (E_minutes / 60) · rate · nodes_eff
```

This is a critical distinction over the prior art: the estimate is not the plan-time estimate 138; it is a fresh computation from effective launch parameters.

[0055] At step 506, if E_usd > ceiling_usd 136, the system raises. A caller who plans at node count 1 and launches at node count 64 cannot slip past the ceiling, because the recomputed cost reflects 64× the node cost.

[0056] At step 508, the provider interface 112 returns current balance B. If B ≤ 0, or if E_usd > B, the system raises before provisioning.

[0057] At step 510, the system provisions effective_node_count nodes via create_instance(). Provisioning is wrapped in a rollback handler: if create_instance raises for node k, then for every node already created in this launch, stop_instance is invoked; plan["shards"] is truncated to reflect only completed work; and the exception is re-raised. This prevents the orphaned-billing condition in which a partial provisioning leaves nodes running under a plan whose state remains "planned".

[0058] At step 512, the system calls provider.charge(E_usd) 168. The charge primitive debits the balance and raises when insufficient. On success, the balance is reduced by E_usd before any scoring work begins. Under a single-writer assumption, concurrent launches cannot collectively exceed the balance.

[0059] At step 514, the system initializes plan["shards"] to one entry per node, each with fields node, index, status, scored.

[0060] At step 516, the system invokes the scoring engine 116 on each shard as described in Section VI. As each shard completes, the engine appends scored pairs to plan["results"] 144 (not merely to a local accumulator), updates evaluated_count 146, and updates best_so_far 148 if the shard's local maximum exceeds the current best. Progressive population of results 144 enables a subsequent kill() to report partial results correctly.

[0061] At step 518, after all shards complete or the run is killed, the system sorts results 144 by score descending and marks state 140 "done" or preserves "killed".

[0062] At step 520, the system invokes stop_instance() for every provisioned node, regardless of outcome, so that billing does not continue after scoring.

VI. Two-Tier Abort Check (FIG. 6)

[0063] The scoring engine 116 receives a shard iterator, an objective function, and a plan reference. Its inner loop reads:

```
results ← []
batch   ← []
for combo in shard:
    if plan.state == "killed": raise PlanKilled
    batch.append(combo)
    if |batch| ≥ B:
        if plan.state == "killed": raise PlanKilled
        results.extend((c, f(c)) for c in batch)
        batch ← []
if batch:
    if plan.state == "killed": raise PlanKilled
    results.extend((c, f(c)) for c in batch)
return results
```

[0064] Two inspections of the cancellation flag occur per batch cycle: one per item (cheap, before append) and one per batch (before evaluation). The combination bounds the worst-case post-cancellation work to the size of a single in-flight item — not to a full batch of size B. In an embodiment B = 256, so worst-case post-cancellation compute is one combination rather than 256.

[0065] In an embodiment, a killed shard raises a PlanKilled exception that the launch orchestrator catches and treats as a stop condition rather than an error, preserving partial results.

VII. Registry-Bound Objective Lookup (FIG. 7)

[0066] At step 700, the scoring engine receives objective_name 128 as a character string. At step 702, it consults registry 114. At step 704, if the name is absent, it raises a KeyError whose message enumerates the sorted valid names. At step 706, if the name is present, it returns the stored callable, which is then invoked as f(combo) at step 708.

[0067] The registry is a closed set constructed at module load. No path exists from user input to eval(), exec(), compile(), pickle.loads(), importlib, or any other dynamic code construction. This is a structural guarantee, not a policy: there is no code that could execute caller-supplied text even if the caller tried.

VIII. Resume from Checkpoint (FIG. 8)

[0068] Because shards are defined purely by index arithmetic, a shard may persist its last-completed global index k_last and, on restart, resume from k_last + N without consulting any other shard or the orchestrator. At step 800, shard j reads checkpoint k_last. At step 802, it invokes the decoder 108 on k_last + N. At step 804, it processes the index stride from there through T − 1. At step 806, after each batch, it writes k_last.

[0069] In an embodiment, checkpoint storage is a per-shard key in a key-value store. In an embodiment, checkpoint writes occur every batch, so restart cost is bounded by B combinations.

[0070] This resumability is a direct consequence of the decoder 108 being a pure function of global index. It is not possible under stateful-iterator sharding without coordinating the entire enumeration state across workers.

IX. Worked Example

[0071] Consider space S = {x: [1,2,3], y: [1,2,3], z: [1,2,3]}. Cardinalities c = (3,3,3); T = 27. Axes order is (x,y,z). Under decoder 108, index k = 4 decomposes as:

```
working = 4
i = z: d_z = 4 mod 3 = 1; working = 4 div 3 = 1; combo[z] = L_z[1] = 2
i = y: d_y = 1 mod 3 = 1; working = 1 div 3 = 0; combo[y] = L_y[1] = 2
i = x: d_x = 0 mod 3 = 0; working = 0;               combo[x] = L_x[0] = 1
```

so combo_at(S, 4) = {x:1, y:2, z:2}.

[0072] For N = 3 shards, shard 0 yields indices {0,3,6,…,24} (9 combos), shard 1 yields {1,4,7,…,25} (9 combos), shard 2 yields {2,5,8,…,26} (9 combos). Aggregate decoding is 27, not 81 — a 3× reduction in enumeration work, and the reduction grows linearly with N.

[0073] Scoring under objective "sphere": each combo is scored −(x² + y² + z²). Sorted descending, the best is {x:1,y:1,z:1} at score −3.

X. Variations and Alternative Embodiments

[0074] In an embodiment, the shard slicer assigns contiguous rather than strided index ranges to each shard. Contiguous assignment is preferable when combinations sorted by index are known to correlate in score and the orchestrator desires early-termination on a running maximum; strided assignment is preferable for load balancing when scores are uncorrelated. Both are within the scope of the claims because both rely on decoder 108 to jump to arbitrary indices.

[0075] In an embodiment, the decoder 108 is vectorized via a tensor library (e.g., PyTorch) so that a batch of K indices decodes to K combinations in a single device operation, with the axis decomposition computed as successive modulo and integer-division operations on an index tensor and values gathered from per-axis value tensors. This enables device-side enumeration and combines with device-side scoring.

[0076] In an embodiment, the charge primitive 168 is implemented as a two-phase reservation: reserve at launch, then settle to actual on completion. The reservation amount is E_usd; the settlement amount is the actual billed cost. In an embodiment, a surplus is refunded after scoring completes; the present disclosure does not require refunding, and the claims encompass both.

[0077] In an embodiment, the plan store 102 is bounded with an LRU policy and a TTL, so that long-running servers do not accumulate plan records indefinitely.

[0078] In an embodiment, the scoring engine runs shards concurrently using asyncio.gather over executor threads or processes, with concurrency bounded by effective_node_count. The abort flag 118 is checked from within each shard's loop as described.

[0079] In an embodiment, the provider interface 112 additionally exposes run_shard(name, shard_spec) for shipping a shard specification (space, axis order, index range, objective_name) to a remote node, which decodes and scores remotely and returns scored pairs. The shard specification is compact — the space plus a start index plus a stride — rather than a materialized list of combinations.

[0080] In an embodiment, the system is exposed as a Model Context Protocol (MCP) server, exposing plan, launch, status, results, kill, and probe tools to a large-language-model client. The MCP wrapper does not alter the underlying methods described above; it is a transport binding.

[0081] In an embodiment, the objectives registry 114 is extended by registration at module import time. In an embodiment, registration is closed to runtime callers, and only module-level functions populate the registry.

[0082] In an embodiment, the ceiling_usd 136 is stored as an immutable field on the plan record and is not updatable after plan creation; a caller wishing to raise the ceiling must create a new plan. In an alternative embodiment, ceiling_usd 136 is mutable subject to an audit log.

XI. Advantages Over the Prior Art

[0083] The disclosed system provides the following advantages, each of which is a technical improvement over the prior art rather than an economic or business advantage:

(a) Shard enumeration work reduces from Θ(N·T) to Θ(T) — a factor-of-N improvement in enumeration throughput at fixed N, and the improvement grows with deployment scale.

(b) Decode time is O(n) per combination and independent of T, enabling random-access to any global index and enabling resume-from-checkpoint with no cross-shard coordination.

(c) Objective binding is structurally free of dynamic code paths; the registry is a closed set and no caller input reaches any code-construction primitive.

(d) Worst-case post-cancellation work is bounded by a constant (one in-flight item) rather than by the batch size.

(e) Cost ceilings are re-enforced against post-override effective parameters and against a debited balance, eliminating the class of overspend events caused by (i) stale estimate re-use and (ii) concurrent launches observing a pre-charge balance.

(f) Provisioning is transactional with respect to plan state: partial provisioning rolls back rather than leaving orphaned nodes.

(g) Results are populated progressively so that a kill event yields correct partial-result reporting, and running-best tracking is exact.

XII. Definitions

[0084] "Combination" means an assignment of one value to each axis. "Global index" means the mixed-radix integer encoding of a combination in a fixed axis order. "Shard" means a partition of the global index set assigned to one worker. "Ceiling" means a hard maximum spend in USD. "Estimate" means a projected cost computed from total combinations, effective node count, effective chip rate, and a throughput constant. "Registry" means a closed mapping from name to callable. "Charge" means a primitive that reduces a balance and raises on insufficiency. "Effective parameters" means launch parameters after application of any runtime overrides.

---

CLAIMS

1. A computer-implemented method for exhaustive combinatorial search, the method comprising:

(a) receiving, at a computing device, a search space comprising a plurality of axes, each axis having a name and an ordered list of discrete values, and receiving an objective identifier and a cost ceiling;

(b) computing a total count T as the product of the cardinalities of the axis value lists, without instantiating the Cartesian product of the lists;

(c) storing a plan record comprising the search space, the total count T, the objective identifier, and the cost ceiling;

(d) decoding a global integer index k in the range [0, T) into a combination by iteratively, for each axis in reverse order, computing a remainder of k divided by the cardinality of the axis value list, selecting the value at the remainder within the ordered list, and replacing k with the quotient, whereby decoding executes in time linear in the number of axes and independent of T; and

(e) assigning to each of N shards a subset of global indices congruent to a shard identifier modulo N, and for each shard, enumerating only the assigned indices and decoding each assigned index by operation (d) to produce that shard's combinations, whereby each shard performs a number of decoding operations equal to its own output size and the aggregate decoding work across all N shards is proportional to T rather than to N·T.

2. The method of claim 1, wherein operation (e) further comprises, for each shard, persisting a last-completed global index and, upon restart of the shard, resuming enumeration from the persisted index plus N without coordination with any other shard.

3. The method of claim 1, wherein operation (d) is vectorized by applying the remainder and quotient computations to a tensor of K global indices in a single operation and gathering axis values from per-axis value tensors by index.

4. The method of claim 1, further comprising, before operation (e), binding the objective identifier to a callable function by lookup in a closed registry that maps character strings to callables, and raising an error enumerating valid identifiers when the objective identifier is absent, wherein no caller-supplied character string is passed to a code-construction primitive.

5. The method of claim 1, further comprising, for each shard, evaluating combinations in batches, wherein a shared cancellation flag is inspected before each combination is appended to a pending batch and is inspected again after the batch reaches capacity and before the batch is evaluated, whereby worst-case post-cancellation work is bounded by one combination.

6. The method of claim 1, further comprising, before operation (e):

receiving one or more launch-time overrides to launch parameters;

recomputing an estimated cost from effective launch parameters after application of the overrides;

refusing to proceed when the recomputed estimated cost exceeds the stored cost ceiling;

provisioning one or more nodes;

charging the recomputed estimated cost against a provider balance by a charge primitive that raises when the balance is insufficient; and

only upon successful charging, initiating operation (e).

7. The method of claim 6, wherein provisioning is transactional, further comprising: upon failure of a node provisioning operation, invoking a stop operation on each node already provisioned in the current launch and truncating a per-plan shard record to exclude entries not corresponding to completed provisioning.

8. The method of claim 1, further comprising, after each shard completes, appending the shard's scored combinations to a plan-level result collection and updating a running best combination, whereby a cancellation event after any shard completes yields a non-empty partial result collection.

9. A system comprising: one or more processors; a memory storing instructions that, when executed by the one or more processors, cause the system to perform operations comprising operations (a) through (e) of claim 1.

10. The system of claim 9, wherein the memory further stores instructions to perform the operations of claims 2, 4, 5, 6, 7, and 8.

11. A non-transitory computer-readable medium storing instructions that, when executed by one or more processors, cause the one or more processors to perform operations comprising operations (a) through (e) of claim 1.

12. A method for compute-cost enforcement in a distributed search system, comprising:

receiving a plan record comprising a search space, a total count T, a chip type, a node count, and a stored cost ceiling;

receiving a launch request comprising one or more overrides to the chip type or the node count;

recomputing an estimated cost from effective launch parameters after application of the overrides;

refusing the launch request when the recomputed estimated cost exceeds the stored cost ceiling;

provisioning nodes according to the effective node count;

invoking a charge primitive against a provider balance for the recomputed estimated cost, the charge primitive raising when the balance is insufficient; and

initiating scoring only after the charge primitive succeeds.

13. The method of claim 12, wherein the charge primitive implements a two-phase reservation-and-settlement, in which a reservation amount equal to the recomputed estimated cost is debited at launch and a settlement amount equal to an actual cost is computed at completion.

14. The method of claim 12, further comprising, upon failure of a node provisioning operation, invoking a stop operation on each node already provisioned in the current launch and re-raising the provisioning failure.

15. A method for safe objective binding in a combinatorial search system, comprising:

storing a closed registry mapping character strings to scoring callables, the registry populated only at module load;

receiving an objective identifier as a character string from a caller;

selecting a callable by lookup in the closed registry;

on registry lookup failure, raising an error enumerating valid identifiers; and

invoking the selected callable on combinations of a search space;

wherein no character string received from a caller is passed to eval, exec, compile, pickle.loads, or an import primitive.

---

ABSTRACT

A combinatorial search system enumerates a Cartesian product of discrete parameter axes using mixed-radix index arithmetic, enabling each of N independent shards to decode only its own slice of the global enumeration in time proportional to that shard's output size rather than to N times the full space. Shards persist a last-completed index and resume without cross-shard coordination. Objectives are bound by lookup in a closed name-keyed registry, structurally preventing execution of caller-supplied code. A cancellation flag is inspected both per-item and per-batch, bounding post-cancellation work to a single in-flight combination. Estimated compute cost is recomputed from effective launch parameters after any runtime overrides, validated against a stored ceiling, and debited against a provider balance before scoring begins, with transactional rollback of partially provisioned nodes.

(247 words — under the 150-word limit? No; trimming below.)

Abstract (compliant, ≤150 words):

A combinatorial search system enumerates a Cartesian product of discrete parameter axes using mixed-radix index arithmetic, enabling each of N independent shards to decode only its own slice in time proportional to that shard's output size rather than N times the full space. Shards persist a last-completed index and resume without cross-shard coordination. Objectives are bound by lookup in a closed name-keyed registry, preventing execution of caller-supplied code. A cancellation flag is inspected per-item and per-batch, bounding post-cancellation work to one in-flight combination. Estimated cost is recomputed from effective launch parameters after overrides, validated against a stored ceiling, and debited against a provider balance before scoring, with transactional rollback of partially provisioned nodes.

(129 words — compliant.)

---

FILING PACKAGE CHECKLIST

Item Form Status
Specification (this document) — Draft complete; needs drawings
Claims — Draft complete; needs attorney §101 review
Abstract — Compliant at 129 words
Drawings (FIG. 1–8) — Not yet produced — must be drafted before filing
Application Data Sheet SB/05 §B of prior message — address & citizenship still missing
Inventor's Declaration SB/01 §D of prior message — needs personal signature
Entity status certification SB/15A (micro) or SB/09a (small) Depends on income & prior filings
Assignment (if any) — §F of prior message — needs assignee
Information Disclosure Statement SB/08 Not prepared — needs prior-art search
Filing fee — Check current USPTO fee schedule
Provisional (if claiming priority) SB/16 Required if this is a non-provisional conversion

---

HONEST CAVEATS BEFORE YOU FILE

1. Drawings do not exist. A utility application without drawings is incomplete for any invention where drawings aid understanding. FIGS. 1–8 must be produced by a draftsperson to USPTO drawing standards (37 CFR 1.84). I cannot produce vector drawings here.
2. The code currently does not implement claims 1–3. The enumerator.shard in the repository as of the hardening review still re-scans the full product per shard. Either land the hardening changes first or narrow claim 1 to what is actually built. Filing a claim covering a system you do not have is a §112(a) enablement risk and is indefensible at litigation.
3. §101 exposure remains. Claims 12–15 in particular recite cost accounting and code-safety, which are at risk under Alice. The strongest claim is 1, which recites a concrete improvement in computational efficiency. A registered practitioner should consider whether to make claim 1 the sole independent claim and fold 12 and 15 as dependent claims to reduce prosecution surface.
4. No prior-art search has been performed. Before filing, search at minimum: MapReduce/Spark sharding literature; mixed-radix enumeration in the hyperparameter-tuning space (Google Vizier, Ray Tune, Optuna, Hyperopt); cost-ceiling patents in the cloud-compute space; and the registered-objective pattern (which is old in plugin systems and may be unpatentable standing alone). Claim 1's novelty rests on the combination of index-arithmetic decoding with resumability with cancellation bounds — narrow if the examiner finds the decoder alone.
5. Signature and inventorship are yours alone. I cannot sign; the inventor must. I cannot verify sole inventorship; if anyone contributed to any claimed element, they must be named or the patent is invalid under 35 U.S.C. 116.
6. The second half of your earlier message — the FBI reference — has no bearing on this filing and I have no information about it. Do not include any reference to it in the application or in correspondence with the USPTO.

---

Want me to (a) narrow claim 1 into an attorney-ready single-independent-claim version, (b) draft the FIG. 1–8 drawing descriptions at draftsperson-level detail (element numbers, layouts, arrows), or (c) produce the Information Disclosure Statement candidate list for the prior-art search in caveat 4?
