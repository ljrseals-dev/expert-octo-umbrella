Brute Genius — Patent Paperwork

First, a necessary clarification on the second half of your message: I have no record, documentation, or knowledge of any FBI inquiry into Brute Genius. I cannot confirm, deny, draft, or role-play anything related to a government investigation. If you've received actual correspondence, contact counsel before responding. The rest of this response is patent drafting only.

Also: I am not a patent attorney. This is a working draft for you to hand to a registered patent practitioner. Software patents face §101 (Alice) subject-matter risk; the claims below are drafted to lean on technical improvements (index arithmetic, abort-latency bounds, ledger enforcement) rather than "do X on a computer."

---

PROVISIONAL PATENT APPLICATION (DRAFT)

Title: System and Method for Exhaustive Combinatorial Search with Index-Arithmetic Sharding, Registry-Bound Objective Evaluation, and Pre-Execution Cost Enforcement

Inventor(s): [TO BE COMPLETED]

Filing entity: [Micro / Small / Large]

---

1. Cross-Reference to Related Applications

None.

2. Field of the Invention

The present disclosure relates to distributed combinatorial search systems, and more particularly to systems and methods for enumerating a Cartesian product of discrete parameter axes across independent, resumable shards while enforcing compute-cost ceilings and preventing execution of caller-supplied code.

3. Background

Exhaustive search over a discrete parameter grid is used in hyperparameter tuning, hardware configuration sweeps, and simulation studies. Existing approaches suffer from three problems:

1. Redundant enumeration. Naive sharding (e.g., index % n_shards == shard_id) re-walks the entire product once per shard, so total iteration cost scales as n_shards × |space| rather than |space|. At large spaces (≥10⁹ combinations) this makes multi-shard execution slower than single-threaded execution.
2. Unsafe objective binding. Allowing callers to supply scoring code as strings invites arbitrary code execution.
3. Post-hoc cost accounting. Cost ceilings are typically checked before execution but not re-validated when execution parameters are overridden, and balance is not debited before work begins, allowing budget overruns.

4. Summary

The disclosed system provides:

· Index-arithmetic sharding in which a mixed-radix decoder maps a global integer index directly to a combination, so that shard i yields exactly its slice of the global enumeration in O(|space| / n_shards) time without re-scanning prior shards.
· Registry-bound objective evaluation in which scoring functions are selected by name from a fixed registry and are never constructed from caller-supplied text.
· Two-tier abort checking in which a cancellation flag is inspected both per combination and immediately before each batch evaluation, bounding post-cancellation work to a single in-flight combination.
· Pre-execution ledger enforcement in which an estimated cost is recomputed from effective launch parameters (including runtime overrides), validated against a stored ceiling, and debited against a provider balance before scoring begins.

5. Brief Description of the Drawings

· FIG. 1 — Flow diagram of plan-phase: space validation, combination counting, ceiling check, plan record creation.
· FIG. 2 — Mixed-radix index decoder mapping global index → combination.
· FIG. 3 — Shard slice enumeration by stride.
· FIG. 4 — Launch-phase flowchart including override recomputation, balance charge, provisioning rollback.
· FIG. 5 — Two-tier abort check timing diagram.

6. Detailed Description

6.1 Mixed-radix index decoding

Given axes A₁, A₂, …, Aₙ with cardinalities c₁, c₂, …, cₙ, the global index k ∈ [0, ∏cᵢ) maps to a combination by successive division:

```
for i from n down to 1:
    d[i] = k mod c[i]
    k = k div c[i]
```

This yields a combination in O(n) time independent of space size, decoupling enumeration cost from decode cost.

6.2 Strided shard slicing

Shard j of N enumerates indices {j, j+N, j+2N, …} < total. Because decoding is direct, each shard performs ceil((total - j)/N) decodes and no work on non-owned indices. Total work across all shards is total, not N × total.

6.3 Registry binding

Objectives are stored in a dictionary {name → callable}. Lookup raises KeyError with the list of valid names on miss. No path exists from caller input to eval, exec, compile, pickle.loads, or importlib.

6.4 Abort latency bounding

The scoring loop inspects a shared cancellation flag:

· before appending each combination to the pending batch (per-item), and
· after the batch fills but before the batch is evaluated (per-batch).

Worst-case post-cancellation work is therefore one in-flight combination, not one in-flight batch.

6.5 Cost enforcement

At plan time, estimated cost is E = (T / (N × r)) / 60 × p × N where T = total combinations, N = node count, r = per-node throughput, p = hourly rate. The plan is refused if E > ceiling. At launch time, overrides to chip or node count trigger recomputation of E from effective parameters and re-validation against the stored ceiling. On success, E is debited from the provider balance via a charge() method that raises when insufficient.

6.6 Independent resumability

Because shard j is defined purely by index arithmetic, a shard can persist its last-completed index k_last and resume from k_last + N with no coordination with other shards or the orchestrator.

7. Claims (for conversion to non-provisional)

1. A computer-implemented method for exhaustive combinatorial search, comprising:
   (a) receiving a search space comprising a plurality of named axes, each axis having an ordered list of discrete values;
   (b) computing a total count as the product of axis cardinalities without materializing the Cartesian product;
   (c) decoding a global integer index into a combination by successive modular division against said cardinalities; and
   (d) assigning to each of N shards the subset of global indices congruent to the shard identifier modulo N, whereby each shard performs a number of decoding operations proportional to its own output size and independent of N.
2. The method of claim 1, wherein the decoding step (c) executes in time linear in the number of axes and independent of the total count.
3. The method of claim 1, further comprising, for each shard, persisting a last-completed index and resuming enumeration from said index plus N upon restart, without communication with other shards.
4. A method for safe objective binding in a combinatorial search system, comprising: receiving an objective identifier as a character string; selecting a callable from a closed registry keyed by said string; and refusing execution when said string is absent from the registry, wherein no caller-supplied text is compiled or evaluated.
5. A method for bounded-latency cancellation in a batched scoring loop, comprising: inspecting a shared cancellation flag before adding each item to a pending batch; inspecting said flag again after the batch reaches capacity and before batch evaluation; and aborting with partial results preserved, whereby post-cancellation work is bounded by the size of a single in-flight item.
6. A method for compute-cost enforcement, comprising: storing a cost ceiling in a plan record at plan-creation time; recomputing an estimated cost at launch time from effective launch parameters including any runtime overrides; refusing launch when the recomputed estimate exceeds the stored ceiling; and, upon acceptance, debiting the estimate from a provider balance before scoring begins.
7. A system comprising one or more processors and memory storing instructions that, when executed, perform the methods of claims 1–6.
8. A non-transitory computer-readable medium storing instructions that, when executed, perform the methods of claims 1–6.

8. Abstract

A combinatorial search system enumerates a Cartesian product of discrete axes using mixed-radix index arithmetic, enabling independent shards to decode only their own slice of the global enumeration in time proportional to their output rather than to the full space. Objectives are bound by registry lookup to prevent execution of caller-supplied code. A shared cancellation flag is inspected both per-item and per-batch to bound abort latency. Estimated compute cost is recomputed from effective launch parameters, validated against a stored ceiling, and debited from a provider balance before scoring begins.

---

Filing checklist

☐ Confirm inventorship (everyone who contributed to a claim)
☐ Prior-art search — start with: MapReduce sharding literature, mixed-radix enumeration, hyperparameter sweep patents (Google Vizier family), Ray Tune, Optuna
☐ Decide provisional vs. non-provisional. Provisional buys 12 months of priority for ~$60–$300 USPTO fee; claims above are non-provisional-grade
☐ Consider defensive publication if budget is tight — the index-sharding and two-tier-abort techniques are narrow enough that publication may be cheaper than prosecution
☐ Do not publish the README or repo publicly before filing if you intend to pursue non-US protection (absolute-novelty jurisdictions)

Caveats

· §101 risk is real. Claims 4–6 in particular may draw Alice rejections; the index-arithmetic claims (1–3) are the strongest because they recite a concrete technical improvement (linear vs. quadratic work).
· Claim 6's charge step is arguably routine accounting. Tie it to the recompute-on-override step to survive obviousness.
· The current code does not implement claims 1–3 as-written (the shard rewrite in the prior hardening prompt does). File only after that lands, or the patent covers a system you don't have.
· The FBI portion of your message: I have nothing to add. Consult counsel if you've received real correspondence.

Hand this to a registered practitioner (USPTO registration required to prosecute). Want me to draft the FIG. 1–5 flow descriptions in more detail, or narrow claim 1 for a stronger §101 posture?
