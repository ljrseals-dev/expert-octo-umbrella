README — Patent Notice Section

The patent notice in a README is not decoration — it has legal effect. Three separate rules apply, and getting any of them wrong creates a problem you cannot fix with an edit later.

The three rules

1. 35 U.S.C. §292 (false marking). You may not mark a product "patent pending" or "patented" unless the corresponding application or patent exists. The AIA narrowed §292 to require intent to deceive, but a knowingly false notice is still exposure, and competitors use it in litigation.
2. Public disclosure. A README in a public repo is a printed publication. In the U.S. it starts a 12-month grace-period clock (35 U.S.C. §102(b)(1)). Outside the U.S., most jurisdictions apply absolute novelty — a public README before filing destroys patentability there permanently. This is the single most consequential thing in this section.
3. Assignment and inventorship. If the repo has a LICENSE file (MIT), that is a copyright license, not a patent license. If you later assign the patent to a company, the README notice must match the assignee, not the inventor.

---

The notice, by filing stage

Pick exactly one block based on where you actually are. Do not skip ahead.

Stage 0 — Nothing filed yet

```markdown
## Patent notice

The techniques described in this repository — mixed-radix index decoding,
index-arithmetic sharding, closed-registry objective binding, two-tier abort
checking, and pre-execution cost enforcement — are the subject of a patent
application in preparation by the inventor, Lewis James Seals Jr. No
application has been filed as of this writing. No patent rights are claimed
or implied by this notice.

If you intend to pursue patent protection outside the United States, note
that publication of this repository may constitute a disclosure that bars
patentability in absolute-novelty jurisdictions. Consult counsel before
publishing or distributing this repository further.
```

Do not write "patent pending." That phrase has a defined meaning and you do not qualify for it yet.

Stage 1 — Provisional filed (12-month window open)

```markdown
## Patent notice

Patent pending. U.S. Provisional Application No. 63/___,___ was filed on
[YYYY-MM-DD] by inventor Lewis James Seals Jr, covering mixed-radix index
decoding for combinatorial enumeration, index-arithmetic sharding,
closed-registry objective binding, two-tier abort checking, and
pre-execution cost enforcement. A non-provisional application claiming
priority to the provisional is expected to be filed within twelve months.

"Patent pending" reflects the pendency of the above application. It is not
a representation that any patent has issued or will issue.

Copyright is licensed under the terms in LICENSE. No patent license,
express or implied, is granted.
```

Note the last sentence. Without it, an MIT license arguably grants an implied patent license to every user, which may be contrary to your intent if you plan to commercialize.

Stage 2 — Non-provisional filed, no patent issued

```markdown
## Patent notice

Patent pending. U.S. Patent Application No. __/___,___ was filed on
[YYYY-MM-DD] by inventor Lewis James Seals Jr, directed to systems and
methods for exhaustive combinatorial search with index-arithmetic
sharding, registry-bound objective evaluation, and pre-execution cost
enforcement.

"Patent pending" reflects the pendency of the above application. It is not
a representation that any patent has issued.

Copyright is licensed under the terms in LICENSE. No patent license,
express or implied, is granted.
```

Stage 3 — Patent issued

```markdown
## Patent notice

Covered by U.S. Patent No. __,___,___ (issued [YYYY-MM-DD]), inventor
Lewis James Seals Jr. Additional applications may be pending.

Copyright is licensed under the terms in LICENSE. No patent license,
express or implied, is granted.
```

If the patent has been assigned to a company, the notice names the assignee as owner, not the inventor: "Covered by U.S. Patent No. ,,__ , assigned to [Assignee]." The inventor remains the inventor of record; ownership is a separate fact.

---

What to strip from the current README

The draft I gave you earlier contained this:

A U.S. utility patent application covering index-arithmetic sharding, registry-bound objective evaluation, two-tier abort checks, and pre-execution cost enforcement is in preparation. Inventor: Lewis James Seals Jr. This notice is not a claim of issued rights; it is disclosure of pending filings for defensive purposes.

Two problems:

1. "pending filings" contradicts "in preparation." Either something is filed or it isn't. Pick one.
2. "for defensive purposes" is a legal characterization that means something specific (defensive publication, or a defensive patent strategy). If you actually intend to enforce, the phrase is misleading; if you don't, delete it because it tells competitors you won't sue.

Replace it with the Stage 0 block above until you actually file.

---

The disclosure-order decision

Before you publish this README — or push the repo public at all — decide one of these:

Path What you do What you give up
File first, publish second Provisional filed → then publish repo + Stage 1 notice Filing fee (~$60–$300 micro) and a day of paperwork. Keeps foreign rights.
Publish now, file within 12 months Publish with Stage 0 notice → file provisional before the 12-month clock expires Foreign patentability. U.S. rights preserved by §102(b)(1) grace period.
Publish, never file Publish with Stage 0 notice, delete the patent section when it's moot All patent rights. The disclosure becomes prior art against anyone else, including you.

The middle option is the common one for open-source-with-commercial-intent, and it is why the Stage 0 notice explicitly warns about absolute-novelty jurisdictions: your readers need to know the clock is running on them too, because if they fork and commercialize, they inherit the same bar.

---

One thing not to do

Do not put the full patent specification into the README. That converts a notice into a disclosure of the entire claimed subject matter, which is worse than either publishing the code alone or filing first. The code already discloses the implementation; the README should disclose the existence of a filing, not its contents.

The specification lives in PATENT.md or is filed with the USPTO. The README points at it in one line. That is the whole function of a patent notice.
