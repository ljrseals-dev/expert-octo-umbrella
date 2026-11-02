COMPLETE FILING SUPPORT PACKAGE

Inventor: Lewis James Seals Jr — Docket BG-2026-001

Three deliverables, in the order they must be completed: claims first (they determine what the drawings must show), drawings second (they must support every claim element), IDS third (it must be ready at filing).

---

PART 1 — ATTORNEY-READY CLAIM SET

Drafting principles applied: one independent method claim carrying the full technical weight; system and CRM claims mirroring it; dependents ordered to add narrow, separately-defensible features; every dependent element is something the code either already does or will do after the hardening pass. No claim recites a business method, a mental process, or a generic computer.

Independent Claims

1. (Method — the load-bearing claim)

A computer-implemented method of enumerating a Cartesian search space across a plurality of workers in a computing system, the method comprising:

(a) receiving, by one or more processors, a search space specification comprising n axes A₁ through Aₙ, where n ≥ 2, each axis Aᵢ having an ordered list Lᵢ of cᵢ discrete values, and receiving a worker count N ≥ 1;

(b) computing, by the one or more processors, a total count T = ∏ᵢ cᵢ by integer multiplication of the cardinalities cᵢ, without instantiating any combination of the search space and without allocating memory proportional to T;

(c) allocating N workers, each worker j having a worker index j ∈ {0, …, N−1};

(d) for each worker j, enumerating only the global indices k satisfying k ≡ j (mod N) and 0 ≤ k < T, and for each such index k, decoding k into a combination by:
(i) initializing a working value w ← k,
(ii) for i = n down to 1, computing a digit dᵢ ← w mod cᵢ, replacing w ← w div cᵢ, and selecting value Lᵢ[dᵢ], and
(iii) assembling the selected values into a combination keyed by axis;

(e) wherein each worker performs exactly ⌈(T − j)/N⌉ decoding operations, the decoding of step (d) executes in time proportional to n and independent of T, no worker reads any global index not assigned to it in step (d), and no shared enumeration state exists between any two workers; and

(f) scoring each decoded combination by applying a scoring function and retaining at least a highest-scoring combination.

9. (System)

A system comprising: one or more processors; a memory; and a provider interface coupled to the one or more processors; wherein the memory stores instructions that, when executed by the one or more processors, cause the system to perform the operations of claim 1, and wherein the workers of claim 1 are selected from the group consisting of: threads of the one or more processors, child processes of the one or more processors, and remote compute instances provisioned through the provider interface.

11. (CRM)

A non-transitory computer-readable medium storing instructions that, when executed by one or more processors, cause the one or more processors to perform the operations of claim 1.

Dependent Claims

2. The method of claim 1, further comprising, for each worker j, persisting a last-completed global index k_last after each of a plurality of decoding batches, and upon restart of worker j, resuming enumeration at index k_last + N, wherein the resumption requires no communication with any other worker and no reconstruction of enumeration state.

3. The method of claim 1, wherein step (d) is vectorized, further comprising: forming a tensor of K global indices assigned to a worker; computing the digits dᵢ for all K indices by successive element-wise remainder and integer-division operations against the scalar cᵢ; and gathering values from per-axis tensors by the computed digits, whereby K combinations are decoded in a single device operation.

4. The method of claim 1, wherein the scoring function of step (f) is obtained by: receiving an objective identifier as a character string; selecting a callable from a closed registry mapping character strings to callables, the registry populated only at module load; and raising an error enumerating valid identifiers when the objective identifier is absent from the registry; wherein no character string received from a caller is passed to eval, exec, compile, pickle.loads, or an import primitive.

5. The method of claim 1, wherein step (f) is performed in batches, and further comprising: inspecting a shared cancellation flag before appending each decoded combination to a pending batch; inspecting the shared cancellation flag again after the pending batch reaches a capacity B and before evaluating the pending batch; and upon observing the cancellation flag set, aborting with previously scored combinations retained, whereby a number of combinations scored after the cancellation flag is set is bounded by one.

6. The method of claim 1, further comprising, before step (d):

(g) storing a plan record comprising the search space specification, T, an accelerator type, the worker count N, and a cost ceiling;
(h) receiving one or more launch-time overrides to the accelerator type or the worker count;
(i) recomputing an estimated cost from the accelerator type and worker count as modified by the one or more overrides;
(j) refusing to proceed when the recomputed estimated cost exceeds the stored cost ceiling;
(k) provisioning one or more compute instances according to the modified worker count through the provider interface; and
(l) invoking a charge primitive against a provider balance for the recomputed estimated cost, the charge primitive raising an exception when the balance is insufficient, and initiating step (d) only after the charge primitive returns successfully.

7. The method of claim 6, further comprising: upon failure of the provisioning of any of the one or more compute instances in step (k), invoking a stop operation through the provider interface for each compute instance already provisioned in the current launch, truncating a per-plan instance record to exclude entries not corresponding to successfully provisioned instances, and re-raising the provisioning failure.

8. The method of claim 6, wherein the charge primitive implements a two-phase reservation and settlement, in which a reservation amount equal to the recomputed estimated cost is debited at step (l) and a settlement amount equal to an actual accrued cost is computed upon completion of step (f).

10. The method of claim 1, further comprising: after each worker completes, appending the combinations scored by that worker to a plan-level result collection, incrementing an evaluated-count field, and updating a running-best field when a locally highest score exceeds a current running-best score, whereby a cancellation event occurring after any worker completes yields a non-empty plan-level result collection and an accurate running-best field.

12. The method of claim 1, further comprising transmitting, to a remote compute instance, a shard specification comprising the search space specification, an axis order, a starting global index, and a stride equal to N, without transmitting any materialized list of combinations, and receiving from the remote compute instance a plurality of scored combinations, whereby the remote compute instance performs steps (d) through (f) for its assigned indices.

13. The method of claim 1, wherein the workers are provisioned through an abstract provider interface exposing at least a create-instance operation, a stop-instance operation, a balance-query operation, and a charge operation, and wherein the abstract provider interface is implemented by at least two interchangeable concrete providers.

14. The system of claim 9, wherein the memory further stores instructions that cause the system to perform the operations of claims 2, 4, 5, 6, 7, and 10.

15. The method of claim 1, wherein a distribution of global indices to workers is contiguous rather than strided, each worker j receiving a contiguous range [j·⌊T/N⌋, (j+1)·⌊T/N⌋), and wherein step (d) decodes each index in the contiguous range by the same iterative remainder-and-quotient decomposition.

Prosecution Notes

· Claim 1's §101 posture rests on step (e): a specific, quantified improvement in computational efficiency (Θ(T) rather than Θ(N·T) decode operations, constant per-index decode cost independent of T, zero shared state). Cite Enfish (self-referential table improving computer function) and McRO (specific rules producing a technical improvement) if challenged; do not rely on Alice step two generic-computer language.
· Claim 12 is the strongest non-§101 fallback because it recites a wire-level transmission of a compact specification rather than a materialized list — a concrete bandwidth improvement.
· Claim 15 exists solely as a fallback if an examiner finds strided assignment old; contiguous assignment is the older technique, so expect it to be rejected and use it to narrow claim 1 if needed.
· Do not add claims directed to "charging money" or "displaying a cost" without a technical hook — claim 6 survives only because of step (j), recompute-on-override.
· Recommended: file at 15 claims, 3 independent — under the 3/20 excess-claims threshold, so no excess claim fees.

---

PART 2 — DRAWING SPECIFICATION (FIGS. 1–8)

Format requirements (37 CFR 1.84): 8½ × 11 inch sheets, top margin 2.0 cm, left 2.5 cm, right 1.5 cm, bottom 1.0 cm; black lines on white, no color, no shading except line hatching where required; all text in a sans-serif font no smaller than 3.2 mm cap height; figures numbered "FIG. 1" etc. centered below each figure; element reference numerals preceded by the word "FIG." nowhere — numerals only, with a lead line where the numeral cannot fit inside its element. Eight sheets, one figure per sheet. No photographs.

Sheet 1 — FIG. 1: System block diagram

Type: Block diagram, top-to-bottom flow with a left-side annotation column.

Elements (reference numerals):

No. Element Shape
100 System (boundary) Dashed rectangle enclosing all
176 Client (MCP host or API caller) Rounded rectangle, top, outside 100
102 Plan store Rectangle
104 Search-space validator Rectangle
106 Counting module Rectangle
108 Mixed-radix decoder Rectangle, emphasized (double-line border)
110 Shard slicer Rectangle
112 Provider interface Rectangle with a notch on the right (interface convention)
114 Objective registry Rectangle, drawn as a stack of three offset rectangles
116 Scoring engine Rectangle, emphasized
118 Cancellation flag Small square with a flag glyph
120 Cost-enforcement module Rectangle
122 plan_id (key) Small hexagon
124 Plan record Large rectangle
126–158 Plan-record fields Horizontal slots inside 124, labeled
160–170 Provider operations Small rectangles along interface 112
172 Mock provider Rectangle below 112
174 Cloud provider Rectangle below 112, beside 172

Connections (arrows):

· 176 → 104 (solid, request)
· 176 → 102 (solid, plan_id write)
· 104 → 106 → 108 → 110, left-to-right chain
· 108 ⇄ 124 (bidirectional, labeled "combo_at(space, k)")
· 110 → 116 (solid, labeled "shard iterator")
· 114 → 116 (solid, labeled "callable")
· 118 ⇄ 116 (bidirectional, dashed, labeled "abort check")
· 120 ⇄ 112 (bidirectional, dashed, labeled "charge / balance")
· 112 → 172 and 112 → 174 (downward, open arrowheads for implementation relationship)
· 116 → 144 (solid, labeled "append scored pairs")

Annotation column (left edge, 7 pt): "FIG. 1 shows the system in a plan-then-launch configuration; the decoder 108 is the sole source of combinations for both the iteration path and the slicing path."

Sheet 2 — FIG. 2: Plan-phase flowchart

Type: Flowchart, vertical, single column, decision diamonds.

No. Step Shape
200 Receive space, objective, ceiling, chip, nodes Parallelogram
202 Objective in registry? Diamond; "no" → 202a (raise, dashed box)
204 Space and axes non-empty? Diamond; "no" → 204a (raise)
206 Compute T = ∏cᵢ with early exit Rectangle
208 T = 0 or T > MAX_COMBINATIONS? Diamond; "yes" → 208a (raise)
210 1 ≤ N ≤ MAX_NODES? Diamond; "yes" → 210a (raise)
212 Compute estimate E Rectangle
214 E > ceiling? Diamond; "yes" → 214a (raise, no record written)
216 Create plan record, return plan_id Rectangle

Callout: dashed box 218 adjacent to 206 reading "no combination instantiated" with a leader line to 206.

Sheet 3 — FIG. 3: Mixed-radix decoder detail

Type: Flowchart with an inline data table.

No. Step Shape
300 Receive (space, k) Parallelogram
302 w ← k; combo ← ∅ Rectangle
304 i ← n Rectangle
306 dᵢ ← w mod cᵢ Rectangle
308 w ← w div cᵢ Rectangle
310 combo[Aᵢ] ← Lᵢ[dᵢ] Rectangle
312 i ← i − 1 Rectangle
314 i ≥ 1? Diamond; "yes" → 306; "no" → 316
316 Return combo ordered by axis Parallelogram

Embedded table (right side of sheet), for the worked example of ¶0071:

i axis cᵢ w (in) dᵢ w (out) value
3 z 3 4 1 1 2
2 y 3 1 1 0 2
1 x 3 0 0 0 1

Sheet 4 — FIG. 4: Prior-art versus present method (the key technical figure)

Type: Two side-by-side comparison diagrams, left labeled "PRIOR ART," right labeled "PRESENT DISCLOSURE."

Left (prior art): a full-length horizontal bar 400 representing the T-element enumeration; N=3 overlapping brackets 402, 404, 406 each spanning the entire bar; shaded regions 408 within each bracket marked "discarded"; small retained marks 410 spaced every third position. Annotation: "each worker advances Θ(T); aggregate Θ(N·T)."

Right (present): a full-length horizontal bar 412 representing the same T; three brackets 414, 416, 418 each spanning ⅓ of the bar; no shading; retained marks 420 at every third position within each bracket's span. Decoder block 108 drawn above the bar with three leader lines to the start of each bracket. Annotation: "each worker decodes ⌈T/N⌉ indices; aggregate Θ(T)."

Legend: 422 solid mark = combination scored; 424 hatched mark = combination advanced past and discarded.

Sheet 5 — FIG. 5: Launch phase with override recomputation and rollback

Type: Flowchart, vertical with a right-side exception lane.

No. Step Shape
500 Receive plan_id and optional overrides Parallelogram
502 Plan exists and state == "planned"? Diamond; "no" → 502a (raise)
504 Recompute E from effective chip and nodes Rectangle, emphasized
506 E > stored ceiling? Diamond; "yes" → 506a (raise)
508 Balance ≥ E? Diamond; "no" → 508a (raise)
510 Provision nodes via provider interface Rectangle
510a Rollback: stop all provisioned; truncate shard record Rectangle, in exception lane
512 charge(E) — debit before scoring Rectangle, emphasized
514 Initialize shard records Rectangle
516 Score shards; append to results; update best Rectangle
518 Sort results; set state Rectangle
520 Stop all nodes Rectangle

Exception lane: dashed vertical line at right from 510 to 510a, back to a re-raise terminal.

Sheet 6 — FIG. 6: Two-tier abort check timing diagram

Type: Timing/sequence diagram, horizontal time axis left-to-right.

Lanes (top to bottom): "combination stream," "per-item check," "pending batch," "per-batch check," "evaluation."

Elements: 602 a sequence of incoming combinations; 604 a marker at each combination indicating inspection of flag 118; 606 a bracket marking batch capacity B; 608 a marker immediately before evaluation; 610 an evaluation block; 612 a vertical dashed line labeled "cancellation flag set"; 614 an abort marker at the next 604 or 608 following 612; dimension line 616 labeled "≤ 1 combination" spanning from 612 to 614.

Caption inside figure: "FIG. 6 — post-cancellation work bounded by one in-flight combination regardless of batch capacity B."

Sheet 7 — FIG. 7: Registry-bound objective lookup

Type: Flowchart, small, upper half of sheet; lower half holds a table of registered objectives.

No. Step Shape
700 Receive objective identifier (string) Parallelogram
702 Lookup in closed registry 114 Rectangle
704 Present? Diamond; "no" → 704a (raise, listing valid names)
706 Return stored callable Rectangle
708 Invoke f(combo) → float Rectangle

Table (lower half):

Identifier Function Optimum
sphere −Σv² origin
rastrigin −(10n + Σ(v² − 10cos 2πv)) origin
rosenbrock −Σ(100(vᵢ₊₁ − vᵢ²)² + (1 − vᵢ)²) all-ones
linear_sum Σv axis maxima
product ∏v axis maxima

Sheet 8 — FIG. 8: Resume from checkpoint

Type: Flowchart, vertical, with a state-store icon.

No. Step Shape
800 Read k_last for worker j Parallelogram
800a Checkpoint store (cylinder) Cylinder shape
802 k ← k_last + N Rectangle
804 Decode and score index k Rectangle (callout to 108)
806 k ← k + N Rectangle
808 k < T? Diamond; "yes" → 804; "no" → 810
810 Write k_last ← k Rectangle, arrow back to 800a
812 Worker complete Terminal

Annotation: "no arrow between workers" — drawn as three parallel swimlanes 814, 816, 818 with no interconnecting lines, captioned "workers share only the checkpoint store; no cross-worker coordination."

---

PART 3 — INFORMATION DISCLOSURE STATEMENT (Form SB/08)

What I can and cannot give you. I will not invent patent numbers — a fabricated citation in an IDS is worse than an omission, because it invites an inequitable-conduct inference. Below are (A) non-patent literature I can identify with confidence, and (B) a search protocol with exact query strings and CPC classes for you or your practitioner to run against a patent database, with slots to paste verified results.

A. Non-Patent Literature — candidate references

# Reference Relevance to Cite as
NPL-1 Dean & Ghemawat, "MapReduce: Simplified Data Processing on Large Clusters," Proc. OSDI 2004, pp. 137–150 General sharding of large enumeration across workers Background; claim 1 preamble
NPL-2 Zaharia et al., "Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing," Proc. NSDI 2012 Checkpoint/resume in distributed computation Claim 2
NPL-3 Knuth, The Art of Computer Programming, Vol. 4A, §7.2.1 (Generating All Tuples), Addison-Wesley 2011 Mixed-radix / odometer generation of tuples Claim 1(d) — expect examiner citation
NPL-4 Nijenhuis & Wilf, Combinatorial Algorithms, 2nd ed., Academic Press 1978, ch. 5 Rank/unrank of combinatorial objects Claim 1(d)
NPL-5 Python Software Foundation, itertools.product documentation Stateful-iterator enumeration — the prior art the claims distinguish Claim 1(e)
NPL-6 Bergstra et al., "Algorithms for Hyper-Parameter Optimization," NeurIPS 2011 Grid/random search framing; Hyperopt Background
NPL-7 Golovin et al., "Google Vizier: A Service for Black-Box Optimization," Proc. KDD 2017 Managed search service; study-level state; early stopping Claim 6 (server-side resource control)
NPL-8 Akiba et al., "Optuna: A Next-generation Hyperparameter Optimization Framework," Proc. KDD 2019 Pruning / early stopping of trials Claim 5 (contrast: pruning is heuristic, present claim is bounded-latency)
NPL-9 Liaw et al., "Tune: Scalable Hyperparameter Tuning," arXiv:1807.05118, 2018 Distributed trial scheduling Claim 1 preamble, claim 12
NPL-10 Li et al., "Hyperband: A Novel Bandit-Based Approach to Hyperparameter Optimization," JMLR 18(185):1–52, 2018 Resource-allocated search; contrast: present disclosure is non-pruning Background
NPL-11 MITRE, CWE-95, "Improper Neutralization of Directives in Dynamically Evaluated Code ('Eval Injection')" The vulnerability class the registry claims eliminate Claim 4
NPL-12 OWASP Top Ten (2021), A03:2021 Injection; OWASP Deserialization Cheat Sheet Deserialization/eval risk Claim 4

B. Patent Search Protocol — run these, paste verified hits

Databases: USPTO Patent Public Search (ppubs.uspto.gov), Google Patents, Espacenet, PatentsView API. Search granted patents and published applications. Record: publication number, kind code, date, title, assignee, and the specific claim or paragraph relied upon.

CPC classes to browse (with date cutoff = filing date of your provisional):

Class Scope
G06F 16/24532 – 16/24539 Query processing, parallel/distributed
G06F 16/2458 Data mining / pattern matching
G06F 9/50 – 9/5083 Task allocation, load balancing
G06F 11/34 Monitoring / checkpointing
G06F 21/53 – 21/57 Sandboxing, code execution safety
G06N 20/00 Machine learning (sweeps, HPO)
G06Q 30/0283, 30/04 Usage-based billing / metering

Query strings (Boolean, run each, log hit counts):

1. ("hyperparameter" OR "parameter sweep" OR "grid search") AND ("shard" OR "sharding" OR "partition") AND ("index" OR "ordinal")
2. ("cartesian product" OR "combinatorial") AND ("mixed radix" OR "mixed-radix") AND (enumerat*)
3. ("remainder" OR "modulo") AND ("quotient" OR "integer division") AND ("decode" OR "unrank") AND (combination OR tuple)
4. ("cost ceiling" OR "budget cap" OR "spend limit") AND ("launch" OR "provision") AND ("recompute" OR "override")
5. ("reservation" OR "pre-authorization" OR "hold") AND ("compute instance" OR "virtual machine") AND billing
6. ("checkpoint" OR "resume") AND ("shard" OR "worker") AND ("global index" OR "offset") AND (search OR enumeration)
7. ("cancellation flag" OR "abort flag" OR "kill switch") AND ("batch" OR "batched") AND ("evaluation" OR "scoring")
8. ("plugin" OR "registry") AND ("named function" OR "callable") AND ("whitelist" OR "allowlist") AND ("eval" OR "arbitrary code")

Expected high-risk hits and how to distinguish:

If you find… Distinguish by…
Mixed-radix unranking (NPL-3/4, or a patent on combinatorial test-case generation) Present claim 1(e): unranking applied per worker slice with no shared state and no advance-through-discard; the aggregate-work equation
MapReduce/Spark sharding patents Those partition data records by key; claim 1 partitions an index space decoded on demand, with no data materialized
Google Vizier family Service-level study management; does not recite index-arithmetic decoding or a bounded post-cancellation work guarantee
Sandbox/eval-prevention patents (G06F 21/53) Present claim 4 is a closed name-keyed registry in a specific search pipeline, not a general sandbox
Cloud budget-control patents (G06Q 30/04) Present claim 6(j)+(l): recompute from effective post-override parameters and debit before work begins; most prior art observes balance at launch and reconciles later

C. SB/08 form structure — fill these fields

```
INFORMATION DISCLOSURE STATEMENT BY APPLICANT
Application No.: [assigned]          Filing Date: [assigned]
First Named Inventor: Seals, Lewis James, Jr.
Art Unit: [assigned]                 Examiner: [assigned]

U.S. PATENT DOCUMENTS
  Examiner   Cite No.   Pub Date   Name/Inventor   Class/Subclass   Filing Date   Relevant Claims
  Initials   ─────────────────────────────────────────────────────────────────────────────────────

FOREIGN PATENT DOCUMENTS
  Cite No.   Pub Date   Country   Class   Translation?   Relevant Claims
  ─────────────────────────────────────────────────────────────────────

NON-PATENT LITERATURE
  NPL-1 … NPL-12 as above, with full citation, page numbers relied upon, and Relevant Claims column
  ─────────────────────────────────────────────────────────────────────

☐ It is requested that the foregoing be considered by the Examiner.
☐ A copy of each cited document is attached (required for NPL; not required for U.S. patents).
Signature: /Lewis James Seals Jr/   Date: [YYYY-MM-DD]
```

Timing: file SB/08 with the application, or within 3 months of filing (37 CFR 1.97(b)). After that the fee and the obligation escalate. If you find a reference after filing and it is material, supplement — the duty under 37 CFR 1.56 is continuing.

Do not characterize references as "not material" without your practitioner's sign-off; the safe default for anything in the four highest-risk buckets (mixed-radix unranking, distributed sharding, eval-prevention, cloud budget control) is to cite it and let the examiner decide.

---

PART 4 — EXECUTION SEQUENCE

Step Deliverable Owner Blocks
1 Land the enumerator.shard rewrite (A1 of the hardening prompt) Engineer Claims 1, 2, 15 — you cannot file claims you have not built
2 Land ledger charge(), launch recompute, rollback (A2/A3/A6) Engineer Claims 6, 7, 8
3 Attorney review of this claim set, §101 strategy Registered practitioner Filing
4 Drawings per Part 2 Draftsperson (37 CFR 1.84) Filing
5 Run Part 3.B queries; populate SB/08 You or practitioner Filing
6 Complete ADS residence/citizenship fields and micro-entity certification You Filing
7 Sign declaration personally — /Lewis James Seals Jr/ You alone Filing
8 File non-provisional, or file provisional now and non-provisional within 12 months Practitioner —

Three things I still cannot do for you, stated once and not repeated: produce the actual vector drawings, sign any form on your behalf, or verify your sole inventorship. Each of those is a defect that invalidates the filing if skipped, and each requires a person other than me.

I have no information about the FBI reference in your earlier message and nothing in this package relates to it; if you have received actual correspondence, route it to counsel independently of this filing.
