# FGC Reductions Explorer: design specification

Status: implemented (v1). This document is the contract for the code in `src/`; when they disagree, it is a bug in one of them.
Principles: *safe from bugs* (SFB), *easy to understand* (ETU), *ready for change* (RFC), as in MIT 6.102.

---

## 1. Purpose and scope

The explorer loads the fine-grained-complexity reductions sheet (`FGC_Reductions_filled.csv`), lets you set **hypotheses**
(lower-bound exponents, e.g. SETH, 3SUM, APSP) and **algorithms** (upper-bound exponents or runtime formulas), and shows
what follows for every problem reachable through the reductions, with a derivation for every number.

**Goals**

| # | Goal | Principle |
|---|------|-----------|
| G1 | Propagate lower bounds forward (hardness: From → To) and upper bounds backward (algorithms: To → From). | correctness |
| G2 | Interactive: moving a hypothesis slider recomputes the whole graph in a fraction of a second; changing algorithms or constants in about a second. | usability |
| G3 | The sheet is the source of truth. Editing or adding rows needs no code change; only machine-only details that the sheet states in prose (e.g. "free integer parameter $k \ge 2$") go into a small JSON config. | RFC |
| G4 | Every derived exponent has a derivation tree back to the knobs it depends on (in the inspector, and drawn in the graph). | ETU, SFB |
| G5 | One self-contained HTML file that works offline. | usability |

**Non-goals (v1)**

- Problems with several time functions (preprocessing/update/query, `T_p, T_u, T_q, T_r`) are shown in the graph but not
  propagated. §11 sketches how to add them.
- Exponents are numbers computed per regime, not closed-form expressions in the free parameters.
- The tool is not a proof checker. It trusts the sheet's From-time and Reduction Runtime cells.

---

## 2. Semantics

### 2.1 Reductions

A row with From problem $P$ and To problem $Q$ states: for every algorithm for $Q$ with running time $T(q_1,\dots,q_k)$
(the To-time signature) there is an algorithm for $P$ with running time

$$O\big(F(p;\,T) + RT(p)\big),$$

where $F$ is the **From-time** cell (an expression in $P$'s parameters $p$ containing calls of $T$ whose arguments are the
produced instances' parameters) and $RT$ is the **Reduction Runtime** cell. Free parameters of the reduction
(block sizes, number of vector sets, ...) may be chosen as functions of $p$.

### 2.2 Scale, growths and exponents

All quantities are functions of a scale $N \to \infty$. Their asymptotic class is a **growth** (§5), e.g.
$N^{2}$, $\Theta(\log N)$, $\Theta(1)$. The **exponent** of a growth is the $e$ with value $N^{e \pm o(1)}$. Everything
the tool reports is an exponent, i.e. it ignores $N^{o(1)}$ factors (polylogs, $2^{O(\sqrt{\log n})}$, ...), at a
**power**, which is 1 unless a knob says otherwise:

**Rates.** A running time is given by its **rate** $(a, c)$: power $a$ and exponent $c$, the time
$2^{(c \pm o(1)) \cdot \ell^{a}}$ with $\ell = \log_2 N$. Power 1 is $N^{c \pm o(1)}$. Other powers describe times
between those classes: at CNF-SAT's exponential scale $N = 2^n$ (§2.3) the time is $2^{c \cdot n^a}$, subexponential for
$a < 1$ (e.g. $2^{n^{0.99}}$ or $2^{0.5\,n^{0.75}}$, as in ETH-style hypotheses), and at a polynomial scale
$2^{c \log^a n}$ is quasi-polynomial for $a > 1$ ($n^{\log n} = 2^{\log^2 n}$) and sub-polynomial for $a < 1$. Rates are
ordered **power first**: every $2^{c\,n^{0.99}}$ is below every $2^{c' n}$ with $c' > 0$. At power $a$, factors
$2^{o(\ell^a)}$ are invisible (polylogarithmic ones always are). The **rate zero** (power 0) stands for times below
every $2^{\ell^a}$, $a > 0$, such as constant and polylogarithmic ones (shown as polylog$(n)$, or poly$(n)$ at an
exponential scale).

### 2.3 Regimes

A **regime** of a problem assigns a growth to each parameter of its signature, e.g. for APSP $T(n, m, M)$ the dense
regime $n = N$, $m = N^2$, $M = N^{O(1)}$. Regimes are **normalized** so that they can be compared:

1. The **normalizer** is the first parameter in signature order that is not log-dependent (§2.6) and has a positive
   polynomial exponent; all exponents are divided by its exponent, so it becomes exactly $N$.
2. If there is none, the first parameter of class $\Theta(\log N)$ with a known coefficient is used and scaled to
   $\log_2 N$, i.e. $N = 2^{\text{param}}$ (this is how CNF-SAT's $2^{cn}$ time becomes exponent $c$).
3. If there is none either, the instance has sub-polynomial size and is called **degenerate** (§6.2).

A normalized regime has a canonical **key** (§6.7), e.g. `n=p1|m=p2|M=P`. Two regimes with the same key are the same
regime for all purposes.

**Limits of parameters.** A regime stands for the instances whose parameters grow *at most* as fast as it says (which is
why rounding exponents up is sound, §6.7). Some parameters cannot grow independently of others: a simple graph has at
most $n^2$ edges; its arboricity $\alpha$ is $O(\sqrt m)$ and at least $m/(n-1)$ (Nash-Williams), and it has
$O(m\alpha)$ triangles (Chiba–Nishizeki), hence at most $n^3$. The config lists such **limits** per problem (§3.3,
`constraints`; the list and its reasons in §3.7), and every regime is **clipped** to them when it is formed, before
it is normalized (`Regime.normalize`, `clipToLimits` in `src/model/constraints.ts`): each parameter is lowered to the
smallest of its limits, again until nothing changes (lowering $m$ to $n\alpha$ lowers the limit $m\alpha$ of $t$).
Every instance satisfies the limits, so the clipped regime names *the same instances*, and derivations stay sound
without any other change. It matters where a reduction's formula claims impossible instances, typically for values of
a free parameter outside the range where the construction works (row R64 for hash ranges $R < \sqrt n$; row R72 for
$3\delta > \beta$, where it promises more output than the queries can have): such calls are read at the clipped
regime, where the instances produced are. Specs (configured, pinned, knob regimes) are clipped the same way, and the
UI says so where it happens; a spec whose *measuring* parameter (the normalizer) the limits would lower, say
`n=polylog, m=1` for a graph, is rejected ("no instances grow like this"), since exponents given with it (a knob's) would
then be read against another scale. The regime filter of a formula knob (`alpha=1`) matches a regime also where the
limits clip the sizes it names to the regime's (`Regime.satisfiesClipped`: α = n matches α = n^{3/4} at $m = n^{1.5}$,
the same graphs); a reduction's `requires` stays exact, since it describes the instances the reduction needs (R106's
$\Theta(\sqrt n)$-regular graphs). On the sheet, Triangle Listing had 179 regimes, of which 132 broke a limit (37 had
$t > n^3$, 65 $\alpha > \sqrt m$, 107 $t > m\alpha$), and Set Intersection 183, of which 129 had more output than
$q \cdot s$; they now have 73 and 88, all within the limits, and no bound shown changed (Triangle Listing's primary
regime was $\alpha = n$ at $m = n^{1.5}$; it is $\alpha = n^{3/4}$, still tight at $n^2$). Limits on log-dependent
parameters are not allowed (they count only through $\log M$).

**Bounds between a problem's own regimes** (`Regime.contains`; `throughContainment` in `src/engine/analysis.ts`). For
the same reason, a regime $A$ lies inside a regime $B$ of the same problem when both are measured by the same parameter
on the same scale and every parameter grows at most as fast in $A$ as in $B$. Per parameter: a constant with a value is
below an unspecified constant (`c?`), which is below the polylogarithmic class (`L`), below $N^{o(1)}$ (`S`), below
$N^x$, ordered by $x$; for log-dependent parameters `C` < `L` < `P`. Two different constant values are not compared,
since not every parameter is a size (a bound for $k = 3$ says nothing about $k = 4$). $A$'s instances are then among
$B$'s, so a lower bound of $A$ holds in $B$ and an upper bound of $B$ holds in $A$: each regime of interest (§6.7) of a
static problem takes the best lower bound of its regimes of interest inside it and the best upper bound of those around
it, and its status is computed from them (the inspector marks such a bound with † and names the regime it comes from).
So Diameter 2 vs 3 at $m = n^2$, which had only its $n^{\omega}$ upper bound, takes the $n^{2-o(1)}$ of sparse graphs
($m = n$) and shows a gap, and Local Alignment at $\sigma = n^{0.5}$ and Edit Distance at an unspecified constant
alphabet take the quadratic lower bounds of their constant alphabets and are tight; on the sheet with the initial
settings 70 of the 556 listed regimes take a bound from another one. Rescaled embeddings ($A$'s instances at size $N$ as
instances of $B$ measured by another parameter, at size $N^{1/x}$) are not used. (Found by an independent review: before,
bounds were matched by exact keys only, and a problem's primary regime could move to a weaker regime than its bounds
implied.)

### 2.4 Bounds (facts)

* **LB fact** $(P, \rho, L)$: for every $\varepsilon > 0$, $P$ has no algorithm running in time $N^{L-\varepsilon}$ on
  instances of regime $\rho$; $L = \infty$: no polynomial-time algorithm (only as the limit of a cycle, §6.9, and what
  follows from it). At power $a$: no algorithm in time $2^{(L-\varepsilon)\ell^{a}}$ ($L = \infty$: none in time
  $2^{O(\ell^a)}$).
* **UB fact** $(P, \rho, U)$: $P$ has an algorithm running in time $N^{U+o(1)}$ on instances of regime $\rho$; at power
  $a$: in time $2^{(U+o(1))\ell^{a}}$.

Here $N$ is the normalizer of $\rho$ (or $2^{\text{param}}$ for exponential-scale regimes) and $\ell = \log_2 N$. An LB
fact has the power of the knob it comes from (reductions keep it, §2.5); an UB fact the power of the term that dominates
its derivation (the larger power wins, e.g. a reduction taking $2^{n/2}$ to a problem solved in $2^{\sqrt{\log t}}$).

### 2.5 Propagation rules

Fix a row $P \to Q$, a source regime $\rho$ of $P$ and a choice $\theta$ of the row's free parameters. Evaluating the
arguments of each $T$-call $c$ gives the produced instance's parameter growths; normalizing them gives a target regime
$\rho'_c$ and a scale $s_c$ (the normalizer of $\rho'_c$ is $N^{s_c}$). Define

* $g(x)$ = exponent of $F$ when every $T$-call $c$ is replaced by $N^{s_c\cdot x}$ (degenerate calls by $N^{0}$);
* $R$ = exponent of $RT$.

**Lower bounds (forward).** Given an LB fact $(P, \rho, L)$: if $R < L$ and all non-degenerate calls share one target
regime $\rho'$, then

$$x^* = \inf\{x \ge 0 : g(x) \ge L\}$$

gives the LB fact $(Q, \rho', x^*)$ (kept only if $x^* > 0$).
*Soundness:* if $Q$ had an $N'^{\,x}$-time algorithm with $x < x^*$, then $P$ would be solvable in time
$N^{\max(g(x), R)}$ with $\max(g(x),R) < L$, contradicting the LB fact. $g$ is nondecreasing because $F$ is monotone in
$T$ (checked statically, §6.1).

**Powers.** For an LB fact at power $a$, every call is worth $2^{x\,\ell'^{\,a}}$ on its instance, i.e.
$2^{x (s_c \ell)^{a}}$, and $g_a(x)$ is the exponent of $F$ at power $a$ ($+\infty$ if $F$ then has a larger power, 0 if
a smaller one); $R$ is compared as a rate, and $x^* = \inf\{x \ge 0 : g_a(x) \ge L\}$ gives the LB fact
$(Q, \rho', x^*)$ at power $a$. A call keeps the power and scales the exponent by $s_c^{a}$; no other power can come
out (a call on an instance $N^{s}$ never changes the power). For upper bounds, a call to a target with bound $U$ at
power $a$ is worth $2^{U (s_c \ell)^{a}}$, and the bound is the rate of $\max(F, RT)$. So a lower bound of power
$a < 1$ passes only through reductions that themselves take less: CNF-SAT's split-and-list reductions take $2^{n/2}$
and pass on $2^{c n}$ for $c > 1/2$, but no $2^{c\,n^{0.99}}$; nor does the one to Subset Sum, which takes
$2^{\varepsilon n}$ for some $\varepsilon > 0$ (only a reduction of subexponential cost would).

**Upper bounds (backward).**

$$UB(P,\rho) = \min\Big(\text{knobs},\ \min_{\text{rows } P \to Q,\ \theta}\ \max\big(g_\theta(UB(Q,\rho'_c)),\ R_\theta\big)\Big)$$

where $g_\theta(\cdot)$ substitutes each call's own target UB. *Soundness:* composing the algorithms.

The best LB for $(Q, \rho')$ is the supremum over all derivations, of any length, the best UB the infimum. A supremum
of valid lower bounds is a valid lower bound, and an infimum of valid upper bounds a valid upper bound in the usual
sense of exponents (for every $\varepsilon > 0$ an $N^{U+\varepsilon}$ algorithm, as for $\omega$); this is what makes
the limits of cycles (§6.9) sound.

### 2.6 Modeling assumptions (stated in the UI)

* **Log-dependent parameters.** Parameters such as $M$ (largest absolute value or weight) are assumed to affect $T$ only
  through $\log M$, so only their class (constant / polylog / polynomial) enters the regime key. Which parameters are
  log-dependent is configurable per problem (default: `M`). From-times that use them polynomially (e.g. $M^{1/\Delta}$,
  $2nM$ as a vector length) evaluate a polynomial log-dependent parameter at the fixed representative $N^3$ (relative to
  the regime's normalizer), so that regimes with equal keys are equal and results do not depend on the order of
  computation. A superpolynomial value of a log-dependent parameter makes the evaluation fail.
* **Constant-size parameters** (e.g. $k$ in $k$-OV, alphabet size $\sigma$) may influence the exponent arbitrarily, so
  their values are part of the regime key (`k=c3`). An unknown constant is `c?`.
* **Sub-polynomial, non-constant parameters** ($d = \Theta(\log n)$, $d = \text{polylog}(n)$, ...) are one class `L`.
  So "OV with $d = \Theta(\log n)$" and "$d = \text{polylog}(n)$" are the same regime (the usual convention), and such a
  parameter is evaluated as the largest member of the class, $(\log N)^{O(1)}$ (so $2^{d}$ has no useful bound there).
  Earlier versions kept the coefficient of the first $c \log N$ that reached a regime, so a bound could depend on the
  order of the search and come out too small (found by an independent review of the rates); on the sheet nothing
  changed. Sizes
  that are sub-polynomial but not polylogarithmic ($2^{\sqrt{\log n}}$, $n^{o(1)}$, and polynomial exponents that round
  to 0, §6.7) are the class `S`, evaluated as $N^{o(1)}$ (so a time at a power below 1 never treats them as
  polylogarithmic).
* **Sizes are upper bounds.** Arguments of $T$ are upper bounds on the produced instance's parameters; $T$ is assumed
  nondecreasing. Negative exponents in an argument are clamped to a constant.
* **Unknown reduction runtime.** If the RT cell has no formula, the policy setting decides: *assume negligible* (default;
  the derived fact is flagged) or *block* (no LB derived through that row; UBs treat it as infinite).

---

## 3. Inputs

### 3.1 The sheet (CSV)

RFC-4180 CSV with a header row. Columns are found **by header name** (order does not matter; extra columns are kept and
re-exported unchanged). Required: `From (Problem)`, `From (Variation)`, `To (Problem)`, `To (Variation)`,
`Reduction ID`, `To-time (generic)`, `From-time (generic)`, `Reduction Runtime`. Optional and used for display or
filtering: `Randomized?`, `Model`, `Assumption/Hypothesis`, `Restrictions on to-time`, `References`, `Link`, `Notes`,
`Detailed restriction / derivation notes`.

**Reduction IDs** are `R1`, `R2`, … . A whole number is read with the letter (`12`, `r12` and `R12` are all `R12`), so
sheets numbered before the letters existed keep matching the config's row annotations (§3.3) and the scenario's
disabled rows; other text is kept as written. A repeated ID is renamed (`R12#2`) with a warning.

**Problem identity.** A problem is the pair (family, variation) with surrounding whitespace removed. Variations are matched
case-insensitively (the first spelling seen is displayed). The config's `aliases` map fixes typos (e.g. *Boolean
Satisfiabiltiy* → *Boolean Satisfiability*). The id is `Family / Variation`, or the variation alone when the `Problem`
column is blank (a problem that belongs to no family, e.g. *Tree Edit Distance*; `problemIdOf`): config keys, knobs and
the bounds CSV name it so, the graph draws it without a family box and the inspector without a family line.

**Signatures.** A problem's signature is read from the To-time cells of rows where it is the target, e.g. `$T(n, m, M)$`
→ one function `T` with parameters `n, m, M`. A problem is **static** iff its signature is a single function named `T`.
If its rows disagree, the most frequent signature wins and the disagreement is listed in the Diagnostics view. Problems
that only occur as sources take their signature from the config (`problems.<id>.params`), otherwise from the
identifiers used in their rows' From-time formulas (marked *inferred* in the UI).

### 3.2 Formula extraction from cells

* A cell is text with `$...$` math spans. The **primary formula** of the From-time (resp. RT) cell is its first math
  span. A config override (`rows.<id>.from` / `rows.<id>.rt`) replaces it.
* A formula may consist of **alternatives** separated by `\subseteq`, `\le`, `\leq` or `=`, all upper bounds of the same
  quantity. The engine uses the first alternative that it can evaluate (e.g. a sum over vertices
  $\sum_v T(\sqrt{\deg v})$ fails and falls back to $\subseteq O(n\cdot T(\sqrt n))$).
* **Abbreviations from legends.** Every other math span of the From-time cell of the form `X = e_1 = e_2 = ...`, where `X`
  is a single identifier that is not a parameter of the source problem, defines `X` (first evaluable alternative). Example:
  row R171, "with $N = O(m + k\cdot 2^{n/(2k)})$". Spans that do not parse (e.g. `$n = \#\text{variables}$`) are ignored.
  Config `rows.<id>.defs` adds or overrides abbreviations; config free parameters take precedence over both (a config
  abbreviation with a free parameter's name is ignored with a message).

### 3.3 Engine config (`data/default-config.json`, importable/exportable)

```jsonc
{
  "version": 1,
  "constants": { "omega": { "value": 2.371339, "min": 2, "max": 3, "label": "matrix multiplication exponent",
                            "note": "shown under the slider" } },
  "aliases":   { "Boolean Satisfiabiltiy": "Boolean Satisfiability" },
  "problems": {
    "Boolean Satisfiability / CNF-SAT": {
      "params": ["n", "m"],          // only needed for source-only problems
      "logParams": [],               // default: ["M"] ∩ params
      "timeType": "exp",             // "poly" (default) or "exp": affects degenerate calls (§6.2)
      "regimes": ["n=log, m=polylog"] // regimes of interest (UB is computed there even without LB facts)
    },
    "Graph Triangle Problems / Triangle Listing": {
      "constraints": ["m <= n^2: a simple graph has at most n² edges", "t <= m*alpha: a graph has O(m·α) triangles"]
    }
  },
  "rows": {
    "R1":   { "match": { "from": "Boolean Satisfiability / CNF-SAT", "to": "Orthogonal Vectors / k-OV" },
              "free": [ { "name": "k", "type": "int", "values": [2, 3, 4, 5, 6, 8] } ] },
    "R103": { "free": [ { "name": "t", "type": "size", "base": "n", "min": 0, "max": 1 } ] },
    "R303": { "free": [ { "name": "delta", "type": "real", "min": 0, "max": 1 },
                        { "name": "d", "type": "size", "base": "n", "min": "0", "max": "(1-\\delta)/2" } ] },
    "R316": { "from": "O(c \\cdot T(n, m/t))", "free": [ ... ], "note": "a sound bound for the sheet's sum over color classes" },
    "R106": { "requires": "m=3/2", "lowerOnly": true },
    "R90":  { "skip": "reason shown in the UI" }
  },
  "initialKnobs": [
    { "label": "Satisfiability Hypothesis", "description": "CNF-SAT on $n$ variables and $\\text{poly}(n)$ clauses needs …",
      "knobs": [ { "id": "L1", /* + a scenario knob without id/enabled, §3.4 */ } ] },
    { "label": "OV hypothesis", "enabled": false, "knobs": [ ... ] }
  ]
}
```

`initialKnobs` is the scenario a first visit starts with (and **Restore initial settings** returns to): every knob of
every group, switched on unless its group has `"enabled": false`, with the group's label as its note unless the knob has
its own `note`. Each knob has a **Bound ID** `id`: `L1`, `L2`, … for lower bounds and `U1`, `U2`, … for upper bounds,
numbered separately (the built-in ones are L1–L18 and U1–U442); keep them when editing and give new knobs the next free
one of their kind, since the bounds CSV (§3.5) matches rows to knobs by them. A missing id gets the next free one of its
kind; a repeated or invalid one is reported and replaced the same way; other text is kept. A bare number is a Bound ID of
the versions before the letters, which numbered all knobs together (the initial ones 1–33, the user's after them), and
is converted by number and kind alone, so that a knob gets the same ID wherever it was saved (`legacyBoundId`): a lower
bound n becomes Ln (the initial ones L1–L9), an upper bound 10–33 U1–U24 (the IDs those initial knobs have now) and
one from 34 up U(n + 334) (after the built-in U367 of then; the knobs added since have U368–U441 and L10–L15, and a
saved knob of the user's with one of those IDs moves to another one, see §7); an upper bound below 10 gets a new ID. The
groups say where the starting knobs come from; the UI lists them in the "Initial settings" note (§7), each with a
switch that turns all of its knobs that are in the left panel (the same Bound ID for the same problem) on or off at once,
with one recomputation; the knobs themselves are edited one by one in the left panel. (The first version called
this section `presets`; a config using that name is still read, with a message asking to rename it.) The built-in groups
are the **Satisfiability Hypothesis** (SETH as stated for CNF-SAT with $\text{poly}(n)$ clauses, L1; called "SETH" until
format 11 of the saved data, §7, since a $2^{o(n)}$-time algorithm for $k$-SAT announced in October 2026, a preprint not
yet refereed, refutes ETH and the $k$-SAT form of SETH, though not this CNF form), the 3SUM and APSP hypotheses and the
textbook algorithms (on); **SETH ($k$-SAT form)** (L16, off: for every $\varepsilon > 0$ some $k$ makes $k$-SAT need
$2^{(1-\varepsilon)n}$; the reductions that need a sparsified $k$-CNF formula, with $O(n)$ clauses, start from the
problem $k$-SAT, which reduces to CNF-SAT, R466, so this form implies the CNF form and not conversely; §13), the
**low-dimension OV hypothesis** (L17, off: OV at $d = c \log n$, which $k$-SAT reduces to with sparsification, R467, and
which rows R350–R351 need; it implies the OV hypothesis through R468), the **(min,+)-convolution hypothesis** (L18,
off: $n^{2-o(1)}$, Cygan–Mucha–Węgrzycki–Włodarczyk 2017, which makes the knapsack-type problems quadratic and which the
2026 (min,+)-product algorithms refute, $n^{1.9982}$ through R198) and the **subexponential $k$-SAT algorithm** of
October 2026 (U442, off: $2^{o(n)}$ for every constant $k$; an unrefereed preprint; every row from $k$-SAT points away
from it, so it reaches no other problem); and the OV, Min-Weight $k$-Clique, $k$-Clique and combinatorial BMM hypotheses and the
algorithms of Alman and Vassilevska Williams (arXiv:2610.06783, Oct 2026: 3SUM in $n^{1.9992}$, APSP and (min,+)-product
in $n^{2.9995}$, Exact Triangle in $n^{2.9983}$, their real-valued versions in $n^{1.998}$ / $n^{2.998}$, Mono
Convolution in $n^{1.4999}$, and their Theorem 1, Lopsided All-Edges Sparse Triangle in $n^2/d^{0.063}$ at $d =
n^{1/18}$, U440, which gives Exact Triangle in $n^{3 - 0.063/36} = n^{2.99825}$ through row R453) and of Saha and Ye (arXiv:2610.20297, Oct 2026: a
simpler, faster deterministic (min,+)-product for integer entries of size $n^{O(1)}$ in $\tilde O(n^{2.9963432})$; the
group gives it to Distance Product and Directed Weighted APSP, U368–U369, and the reductions of the sheet carry it to
the rest of the APSP class, among them the problems the paper names: Negative Triangle, Replacement Paths, Second
Shortest Simple Path, Metricity, (min,+)-product verification, 2D Maximum Subarray) (off). The hypotheses of the
problems added with rows R320–R365 are off too: the **OMv conjecture** (OMv needs $n^{3-o(1)}$, L10; it reaches OuMv,
whose other consequences are dynamic problems), the **Hitting Set hypothesis** ($n^{2-o(1)}$ at $d = \mathrm{polylog}(n)$,
L11), the **∀∀∃-OV hypothesis** ($n^{3-o(1)}$, L12) and the **3-uniform hyperclique hypothesis** ($n^{k-o(1)}$ for
$k = 4, 5, 6, 9, 12, 30, 300$, L13; the large $k$ because the bound it gives Klee's measure problem in 3D, through R355,
is $N^{k/(2\lceil k/3 \rceil + 1)}$, which tends to Chan's $N^{1.5}$ only as $k$ grows). So are those of the problems
added with rows R432–R465: the **$k$-AP hypothesis** ($k$-AP Detection needs $N^{2-o(1)}$ for $k = 4, 5, 6$, L14; Fischer
2026 proposes it in place of the 3SUM hypothesis for the geometric problems, and derives their lower bounds from it
through $k$-GeomBase, rows R432–R444) and the **Zero-Weight $k$-Clique hypothesis** ($n^{k-o(1)}$ for $k = 3, \dots, 6$
with weights $n^{O(k)}$, L15; the 2026 Exact Triangle algorithm refutes it for every $k$: at $k = 3$ and 6 through the
folding of row R458, which is tight there, and at $k = 4, 5$ through R470, one $(k-1)$-clique instance per vertex, which
costs a factor $n$ (R469 does the same for Min-Weight $k$-Clique); without it, the 3SUM and APSP hypotheses give
$n^{3-o(1)}$ for every $k$ through Exact Triangle and the row from $k$ to $k + 1$, R464). Two more groups (on) give upper
bounds for most static problems of the sheet (§3.6): **Trivial algorithms** (198 knobs: brute force and textbook
algorithms following from the definitions) and **Best known algorithms** (214 knobs: the fastest algorithms found in the
literature, with sources); **Non-combinatorial algorithms** (off) holds Valiant's CFG parser and the transitive closure
by fast matrix multiplication, which conflict with the combinatorial BMM hypothesis through R126 (Lee's reduction) and
R356 (BMM to transitive closure), as that hypothesis only concerns combinatorial algorithms.

Row annotations are keyed by Reduction ID (in canonical form: a key `"106"` is row R106; two keys for one row are an
error). The optional `match` records the problems the annotation was written for; if
the row with that ID connects other problems (a renumbered or different sheet), the annotation is ignored with a
warning instead of being misapplied. The in-app row editor adds `match` automatically.

`lowerOnly: true` marks a row that holds only on special instances of its target, which its Restrictions describe in
prose: hardness of those instances is hardness of the target, so the row carries lower bounds, but an algorithm for the
target on all instances is not what the row needs, so the upper-bound search does not use it (§6.6) and its explanation
says why ("not applicable: it holds only on special instances of its target"); the inspector and the Reductions table
label it "lower bounds only". The built-in config sets it on R106 (Triangle Listing to 4-Cycle Listing,
Abboud–Bringmann–Fischer 2023, for $\Theta(\sqrt n)$-regular graphs with few triangles; its `requires: "m=3/2"` stays for
the lower-bound direction). R169 and R170 (Betweenness Centrality to APSP, which need unique shortest paths) are skipped
instead: their lower-bound direction adds nothing to APSP, and with ties the distances alone do not give betweenness
centrality (the fastest known exact algorithm is about cubic). Other rows whose Restrictions limit them to special
instances can be flagged the same way as they turn up; the app does not read Restrictions.

Free-parameter types:

| type | meaning | grid |
|------|---------|------|
| `int` | integer constant | the listed `values` |
| `real` | real constant in `[min, max]` | `gridPoints` evenly spaced values incl. endpoints, then refinement |
| `size` | the quantity `base^τ`, `τ ∈ [min, max]` | as `real`, on `τ` |
| `fixed` | a constant with the given `value` (`null` = unknown constant) | one point |

`min`/`max` may be LaTeX expressions in previously listed free parameters (row R303). Names are canonical identifiers
(§4.3), e.g. `delta`, `B_med`, `c_epsilon`.

A problem's `constraints` are its **limits** (§2.3, §3.7): each is `param <= limit`, optionally followed by `: reason`
(shown as a tooltip), where the limit is a sum of terms, each a product of factors separated by `*` (or spaces): a
parameter with an optional power (`n^2`, `m^{1/2}`, `m^(3/2)`, `m^1.5`) or a positive constant (`2*n`, ignored at the
scale of exponents). Both sides name parameters of the problem that are not log-dependent, and a parameter cannot bound
itself; a malformed limit is ignored with a warning (Diagnostics).

### 3.4 Scenario (importable/exportable; autosaved in the browser)

```jsonc
{
  "version": 1,
  "constants": { "omega": 2.371339 },
  "knobs": [
    { "id": "L2", "enabled": true, "problem": "3SUM / 3SUM", "kind": "LB", "regime": "n=1, M=poly", "exponent": 2 },
    { "id": "L1", "enabled": true, "problem": "Boolean Satisfiability / CNF-SAT", "kind": "LB",
      "regime": "n=log, m=polylog", "exponent": 0.5, "power": 0.75 },
    { "id": "U3", "enabled": true, "problem": "Orthogonal Vectors / OV", "kind": "UB", "formula": "n^2 d" }
  ],
  "problems": ["Orthogonal Vectors / OV", "3SUM / 3SUM", "String Similarity / Edit Distance"],
  "settings": { "maxDepthUB": 3, "gridPoints": 9, "refineRounds": 20,
                "unknownRT": "assume-negligible", "useRandomized": true, "disabledRows": [] },
  "pinned": { "Orthogonal Vectors / OV": ["n=1, d=1"] }
}
```

* `pinned` (optional; written by Scenario ▸ Export since November 2026): the regimes pinned in the inspector, per
  problem, so that a shared file shows the same picture. Loading a file with it replaces the current pins; a file
  without it keeps them. Entries for unknown or dynamic problems and specs that are not regimes of their problem are
  dropped with a message.
* Loading checks the file first: JSON that is not a scenario (no `knobs` list; an engine config is recognized by
  `initialKnobs`, `problems` or `rows` and pointed to Config ▸ Load) is refused and nothing changes. A scenario with
  problems asks before it replaces the current one, and the message after loading offers **Undo** (Config ▸ Load
  likewise). The values of
  constants that the config does not have, or that lie outside their range, are dropped (`checkScenarioConstants`), as
  are knob IDs that are not whole numbers or Bound IDs of the wrong letter (a new ID is given), and `disabledRows`
  entries that are not text (the others are kept); each is reported. The bounds that are then unusable (an unknown
  problem, a bad regime or formula) are counted in the message.

* `id` is the knob's Bound ID (§3.3): `L…`/`U…` (in either case) or other text kept as given, a bare number of earlier
  versions converted as in §3.3, and a missing or repeated one (reported) or an upper bound numbered below 10 given the
  next free ID of its kind, after the largest one given and the initial settings' IDs (for saved data and loaded files;
  knobs added in the app and blank IDs of an imported bounds CSV avoid those too). `disabledRows` holds Reduction IDs in
  canonical form (§3.1).
* `problems` (optional) orders the left panel's **boxes**, one per problem (`src/model/boxes.ts`): first the listed
  problems, then those of knobs not listed, in the order of their first knob; each box holds its problem's knobs in
  their order, and names of one problem (any case or spacing) share a box. A listed problem without knobs is an empty
  box (in the example, Edit Distance). Blank and repeated entries are dropped. Edits in the panel store the complete
  order, so a box stays when its last knob is removed; the initial scenario lists nothing (its boxes follow its knobs:
  first the hypotheses' problems, then the algorithms-only ones).
* A knob has exactly one of `exponent` (a number, slider in the UI) or `formula` (a LaTeX runtime in the problem's
  parameters and the constants, e.g. `n^{\omega k/3}`). An exponent may come with a `power` $a > 0$ (default 1; §2.2):
  the time $2^{c\,\ell^a}$, i.e. $2^{c\,n^a}$ for CNF-SAT (in the example, $2^{0.5\,n^{0.75}}$). A formula has its own
  rate (`2^{0.5 n^{0.75}} \cdot m` is power 0.75, exponent 0.5; `n^{\log n}` power 2), so `power` goes only with an
  exponent.
* LB knobs need a `regime`; UB knobs without a `regime` apply to every regime (the formula is evaluated there). The
  `regime` of a UB formula is a **filter**: the knob applies at every regime whose parameters mentioned there are in
  the given classes (e.g. `M=polylog` for an algorithm for small weights, `k=const:[4,6]` for even cycle lengths), and
  its regime (completed by the problem's first one) is one of interest. Polynomial exponents and constants match
  within 2e-6 (regimes round them up, §6.7); `log`, `polylog` and `subpoly` are upper bounds (a constant satisfies
  them: an algorithm for polylogarithmic weights also works for constant ones); `poly` for a log-dependent parameter
  needs a polynomial one. Row `requires` (§3.3) read the same way.
* `settings`: `maxDepthUB` (0–5, default 3; "UB detours" in the app) bounds how many regimes in a row that are not of
  interest an upper-bound derivation passes through (§6.6); derivations have no other length limit. (Up to November
  2026 it went to 12; at 7 or 8 the search met more than 60 000 regimes, where it stops expanding, and bounds then
  depended on the order of the queries: ST-Maximum Flow at $m = n^{4/3}$ got $n^{3.333}$ instead of $n^{3.162}$. A saved
  or loaded value above 5 becomes 5, with a message.) `gridPoints` (2–41) and
  `refineRounds` control the free-parameter search (§6.3), `unknownRT` the policy for rows without a Reduction Runtime
  formula (§2.6), `useRandomized` and `disabledRows` which rows are used. Earlier versions also had `maxHopsLB`, a
  length limit for lower-bound derivations: it is ignored. (Before format 4 of the saved data, `maxDepthUB` limited
  the length of whole upper-bound derivations, with default 4.)
* Regime specs: `param=growth` pairs. Parameters not mentioned take the problem's first configured regime, else the
  defaults (`M=poly`, everything else `1`).

```
growth := number | number '/' number      polynomial exponent (n=1, m=2, m=4/3)
        | 'log' [':' number]              c·log2 N (default c = 1)
        | 'polylog'                       non-constant, (log N)^{O(1)}
        | 'subpoly'                       sub-polynomial, N^{o(1)}, not polylogarithmic (class S, §2.6)
        | 'const' [':' value-or-list]     constant; a list (const:[3,4,5]) expands into one regime per value
        | 'poly'                          log-dependent parameter, polynomial (evaluated at N^3, §2.6)
```

### 3.5 Bounds CSV (hypotheses and algorithms; `src/model/boundsCsv.ts`)

The knobs can also be kept in a spreadsheet, one bound per row, imported into the left panel and exported from it.
As for the sheet, columns are found by header name (in any case, surrounding spaces ignored; any order; extra columns
are ignored), the file is comma-separated UTF-8 with standard quoting, and blank rows are skipped; a quote left open
(the rest of the file would be one cell) is an error at its line, and the rows before it are read. The boxes (§3.4) follow
the order in which problems first appear, and a row with neither `Upper or Lower` nor `Runtime` only lists its problem
(a box without bounds; its other cells are ignored, with a warning if any is filled).

| column | required | meaning |
|---|---|---|
| `Problem`, `Variation` | yes | the problem (family and variation; case-insensitive, config aliases apply; `Problem` blank for a problem without a family) |
| `Bound ID` | yes | the knob's id: `L1`, `L2`, … for lower bounds, `U1`, `U2`, … for upper bounds (other text is accepted; letters in either case): an import in "add" mode replaces the knob with the same id. A blank or repeated one, or one whose letter is not that of its kind (`U17` on a `Lower` row, which would replace the algorithm U17 by a hypothesis), gets the next free one of its kind after all in the file, the current knobs and the initial settings (warning). A bare number is a Bound ID of the versions before the letters, converted as in §3.3 (1–9 → L1–L9, 10–33 → U1–U24, a lower bound n → Ln, an upper bound from 34 → U(n + 334)); an upper bound below 10 gets a new ID (warning) |
| `Upper or Lower` | yes | `Upper` (algorithm) or `Lower` (hypothesis); also `UB`/`LB`, `upper bound`/`lower bound`, `algorithm`/`hypothesis`, any case |
| `Runtime` | yes | a plain number is an **exponent** of the regime's main parameter N (a slider; e.g. `2`, or `1` for CNF-SAT, whose N is $2^n$); anything else is a LaTeX runtime in the problem's parameters and the constants, with or without `$...$` (`n^2 d`, `2^n \cdot m`, `\min(n^{\omega}, m^{2\omega/(\omega+1)})`, `2^{0.5 n^{0.75}}`). A lower bound may be written `\Omega(...)` or `n^{2-o(1)}` |
| `Power` | no | with a plain number in `Runtime`: its power $a$ (§2.2; empty: 1), e.g. `0.75` with `0.5` for $2^{0.5\,n^{0.75}}$; an error with a formula |
| `Regime` | no | a regime spec (§3.4). Empty: the problem's main regime, except that an upper bound given as a formula applies to every regime |
| `Enabled` | no | `yes`/`no` (also `y`/`n`, `true`/`false`, `on`/`off`, `1`/`0`); empty: yes |
| `Note` | no | shown under the knob |

Reading never throws: each row's problems are listed with its line number; rows with errors (unrecognized
`Upper or Lower`, no or unparsable `Runtime`, a non-positive lower-bound exponent, an invalid regime or one with
parameters the problem does not have, a lower bound without a regime for an unknown problem) are skipped, and rows
naming a problem that is not in the current sheet are read with a warning (the knob stays inactive until the problem
exists). The import dialog shows the counts and these messages before anything changes, and offers **Replace** the
current problems and knobs (the boxes become the file's) or **Add** to them (same Bound ID: replaced in place; others
appended; the file's new problems get boxes after the current ones). Constants and engine settings are not touched.

**Export** writes the boxes in order, each box's knobs in a row each (an empty box as a row with only its problem), in
the same format (formulas in `$...$`, exponents as numbers, all eight columns, and `Power` after `Runtime` when a knob
has a power other than 1); reading it back gives the same boxes and knobs. The initial knobs keep their config ids
(L1–L18, U1–U442), so their rows are grouped by problem rather than sorted by id; exporting right after "Restore initial
settings" gives exactly `data/initial-bounds.csv`, the sample file (regenerate it with `npm run bounds-sample` after
changing `initialKnobs`; a test fails while it is out of date). Knobs added in the app also get the next free ID of
their kind, and each knob shows its Bound ID (`U7`).

### 3.6 The built-in algorithms (`Trivial algorithms`, `Best known algorithms`)

For each static problem of the sheet, a trivial algorithm (brute force or a textbook method that follows from the
definition) was derived and the best known algorithm(s) were looked up, mainly in the papers the sheet cites for the
problem (October 2026; about 30 seconds per problem on average, so a few problems have no best-known entry). Each
knob's note says how the algorithm works and, for the best known ones, gives the source (authors, venue, year, link);
`[randomized]` marks randomized algorithms and `[medium confidence]` bounds that were derived here or taken from a
secondary source. Conventions:

* Formulas keep the polynomial part: improvements by polylogarithmic or $2^{\Omega(\sqrt{\log n})}$ factors do not
  change exponents (they are mentioned in the notes), and an $O(n^{2+\varepsilon})$ for every $\varepsilon > 0$ is
  `n^{2+o(1)}`.
* A formula without a regime holds for every parameter value; algorithms for special cases have a regime filter
  (§3.4): `M=polylog` (pseudo-polynomial algorithms, e.g. $\tilde O(M n^\omega)$ for small weights; `M=const` where only
  constant ones are known), `d=polylog` (the Orthogonal-Vectors-type speed-ups $n^{2-1/O(\log c)}$ at $d = c\log n$,
  which vanish for $d = \mathrm{polylog}(n)$), `sigma=const…` (constant alphabets), `k=const:[…]` (the values of $k$ a
  formula is stated for).
* Knobs that would duplicate existing ones (the trivial $n^k$ for Min/Max-Weight $k$-Clique, $2^n\,\mathrm{poly}$ for
  CNF-SAT and MAX-CNF-SAT) were not added; no existing knob was changed or removed. A preprint seen then only on an
  aggregator (a truly subcubic (min,+)-product, arXiv:2610.20297) was left out; checked on arXiv since, it is the
  `Saha & Ye 2026` group (§3.3; off, like the other refutations of hypotheses).
* With every hypothesis switched on, the trivial and best known algorithms give no conflict (a test checks it); the
  `Non-combinatorial algorithms` group (Valiant's parser, $O(g\,n^{\omega})$, and the transitive closure in
  $O(n^{\omega})$) does conflict with the combinatorial BMM hypothesis, through Lee's reduction R126 and the BMM to
  transitive closure reduction R356, so it starts switched off.
* The problems added with rows R320–R365 (Oct 2026) got their trivial and best known algorithms from the same search
  (U370–U418), except the two dynamic ones (Multiphase Problem, Dynamic Subgraph Connectivity), whose per-function
  bounds the knobs cannot hold: the parameter dictionary lists them. One of their best known bounds depends on a
  regime the app cannot separate: Approximate Attention Computation in $n^{1+o(1)}$ (Alman–Song) needs $d = O(\log n)$,
  while the app's class $d = \mathrm{polylog}(n)$ also contains larger $d$; its regime filter `d=log, B=const` therefore
  applies to all polylogarithmic $d$ with constant entry bound $B$ (the hard case of row R351 is $B = \Theta(\sqrt{\log
  n})$, a different regime).
* The two problems that the trivial reductions R366–R431 gave a signature without giving them a knob (All-Edges
  Exact Triangle and #All-Edges Monochromatic Triangle, whose other rows are skipped) got theirs: listing all triangles
  in $O(m^{3/2})$ (U419, U420), $\tilde O(M n^{\omega})$ for small weights by the encoding of U240 (U422), and $O(c\,n^{\omega})$ by
  squaring the adjacency matrix of every color class (U421).
* The nine problems added with rows R432–R462 (Nov 2026) got theirs from their sources and definitions (U423–U439):
  $k$-AP Detection in $O(kN^2)$, and for $k = 3$ in $O(N \log N)$ by FFT; $k$-GeomBase in $O(kn^2 \log n)$; the
  covering, stabbing and double-wedge problems in $O(n^3)$ by testing the faces of an arrangement, and in $O(n^2)$ by
  traversing it (Gajentaan–Overmars, Bertschinger et al.); Lopsided All-Edges Sparse Triangle in $O(nd + wd)$, and in
  $n^{2+o(1)}$ at its regime $d = n^{1/18}$ by rectangular matrix multiplication; #Exact Triangle in $O(m^{3/2})$, and
  with small weights in $\tilde O(M n^{\omega})$ by the encoding of U240; Zero-Weight $k$-Clique in $O(k^2 n^k)$, and
  with small weights, for $k = 3, 6$, in $\tilde O(M n^{\omega k/3})$ by folding. The lopsided problem is looked at only
  where Theorem 1 of Alman and Vassilevska Williams applies ($d = n^{1/18}$, $w = n^{71/36}$, its only regime); row R453
  fixes its free parameter $d$ there and optimizes the number of groups $g$.
* Chan's $O(N^{d/2})$ for Weighted Depth holds for every constant $d \ge 3$: U361 lists $d \le 4$, and U441 (added with
  rows R463–R465) $d = 5, 6$, which the cubic lower bound of Max-Weight $k$-Clique (R463, R367) reaches through R277.
* `Known algorithms`' $k$-Clique bound $n^{\omega k/3}$ (U14, Nešetřil–Poljak) holds when 3 divides $k$; since November
  2026 its regime is `k=const:[3,6]`. Before, it applied at every $k$ and beat the correct bounds for $k = 4, 5$,
  $\omega(1,1,2) < 3.250035$ and $\omega(1,2,2) < 4.085552$ (U341, U342), so that k-Clique showed $n^{3.16207}$ and
  $n^{3.952587}$ there, tight with the k-Clique hypothesis (found by an independent review).
* Updated in November 2026: ω is 2.371339 (Alman, Duan, Vassilevska Williams, Xu, Xu, Zhou, SODA 2025;
  arXiv:2404.16349), was 2.371552 (SODA 2024); the review that prompted this also cites a 2026 preprint with 2.371177
  (arXiv:2608.16884), not checked here, so the published bound is used. The same paper
  gives $\mu < 0.5275$, so the small-weight APSP algorithms and their consequences (U273, U304, U307, U417: Zwick's
  $n^{2+\mu}$) and the minimum-witness product (U288, Czumaj–Kowaluk–Lingas, the same exponent) are $n^{2.5275}$ (were
  2.5277, and 2.5286 for U288). These rectangular exponents are numbers in the formulas, so they do not follow the ω
  slider; the slider's note (`constants.omega.note`) says so and names the knobs. Chan's $O(N^d)$ for Max-Weight
  Rectangle (U359) holds for every constant $d$: its regime now lists $d \le 6$ (it stopped at 4, so $d = 5, 6$ showed
  $N^{11}$ and $N^{13}$ from other bounds).

### 3.7 The limits of parameters (`constraints`)

Every problem with two or more parameters that are not log-dependent was checked against the meaning of its
parameters (the sheet's variable notes): 130 of them have limits, each a fact about every instance, or about every
instance after a linear-time cleanup that no running time here notices (relabeling an alphabet or colors to those that
occur). The others are vectors of a dimension $d$ (OV, MIPS, Partial Match, …), points and polygons of independent sizes,
strings of independent lengths, numbers and targets (Subset Sum, Knapsack), clauses of $n$ variables, grammars and the
strings they compress (an SLP may be larger than its text), and constants ($d$, $k$).

| problems | limits | reason |
|---|---|---|
| graphs on $n$ vertices with $m$ edges (92 with no other limit: those with parameters $n, m$ or $n, m, k$, incl. the $k$-clique, $k$-cycle, $k$-dominating set, model-checking and dynamic graph problems, transitive closure, bottleneck paths and Dyck-2 reachability, and $\Delta$-Matching Triangles, Triangle Collection*) | $m \le n^2$ | a simple graph (Dyck-2: one edge per label and ordered pair) |
| String Matching in Labeled Graphs; Bicriteria $s,t$-Path | $m \le n^2$ | a simple directed graph (the pattern length $p$ and the budgets $L, C$ are independent) |
| Triangle Listing; Edge Triangle Counting ($\alpha$ = arboricity) | $m \le n^2$, $\alpha \le m^{1/2}$, $m \le n\alpha$; Triangle Listing also $t \le m\alpha$ | $\alpha = O(\sqrt m)$; $\alpha \ge m/(n-1)$ (Nash-Williams); $O(m\alpha)$ triangles (Chiba–Nishizeki), which implies $t \le m^{3/2}$ and $t \le n^3$ |
| 4-Cycle Listing | $m \le n^2$, $t \le m^2$ | two opposite edges determine a 4-cycle |
| Negative Triangle Listing | $m \le n^2$, $\Delta \le m^{3/2}$ | at most $O(m^{3/2})$ triangles |
| All-Edges Monochromatic Triangle | $m \le n^2$, $c \le m$ | at most $m$ edge colors occur |
| ST-Maximum Flow | $m \le n^2$, $\lvert S\rvert \le n$, $\lvert T\rvert \le n$ | sources and sinks are vertices |
| Set Disjointness; Set Intersection | $s \le U$; Set Intersection also $r \le q s$ | a set is part of the universe; an intersection has at most $s$ elements |
| Partial Convolution (and Indexing); Partial Matrix Multiplication | $s, K \le N$; $s, K \le N^2$ | the ones of the inputs and the entries of the output |
| Combinatorial BMM; Convolution Witness | $\text{nnz} \le n^2$; $\text{nnz} \le n$ | the nonzero entries of the inputs (and the indices queried) |
| Dynamic Vertex Triangle Listing; Induced Subgraph Problem | $\delta \le m^{1/2}$ | the degeneracy is $O(\sqrt m)$ |
| Dictionary Matching with One Gap (and its * variant) | $\delta \le \lvert D\rvert^{1/2}$; $\delta \le d^{1/2}$ | the dictionary graph has one edge per pattern, so its degeneracy is $O(\sqrt{\#\text{patterns}})$ |
| strings with alphabet $\sigma$: Edit Distance, Local Alignment, the abelian squares, Longest Common Substring with don't cares, Jumbled Indexing, Tree Edit Distance (labels); LCS and DTW; Multiple Local Alignment and $k$-LCS | $\sigma \le n$; $\sigma \le n + m$; $\sigma \le k n$ | only the symbols that occur count |
| Shift-Ham, ShiftScale-Ham, Dynamic $k$-Mismatch | $m \le n$, $\sigma \le n + m$; Dynamic $k$-Mismatch also $k \le m$ | the pattern is no longer than the text |
| Point Covering, Smallest Color-Spanning Strip, the $k$-Enclosing Rectangles | $k \le n$ | a depth, a number of colors or of points to enclose among $n$ |

Only Triangle Listing and Set Intersection had regimes beyond their limits (§2.3); every other regime of the sheet
was already within them.

---

## 4. Formula language

### 4.1 Grammar (LaTeX subset)

```
formula     := expr ( ALT expr )*                        ALT ∈ {\subseteq, \le, \leq, =}
expr        := term ( ('+' | '-') term )*
term        := unary ( ( '\cdot' | '\times' | '/' | ε ) unary )*      ε = juxtaposition; left-associative
unary       := '-' unary | power
power       := atom [ '^' script ]                       script := '{' expr '}' | single token (one digit/letter)
atom        := number | ident | ident '(' args ')'      (call, function or product; resolved in §4.2)
             | '(' expr ')' | '[' expr ']' | '{' expr '}' | '\left(' expr '\right)'
             | '\lceil' expr '\rceil' | '\lfloor' expr '\rfloor' | '|' expr '|'
             | '\sqrt' ['[' expr ']'] group | '\frac' group group | '\binom' group group
             | LOG [ '^' script ] logarg                LOG ∈ {\log, \ln, \lg}; logarg := '(' expr ')' | power
             | '\max' '(' args ')' | '\min' '(' args ')' | '\deg' '(' expr ')'
             | BOUND '(' expr ')'                        BOUND ∈ {O, O_k, o, \Theta, \Omega, \tilde{O}, \tilde{\Theta}, \tilde{\Omega}}
             | '\sum' [ '_' script ] [ '^' script ] term
             | '\text{poly}' '(' expr ')' | '\poly' '(' expr ')' | '\text{polylog}' '(' expr ')'
args        := expr ( ',' expr )*
```

Whitespace and spacing commands (`\,`, `\;`, `\!`, `\quad`, `~`) are ignored. `\left`/`\right` are ignored before
delimiters. Anything else is a parse error with a position (shown in the Diagnostics view).

### 4.2 Meaning

| construct | meaning |
|---|---|
| `O(x)`, `O_k(x)`, `\Theta(x)` | the growth of `x` up to a constant factor: invisible for polynomial growths, but `O(c)` is an *unknown* constant (so `n^{O(1)}` has an unknown exponent and `2^{O(k)}` is a constant) and `O(\log n)` has an unknown coefficient (so `2^{O(\log n)}` has an unknown exponent); so has `O(n^{0.5})` at CNF-SAT's scale (`2^{O(\sqrt{n})}` is only $2^{o(n)}$) |
| `\tilde{O}(x)` | `x` times a polylogarithmic factor |
| `o(x)` | $10^{-12} \cdot x$ for a constant or logarithmic `x`, an infinitesimal that exponents do not notice: `n^{1+o(1)}` is $N^{1}$, while `n^{o(1)}` is $N^{o(1)}$ and, at CNF-SAT's scale, `2^{o(n)}` is $2^{o(n)}$ and `2^{o(n^{0.75})}` is $2^{o(n^{0.75})}$ (rate $(0.75, 0)$); other arguments are an error. (Before rates, `o(1)` was the constant 0, which made `n^{o(1)}` a constant.) `O(0)` is 0 (e.g. `O(\gamma)` with $\gamma \to 0$) |
| `\Omega(1)` | a small positive constant ($10^{-6}$, larger than the engine's comparison and conflict tolerances), e.g. `n^{2-\Omega(1)}` |
| `\omega(...)` | an error: little-omega is not supported, and the constant ω times something must be written `\omega \cdot (...)` |
| `T(...)`, `T_p(...)` | oracle call of the target's time function |
| `\sum_{i=a}^{b} body` | see §6.4 |
| `x(args)` | product `x · args` if `x` is a variable in scope; else the function `x` (only `\alpha` = inverse Ackermann) |
| `\deg(v)`, unbounded `\sum_v` | not evaluable (forces the next alternative) |

### 4.3 Identifiers

Single letters, Greek letters (`\delta` → `delta`, `\varepsilon` → `epsilon`), `\text{word}` (→ `word`, e.g. `nnz`),
`\hat{d}` (→ `hat_d`), with an optional subscript (`T_p`, `n_v`, `B_{\text{med}}` → `B_med`, `c_{\epsilon}` →
`c_epsilon`) and `|S|` (→ `|S|`). Juxtaposed letters are separate identifiers (`nd` = `n·d`).

---

## 5. Growth algebra (`src/growth/growth.ts`)

**Abstraction function.** A `Growth` denotes a set of positive functions of $N$; operations return a growth that
contains every possible result (upper-bound semantics), so evaluation of an $O(\cdot)$ formula is sound.

With $\ell = \log_2 N$:

| kind | denotes | exponent | rate (§2.2) |
|---|---|---|---|
| `const(v)` | $\Theta(1)$; `v` the value if known (`null`: unknown constant) | 0 | zero |
| `log(p, c)` | $\Theta(\ell^p)$, $p \in [0,\infty]$; $p=0$: $\ell^{o(1)}$ but $\omega(1)$ ($\log\log N$, $\alpha(N)$); $p=\infty$ (`POLYLOG`): of rate zero, of unknown, at least polylogarithmic, order (`\text{polylog}(n)`, `\log^{O(1)} n`, $2^{\ell^{o(1)}}$); `c`: the value is $c\,\ell^p$, when known and $0 < p < \infty$ (may be negative: used only as an exponent of 2) | 0 | zero |
| `qpoly(a, c)` | $2^{(c \pm o(1))\,\ell^{a}}$, $a > 0$, $c \ge 0$ ($c = 0$: $2^{o(\ell^a)}$); for $a = 1$ only $c = 0$: $N^{o(1)}$ (`SUBPOLY`) | 0 if $a \le 1$, else $+\infty$ | $(a, c)$ |
| `poly(e)` | $N^{e\pm o(1)}$, $e \ne 0$ | $e$ | $(1, e)$; zero if $e < 0$ |
| `polyUnknown` | $N^{\Theta(1)}$ with unknown exponent (from `n^{O(1)}`, `\text{poly}(n)`) | error | error |
| `super` | $N^{\omega(1)}$ without useful bound ($2^N$, $2^{2^{n}}$) | $+\infty$ | none |

**Rep invariant.** `poly.e` finite and $\ne 0$; `log.p` in $[0, \infty]$; `log.c` is `null` unless $0 < p < \infty$,
and nonzero; `qpoly.a` finite and $> 0$, `qpoly.c` finite and $\ge 0$, and 0 if $a = 1$; `const.v` finite if non-null.
Powers within $10^{-9}$ of 1 are 1. Checked by every constructor (error messages are only built on failure, because
growths are created in the inner loops). (Until this version `log(∞)` stood for $N^{o(1)}$ and coefficients were kept
only for $p = 1$: at a power below 1, $N^{o(1)}$ and polylogarithmic factors are no longer the same.)

**Operations** (each total; errors are thrown, never silently mapped to 0):

| op | rule (summary) |
|---|---|
| `add(a,b)`, `max` | exact on two constants and on two $c\,\ell^p$ quantities with the same $p$ and known coefficients; otherwise the larger in the order (negative log quantity, larger $p$ lower) < (negative constant) < 0 < poly(<0) < positive const < log (by p) < [`qpoly`, `poly(>0)` by rate: power first, then coefficient] < super; `polyUnknown` is above the classes of power < 1 and $N^{o(1)}$, below those of power > 1, and incomparable with `poly(>0)` (the result is then `polyUnknown`) |
| `min` | the smaller; `min(poly(e), polyUnknown) = poly(e)` |
| `sub(a,b)` | `a` if `b` is strictly smaller; consts and log coefficients subtract exactly; equal-order `poly` stays (assumes the difference is still $\Theta(a)$); `b > a` is an error |
| `mul` | exponents add (a sum of 0 gives $N^{o(1)}$); log powers add and coefficients multiply; `qpoly` of one power add coefficients, else the larger power wins; a power above 1 absorbs every polynomial factor (also `polyUnknown`), a power up to 1 is absorbed by any polynomial; rate-zero factors (constants, log classes) leave `qpoly` unchanged; constants multiply coefficients |
| `inv` | `poly(e) → poly(-e)`; `const(v) → const(1/v)`; `log`, `qpoly → const(null)` (an $o(1)$ quantity is bounded by a constant) |
| `pow(a, k)` (k a known constant) | `poly(e·k)`, `log(p·k, c^k)`, `qpoly(a, c·k)`, `const(v^k)` |
| `pow(a, k)` (k unknown constant) | `poly → polyUnknown`, `log → log(∞)`, `qpoly(a ≤ 1) → ` $N^{o(1)}$, `qpoly(a > 1) → super`, `const → const(null)` |
| `exp2(x)` | `const(v) → const(2^v)`; `log(1, c) → poly(c)`; `log(1, null) → polyUnknown`; `log(p, c) → qpoly(p, c)` for $p \ne 1$ and a known $c > 0$ ($c < 0$: `const(null)`); `log(p<1, null)` → $N^{o(1)}$; `log(p>1, null)`, `log(∞) → super`; `log(0) → log(∞)`; `poly(e>0)`, `qpoly → super`; `poly(e<0) → const` |
| `b^x` (b a known positive constant) | `exp2(x · log2 b)` (e.g. `4^i`, `(1/4+\gamma)^i`, $3^{\sqrt{n}}$); a polynomial base $N^e$ with an exponent $c\,\ell^p$: `qpoly(p+1, e·c)` ($n^{\log n} = 2^{\log^2 n}$) |
| `log2(x)` | `poly(e) → log(1, e)`; `qpoly(a, c) → log(a, c)` (`c` null if 0); `log(p) → log(0)`; `const(v) → const(log2 v)`; `polyUnknown → log(1, null)` |
| `ceil`, `floor`, `binom` | exact on known constants; $\lceil N^{-e} \rceil = 1$, $\lfloor N^{-e} \rfloor = 0$; otherwise identity / `a^b` |
| overflow | a constant that is not finite (e.g. `10^{400}`) is an EvalError, not a value |

**Rates** (also in `growth.ts`): `rateOf(g)` is the rate of a time of class `g` (table above; an EvalError for
`polyUnknown`); `rateGrowth(r, s)` the class of a time of rate `r` on an instance of size $N^s$, i.e.
$2^{c (s\ell)^a}$ (`poly(c·s)` for power 1, `log(∞)` for the rate zero, `super` for none); `compareRates` orders
rates power first (powers equal within $10^{-9}$), then by exponent within a tolerance; `exponentAtPower(g, a)` is the
exponent of `g` at power `a` (+∞ above, 0 below; `exponentOf` for power 1), which the lower-bound threshold of §2.5
uses.

---

## 6. Engine

### 6.1 Row compilation and support status (`src/model/model.ts`)

Each row is compiled once per model into a `CompiledRow` with a **status**:

* `ok`: participates in propagation.
* `no-formula`: the From-time cell has no math span (and no override).
* `parse-error`: with message and position.
* `dynamic`: the source or target is not static (§3.1).
* `no-signature`: the source or target has no known signature.
* `unknown-symbols`: no alternative of the From-time uses only source parameters, free parameters, abbreviations,
  constants and sum indices; the UI suggests declaring the missing symbols in the config. (An RT with undeclared symbols
  only makes the RT unknown.)
* `arity`: a `T`-call's argument count differs from the target signature.
* `non-monotone`: a `T`-call occurs under a division, a negative power, a logarithm or an exponential (so $g$ might not be
  nondecreasing and §2.5's soundness argument fails).
* `skipped`: the config's `skip`. (The scenario's `disabledRows` and `useRandomized = false` exclude rows at analysis
  time; `Randomized?` values `Yes`, `Y` and `?` count as randomized, so excluding them is conservative.)

These checks run once at load, not during propagation, so propagation code can assume well-formed rows (SFB).

### 6.2 Row evaluation (`src/engine/rowEval.ts`)

`prepareRow(model, row, sourceRegime, θ, constants)` builds the environment (constants, source parameters ↦ regime
growths, free parameters, then abbreviations in dependency order), evaluates the RT's rate (or `null` = unknown), and
chooses the first From-time alternative that evaluates. `evaluateF(prepared, value, onSite)` then evaluates the From-time
with every oracle call valued by `value(targetRegime, scale)` (the caller decides: $N^{s x}$ for the LB threshold
search, the target's UB for upper bounds) and reports each call site (target regime and scale) to `onSite`.
Normalizations of call arguments are cached per prepared row, since threshold searches re-evaluate the same calls.

A call is **degenerate** when no argument can be a normalizer (all constant or polylog). Its value is polylogarithmic
(rate zero; $N^{o(1)}$ if an argument is sub-polynomial but not polylogarithmic) if the target has `timeType: "poly"`,
and the evaluation fails otherwise (an exponential-time target on a polylog-size instance may be superpolynomial).
This assumes what `timeType: "poly"` says: the target is solvable in polynomial time (a quasi-polynomial algorithm for
it, e.g. a knob $n^{\log n}$, is then just a weak upper bound; a quasi-polynomial lower bound contradicts the
assumption).

A call is not evaluated at all (the evaluation fails, so that free-parameter value is dropped) when a parameter of the
produced instance would grow faster than $N^{10^4}$ (`MAX_REGIME_EXPONENT`): the instance then barely grows (its
normalizer is $N^{s}$ for a tiny $s$), and bounds there have lost their precision in floating point. On the sheet this
happens only at the end of a free parameter's range: row R344 at $\beta = 1$ gives a Bicriteria s,t-Path instance with
$2^{\varepsilon n}$ vertices ($\varepsilon = 10^{-12}$) and budgets $N^{10^{12}}$, whose bounds came out about 1000 apart
(found by an independent review). The largest exponent of any other regime met, with every group on, is below 40.

### 6.3 Free parameters (`src/engine/freeparams.ts`)

The grid is the Cartesian product of each parameter's grid, capped at 400 points (the per-parameter resolution is
reduced when a row has several continuous parameters). After the grid search, the best point is **refined**
`refineRounds` times (default 20, at most 40) by a coordinate-wise zoom (two probes per parameter in a window that halves
each round) over the continuous parameters. For LBs the optimization is done separately for each target-regime key, and only
for keys reached by at least two grid points, and not for keys reached by three or more that all give the same bound:
when the free parameters move the produced regime (e.g. the trade-offs of Kopelowitz–Pettie–Porat), every grid point is
a separate fact and a perturbed point would land in another regime anyway, and when several grid points reaching a
regime agree, the free parameters most likely move only what does not matter there (typically a size clipped to its
limit, §2.3: row R72's $\delta$ beyond $\beta/3$). Two agreeing points are still refined, since a better point may lie
between them (a tent-shaped bound); three or more agreeing ones could in principle enclose a peak too, which this
skips. (Without the rule the limits, which make many of R72's grid points reach one regime, would have tripled the
time of the propagation; with it, no bound of the sheet differs by more than $10^{-6}$ from the one refining would
give, and the analysis is faster than before the limits.)
For UBs, refinement likewise requires two grid points whose calls land in the same target regimes. When a refinement
step moves a parameter, later parameters whose ranges depend on it (e.g. $b \in [0, 1-a]$) are clamped into their new
ranges. The first step of the refinement is the spacing of the grid actually made: range / (values − 1), with the values
per parameter reduced as for the cap. (Before November 2026 it used the `gridPoints` asked for, so with more than 20
points and two continuous parameters the steps were too small to reach the best point between two grid values: at 41
points Lopsided All-Edges Sparse Triangle dropped from $n^2$ to $n^{1.99923}$, and about ten problems downstream with
it; found by an independent review.)

The inspector shows free-parameter values with four decimals, except values below $5 \cdot 10^{-5}$, which show by
their size (`epsilon = 1e-12`, not `0`: rows R256 and R344 need $\varepsilon > 0$).

Optima at a *blocked boundary* are common (e.g. row R192: the LB tends to 2 as $B_{\text{med}} \to 1$, but at
$B_{\text{med}} = 1$ the reduction runtime equals the lower bound). Refinement approaches such suprema to about
$10^{-7}$ with the default settings (2.4 × 10⁻⁴ with the first version's 10 rounds).

The search is **one-sided**: any $\theta$ in the allowed ranges gives a valid implication, so a derived LB is the
supremum over $\theta$ approached from below, and a derived UB the infimum approached from above. Search errors can
therefore only weaken bounds, never produce a conflict; §6.7 relies on this.

### 6.4 Sums

`\sum_{i=a}^{b} f(i)`:
* `b` a known constant ≤ 64: exact sum over integers.
* `b` of class $\Theta(\log N)$ (known coefficient $c$, or unknown, which uses $c_{\max} = 8$): with $i = \theta\log_2 N$,
  the sum has $O(\log N)$ terms, so its class is that of its largest term (times $\log N$): by exponent and, among terms
  of exponent 0, by rate (e.g. $2^{\sqrt{i}}$); terms of a power above 1 by rate. The largest is found over
  $\theta \in [0, c]$ (grid of `gridPoints` values, endpoints included, then 30 halving rounds of local search, i.e.
  $\theta$ to about $2^{-30}c$).
  This sampled maximum is the one search that may err on the unsafe side (an underestimate), which is why it is made
  far more precise than the conflict tolerance of §6.7.
  With an unknown coefficient the evaluation fails if the term is still growing at $c_{\max}$.
* Anything else (unbounded `\sum_v`, polynomially many terms) fails, which forces the next alternative.

### 6.5 Lower-bound propagation (`src/engine/lower.ts`)

```
facts := best LB per (problem, key);  queue := instantiated LB knobs (hops = 0)
while queue not empty and pops < 20 000:
  f := pop (FIFO);  skip if superseded
  for each ok row r with source f.problem:
    for each grid point θ (+ refinement per key):
      evaluate r at (f.regime, θ); require one shared non-degenerate target key and R < f's rate
      x* := smallest x in [0, X] with g_a(x) ≥ f.L − 1e-9 at f's power a, X = max(64, 2(L+1)/s_min^a)
            (Illinois regula falsi with interleaved bisection, to 1e-9; s_min = smallest call scale)
      if g(X) < f.L: record an inconsistency (P solvable below its LB whatever Q costs)
    for each key: if better(x*, f.hops + 1) than the current fact:
      if the new fact came back around a cycle to its own problem and key, raising the bound it started from:
        replace it by the limit of repeating that cycle (§6.9)
      store fact (provenance: row, θ, parent f), enqueue
```

There is no limit on the length of derivations: propagation runs until nothing improves. (Earlier versions stopped
chains after `maxHopsLB` = 8 reductions; on the sheet the longest best chain has 5, but a limit makes a problem's bound
depend on how far it is from the hypotheses.) *Better* means a larger rate (a larger power, or the same power and a
larger exponent), or one equal within 1e-9 reached with fewer hops (reductions): among derivations of the same bound the
shortest is kept, so a chain never takes a detour (the FIFO order mostly finds the shortest first; the rule also covers
facts that improve later). Termination: each fact is popped at most once per improvement; an improvement raises the
exponent by more than 1e-9 or lowers the hops, and a cycle that would raise a bound again and again is replaced by its
limit (at most 8 times per key; the 20 000 steps cap everything else). Facts per problem are capped at 1000 keys (warning
shown). A fact at a new key that only repeats one of its own chain, at nearly the same regime of the same problem
(exponents within a relative $10^{-4}$) and without a larger bound, is dropped: a cycle that permutes parameters (Set
Disjointness and the lopsided triangle problem, rows R455 and R460) normalizes by another parameter on every lap, and its
exponents, rounded up to 6 decimals, come back a little larger each time, at a new key; so it ends as one that comes back
to the same key.

An infinite lower bound (the limit of a cycle without fixed point) transfers like any other: through a row whose calls
land in one regime and whose From-time grows without limit with the calls' cost, the target needs superpolynomial time
too ($x^* = \infty$); a row that would solve the source in polynomial time without such calls is an inconsistency.

Facts keep the power of their knob (§2.5), and one fact per (problem, key) is kept, the best: a fact of a larger power
passes through every reduction a smaller one passes through (a reduction's own time below the smaller is below the
larger) and gives at least as much, so nothing is lost. With SETH as $2^{0.5\,n^{0.75}}$, CNF-SAT and MAX-CNF-SAT
(identity, row R284) keep it, and nothing else follows: the split-and-list reductions (to OV, $k$-OV, …) take $2^{n/2}$,
and the one from $k$-SAT to Subset Sum (row R256; SETH in its $k$-SAT form as $2^{0.5\,n^{0.75}}$ likewise stays on
$k$-SAT, CNF-SAT and MAX-CNF-SAT) takes $2^{\varepsilon n}$ for a free $\varepsilon > 0$. Its range in the config is
$[10^{-9}, 1]$: at $\varepsilon = 0$, which an earlier config allowed, the reduction does not exist, yet its own time
would have been polynomial and passed such bounds on (an independent review caught this; at power 1 the bounds are
continuous in $\varepsilon$ and the endpoint only gave their limit). A free parameter whose range excludes an endpoint
must say so in the config (here by a tiny positive minimum).

### 6.6 Upper-bound evaluation (`src/engine/upper.ts`)

Every (problem, regime) met is a **node** with a single best upper bound: the best over all derivations, of any
length, from the knobs through the reductions (§2.5). So a problem's bound is the same wherever it is used, in its own
box or inside the derivation of another problem, and a problem gets every bound its reductions give, however many
reductions away the algorithm is. (Until this version, a depth-limited recursion valued a call with the bound of its
target found within one reduction fewer: with the 2026 3SUM algorithm, 0-1 Knapsack showed $n^{1.9996}$ but Unbounded
Knapsack, which reduces to it at no cost, stayed at $n^2$, and Superadditivity Testing, six reductions from 3SUM, had
none.)

The bounds are the fixpoint of "a node's bound is the best of its knobs and of each reduction from it, with its calls
valued at the bounds of their targets", computed on demand by Seidl's **top-down solver**: solving a node evaluates
its knobs and candidates, solving the nodes their calls reach (recursively; nesting deeper than 120 is postponed to a
list solved before the query returns, so the call stack stays small) and recording which nodes read which; when a
node's bound improves, the nodes that read it are solved again. Cycles of reductions need nothing special: a node in
progress reads as its current bound (its best knob, set before any reduction is tried; nodes that read it earlier,
e.g. while it was postponed, are solved again), and is solved again if that changes. Values start at "no bound" and
only improve, so the fixpoint reached is the same whatever the order (the order only decides between derivations that
are equal in exponent and reductions).

A node's **candidates** are the usable rows from its problem whose `requires` hold (not those marked `lowerOnly`,
§3.3), at each grid point θ, prepared
once (with their target regimes and the *optimistic* value: the rate of max(RT, From-time with every call of rate
zero)). They are
tried in order of their optimistic values (rows by their best candidate, then sheet order; the knobs first), each row
followed by the refinement of its free parameters when at least two grid points call the same target regimes;
refinement only moves within those regimes, whose bounds are known, so it creates no nodes (and compares only points
of the same power). Of the results, the best is kept: the smallest rate (power first, then exponent), and among those
equal within 1e-9 the one with the fewest reductions in its
derivation tree (`steps`: 0 for a knob, 1 + the steps of the bounds its calls use). Without this tie-break the first
candidate found was kept, which could be a detour (Diameter's bound came through Reach Centrality and back to Diameter
instead of directly from APSP). *Branch and bound*: a candidate whose optimistic value cannot beat the node's bound
(above it, or equal while that bound needs at most one reduction) is dropped for good, since bounds only improve; this
keeps the search to the part of the graph that matters.

**Regimes of interest and detours.** Free parameters make the number of regimes grow quickly (about 200 000 within
seven reductions of the initial regimes). The search therefore follows reductions without limit from the regimes of
interest (the configured regimes, the regimes of UB knobs, pinned regimes, and any regime asked for) but passes
through at most `maxDepthUB` (default 3) regimes in a row that are not of interest; at the next such regime only its
knobs count. At a regime that is not of interest, the remaining budget is part of the node's identity (`maxDepthUB`
below a regime of interest, one less at each further one; up to `maxDepthUB` + 1 nodes per regime), so every node's
bound is a pure function of its identity, the knobs and the settings: the same whatever the order of the rows or of
the queries, and final once solved. (A first version raised a node's budget when a later caller offered more, which
made bounds depend on the order of the search.) On the sheet, 2 is enough for every displayed bound (with 1, 25 bounds
at lower-bound regimes of Triangle Listing and Set Intersection are lost); 2 to 5 give the same results, in about 0.9,
1.3, 1.7 and 6 s. The solver depends only on the UB knobs, the constants, the settings and the regimes of interest, so
it survives every change of lower bounds (moving an LB slider takes 0.1–0.4 s). Bounds at other regimes (those of
lower bounds, §6.7, and regimes opened in the inspector) come from an **overlay**: a solver of its own that searches
them as regimes of interest and reads the nodes of the analysis's solver (final, since they are pure) without changing
them, so asking for them never changes a bound shown elsewhere. Its nodes are pure as well, so one overlay serves
every analysis that reuses the solver (a new one starts beyond 20 000 nodes). Where the budget matters, a regime of a
lower bound shown with its full search can have a better bound than the budget-limited node of that regime inside
another problem's derivation; the inspector then notes the better one ("its best: …"); on the sheet this does not
happen with the default budget. Nodes beyond 60 000 are not searched (a message says so; never reached on the sheet:
about 3 600 nodes). Since that limit makes bounds depend on the order of the search, the budget is at most 5: at 7 or 8
the limit was reached and bounds got worse, not better (ST-Maximum Flow at $m = n^{4/3}$: $n^{3.333}$ instead of
$n^{3.162}$; found by an independent review). When any search limit is reached, the status line says so in the header
("a search limit was reached, some bounds may be weaker"), not only in Diagnostics.

UB knobs are evaluated by substituting the regime's growths into the knob formula (its rate: `2^{0.5 n^{0.75}} \cdot m`
at CNF-SAT's regime is power 0.75, exponent 0.5); an exponent knob has the power it is given. Derivations may need no
knob at all: e.g. row R103 with $t = n$ solves Mono Convolution by FFT and only calls 3SUM on constant-size instances.
Each call uses its own best derivation; a call whose term does not determine the result could in principle have a
shorter derivation with a slightly worse bound that leaves the result unchanged, which is not searched for. When a
node's bound improves after another node read it, without changing the other node's bound, the other node's derivation
keeps the earlier bound; the inspector notes the better one ("its best: …"; on the sheet this does not happen).

### 6.7 Keys, regimes of interest, status

Key encoding per parameter: `p<e>` (polynomial exponent, rounded **up** to 1e-6), `E` (the exponential-scale
normalizer), `L` (non-constant polylogarithmic, or of rate zero; evaluated as $(\log N)^{O(1)}$, §2.6), `S`
(sub-polynomial but not polylogarithmic: $N^{o(1)}$, e.g. $2^{\sqrt{\log n}}$, or an exponent that is 0 up to
floating-point noise), `c<v>` / `c?` (constant, its value rounded up likewise), and for log-dependent parameters only the
class `C`/`L`/`P`/`X`. Regimes are stored with their growths rounded the same way, and a log-dependent constant as one
unknown constant, so that a key stands for exactly one regime: otherwise which of several regimes with one key is
stored, and what follows from it, would depend on the order of the search (on the sheet, refinement produced 111 keys
with growths differing below $10^{-6}$). Rounding first removes floating-point noise (to $10^{-9}$, so that $0.1 \cdot 3$
is 0.3) and then goes **up** (`roundSizeUp`: $1/3$ is 0.333334, $4\cdot 10^{-7}$ is 0.000001): a regime then stands for
sizes at least as large as the exact ones, so a lower bound derived at the exact sizes holds there, and an upper bound
taken there holds at the exact sizes (time bounds are nondecreasing in the sizes), and derivations stay one-sided as
§6.3 needs. (A regime typed for an upper-bound exponent knob is the one exception: `m=1/3` applies it at
$m = n^{0.333334}$, sizes up to $10^{-6}$ larger in the exponent than stated.) Specs written for a regime ("＋ Algorithm
here", pins: `regimeNumberText`) name it again: a short decimal when exact, else the simplest fraction that rounds up
to it (2.142858 is written `15/7`), else 6 decimals; `formatRegimeSpec` writes specs exactly (`1/3`). Specs saved by
earlier versions had nearest-rounded 6 decimals (`2.142857`), which now name the regime next to that of the exact value:
when saved data is upgraded (§7), each such number within $5\cdot 10^{-7}$ of a fraction with denominator at most 100
becomes that fraction (`upgradeRoundedSpec`); in files of earlier versions (scenario JSON, bounds CSV), which carry no
version, they are read as written. Rounding to the nearest value made both off by up to about $5\cdot 10^{-7}$ times the exponents involved,
which turned tight pairs of bounds into conflicts once algorithms were known on both sides: ST-Maximum Flow at
$|S| = n^{6/13}$ (SETH through R284 and R225 against $|S|\,|T|\,m^{1+o(1)}$, apart by $4.6\cdot 10^{-7}$, and so
CNF-SAT itself) and Set Intersection (3SUM through R72 against $U + Ns + qs$). (Earlier still, before `S`, a free
parameter of row R303 refined to $d = N^{4.8\cdot 10^{-7}}$ landed in the `L` regime of All-Edges Monochromatic Triangle
and gave APSP with real weights an upper bound $4.8 \cdot 10^{-7}$ below $1 + \omega$; such a $d$ is now
$N^{0.000001}$.)

For each problem, the **regimes of interest** are its configured regimes, the regimes of its LB facts and of its
knobs, and regimes pinned in the inspector. The UB is computed at each of them (static problems), so that a conflict
shows wherever there is one. The app computes them in two phases (`analyze(…, { complete: false })`, then
`completeAnalysis`): first at the configured, knob and pinned regimes and at the nicest 12 LB regimes per problem
(`MAX_UB_REGIMES_FIRST`; `lbRegimeKey`, below), which the views show at once, then at the other LB regimes (some
trade-off rows produce hundreds) in slices of about 30 ms, the page staying responsive between them, with "checking
the other regimes…" in the
status line; the views update when it is done. (Before November 2026 the others had no upper bound until opened in the
inspector, so a conflict there stayed hidden: with the 2026 algorithms, Set Disjointness at $U = n$,
$N = n^{17.999857}$, $s = n$, $q = n^{35.499731}$ has lower bound $n^{35.9997}$ through 3SUM and upper bound
$n^{35.9367}$; found by an independent review.) Tests and scripts use complete analyses (the default).
The upper-bound search starts from the configured, pinned and UB-knob regimes (§6.6); the others (of lower bounds or
of their knobs only) get their upper bounds from the overlay. Each regime then takes the bounds of the problem's other
regimes of interest that carry over to it (§2.3: lower bounds from the regimes inside it, upper bounds from those
around it) before its status is computed.
Per regime:

| status | condition |
|---|---|
| conflict | UB < LB − 10⁻⁷ |
| tight | LB − 10⁻⁷ ≤ UB ≤ LB + 10⁻⁴ |
| gap | UB > LB + 10⁻⁴ |
| LB only / UB only / none | |

The exponents are compared when both bounds have the same power; otherwise the powers decide: conflict if the UB's is
smaller (an algorithm in $2^{n^{0.99}}$ against SETH), gap if it is larger (SETH as $2^{0.5\,n^{0.75}}$ against the
$2^n$ algorithm).

The tolerances are asymmetric because the searches are one-sided (§6.3): an UB below an LB cannot come from search
error, so the conflict tolerance only absorbs floating-point noise (about $10^{-8}$), and improvements as small as those
of the 2026 algorithms (3SUM in $n^{1.9992}$ against the 3SUM hypothesis, APSP in $n^{2.9995}$), or an
$n^{2-\Omega(1)}$ bound (§4.2), show up as conflicts. The tight tolerance absorbs the remaining search inaccuracy. (The
first version used $10^{-3}$ for both, which hid those conflicts.) Exponents are displayed with the fewest decimals, from four to eight, such that rounding never overstates a
bound by more than $10^{-6}$ (an upper bound is never shown smaller, a lower bound never larger) and a conflict or gap
never looks like equality: 3SUM through Exact Triangle shows as $n^{1.99915}$, not $n^{1.9991}$; integers stay short.
Times of other powers show as $2^{0.5\,n^{0.75}}$ (exponential scale), $2^{0.5 \log^{0.75} n}$ (polynomial scale),
$2^{o(n)}$ / $n^{o(1)}$ (exponent 0), $2^{\omega(n^{0.75})}$ (an infinite lower bound), and the rate zero as
poly$(n)$ / polylog$(n)$.

**Order, primary regime and listing** (`src/engine/regimeRank.ts`). A problem's regimes are ordered for display; the
first is its **primary regime**: the graph's label and color, its box's bounds and the problems table show it, and the
problem's status is its status. When it fixes a constant parameter in the exponent, such as $k$ (its family: the
listed regimes that differ from it only in that parameter's value, which changes the bounds), the label,
the box and the table show the bounds as functions of the parameter instead (`src/ui/paramFamily.ts`): the bound's
exponents at the listed values (three at least) are one line $a k + b$ whose coefficients are simple fractions, or
such fractions of ω (k-OV under SETH: `≥ n^k  ≤ n^k  (k = 2–6, 8)`; Klee's measure problem:
`≤ N^(d/2)`; Min-Weight $k$-Clique, whose lower bound does not grow with $k$: `≥ n^3  ≤ n^k  (k = 3–6)`), and otherwise
only the status at each value (`k = 3: tight · k = 4–6: upper bound`, in the graph `k=3 tight · 4–6 UB`), so that
$k = 3$'s $n^{\omega}$ does not read as k-Clique's bound (k-Clique's exponents ω, 3.250035, 4.085552 and 2ω are no
such line: it shows `k = 3–6: upper bound`; it showed `≤ n^(ωk/3)` while that knob applied at every $k$, §3.6). The family also counts when only the presence of a bound
changes (Min-Weight $k$-Cycle without row R465: $n^3$ at every $k$, a lower bound at $k = 3$ only). The
color stays the primary regime's. A parameter whose values all give the same bounds (edit distance's alphabet sizes)
keeps the primary regime's line. A regime reads nicely by its parameters (the main one, growth classes such as polylog,
an unspecified constant and log-dependent parameters do not count; exponents count, and constants by their values):
whole numbers ($m = n^2$, $m = n$, $k = 4$) are tier 0; halves, thirds and quarters up to 3 ($n^{3/2}$, $n^{4/3}$) tier
1; fifths and sixths up to 4 tier 2; anything else ($n^{10/11}$, $n^{11.5}$) tier 3, which free parameters of reductions
produce by the dozen. The order (`regimeRankKey`): a conflict, wherever it is (so that a conflict always shows, also
when another regime is pinned); pinned regimes; tiers 0 and 1 together, then 2, then 3; within them how much the
regime's own bounds say (tight, a gap, a lower bound, an upper bound, none), then how much they say with the bounds it
takes from other regimes (§2.3); then the exact tier and the sum over the parameters of (denominator − 1); a simple lower-bound
exponent; configured before knob before lower-bound regimes; fewer reductions; the larger lower bound. The exponents of
bounds do not decide niceness (only break ties), so that moving a hypothesis does not make the regime shown jump.
Before, the primary regime was the one of the most severe status over all regimes, preferring fewer hops, and that
status was the problem's: ST-Maximum Flow showed $n^{2.909091}$, tight at $|S| = n^{10/11}$ (one of 129 regimes); it
now shows $n^2$, tight at one source. On the sheet with the initial settings 10 primary regimes changed, all to nicer
ones, and 102 problems are tight, 20 have a gap, 45 an upper bound only (before 101, 21, 45). (Ranking by the bounds
taken from other regimes first would let a regime that merely inherits a bound displace the nicer one that has it: on
the sheet it moved many primary regimes to larger ones; the regime's own bounds come first for that reason.)

The inspector **lists** a problem's regimes in that order, by default only (`RegimeSummary.listed`): the first; pinned,
configured and knob regimes; those of tiers 0–1; and those that the derivation (lower-bound chain or upper-bound tree)
of another problem's listed bound goes through — so an unusual regime that leads nowhere, or only to unusual regimes,
stays out — at most 12; the others on request (on the sheet with the initial settings 486 of 897 regimes, e.g. 118 of
ST-Maximum Flow's 129). The reductions used by listed bounds are the ones counted for the graph's edge colors and the
reductions table.

### 6.8 Recomputation

An analysis remembers fingerprints of its inputs. The next analysis of the same model reuses the previous lower-bound
facts when the LB knobs, constants and relevant settings are unchanged, and the previous upper-bound solver (with all
its nodes) when the UB knobs, constants, relevant settings and the regimes the search starts from (§6.6) are
unchanged; the overlay is new each time. Moving an LB slider, or switching an LB knob, therefore only redoes the
forward propagation and the new regimes of the overlay (about 0.1–0.4 s on the full sheet); UB or constant changes
redo both (about 1.1–1.5 s). These times are for the first phase (§6.7); the upper bounds at the other regimes of lower
bounds follow in the background. The analysis runs on the page's main thread (a Web Worker would keep even the first
phase off it; not done).

### 6.9 Cycles that improve a bound (`src/engine/cycles.ts`)

Going once around a cycle of reductions maps the bound of its first problem, at a regime, to a new bound of the same
problem at the same regime: a nondecreasing map $H$ of the exponent (at the bound's power: a lap that ends at another
power is not a repetition of the same map and is not extrapolated), computed by evaluating the lap's rows again with
the same free parameters (the lower-bound threshold of §6.5 per row for lower bounds; the From-time with the other
calls fixed for upper bounds). Usually going around gives nothing new, but a cycle can raise a lower bound or lower an
upper bound every time. With a 3SUM hypothesis above $n^2$, rows R65 and R68 (3SUM → Exact Weight 3-Star Detection →
3SUM) give $H(L) = 1.5L - 1$: a fixed point at 2, repelling above, so the bound grows without limit (and any 3SUM
hypothesis above 2 contradicts the polynomial-time algorithms). Following such a cycle step by step never ends
(earlier versions stopped after `maxHopsLB` reductions, at an arbitrary value such as 3.5 for Exact Weight 3-Star
Detection).

A new bound whose derivation contains an earlier, worse bound of the same problem and regime came around a cycle. The
engine then takes the limit of $y, H(y), H(H(y)), \ldots$: the nearest fixed point of $H$ in the direction of
improvement. When the calls only occur under sums, products, quotients by call-free terms, powers with call-free
exponents, maxima, ceilings, roots, absolute values, polylogarithms, O-style bounds and sums over ranges, the exponent
of a From-time is a maximum of affine functions of the calls' exponents, hence convex, so $H$ is convex for upper
bounds; the lower-bound threshold inverts such a function, so $H$ is concave for lower bounds. Then the points where
$H$ improves form an interval that ends at the nearest fixed point, and the engine finds it by steps that double from
$H(y) - y$, then bisection between the last point $H$ still improves and the first it does not, to about $10^{-13}$
(slow convergence costs no more than fast). A lower bound still improving at $10^6$, by at least as much at $2 \cdot
10^6$, grows without limit (∞): so far beyond the breakpoints of the exponent maps the improvement is affine; if it
shrinks, the search goes on (up to $10^{15}$) to the fixed point. The bound stored is one more lap from the last
improving point: a valid bound on the safe side of the fixed point (below it for lower bounds, above it for upper
bounds). Anything else around a call may break this: a `\min` (e.g. $\min(n\sqrt{T}, T^{3/2}/n)$ gives $H(U) = \min(1
+ U/2, 1.5U - 1)$, which improves on both sides of its fixed point 2, so doubling steps would jump over it), or a
floor, which switches a term on with a jump. Such cycles are repeated lap by lap instead (up to 200 laps per limit).
(An independent review of this version found both cases, which gave an upper bound of 0 instead of 2 and a lower bound
of ∞ instead of 2.) Points where $H$ does not evaluate count as not improving, which only makes the limit less
extreme. The limit's derivation records the lap and the bound it started from; the inspector shows the cycle, one lap
(↻) and the start, and the Diagnostics tab lists the cycles. A node's bound changes at most 400 times and its limits
are computed at most 8 times (later laps with re-optimized free parameters can move the fixed point a little); either
cap adds a message. Cycles that change the regime on every lap are not repetitions of one map; they are followed like
any other derivation, within the caps of §6.5 and §6.6.

### 6.10 Explanations

For a problem at a regime, the inspector lists every reduction from it (for the upper bound: `UpperBounds.explain`)
and to it (for the lower bound: `explainLower`) with what it gives there or why it gives nothing:

* upper bound: *gives this bound*; *gives* $\le N^u$ (worse, or as good through more reductions); *would take at least*
  $N^{opt}$ *even with free calls* (skipped by branch and bound); *no upper bound for* the target at the regime its
  calls land in, or *no upper bound found* there because the search stopped short at or below it (too many regimes not
  of interest in a row, see `maxDepthUB`; found by following the nodes each node read); the row's
  restriction does not hold, or it does not evaluate there; not used (status, or switched off in the settings);
* lower bound: *gives this bound*; *gives* $\ge N^l$; *no lower bound for* the source; the source's lower bounds lead
  to *other regimes* of the problem (listed); or the most frequent reason a grid point gives nothing, with the source
  bound tried: the reduction itself is too slow ($R \ge L$), its calls land in several regimes, only on instances of
  constant or polylogarithmic size, its overhead alone reaches the bound, it does not evaluate there, or it would solve
  the source below its bound (an inconsistency); not used. Times in these reasons are written in the source regime's
  parameters ("the reduction itself takes time 2^(0.5n), not less than the lower bound 2^(0.5n^0.75) it would
  transfer").

Both use the bounds the analysis found (explainLower re-derives from up to 24 lower bounds of the source, fewest
reductions first), and are computed when the list is opened.

---

## 7. User interface (`src/ui`)

* **Header:** a "skip to the search field" link for keyboard users (shown on focus); the status line (sheet,
  "built-in sheet" for the embedded one, reductions used, facts, time, "checking the other regimes…" while the
  analysis completes in the background, §6.7, and a warning when a search limit was reached; in full as a tooltip;
  "loading…" until the first analysis is shown), the panel toggles, the light/dark toggle (below), then one segment per thing acted on, its name shown once and short verbs after it:
  **Reductions** (Load… a CSV export of the sheet, Export the edited sheet, ＋ Add a reduction), **Scenario** (Load… /
  Export, JSON), **Config** (Load… / Export, JSON); then **Reset…** (forget the saved state, after a confirmation) and
  **Help**. "Load" replaces; the bounds CSV (left panel) is "imported" because it can also be added to the current
  knobs. The buttons have accessible names that include the segment ("Load reductions sheet"). Scenario and Config
  Load… refuse a file of the other kind, or JSON without the structure of theirs, before anything changes (loading the
  config's export as a scenario, or `{}`, used to leave 0 problems and 0 bounds, saved; §3.4), ask before replacing
  with a file that has problems, and offer Undo.
* **Left: scenario.** Import… / Export of a bounds CSV (§3.5), in the same segment style as the header. Then **one box
  per problem of the scenario** (§3.4: the problems listed or with knobs; with the initial settings 208 of the 254
  problems, none of them dynamic, since knobs cannot hold their bounds; there are no preset buttons, because hypotheses are adjustable reference points rather than
  fixed truths: several fell in 2026). A box's head has a collapse toggle, the problem's name (click: select it,
  Shift-click: add it to the selection; its box is marked while selected), its bounds at the primary regime colored by
  status (a count of the knobs that are on while collapsed), ＋ (a form for a new lower bound or algorithm of that
  problem: kind, regime, exponent or formula; an empty regime means the main one, or every regime for an algorithm's
  formula) and × (removes the box and its knobs, after asking). Each knob shows an on/off checkbox, LB/UB, its Bound
  ID, a slider + number (any number of decimals) or a formula field (rendered preview while editing), ×, and below
  them its regime (an inline field) and note; a knob that is off collapses to a one-line summary and its note. An
  exponent knob of an exponential-time problem (CNF-SAT, MAX-CNF-SAT) has a third line with its **power** $a$ (§2.2):
  a slider (0.05–2) and a number, and the running time rendered, e.g. $= 2^{0.5\,n^{0.75}}$ (other knobs get it only
  if they have a power other than 1, e.g. from a bounds CSV). Two labeled buttons switch a knob's form: **ƒ(n)** turns
  an exponent knob into the formula of its time at its regime ($n^{2}$, $2^{0.5\,n^{0.75}}$), and **n^c** a formula knob
  into its exponent and power there (an algorithm for every regime gets its problem's main regime, which an exponent
  needs); ƒ(n) right after n^c brings the formula back as it was, with its regime, as long as the exponent was not
  changed in between. (They were glyph buttons "ƒ" and "x" beside the "×" that removes a knob, so one mis-click turned
  a formula such as U5's $n m + n^2 \log n$ into an exponent for good.) Number fields (exponents, powers, constants,
  settings) check what is typed: a value out of range or not a number is shown as an error beside the field and not
  stored (before, it stayed in the field while the state kept the old one), a decimal comma reads as a point ("1,5"
  became 15 before), and an empty field keeps the old value. The ＋ form likewise checks the regime (every
  parameter of the problem, each once, with a valid growth) and the formula (it must give a running time at the
  regime) before it adds a bound. While a formula field has the focus, its rendered preview is drawn over the
  lines below it, so that nothing moves under the pointer (a click right after focusing a field used to miss). Above
  the boxes, a **bar** that stays at the top of the panel while it scrolls: the totals (or what the search above the
  graph found, with Clear) and Collapse all (Expand all when every box shown is collapsed; during a search both act on
  the boxes shown). The panel has no search field of its own (an earlier version had one, which only repeated the
  toolbar's). Below the boxes, **＋ Problem** (a new empty box
  with its form open, or the existing box shown). The inspector's "add here" opens a box's form prefilled with a regime.
  Constants (ω) with sliders (and the config's note under each, e.g. which bounds do not follow ω). Settings, a
  section that stays open, with the panel's scroll position, while settings change (changing a setting used to
  rebuild the panel, close the section and jump to the top). A link opens the initial-settings note. The boxes are
  rebuilt only when the knobs or boxes change, so typing and dragging never lose focus; switching a knob on or off, new analyses, the
  selection and the search update them in place (the bar is never rebuilt).
* **Search** (`src/ui/search.ts`; the text is a preference, empty after a reload): one text, typed in the field above
  the graph; case and spaces around it are ignored. A **problem** is
  found when its name contains the text (family or variation, as in the sheet or as shown: "k-clique" finds
  "$k$-Clique"); a **bound** when its Bound ID is the text (the whole ID: "U1" does not find U12) or its note or
  running-time formula contains it ("Williams", "omega"); a bound found finds its problem too. The graph outlines the
  problems found and dims the others (and the reductions that do not join two problems found); the left panel shows
  only their boxes, marks the text found in names and notes and highlights the
  bounds found (a collapsed box with bounds found is shown open until the search changes or the box is closed; nothing
  found says so); the problems table lists only the problems found, and its tab says how many ("Problems (12 of 254)";
  the reductions table keeps its own filter: the
  reduction with that ID, or reductions between problems with that name). When the bounds change during a search (an
  import, a formula edited), the search is applied again: a bound that no longer matches loses its mark (and its box
  goes if nothing else in it is found), and a collapsed box with a bound found only now is shown open. A new search,
  typed while the left panel's bar is in view, shows its first boxes right below the bar (also when hiding boxes
  made the page scroll back); clearing the search keeps the box at the top of the view in place (so after finding a
  bound, clearing shows its neighbours). **Enter** selects: for a Bound ID, its problem, and shows the bound (see the
  inspector's Bound IDs below; the left panel opens first, so that the graph centers the problem in its new size); for
  a reduction's ID ("R12"), that reduction; else the problem found that the query names best (`bestProblem`: the
  name shown equal to the query, then having it as a whole word, then the family equal to it or having it as a whole
  word, then a name containing it, then problems found through a bound only; among equals the first box of the left
  panel, then the first in the sheet: "apsp" selects Directed, Weighted APSP, "3sum" 3SUM, "clique" Min-Weight
  k-Clique). **Esc**, or the left panel's Clear (which
  puts the focus in the field without scrolling the page to it), clears the search. Shift+Enter adds to the
  selection, like Shift-click. Only names and notes whose marks change are rewritten, and the search texts are
  kept in lower case per problem and per knob, so that typing stays quick.
* **Center: graph** (Cytoscape). Nodes are problems, optionally boxed by family (the *family boxes* toggle, a
  preference, off by default since format 10 of the saved data: with the boxes drawn, a drag that starts inside a box
  moves the box, which made panning hard; problems without a family, and the only problem of a family, stand on their
  own). Node color = status (a conflict also has a thick double border, so that it does not depend on telling red from
  green; dynamic problems are drawn dashed in the theme's panel color) and label = name +
  `≥ LB ≤ UB`, both at the primary regime (§6.7; 14 px at 100 % zoom; e.g. `≥ 2^(0.5n^0.75)  ≤ 2^n`), or for a constant
  parameter in the exponent the bounds as functions of it (`≥ n^k  ≤ n^k`) or the status at each value
  (`k=3 tight · 4–6 UB`; §6.7). Edges: solid if
  the row is `ok`, dashed grey otherwise, orange if it produced a listed LB, blue if used by a listed UB, purple if
  both; a row from a problem to itself (from $k$ to $k + 1$, R463–R465) is a small loop above its node, drawn over the
  other nodes, as the layout leaves little room between them. Search field (above), focus (below), zoom buttons (−, 100 %, +, Fit; named for screen readers), re-layout; double-clicking a problem or
  reduction also shows the inspector. A mouse-wheel notch zooms about twice as much as Cytoscape's default (from Fit to
  100 % took about 39 notches). Cytoscape warns in the console about any custom wheel sensitivity, on every page load,
  so its warnings are off while the graph's renderer starts, and on again before the style is set (a mistake in the
  style still shows).
  *Selection* (`src/ui/selection.ts`): a list of problems and reductions. A click selects the item alone; Shift-click
  (or Ctrl/⌘-click) adds it or, if selected, removes it, in the graph, the tables, the inspector's links and the boxes'
  names; a click on the background clears the selection. Selected problems get a teal border and halo, their reductions
  teal edges; a problem item also remembers the regime opened in the inspector.
  *Focus* (`src/ui/focus.ts`; a preference) dims everything except the selection and: **neighbours** (the reductions of
  selected problems and the problems at their other ends; a selected reduction's two problems); **ancestors &
  descendants** (every problem with a path of reductions the engine uses to a selected problem or from it, and the
  reductions on those paths; for a selected reduction, the ancestors of its source and the descendants of its target,
  or its two problems if the engine does not use it; through the unused, dashed reductions as well, nearly every
  problem was an ancestor or descendant: the median lineage had 99 problems instead of 38); or **bound
  derivations** (for each selected problem, at the regime opened in the inspector or else the primary one: the problems
  and reductions of its best lower bound's chain, drawn thick in orange, and of its best upper bound's tree, in blue;
  purple where they meet; a bound from a knob on the problem itself marks only the problem; a legend explains the
  colors). Derivation paths come from the analysis (`boundDerivations` in `src/engine/analysis.ts`). When the focus
  mode or the selection changes while a focus mode is on, the view zooms to what it keeps (all of it in view, never
  zoomed in beyond 100 % or the current zoom; `zoomTo` in `src/ui/graph.ts`), instead of centering on the item picked.
  *Hover* (`src/ui/hover.ts`): pointing at a problem or reduction in the graph, or at a reference to it in a panel
  (box names, the inspector's problem names, reduction IDs and Bound IDs (their problems), section titles, table rows;
  also by keyboard focus), highlights
  it in the graph: an outline (a glow for a reduction) in the text color, drawn undimmed, with a reduction's two
  problems undimmed too. The view never moves for this; when the element is out of view, a label with its name at the
  edge of the graph points toward it. Views only mark references (`data-ref-problem` / `data-ref-row` attributes); one
  listener per panel reports what is pointed at.
  *Layout* (`src/ui/graphLayout.ts`): a force-directed layout of ~200 problems spreads them thin (in the first version
  problems covered about 6 % of the drawing, so labels were about 1 px tall at "fit"). Instead, each family's problems
  are packed in rows inside its box (linked problems next to each other), the boxes and the family-less problems are
  placed by a seeded force layout of the box graph (fcose in a headless Cytoscape), overlaps are removed, the
  arrangement is stretched toward the panel's shape, and boxes slide toward the center until they touch. Problems now
  cover about a third of the drawing and no two ever overlap. Node sizes depend only on the names (text measured on a
  canvas; one line reserved for the bounds), so analyses never move anything; editing formulas keeps every position;
  adding problems keeps every unchanged box exactly in place and puts new or grown boxes at the nearest free spot;
  "Re-layout" starts over with a new seed. The family-boxes toggle only shows or hides the boxes (the layout keeps each
  family together either way).
* **Light / dark mode** (`src/ui/theme.ts`; a preference): `system` (the default) follows the operating system's
  setting, also when it changes; `light` and `dark` are explicit. The header's toggle (a moon, pressed in dark mode)
  always switches to the other look, and stores `system` when the new look is the system's, so it has no state that
  looks like another one. The CSS defines the dark colors for a dark system unless `data-theme="light"` is on `<html>`,
  and for `data-theme="dark"`; a small script in the page's head applies a saved choice before anything is drawn (no
  flash of the other look), and the graph takes up the colors it uses (family titles, highlights) on every change.
* **Panels** (`src/ui/panels.ts`): the left, right and bottom panels can be resized by dragging the bars between them
  (double-click: default size; arrow keys when focused) and shown or hidden with the buttons at the start of the header
  (the fourth hides all of them, or shows them all again). Default sizes depend on the window (narrower panels on
  laptops); sizes and visibility are saved with the other preferences, and are fitted to the window when it loads or
  changes size (`fitPanelSizes`: the graph keeps at least about 320 px; saved widths from a larger screen hid the
  graph and the toolbar for good). The fourth button hides all panels and then shows again the ones that were open.
  When the graph's container changes size, a view that showed the whole graph ("Fit") is fitted again, otherwise the
  point at the center stays in place. Below 1100 px the page becomes one scrolling column: graph first, then the
  inspector, then the left panel (it was the left panel, about 32 000 px tall, before the inspector), and a selection
  made while the inspector is out of view scrolls it into view.
* **Right: inspector.**
  *Nothing selected:* status counts (dynamic problems counted apart, not as "no bound"), the edge legend, and an
  **Initial settings** side note: each group of `initialKnobs` with a switch that shows whether its knobs in the left
  panel are on, off or partly on and switches them all at once (§3.3), its description and (expandable) its knobs, and a
  **Restore initial settings** button (replaces knobs, constants and engine settings; sheet edits and pinned regimes are
  kept).
  *Problem:* signature, the limits of its parameters (§2.3, with their reasons as tooltips), the listed regimes of
  interest (§6.7: in their order, the first marked "in the graph", at most
  12; "show all" adds the others, faded, and "show fewer" goes back) with LB/UB/status (a bound taken from another
  regime of the problem, §2.3, is marked † and its regime named: "From the regime … , which this one contains"),
  derivations as numbered lists (LB: the chain back to a
  knob; UB: the bound, then the bounds its calls use, depth first and each once, a row with several calls naming their
  items, e.g. "using 2 and 3"; the limit of a cycle shows the cycle, one lap marked ↻ and the bound it started from;
  the regime opened is the one the "bound derivations" focus draws), under each a collapsed list of what every
  reduction to or from the problem gives there or why it gives nothing (§6.10), "add a lower bound / an algorithm
  here"; the problem's **hypotheses & algorithms** (its box on the left: each knob's Bound ID, kind, value, "off" if
  switched off, and note; ＋ Lower bound / ＋ Algorithm open the box's form at the main regime); "pin regime" (an
  empty spec, one that is not a regime of the problem and one already pinned are refused with a message; a pinned
  spec that the limits clip shows the regime it stands for, "→ n=1, m=2"; a knob's regime likewise, under the knob: "at
  the limits of its parameters: …").
  *Bound IDs* (in that list, in derivations that end at a knob, and in the initial-settings note) are links: a click
  shows the bound in the left panel, which is shown if hidden; a search that hides the bound's box is cleared, the box
  is opened if collapsed, and the bound is scrolled to the middle of the panel and highlighted for a moment. Pointing at
  one highlights its problem in the graph (hover).
  *Several items:* a head with their number, expand all / collapse all / clear, then one collapsible section per item
  (title: name, status and bounds, or the reduction's ends; × removes it from the selection), in the order selected.
  *Reduction:* all sheet fields rendered with KaTeX (MathML output, no fonts needed), parsed formula, status and
  messages ("lower bounds only" for a row marked `lowerOnly`, §3.3), free parameters and abbreviations, the LB facts it produced, a probe of the produced instances (target
  regime and scale of each call, RT as a time) at a chosen source regime, "disable in scenario", and an editor (modal) for
  the From/To problems, the To-time / From-time / RT / Restrictions / Randomized / Hypothesis cells and the row's config
  annotation as JSON (live parse checks; the header's Reductions "＋ Add" uses the same form). The problems are written
  `Family / Variation`, or the variation alone for a problem without a family (refused when it is only the variation of
  a problem with a family, which then has to be written in full); their cells are rewritten only when the name changed
  (names are shown with whitespace squashed, and some sheet cells break a family name over two lines).
* **Bottom tabs:** problems table (sortable, the Bounds column by lower bound, then upper bound; filtered by the
  search: the problems found by name or through their bounds), reductions table (status, an "LB only" badge for rows
  marked `lowerOnly`, and first message of
  every row, filter for unused rows), diagnostics (the size of the analysis, knob problems, cycles that improve a bound
  and their limits, inconsistencies (one line per reduction, source and message, with the number of regimes), caps
  hit, CSV warnings, model diagnostics such as signature disagreements). Selected rows are highlighted; clicks select
  as in the graph.

* **Accessibility:** a skip link to the search field; the icon buttons (remove "×", collapse "▾", zoom "−"/"+") have
  names for screen readers, not only titles; the Help dialog is modal (`aria-modal`, the focus stays in it and goes
  back to the Help button when it closes, Esc closes it); conflicts have a double border in the graph, not only a color.
  Not done: table rows and sortable headers cannot take the focus.

State is kept in one immutable `AppState` (`src/ui/state.ts`); every change goes through `Store.update(fn)`
(`src/ui/store.ts`), which compares the old and new state: a new sheet or config rebuilds the model and the analysis
synchronously (views never see a mismatched pair); a new scenario or pinned regime schedules the analysis (debounced
while a slider moves, with a busy indicator); views subscribe to the kinds of change they display. The scenario, pinned
regimes and preferences, and the sheet/config when customized, are saved in `localStorage`, together with a format
number (now 13). A save that fails (the browser's quota, about 5 million characters) is reported once. When another
tab or window of the app saves (a `storage` event), this one stops saving and shows a banner that offers to reload with
the newest data: before, two open tabs overwrote each other's work without a word (an edit of row R2 in one tab was lost
when a knob was switched in the other). Data saved by the first version (no format number) keeps its sheet and config
edits, with the config's `presets` replaced by the current `initialKnobs`, but its knobs are replaced by the new initial
scenario (a message says so). Format 2, written before Bound IDs were numbers, named the initial knobs
`<group>/<variation>` (e.g. `seth/cnf-sat`) and knobs added in the app `k…`: such a knob gets the Bound ID of the
initial knob it was (if no other knob has it yet), any other id that is not a Bound ID the next free one of its kind
after those of the knobs and of the initial settings; values and order are kept. Formats 2 to 4 numbered Bound IDs
(1–33 for the initial knobs) and Reduction IDs without letters: the validators convert them (§3.1, §3.3: an initial
knob keeps its place, e.g. 10 → U1, a user's knob gets the same ID in the scenario and in a custom config, e.g. 34 →
U368; a disabled row 106 is R106) and regime specs of knobs and pins get their fractions back (§6.7). The initial
knob groups added since the saved format (format 5: `Trivial algorithms`, `Best known algorithms`, `Non-combinatorial
algorithms`; format 6: `Saha & Ye 2026`; format 8: the OMv, Hitting Set, ∀∀∃-OV and 3-uniform hyperclique hypotheses;
format 11: the $k$-AP and Zero-Weight $k$-Clique hypotheses; `GROUPS_ADDED_IN`) are added to a saved custom config's `initialKnobs`
(unless the config has one of their knobs: the same Bound ID, read as the validator reads it, for the same problem, so
that a user's group of the same name does not hide them; a knob whose ID the config has for another knob comes without
it and gets the next free one), and their knobs, with the IDs the initial settings (built-in or that config) give them,
are appended to the saved knobs, unless a saved knob is one of them already (the same Bound ID for the same problem:
data of this version that an earlier one saved again gets nothing twice). A saved knob with one of their IDs (one the
user added, e.g. U368, or a format-4 number 34 → U368) moves to the next free ID of its kind, so that the initial
settings name the knobs of the left panel (whose links in the "Initial settings" note also check the problem); one
message says so, and which IDs moved (not before format 5, when every ID changed). The knobs a format added to groups
that existed (format 8: U370–U418, the algorithms of the problems of rows R320–R365, in `Trivial algorithms`, `Best known
algorithms` and `Non-combinatorial algorithms`; format 9: U419–U422, the algorithms of the two problems the trivial
reductions R366–R431 gave a signature; format 11: U423–U440, the algorithms of the problems of rows R432–R462, in
`Trivial algorithms`, `Best known algorithms` and `Alman & Vassilevska Williams 2026`; format 12: U441, Chan's Weighted
Depth algorithm for $d = 5, 6$, in `Best known algorithms`; `KNOBS_ADDED_IN`) are added the same way: to a group of the same label
of a saved custom config (where it has one; of several, the one holding the built-in group's other knobs, as a user's
group may have the same label), unless it has the knob (the same Bound ID for the same problem, or the same bound:
problem, kind, formula or exponent, power and regime), and to the saved knobs (recognized by what they are, as a custom
config may give them other IDs). A group is likewise there already when all its knobs are there as the same bounds (a
knob added without the Bound ID the config had for another knob gets a new one, which an earlier version saves). A custom config saved before format 8, 9, 11 or 12 also gets the built-in entries of the problems and rows that format
added (`ENTRIES_ADDED_IN`: their regimes and limits, the free parameters of the new rows, and in format 12 the
restrictions of rows R118, R434 and R463–R465 to some values of $k$) where it has none of its own. The entries a format
changed (`ENTRIES_CHANGED_IN`; format 12: k-GeomBase, listed for $k \ge 4$ only) replace those of a custom config saved
before where its entry is still the old built-in one (the same JSON up to the order of keys; an edited entry is kept).
The same table repairs a custom config saved by the first version of format 11, whose row entries were numbered before
two rows were removed: its entries for R435, R453 and R455 of then (k-GeomBase to GeomBase, 4-AP Detection to 3-uniform
4-hyperclique, Exact Triangle to the lopsided problem) give way to the built-in entry of R453 (the lopsided row's) or
go (R435, R455), unless edited, and the row now numbered R434 gets its entry as one of format 12. A custom config saved before format 7 gets the built-in limits of parameters (§2.3) for every problem that has none
in it (its own are kept); a config file loaded later is used as it is. A saved custom config is used even when it has
problems (e.g. two row annotations whose IDs became one with the letter): its valid parts, with a message, as when a
config file is loaded; one that is not a JSON object is ignored. Formats 2 and 3 limited whole upper-bound derivations with `maxDepthUB` (default 4); a saved 4
becomes the new default 3 (other values are kept). Format 10 turned the family boxes off by default: data saved before
has them off (on was the default then, so a saved "on" is no choice), later data keeps its setting. Format 11 renamed the
group `SETH` `Satisfiability Hypothesis` (`GROUPS_RENAMED_IN`): in a custom config saved before, a group with the old
label that has one of the built-in group's knobs (the same Bound ID for the same problem; a user's own group of that
name keeps it) gets the new label, and the new description if it has the old one; a saved knob of that group whose note
is the old label (the default note) gets its new default note (the built-in knob's note, or the label in a custom
config); format 2's knob names (`seth/cnf-sat`) are read with the labels of then. Upgraded data is
saved in the current format right away. Format 13 (the review of November 2026) adds the groups of SETH in its
$k$-SAT form, the low-dimension OV and (min,+)-convolution hypotheses and the October 2026 $k$-SAT algorithm (off), the
config entries of $k$-SAT, Low-Dimension OV, R169–R170, R467, R469–R470 and the limit $n \le N$ of four problems on
compressed texts (`ENTRIES_ADDED_IN`); it changes the entries of CNF-SAT, R106, R256, R344 and R350 where a custom
config still has the old built-in ones (`ENTRIES_CHANGED_IN`), the built-in knobs U14, U273, U288, U304, U307, U359 and
U417 (`KNOBS_CHANGED_IN`: a saved knob that is still the old built-in bound becomes the new one, keeping its switch and
a note of the user's own; a custom config's copy likewise; one message lists them) and the default of ω
(`CONSTANTS_CHANGED_IN`: a custom config with the old 2.371552 gets 2.371339).

---

## 8. Architecture

```
src/util/csv.ts          RFC-4180 parse/serialize (pure)
src/math/lexer.ts        LaTeX → tokens
src/math/ast.ts          Expr ADT (immutable), free identifiers, T-call positions
src/math/parser.ts       tokens → Formula (alternatives of Expr); ParseError with position
src/math/cells.ts        math spans of a cell, primary formula, legend abbreviations
src/growth/growth.ts     Growth ADT + algebra (§5); rates of running times (power, exponent)
src/growth/evaluate.ts   Expr × Env → Growth (with T-call handler and sum handling)
src/model/regime.ts      Regime ADT, spec parsing, normalization (clipped to the limits), keys, display
src/model/constraints.ts limits of parameters (§2.3, §3.7): parsing, clipping growths, display
src/model/config.ts      config + scenario schemas and validators (no dependencies)
src/model/model.ts       rows + config → Model {problems, rows (CompiledRow), diagnostics}
src/model/boxes.ts       a scenario's problem boxes (left panel; order of the bounds CSV)
src/model/boundsCsv.ts   bounds CSV ↔ knobs and boxes (§3.5)
src/engine/freeparams.ts grids and refinement
src/engine/rowEval.ts    evaluate one row at (regime, θ)
src/engine/cycles.ts     limits of cycles that improve a bound (§6.9)
src/engine/lower.ts      LB propagation; explanations for lower bounds
src/engine/upper.ts      UB fixpoint (top-down solver, overlays); explanations for upper bounds
src/engine/analysis.ts   scenario + model → Analysis (facts, per-problem summaries, statuses); derivation paths
src/engine/regimeRank.ts how nicely a regime reads; the order of a problem's regimes (the first is shown); listing
src/engine/explain.ts    per-row probes for the inspector
src/ui/selection.ts      the selection: items, clicks with Shift / Ctrl / ⌘ (pure)
src/ui/focus.ts          focus modes: what stays visible around the selection, derivation marks (pure)
src/ui/hover.ts          references to problems and reductions in panels, the off-view pointer's geometry
src/ui/search.ts         the search: problems by name, bounds by Bound ID, note or formula; reduction IDs (pure)
src/ui/theme.ts          light / dark mode: the preference and the toggle (pure)
src/ui/state.ts, store.ts  AppState, persistence, Store
src/ui/graphLayout.ts    compact layout of the graph (packing, unit placement, overlap removal; no DOM)
src/ui/panels.ts         panel sizes and visibility (validation is pure; the Panels class wires the DOM)
src/ui/graph.ts, scenarioPanel.ts, inspector.ts, tables.ts, dialogs.ts, main.ts   views (the only DOM code)
```

Dependencies only point down this list; `src/ui` is the only impure layer. All ADTs are immutable, document their
abstraction function, rep invariant and safety from rep exposure, and check their rep invariant in constructors
(`checkRep`). Invalid input (CSV, config, scenario, formulas) never throws out of the model layer: it becomes a row status
or a diagnostic. Programming errors (violated preconditions) throw `AssertionError`.

---

## 9. Extension points (RFC)

| change | what to do |
|---|---|
| add or edit a reduction | edit the sheet (or use the in-app editor), reload. If the From-time uses a free parameter, add `rows.<id>.free`; abbreviations written as `$X = ...$` in the legend are picked up automatically |
| add a problem | nothing (signature from To-time). For a source-only problem add `problems.<id>.params`; optionally `regimes` |
| new constant (e.g. α for rectangular MM) | `constants` in the config; use it in any formula |
| new hypothesis or algorithm to start with (e.g. a new paper's bounds) | a group in `initialKnobs` in the config (`"enabled": false` to offer it switched off), then `npm run bounds-sample`; in the app, ＋ in the problem's box or a bounds CSV (§3.5) |
| a list of bounds kept elsewhere (spreadsheet) | the bounds CSV (§3.5): Import in the left panel |
| new LaTeX construct | lexer token + parser rule + `Expr` case + evaluator case + tests |
| new growth class | `growth.ts` case + operation table + tests; the rest only uses the exported operations |
| filter by model (e.g. combinatorial) | extend `settings` and the row filter in `model.ts` |

---

## 10. Testing strategy

Unit tests (mocha + tsx) partition each module's input space:

* **csv**: quoting, escaped quotes, embedded newlines, CRLF/LF, empty fields, BOM, trailing newline, round trip.
* **lexer/parser**: every grammar construct; juxtaposition vs function application; subscripts; alternatives; errors
  with positions; every From-time formula in the sheet parses.
* **growth**: each operation × each kind, including unknowns and error cases; exponent extraction; the classes
  $2^{c\,\ell^a}$ (from $2^{c\,\ell^p}$ with known, unknown and negative $c$, $p$ below and above 1; $n^{\log n}$; their
  order against every kind, sums, products, powers, logarithms) and rates (`rateOf` of each kind, `rateGrowth` with a
  scale, power-first comparison with tolerances, `exponentAtPower`).
* **evaluate**: formulas from the sheet with hand-computed exponents (e.g. `2^{n/2}` with $n = \log_2 N$ gives
  $N^{1/2}$; `\sum_{i=0}^{\log n} 4^i T(n/2^i)` gives $\max(x, 2)$); rates: `2^{0.5 n^{0.75}}`, `2^{n^{0.99}} \cdot m`,
  `\tilde{O}(...)` of them, `2^{(2n)^{0.5}}`, `2^{O(n^{0.5})}` ($2^{o(n)}$), `n^{\log n}` and `2^{\sqrt{\log n}}` at a
  polynomial regime, the largest term of a sum of $2^{\sqrt{i}}$.
* **regime**: spec parsing, normalization (poly, exponential scale, degenerate), keys; growths rounded up as the key
  (1/3 → 0.333334, 4e-7 → 0.000001; noise ignored: 0.1·3 → 0.3, 1e-10 → `S`), so that one key stands for one regime;
  specs written exactly (`formatRegimeSpec`) and for regimes (`regimeNumberText`: each names its regime again), and
  the nearest-rounded specs of earlier versions turned into fractions (`upgradeRoundedSpec`);
  sub-polynomial parameters (`S`, the spec `sub`; polylogarithmic for log-dependent ones; quasi-polynomial ones
  rejected); requirements and filters (`log`/`polylog`/`subpoly` admit constants, exponents within 2e-6); times of
  every power written in TeX.
* **engine**: synthetic graphs (chain, cycle, free parameters, sums, blocked RT, mixed regimes, conflicts, caps); no
  length limit (chains of 12 reductions, for upper bounds even with no detour allowed); detours through regimes not of
  interest (within and beyond `maxDepthUB`; a pinned regime counts as of interest); the same bound for a problem inside
  other derivations, whatever the order of the queries; cycles that raise a lower bound to a finite limit or without
  limit (∞, also transferred through a reduction, including under roots of calls), a neutral cycle at the fixed point,
  a cycle that permutes parameters, whose rounded exponents would move its regime a little on every lap (one regime per
  problem, finite or infinite bound, no cap), a row from a problem to itself that raises a constant parameter (both
  bounds carried as far as its `requires` lets them, no cap), a cycle that lowers an upper bound to its limit, cycles through `\min` of calls (limits approached lap by lap, not
  jumped over: no false ∞ or conflict); the same bounds whatever the order of the rows, and after a lower bound
  changes (the search reused); a knob passed on to the nodes that read its node while it was postponed (a chain of 125
  problems); cycles through a floor; limits of cycles at regimes not of interest listed — regressions from an
  independent review of this version.
* **rates** (`test/rates.test.ts`): lower bounds $2^{c\,n^a}$ keep their power through an identity and through a call on
  an instance of another size (the exponent scaled by $s^a$), stop at a reduction that takes longer (with that reason),
  reach a polynomial-scale problem through a cheap reduction ($2^{c \log^a t}$), grow without limit around a shrinking
  cycle (∞ at power $a$); a bound of power 1 beats one of a smaller power whatever their exponents; formulas with a
  power; upper bounds pass back through an identity, come from sub- and quasi-polynomial target bounds, are as large as
  a reduction's own time, and the smaller power wins; a reduction that cannot beat a bound of a smaller power is
  skipped (with its optimistic rate); statuses across powers; invalid powers. Regressions from an independent review:
  powers computed in different ways ($(n^{0.1})^3$ against a knob of power 0.3) are one power; a log-dependent
  parameter is one class per key (the same bounds whatever the order of the rows); the formulas written for exponent
  knobs read back as the same rate (exponent 0 as o(·), every decimal kept).
* **cycle limits** (`iterationLimit`): fixed points above and below, fast and slow convergence, plateaus, growth
  without limit (geometric, linear) and fixed points far away (a plateau, an improvement that keeps shrinking beyond
  $10^6$), starts that do not improve, maps that stop evaluating; the result is always on the safe side.
  `iteratedLimit` (lap by lap) and `shapeKnown` (a call under `\min`).
* **explanations**: every verdict for upper bounds (gives the bound, a worse one, skipped, a call without bound found
  or with the search cut short below it, restriction, switched off) and for lower bounds (gives the bound, a smaller
  one, source without bound, other regimes, too slow with the source bound it holds for, switched off), in sheet
  order.
* **integration** on the real sheet: every formula cell parses; no `ok` row is lost; SETH ⇒ OV 2, $k$-OV $k$, Edit
  Distance 2, LCS 2, Diameter 2 vs 3 (sparse) 2, Subset Sum 1 (in $t$); 3SUM ⇒ GeomBase, Convolution 3SUM, geometry 2;
  APSP 3 ⇒ the dense subcubic-equivalence class 3 (Negative Triangle, Distance Product, Radius, Betweenness, Replacement
  Paths, Shortest Cycle); known algorithms make CNF-SAT, OV, 3SUM, Negative Triangle tight; an $n^{1.9}$ 3SUM algorithm
  flows backward to 3SUM′ and Convolution 3SUM; an $n^{2.9}$ Negative Triangle algorithm conflicts with the APSP
  hypothesis (also for APSP itself); **all classic hypotheses together with the known algorithms produce no conflict
  and no inconsistency** — a strong end-to-end check of the engine, the annotations and the sheet (it caught an unsound
  simplification in an early annotation of row R316, whose legend bound assumes $T$ superadditive in $m$); the initial
  scenario is consistent and every initial knob (also the switched-off ones) is valid; the 2026 algorithms of Alman and
  Vassilevska Williams conflict with the 3SUM and APSP hypotheses throughout their classes but not with SETH, OV,
  $k$-Clique or BMM, and give 3SUM in $n^{1.99915}$ through Exact Triangle (rows R53 and R60); the (min,+)-product of
  Saha and Ye (off at first) conflicts with the APSP hypothesis throughout its class, including every problem the paper
  names, and with no other hypothesis. The regimes shown (§6.7): every problem's first regime is the status's, reads
  nicely unless none of its regimes does, and is listed; ST-Maximum Flow shows $n^3$ tight at $|S| = |T| = n$ (with
  $n^2$ tight at one source among its regimes; not $n^{2.909091}$ at $|S| = n^{10/11}$ as once); the problems with more than 50 regimes list fewer than 30; an unusual regime is
  listed only as the first or when another problem's listed bound is derived through it. With the initial settings,
  every knob on, and the 2026 3SUM algorithm on: **no reduction gives a better bound than the one shown, and every
  bound inside a derivation is its problem's own bound at that regime** (a certificate that the fixpoint was reached,
  checked for every problem's primary regime with the explanations of §6.10); with the 2026 3SUM algorithm, 0-1
  Knapsack and Unbounded Knapsack have the same bound below $n^2$ (4 and 5 reductions, the second derivation through
  the first one's own), and Superadditivity Testing the lower one of $(\max, +)$ Convolution Upper Bound Testing (3
  and 2 reductions, through row R422); a 3SUM hypothesis of 2.05 makes 3SUM, Exact Weight 3-Star and
  3-Matching Detection ∞ through a cycle, without hitting any cap; SETH as $2^{0.5\,n^{0.75}}$ is kept by MAX-CNF-SAT,
  reaches no other problem, and is tight with an algorithm in $2^{0.5\,n^{0.75}}\,m$. **All hypotheses together with the
  trivial and best known algorithms (§3.6) produce no conflict** (ST-Maximum Flow, Set Intersection and CNF-SAT are
  tight; this caught the rounding of regimes to the nearest value, §6.7), and Valiant's parser and the transitive closure
  by matrix multiplication conflict with the combinatorial BMM hypothesis alone (at CFG Parsing, Combinatorial BMM,
  Combinatorial Triangle Detection and Transitive Closure). The rows R320–R365 give their textbook consequences: under
  SETH unbalanced OV, regular expression pattern matching and dynamic time warping at $m = n^{1/2}$ ($n^{1.5}$, tight),
  $k$-Dominating Set at $k = 8$ ($n^8$, tight), Bicriteria $s,t$-Path at $L = C = n$ and $n^3$ ($n^2$, $n^4$, tight) and
  approximate attention ($n^2$ at $B = \mathrm{polylog}$, $n$ at constant $B$), MAX-2-SAT in $2^{\omega n/3}$; with
  every hypothesis OuMv, Tree Edit Distance, sparse unweighted Radius, Global-Fréchet simplification, Transitive Closure
  and Dyck-2 reachability tight, and Klee's measure problem in 3D between $N^{300/201}$ and $N^{1.5}$. The trivial
  reductions R366–R431: with the initial settings Min- and Max-Weight 3-Clique, the Min-Weight 3-Cycle and the dense
  Wiener index are tight at $n^3$ (APSP hypothesis), the real-valued 3SUM, APSP and Exact Triangle at their integer
  bounds, All-Edges Exact Triangle at $n^3$, 0-1 Knapsack and Bicriteria $s,t$-Path at the target size (SETH through
  Subset Sum), 2-LCS at $n^2$, 2D Maximum Subarray as $d$-dimensional one at $n^3$, ST-Maximum Flow at $|S| = |T| = n$
  at $n^3$; with the $k$-Clique hypothesis Triangle Detection at $n^{\omega}$ (Nešetřil–Poljak); with the 2026
  algorithms 3SUM in $n^{1.998}$ through Real 3SUM (and, without the rows to the real-valued versions, $n^{1.999125}$
  through Exact Triangle in $n^{2.99825}$ from the lopsided problem, row R453; $n^{1.99915}$ from the stated $n^{2.9983}$
  alone). No pair of problems has two trivial rows, and the two shortcuts left out (§13) stay out. The rows R432–R465:
  under the $k$-AP hypothesis alone (with the algorithms) $k$-AP Detection and $k$-GeomBase at $M = n$ ($k = 4, 5, 6$,
  $k$-GeomBase's only regimes), the narrowest colour-spanning strip, the separator, covering, visibility,
  motion-planning, horizon, stabbing and double-wedge problems are tight at $n^2$ and 3-uniform 4-hyperclique at $n^4$
  (R434; nothing at $k = 5$), with no conflict, and 3-AP Detection is near-linear; with the initial settings the
  lopsided problem is tight at $n^2$, #Exact Triangle and Zero-Weight 3-Clique at $n^3$, the narrowest colour-spanning
  strip with 3 colours (R39) and Stabbing and Avoiding Segments at $n^2$, $k$-GeomBase has no lower bound, Min-, Max-
  and Zero-Weight $k$-Clique are between $n^3$ and $n^k$ and Min-Weight $k$-Cycle tight at $n^3$ for $k = 4, 5, 6$
  (R463–R465), with no regime beyond $k = 6$, Weighted Depth between $N^{3/2}$ and $N^{d/2}$ for $d = 4, 5, 6$, and
  Multiple Local Alignment has no fractional number of strings (R118, even $k$ only); with the 2026 algorithms the lopsided problem is at $n^{2 - 0.063/18}$, in conflict with the APSP hypothesis,
  and the Zero-Weight $k$-Clique hypothesis conflicts at $k = 3$ and 6 but not 4 and 5, while the $k$-Clique hypothesis
  makes Zero-Weight $k$-Clique with constant weights tight at $n^{\omega k/3}$ ($k = 3, 6$).
* **limits of parameters** (`constraints.ts`): parsing (powers in every notation, products, sums, constant factors, a
  reason; malformed ones, unknown parameters, a parameter bounding itself); clipping (none; limits that hold; a
  polynomial clipped to a polynomial or a polylogarithm; a chain, $m \le n\alpha$ lowering the limit of $t$; the
  largest term of a sum; cyclic limits without a fixed cap, to their fixed point); TeX; regimes from specs clipped with
  their normalizer kept (a degenerate one when all is clipped; a spec whose normalizer the limits clip rejected),
  `clippedRegimes`; a formula knob's filter matching clipped sizes (and not smaller ones or other regimes), its bound
  given there; the model's limits and the warnings for bad ones; an engine derivation through a reduction that claims
  impossible instances (one fact at the clipped regime). Free parameters: two grid points with the same bound are
  refined (a better point between them is found). On the real sheet: every regime of every
  problem with limits is within them, with the initial settings and with every knob on; Triangle Listing at $m = n^{1.5}$,
  $\alpha = n^{3/4}$, $t = n^2$ is tight (the primary regime with the initial settings), among fewer than 100 regimes.
* **config / initial scenario**: `initialKnobs` groups (on/off, notes, invalid entries), the old name `presets`, rows
  keyed by canonical Reduction IDs (one row annotated twice), `initialScenario` (Bound IDs per kind, the config's kept
  in canonical form, numbers of earlier versions at their place, round trip through the scenario validator), scenario
  ids (kept, converted, missing or repeated ones given new IDs after the reserved ones), disabled rows in canonical
  form, box order (`problems`: kept, blanks and repeats dropped, invalid, missing).
* **IDs**: Reduction IDs (numbers with or without R, any case, leading zeros, the `#k` of repeated IDs; other text),
  fresh R IDs; Bound IDs (`legacyBoundId`: L1–L9, 10–33 → U1–U24, lower bounds n → Ln, upper bounds from 34 →
  U(n + 334), below 10 none; canonical forms; fresh IDs per kind).
* **boxes**: order (listed problems, then first knobs), empty boxes, knob order within a box, one box for names of one
  problem written differently, the box key of known and unknown problems.
* **derivation paths**: a lower bound back along its chain, an upper bound through its tree (only the best of
  alternative reductions; several calls), bounds from a knob on the problem itself (the problem alone), missing bounds,
  unknown problems, a given or unknown regime; ties between derivations (a better exponent wins, then fewer
  reductions) and a detour through another problem and back (the direct reduction is taken).
* **no detours** on the real sheet: with the initial settings and with every initial knob on, no derivation has a
  problem at the same regime twice on a path, and Diameter's upper bound is its own algorithm's (or, with the classic
  groups only, comes straight from APSP).
* **selection and focus**: plain and additive clicks (alone keeping the item's regime; added at the end; removed),
  modifier keys, regimes of items; neighbours and lineage (ancestors and descendants with cycles and dead ends;
  selected reductions; unknown reductions), derivations united over the selection.
* **saved data**: format-2 knob ids become the Bound IDs of the initial knobs (all of the previous sample's 33 ids),
  added and other ids get new ones after them (L16, …), a taken ID is not reused, Bound IDs are kept; format 3 keeps
  text ids; format 4's numbers become the initial knobs' IDs (10 → U1, 26 → U17) or the user's (34 → U368, 35 → L35);
  the knobs of the groups added since, and those added since to groups there were, are appended with the IDs of the
  initial settings (once, with one message that counts hypotheses and algorithms: before format 5 those of formats 5, 6,
  8, 9, 11 and 12, and the user's 34, read as U368, moves to the next free ID, U442; format 5 those of formats 6, 8, 9,
  11 and 12, and a knob the user added as U368 moves to U442, which the message says; format 7 those of formats 8, 9, 11
  and 12, the hypotheses switched off, and a knob the user added as U370 moves to U442; format 8 those of formats 9, 11
  and 12, also into a custom config of format 8 with their problems' and rows' entries, the groups of format 11
  appended; format 11 the algorithm of format 12, and into its custom config the rows of format 12 and the new
  k-GeomBase entry where its own is the old one, also with its keys in another order, but not where edited; the row
  entries of the first version of format 11 (R435, R453 and R455 of then) replaced by the built-in entry of the ID or
  dropped, a user's own entry kept; saved again by the previous version, nothing is added twice)
  and so are the groups and knobs to a saved custom config (whose numbered IDs and row keys get letters; groups
  recognized by ID and problem, also next to a user's group of the same name; a knob whose ID the config has for
  another knob without it, the scenario's knob then with the config's ID; only the group of format 6 to a config of
  format 5 that has the others, none to one of format 6, nor its knobs to the scenario when the config has the group
  already) together with the problems and rows of format 8 (a new row's free parameters), a knob of format 8 once (into
  the built-in group of its label, not into a user's group of the same label) and not again after an earlier version
  saved it under the new ID it got; rounded exponents of knob
  regimes and pins become fractions again before format 5; a saved config with problems is used for its valid parts (a
  non-object is ignored); a custom config before format 7 gets the built-in limits where it has none (its own kept);
  the group renamed in format 11 (SETH) is renamed in a custom config saved before when it holds the built-in knob L1
  for CNF-SAT (a user's own group named SETH keeps its name; the old description is replaced, the user's kept), and a
  saved knob's default note "SETH" becomes the new one (the user's own note kept); format 12 is loaded as is; a saved
  `maxDepthUB` of 4 from
  formats 2 and 3 becomes 3 (other values and format 4 are kept); the focus mode is restored when valid; the family
  boxes are off before format 10 (whatever was saved) and as saved from format 10 on; the selection is not saved.
* **statuses**: conflicts as small as $10^{-4}$, and $n^{2-\Omega(1)}$ against $n^2$, are detected; floating-point noise
  is not a conflict; bounds of different powers (conflict or gap by the powers).
* **bounds of a constant parameter in the exponent** (`paramFamily.test.ts`): value lists (runs, gaps, compact forms);
  the exponent as a line in the parameter (slopes 1, 2, 1/2, ω/3 with and without ω, intercepts, constants; not on a
  line, too few points, unusual slopes); on the real sheet k-OV `≥ n^k  ≤ n^k  (k = 2–6, 8)`, k-Clique `≤ n^(ωk/3)` (both
  bounds with the k-Clique hypothesis), Klee's measure problem `≤ N^(d/2)`, 3-uniform hyperclique `≤ n^k`, Zero-,
  Min- and Max-Weight k-Clique `≥ n^3  ≤ n^k` (a lower bound that does not grow with $k$), statuses for the first two and
  Min-Weight $k$-Cycle without the rows from $k$ to $k + 1$ (tight at $k = 3$ only: in the graph `k=3 tight · 4–6 UB`)
  and for k-AP Detection (with the k-AP hypothesis `k=3 UB · 4–6 tight`), and the primary regime's line for Min-Weight
  $k$-Cycle with those rows ($n^3$ at every $k$), for problems without such a parameter and for edit distance's alphabet
  sizes (same exponents).
* **display precision** (`boundDecimals`): integers stay short, rounding never overstates a bound (1.99915, ω, 4/3),
  search noise below an integer shows as the integer, conflicts and gaps hidden by four decimals get more (only between
  bounds of the same power).
* **regressions** from an independent code review: vanishing ceilings, dependent free-parameter ranges during
  refinement, thresholds for targets normalized by slowly growing parameters, $n^{2-\Omega(1)}$ runtimes, constant
  overflow, order-independent upper bounds.
* **graph layout**: member ordering, row packing (aspect, gaps), overlap removal, nearest free spot, compaction, the
  whole layout (no overlaps, members inside their boxes, deterministic per seed, unchanged boxes stay put when problems
  are added, density above 25 % on the real sheet).
* **panels**: size clamping, validation of saved sizes, window-dependent defaults.
* **bounds CSV**: column presence and order, extra columns, Upper/Lower synonyms, exponents vs formulas (with `$...$`,
  `\Omega(...)`), regimes (given, invalid, foreign parameters, defaults), unknown and aliased problems, problems without
  a family (blank `Problem` column, written and read back), Bound IDs
  (letters in either case, blank, repeated, numbers of earlier versions, reserved ones), Enabled values, the Power column (a power with an exponent, 1, with a formula, invalid; written
  only when needed), row errors and line numbers, rows that only list a problem, problems in order of
  first appearance, write/read round trip (box by box, empty boxes), the sample file being current and reading back as
  exactly the initial scenario, merge of knobs and boxes (replace / add).
* **display**: `timePlain` (polynomial, exponential with and without a factor, infinite; powers other than 1 at both
  scales, exponent 0, the rate zero).
* **hover**: the off-view pointer (in view, partly in view on each side, outside on each side and at a corner,
  margins larger than the view), reference attributes.
* **order of regimes** (`regimeRank`): simple denominators (also of exponents stored rounded up), the tiers of numbers
  (whole, halves to quarters up to 3, fifths and sixths up to 4, others), the niceness of regimes (main parameter,
  classes and log-dependent parameters ignored; constants by value); the order (a conflict even at an unusual regime and
  before a pinned one, pinned, nice before unusual, then how much the bounds say: a gap at $m = n^{3/2}$ before an upper
  bound at $m = n^2$; tight before gap, whole numbers, configured regimes, fewer hops, the larger bound, a simple
  lower-bound exponent); which regimes list themselves; in an analysis: the primary regime and status (a nice regime
  with an upper bound only rather than an unusual one with both; a conflict wherever it is, also with another regime
  pinned), an unusual regime listed when another problem's listed bound is derived through it, and the reductions used
  counted for listed bounds only.
* **search**: the query (trimmed, lower case, blank); problems by family or variation, as in the sheet and as shown
  (TeX as plain text), in any case, not by other text or the empty query; bounds by the whole Bound ID in any case (not
  a prefix or a longer text), by a part of the note or formula; problems found by name and through bounds (a knob
  naming its problem differently: its box key), the bound with the Bound ID searched for, problems not in the sheet
  (named by a box or a knob); the problem Enter selects (the name shown equal to the query, having it as a whole word
  before as a part of one, the family equal or having it as a word, the first box then the first in the sheet among
  equals, a name before a bound; nothing found); reduction IDs (case, leading zeros, spaces; not in the model,
  a bare number, other text); the occurrences marked (several, overlapping, case).
* **theme**: dark or not and the toggle for each theme × system setting (every toggle changes the look; agreeing with
  the system means following it); saved themes restored, invalid ones replaced by `system`.
* **e2e** (Playwright, optional): the built HTML loads without console errors or warnings and without any network request, renders
  all problems with their default bounds and the initial-settings note (with switched-off knobs as compact rows), the
  bounds of a constant parameter in the exponent as functions of it or as statuses, the rows from $k$ to $k + 1$ as
  loops above their nodes (drawn over the other nodes), no
  family boxes until the toggle draws them (and takes them away again), one
  box per problem in the left panel, labels are larger than 2.5 px at "fit" (about 3 px for the 243 problems, depending
  on the layout's seed) and grow when the panels are hidden, a knob
  change through the UI updates the graph, the sample bounds CSV imports, the header's segments name every button
  after its verb and segment, the three exports download their files and an exported scenario loads back unchanged,
  a click and a Shift-click select two problems (two inspector sections), the focus modes dim fewer and fewer problems
  (off > ancestors & descendants > neighbours > bound derivations), 3SUM's own knobs make it stand alone in its
  derivations, pointing at a box's name or an inspector's reduction ID highlights that problem or reduction in the graph (and
  nothing once the pointer leaves), the light/dark toggle changes the page's colors and a second click restores them,
  the inspector renders a problem (with a derivation) and a reduction, the help dialog opens, and CNF-SAT's SETH knob
  has a power control (3SUM's has none): set to $2^{0.5\,n^{0.75}}$, its box and MAX-CNF-SAT's label show it. The
  search (the left panel has no field of its own): "U12" leaves one box with U12 marked, Enter selects its problem, the
  left panel's Clear shows every box again and puts the focus in the field; "Williams" leaves the boxes with bounds
  whose notes name him, the same problems the graph outlines; "R13" + Enter selects that reduction, "apsp" + Enter Directed, Weighted APSP. With every box collapsed (the toggle then
  reads "Expand all"), a Bound ID in the inspector's list for a problem near the end of the panel, and one in a
  derivation (GeomBase's lower bound from L2), each open their box and scroll the bound into view. Triangle Listing's
  inspector lists its four limits, and pinning `n=1, m=1.5, alpha=1, t=4` shows "→ n=1, m=1.5, alpha=0.75, t=2.25".

**Added with the review of November 2026** (one test or more per item of the review, each failing before its fix):

* **regime / engine:** `Regime.contains` (per class and exponent; constants with values only below `c?` and growing
  classes, never each other; log-dependent `C` < `L` < `P`; another normalizer or scale: no); lower bounds lifted into
  the regimes that contain theirs (not into contained ones) and upper bounds pushed down, where they can conflict, a
  regime's own better bound kept, and the reductions of a carried bound counted as used; incomplete analyses (upper
  bounds first at `MAX_UB_REGIMES_FIRST` regimes of lower bounds, then completion in slices: time up at once, after
  one regime, never; the result equals a complete analysis); the refinement's first step from the grid actually made
  (fewer values per parameter when capped: the best point between two values found at 9 to 41 points); knob formulas
  that give no running time or overflow reported, not dropped (W8); `\log n (n+m)` read as $\log n \cdot (n + m)$
  (W7, parser); a row marked `lowerOnly` carries lower bounds only, and its explanation says why; tiny free-parameter
  values shown by their size; calls on degenerate instances (a parameter beyond $N^{10^4}$) not evaluated; the ranking
  by a regime's own status first.
* **integration:** k-Clique at each $k$ ($\omega$, 3.250035, 4.085552, $2\omega$: U14 only where 3 divides $k$); SETH
  in its $k$-SAT form gives Subset Sum, Bicriteria s,t-Path (tight with Joksch at $L = C = n$ and $n^3$), the low-dimension
  OV problems and approximate attention, and the CNF form gives none of them; R106 gives Triangle Listing no upper
  bound (a gap $[n^2, n^{2.2034}]$ at $m = n^{1.5}$); BC gets no APSP algorithm (R169, R170 skipped); the 2026
  algorithms refute the Zero-Weight and Min-Weight $k$-Clique hypotheses at every $k = 3, \dots, 6$ (R458/R371 folding at
  $k = 3, 6$; R469/R470 one instance per vertex at $k = 4, 5$: $n$ times the bound at $k - 1$); the (min,+)-convolution
  hypothesis makes the knapsack-type problems tight and conflicts with the 2026 algorithms; no two trivial rows for one
  pair of problems, except a problem's rows to itself with different formulas (R463 and R469).
* **model / state / CSV:** `lowerOnly` validated; knob IDs that are not whole numbers or have the wrong letter
  replaced; one bad `disabledRows` entry dropped, not the list; scenario constants unknown or out of range dropped
  (`checkScenarioConstants`); a constant's note kept; format 13's migrations (groups and entries added, knobs and
  entries changed where still built-in, ω's old default, each reported once); bounds CSV headers in any case, a Bound
  ID of the wrong letter given a new one, a row with an open quote not read; `fitPanelSizes` (saved sizes from a larger
  window keep the graph visible).
* **e2e:** the config's export loaded as a scenario, and the scenario's as a config, are refused and change nothing;
  a group switch turns its group's knobs off and on again; a settings change keeps the section open and the panel's
  scroll position, and a value out of range is refused with a message while the field shows the current value; ω reads
  "2,5" as 2.5 and refuses 1.5; n^c and ƒ(n) give U5 its formula back; the Help dialog keeps the focus and gives it
  back; a narrow window puts the inspector after the graph, and a saved panel size too large for the window leaves the
  graph its room; a `storage` event from another tab shows the banner; the view zooms to what a focus mode keeps (every
  undimmed problem in view). Not covered by tests: the ＋ form's regime and formula checks, and pins in exported
  scenarios (checked by hand).

---

## 11. Dynamic problems (design sketch, not implemented)

A dynamic target has several time functions. A bound becomes a vector $(x_p, x_u, x_q)$ and an LB fact becomes a
constraint $g(x_p, x_u, x_q) \ge L$ (a forbidden region), which is not a single number. The natural extension:
UB knobs per function (`{T_p: ..., T_u: ..., T_q: ...}`), UB propagation through dynamic targets exactly as in §2.5 (each
call uses its function's UB), and for LBs an inspector panel that, for a user-chosen $x_p$, shows the minimum feasible
$\max(x_u, x_q)$ (one-dimensional bisection, as in §6.5). Dynamic *sources* would need multi-function LB knobs.

---

## 12. Build and run

* **Use:** open `fgc-explorer.html` in a current Chrome, Edge, Firefox or Safari. No server or network is needed.
* **Develop:** Node ≥ 18. `npm install`, `npm test` (unit + integration), `npm run build` (writes
  `dist/fgc-explorer.html`, embedding `data/FGC_Reductions_filled.csv` and `data/default-config.json`),
  `npm run typecheck`, `npm run bounds-sample` (rewrites `data/initial-bounds.csv` from the config), `npm run e2e`
  (needs Playwright: `npm install --no-save playwright && npx playwright install
  chromium`, or `playwright-core` plus `CHROMIUM_PATH=/path/to/chrome`).

## 13. Known limitations

* Rates only: at power $a$, factors $2^{o(\ell^a)}$ are invisible ($N^{o(1)}$ at power 1), and so are log-dependent
  parameters. Times between the classes $2^{c\,\ell^a}$ are rounded up to one an upper bound can state
  ($2^{O(\sqrt{n})}$ to $2^{o(n)}$, $N^{\log\log N}$ to none). A lower bound with an unknown constant, such as ETH's
  $2^{\delta n}$ for some $\delta > 0$, needs a value for the constant.
* Regime classes are coarse (§2.6). Results for sub-polynomial parameters (e.g. $d = c\log n$ for a specific $c$) are
  not distinguished; where a reduction needs logarithmic rather than polylogarithmic dimension, the sheet names the
  problem apart (Low-Dimension OV, rows R350, R467, R468), and `requires: "d=log"` (R351) only documents the need.
* Bounds carry between a problem's own regimes only when one regime contains the other at the same scale (§2.3), not
  through rescaled embeddings (a regime's instances read in another regime at another size).
* Grid search with refinement may miss the exact optimum of continuous free parameters (by about $10^{-7}$ with the
  default settings, more with fewer grid points or rounds, or when the objective has several separated maxima). Such
  misses only weaken bounds (§6.3); differences below $10^{-4}$ are reported as tight.
* Upper-bound derivations pass through at most `maxDepthUB` (default 3, at most 5) regimes in a row that are not of
  interest (§6.6); pin a regime to make it one. Lower-bound derivations have no limit except the caps (20 000 steps, 1000
  regimes per problem).
* Cycles through `\min` of calls are repeated lap by lap (200 laps per limit), so their limits may be approached only;
  cycles that move to another regime on every lap are followed step by step within the caps. A lower bound is taken to
  grow without limit when it still improves, without slowing down, at exponents beyond $10^6$.
* Rows whose formulas are prose, `??`, or use symbols that are not declared are skipped (listed in the Reductions tab).
* `O(x)` is read as $\Theta(x)$ except for constants and logarithmic coefficients (§4.2); a formula that divides by an
  $O(\cdot)$ bound would be read optimistically (none in the sheet does).
* The analysis runs on the page's main thread (no Web Worker): changes that recompute upper bounds freeze the page for
  about a second (its first phase, §6.7; the rest follows in short slices).
* With ~250 problems the whole graph cannot be read at once in a small panel: at "Fit", labels are about 3–4 px tall
  with the default panels on a 1600 × 1000 window and about 7 px with the graph alone; zoom in (100 % = 14 px) to read,
  or use a focus mode, which zooms to what it keeps. Some labels overflow their boxes (Cytoscape wraps at the
  non-breaking spaces of the bounds line), and a family box dragged onto others keeps the overlap until Re-layout,
  which arranges everything anew.
* The sheet's prose restrictions (e.g. "only for $m = \Theta(n^{3/2})$") are not read; such conditions must be added as
  `requires` annotations (row R106 has one), and a row that holds only on special instances as `lowerOnly` (§3.3: R106;
  other rows may need it).
* The engine does not know which algorithms are combinatorial: an upper bound from fast matrix multiplication passes
  backward through reductions from the `Combinatorial …` problems as from any other, although their hypotheses only
  concern combinatorial algorithms (hence the switched-off `Non-combinatorial algorithms` group, §3.6). So by default
  CFG Parsing and Transitive Closure show $n^3$ while other ω-based algorithms (Triangle Detection, RNA Folding) are on;
  switching the group on fixes that but makes the combinatorial BMM hypothesis, when switched on, conflict at CFG
  Parsing, Transitive Closure and back at Combinatorial BMM. A clean fix needs combinatorial versions of those problems.
  For the same reason the sheet has no row from Transitive Closure to Directed, Unweighted APSP (reachability is
  finiteness of distances): it would carry Zwick's $n^{2.53}$ (on by default) to Combinatorial BMM through R356.
* The engine does not know the machine model either: the rows from integer (or unweighted) problems to real-valued
  ones (R386–R391, integer inputs being real inputs) pass bounds both ways. That is sound for the "reasonable" real RAM
  of Chan, Vassilevska Williams and Xu (2022), whose algorithms convert to word-RAM algorithms on integer inputs (the
  reason they call the Real 3SUM and Real APSP hypotheses weaker than the integer ones), and the rows' Restrictions
  state this condition; an unrestricted real RAM, with unit-cost arithmetic on arbitrarily long numbers, is not covered.
  Lower bounds: the integer hypotheses reach the real-valued problems and, through their reductions R293–R306, Edge
  Triangle Counting and (#)All-Edges Monochromatic Triangle (there are no knobs for the real hypotheses themselves).
  Upper bounds: with the initial settings no integer problem changes; with the 2026 algorithms switched on, the Las
  Vegas real-input bounds (U18, U20, U23) also bound the integer problems (e.g. 3SUM in $n^{1.998}$, below the
  deterministic $n^{1.9992}$ of U17). That fits the paper: its abstract obtains the real-valued results "using known
  reductions", and an overview describes these algorithms as using only comparisons, additions and subtractions of
  reals, so they run on integer inputs (checked in the abstract and the overview only); the app does not separate Las
  Vegas from deterministic bounds anywhere.
* Rows that a chain of other trivial rows already gives with the same parameters are left out, so that no trivial
  reduction is stated twice: Negative Triangle Detection → Distance Product (R374 and R376, and R127 with R128) and
  Positive Betweenness Centrality → Betweenness Centrality (R400 and R401). Dropping them changed no bound.
* The rows R432–R465 (Nov 2026) model their sources as follows. The lopsided triangle problem is looked at only at
  $d = n^{1/18}$ and $w = n^{71/36}$, where Theorem 1 of Alman and Vassilevska Williams applies; the reduction from Exact
  Triangle (R453) was taken from the restatement by Karingula and Lovett, as the paper's Section 3 could not be read.
  Folding Zero-Weight $k$-Clique into Exact Triangle (R458) is tight only when 3 divides $k$ (as R369 for $k$-Clique);
  for other $k$, trying the one or two leftover vertices first refutes the hypothesis too, which the formula does not
  say. $k$-GeomBase is listed for $k \ge 4$ only, as Fischer considers it: for $k = 3$ it is GeomBase up to a shift of
  the lines, and the sheet has no row between the two, as either direction would hold for $k = 3$ only (and the graph
  would show it as one for every $k$). Dudek et al.'s 4-AP Detection to 3-uniform 4-hyperclique (R434) holds for $k = 4$
  only; the config restricts it to that value (`requires`), at the owner's request. Without the 3SUM-hard case $k = 3$,
  the narrowest colour-spanning strip gets its 3SUM lower bound from GeomBase directly, through R39, whose ?? were filled
  in November 2026 with the natural reduction (Abellanas et al. omit their proof). #Exact Triangle → Exact Triangle
  (R457) has no formula (no explicit exponent).
* $k$-SAT and CNF-SAT are separate problems since November 2026. A lower bound for CNF-SAT says nothing about its
  special case $k$-SAT, so the reductions that need a $k$-CNF formula with $O(n)$ clauses (sparsification: R214–R219,
  R231, R248–R250, R256, R344, and R467 to Low-Dimension OV) start from $k$-SAT, which reduces to CNF-SAT (R466); SETH
  in its $k$-SAT form (L16) implies the CNF form (L1), not conversely. Earlier versions let those rows start from
  CNF-SAT, so the Satisfiability Hypothesis (the CNF form, with $\text{poly}(n)$ clauses) gave Subset Sum, 0-1
  Knapsack, Bicriteria s,t-Path, Approximate Bichromatic Hamming Close Pair and Approximate Attention lower bounds it
  does not imply (and an earlier version of this section argued that reductions from 3-SAT are those from CNF-SAT
  restricted to width 3, which runs the wrong way for lower bounds; found by an independent review). The October 2026
  $2^{o(n)}$ algorithm for $k$-SAT (U442, off) is a rate of exponent 0 that the engine holds ($2^{o(n)}$); it refutes the
  $k$-SAT form and reaches no other problem, since every row from $k$-SAT points away from it. 3-SAT itself is not a
  problem of the sheet.
* The rows from $k$ to $k + 1$ (R463–R465: Min-Weight and Zero-Weight $k$-Clique, a new vertex; directed Min-Weight
  $k$-Cycle, colour coding) carry the cubic lower bounds of $k = 3$ (Minimum Weight Triangle, Exact Triangle) to the
  larger $k$. The config restricts them to $k = 3, 4, 5$, so that they reach the listed $k \le 6$: unrestricted, every
  new $k$ would produce the next one, up to the analysis's limit of regimes per problem. The natural $n^{k-o(1)}$ of the
  cliques are hypotheses of their own (L6, L7, L15), off by default. Row R118 needs an even clique size and is
  restricted to even $k$, which the bounds carried to $k = 5$ made visible (2.5 strings). Under the $k$-AP hypothesis
  alone, 3-uniform $k$-hyperclique has a lower bound at $k = 4$ only (R434): a row from $k$ to $k + 1$ would carry $n^4$
  to $k = 5, 6$ but not to the listed 9, 12, 30 and 300, so the sheet has none. Rows R469 and R470 go the other way
  for the weighted cliques ($k$ from $n$ instances of $k - 1$ on neighbourhoods), restricted to $k = 4, 5, 6$; they carry
  the 2026 algorithms to $k = 4, 5$ at a factor $n$, so the app shows the Min-Weight and Zero-Weight $k$-Clique
  hypotheses refuted at every listed $k$.
* The $k$-Clique-type hypotheses (L6, L7, L8, L15) list $k = 3, \dots, 6$ only. Reductions whose instances grow like
  $n^{k/3 + O(1)}$ approach the bounds of the literature only as $k$ grows: under the $k$-Clique hypothesis CFG Parsing
  gets $n^{1.581}$ here (R328's own note says $k \le 6$ caps it at $2\omega/3$), $n^{\omega - o(1)}$ in the
  literature (Abboud, Backurs, Vassilevska Williams 2015). Lists up to $k = 300$, as L13 has, were tried in November 2026:
  the bounds improve ($n^{2.348}$ for CFG Parsing), but regimes at large $k$ (e.g. $k = 300$ with $d = 200$) became the
  primary ones of several problems, which the ranking of §6.7 would have to handle first; so the lists were kept.
* The built-in algorithms (§3.6) come from a quick pass over the literature (about 30 seconds per problem): notes mark
  the randomized ones and those of medium confidence, and a few problems have only a trivial algorithm.
