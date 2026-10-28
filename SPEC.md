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

**Reduction IDs** are `R1`, `R2`, … . A whole number is read with the letter (`12`, `r12` and `R012` are all `R12`), so
sheets numbered before the letters existed keep matching the config's row annotations (§3.3) and the scenario's
disabled rows; other text is kept as written. A repeated ID is renamed (`R12#2`) with a warning.

**Problem identity.** A problem is the pair (family, variation) with surrounding whitespace removed. Variations are matched
case-insensitively (the first spelling seen is displayed). The config's `aliases` map fixes typos (e.g. *Boolean
Satisfiabiltiy* → *Boolean Satisfiability*). The id is `Family / Variation`.

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
  "constants": { "omega": { "value": 2.371552, "min": 2, "max": 3, "label": "matrix multiplication exponent" } },
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
    "R90":  { "skip": "reason shown in the UI" }
  },
  "initialKnobs": [
    { "label": "SETH", "description": "CNF-SAT needs $2^{(1-o(1))n}$ time",
      "knobs": [ { "id": "L1", /* + a scenario knob without id/enabled, §3.4 */ } ] },
    { "label": "OV hypothesis", "enabled": false, "knobs": [ ... ] }
  ]
}
```

`initialKnobs` is the scenario a first visit starts with (and **Restore initial settings** returns to): every knob of
every group, switched on unless its group has `"enabled": false`, with the group's label as its note unless the knob has
its own `note`. Each knob has a **Bound ID** `id`: `L1`, `L2`, … for lower bounds and `U1`, `U2`, … for upper bounds,
numbered separately (the built-in ones are L1–L9 and U1–U367); keep them when editing and give new knobs the next free
one of their kind, since the bounds CSV (§3.5) matches rows to knobs by them. A missing id gets the next free one of its
kind; a repeated or invalid one is reported and replaced the same way; other text is kept. A bare number is a Bound ID of
the versions before the letters, which numbered all knobs together (the initial ones 1–33, the user's after them), and
is converted by number and kind alone, so that a knob gets the same ID wherever it was saved (`legacyBoundId`): a lower
bound n becomes Ln (the initial ones L1–L9), an upper bound 10–33 U1–U24 (the IDs those initial knobs have now) and
one from 34 up U(n + 334) (after the built-in U367 of then; the Saha & Ye knobs added since have U368 and U369, and a
saved knob of the user's with one of those IDs moves to another one, see §7); an upper bound below 10 gets a new ID. The
groups are only a description of where the starting knobs come from; the UI shows them in the "Initial settings" note
(§7) but has no per-group toggles, so each knob is edited on its own. (The first version called
this section `presets`; a config using that name is still read, with a message asking to rename it.) The built-in groups
are SETH, the 3SUM and APSP hypotheses and the textbook algorithms (on), and the OV, Min-Weight $k$-Clique, $k$-Clique
and combinatorial BMM hypotheses and the algorithms of Alman and Vassilevska Williams (arXiv:2610.06783, Oct 2026: 3SUM
in $n^{1.9992}$, APSP and (min,+)-product in $n^{2.9995}$, Exact Triangle in $n^{2.9983}$, their real-valued versions in
$n^{1.998}$ / $n^{2.998}$, Mono Convolution in $n^{1.4999}$) and of Saha and Ye (arXiv:2610.20297, Oct 2026: a
simpler, faster deterministic (min,+)-product for integer entries of size $n^{O(1)}$ in $\tilde O(n^{2.9963432})$; the
group gives it to Distance Product and Directed Weighted APSP, U368–U369, and the reductions of the sheet carry it to
the rest of the APSP class, among them the problems the paper names: Negative Triangle, Replacement Paths, Second
Shortest Simple Path, Metricity, (min,+)-product verification, 2D Maximum Subarray) (off). Two more groups (on) give upper bounds for most
static problems of the sheet (§3.6): **Trivial algorithms** (160 knobs: brute force and textbook algorithms following
from the definitions) and **Best known algorithms** (182 knobs: the fastest algorithms found in the literature, with
sources); **Non-combinatorial algorithms** (off) holds Valiant's CFG parser, which conflicts with the combinatorial BMM
hypothesis through R126 (Lee's reduction), as that hypothesis only concerns combinatorial algorithms.

Row annotations are keyed by Reduction ID (in canonical form: a key `"106"` is row R106; two keys for one row are an
error). The optional `match` records the problems the annotation was written for; if
the row with that ID connects other problems (a renumbered or different sheet), the annotation is ignored with a
warning instead of being misapplied. The in-app row editor adds `match` automatically.

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
  "constants": { "omega": 2.371552 },
  "knobs": [
    { "id": "L2", "enabled": true, "problem": "3SUM / 3SUM", "kind": "LB", "regime": "n=1, M=poly", "exponent": 2 },
    { "id": "L1", "enabled": true, "problem": "Boolean Satisfiability / CNF-SAT", "kind": "LB",
      "regime": "n=log, m=polylog", "exponent": 0.5, "power": 0.75 },
    { "id": "U3", "enabled": true, "problem": "Orthogonal Vectors / OV", "kind": "UB", "formula": "n^2 d" }
  ],
  "problems": ["Orthogonal Vectors / OV", "3SUM / 3SUM", "String Similarity / Edit Distance"],
  "settings": { "maxDepthUB": 3, "gridPoints": 9, "refineRounds": 20,
                "unknownRT": "assume-negligible", "useRandomized": true, "disabledRows": [] }
}
```

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
* `settings`: `maxDepthUB` (0–12, default 3; "UB detours" in the app) bounds how many regimes in a row that are not of
  interest an upper-bound derivation passes through (§6.6); derivations have no other length limit. `gridPoints` and
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
As for the sheet, columns are found by header name (exact, surrounding spaces ignored; any order; extra columns are
ignored), the file is comma-separated UTF-8 with standard quoting, and blank rows are skipped. The boxes (§3.4) follow
the order in which problems first appear, and a row with neither `Upper or Lower` nor `Runtime` only lists its problem
(a box without bounds; its other cells are ignored, with a warning if any is filled).

| column | required | meaning |
|---|---|---|
| `Problem`, `Variation` | yes | the problem (family and variation; case-insensitive, config aliases apply) |
| `Bound ID` | yes | the knob's id: `L1`, `L2`, … for lower bounds, `U1`, `U2`, … for upper bounds (other text is accepted; letters in either case): an import in "add" mode replaces the knob with the same id. A blank or repeated one gets the next free one of its kind after all in the file, the current knobs and the initial settings (warning). A bare number is a Bound ID of the versions before the letters, converted as in §3.3 (1–9 → L1–L9, 10–33 → U1–U24, a lower bound n → Ln, an upper bound from 34 → U(n + 334)); an upper bound below 10 gets a new ID (warning) |
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
(L1–L9, U1–U367), so their rows are grouped by problem rather than sorted by id; exporting right after "Restore initial
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
  `Non-combinatorial algorithms` group (Valiant's parser, $O(g\,n^{\omega})$) does conflict with the combinatorial BMM
  hypothesis, through Lee's reduction R126, so it starts switched off.
* Known caveat about an older knob: `Known algorithms`' $k$-Clique bound $n^{\omega k/3}$ is achieved for $k$ divisible
  by 3 only; for $k = 4, 5$ the best known exponents are $\omega(1,1,2) < 3.250035$ and $\omega(1,2,2) < 4.085552$
  (`Best known algorithms`), so that knob is optimistic there.

### 3.7 The limits of parameters (`constraints`)

Every problem with two or more parameters that are not log-dependent was checked against the meaning of its
parameters (the sheet's variable notes): 120 of them have limits, each a fact about every instance, or about every
instance after a linear-time cleanup that no running time here notices (relabeling an alphabet or colors to those that
occur). The others are vectors of a dimension $d$ (OV, MIPS, Partial Match, …), points and polygons of independent sizes,
strings of independent lengths, numbers and targets (Subset Sum, Knapsack), clauses of $n$ variables, grammars and the
strings they compress (an SLP may be larger than its text), and constants ($d$, $k$).

| problems | limits | reason |
|---|---|---|
| graphs on $n$ vertices with $m$ edges (86 with no other limit: those with parameters $n, m$ or $n, m, k$, incl. the $k$-clique, $k$-cycle, model-checking and dynamic graph problems, and $\Delta$-Matching Triangles, Triangle Collection*) | $m \le n^2$ | a simple graph |
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
| strings with alphabet $\sigma$: Edit Distance, Local Alignment, the abelian squares, Longest Common Substring with don't cares, Jumbled Indexing; LCS and DTW; Multiple Local Alignment | $\sigma \le n$; $\sigma \le n + m$; $\sigma \le k n$ | only the symbols that occur count |
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
ranges.

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
limit (at most 8 times per key; the 20 000 steps cap everything else). Facts per problem are capped at 400 keys (warning
shown).

An infinite lower bound (the limit of a cycle without fixed point) transfers like any other: through a row whose calls
land in one regime and whose From-time grows without limit with the calls' cost, the target needs superpolynomial time
too ($x^* = \infty$); a row that would solve the source in polynomial time without such calls is an inconsistency.

Facts keep the power of their knob (§2.5), and one fact per (problem, key) is kept, the best: a fact of a larger power
passes through every reduction a smaller one passes through (a reduction's own time below the smaller is below the
larger) and gives at least as much, so nothing is lost. With SETH as $2^{0.5\,n^{0.75}}$, CNF-SAT and MAX-CNF-SAT
(identity, row R284) keep it, and nothing else follows: the split-and-list reductions (to OV, $k$-OV, …) take $2^{n/2}$,
and the one to Subset Sum (row R256) takes $2^{\varepsilon n}$ for a free $\varepsilon > 0$. Its range in the config is
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

A node's **candidates** are the usable rows from its problem whose `requires` hold, at each grid point θ, prepared
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
about 3 600 nodes).

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
knobs, and regimes pinned in the inspector. The UB is computed at each of them, except at LB regimes beyond the first 12
per problem (the nicest first, see below: `lbRegimeKey`; some trade-off rows produce hundreds): those are computed when
opened in the inspector.
The upper-bound search starts from the configured, pinned and UB-knob regimes (§6.6); the others (of lower bounds or
of their knobs only) get their upper bounds from the overlay.
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
problem's status is its status. A regime reads nicely by its parameters (the main one, growth classes such as polylog,
an unspecified constant and log-dependent parameters do not count; exponents count, and constants by their values):
whole numbers ($m = n^2$, $m = n$, $k = 4$) are tier 0; halves, thirds and quarters up to 3 ($n^{3/2}$, $n^{4/3}$) tier
1; fifths and sixths up to 4 tier 2; anything else ($n^{10/11}$, $n^{11.5}$) tier 3, which free parameters of reductions
produce by the dozen. The order (`regimeRankKey`): a conflict, wherever it is (so that a conflict always shows, also
when another regime is pinned); pinned regimes; tiers 0 and 1 together, then 2, then 3; within them how much the bounds say (tight, a gap, a lower bound,
an upper bound, none); then the exact tier and the sum over the parameters of (denominator − 1); a simple lower-bound
exponent; configured before knob before lower-bound regimes; fewer reductions; the larger lower bound. The exponents of
bounds do not decide niceness (only break ties), so that moving a hypothesis does not make the regime shown jump.
Before, the primary regime was the one of the most severe status over all regimes, preferring fewer hops, and that
status was the problem's: ST-Maximum Flow showed $n^{2.909091}$, tight at $|S| = n^{10/11}$ (one of 129 regimes); it
now shows $n^2$, tight at one source. On the sheet with the initial settings 10 primary regimes changed, all to nicer
ones, and 102 problems are tight, 20 have a gap, 45 an upper bound only (before 101, 21, 45).

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
redo both (about 1.1–1.5 s).

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

* **Header:** the status line (sheet, "built-in sheet" for the embedded one, reductions used, facts, time; in full as
  a tooltip), the panel toggles, the light/dark toggle (below), then one segment per thing acted on, its name shown once and short verbs after it:
  **Reductions** (Load… a CSV export of the sheet, Export the edited sheet, ＋ Add a reduction), **Scenario** (Load… /
  Export, JSON), **Config** (Load… / Export, JSON); then **Reset…** (forget the saved state, after a confirmation) and
  **Help**. "Load" replaces; the bounds CSV (left panel) is "imported" because it can also be added to the current
  knobs. The buttons have accessible names that include the segment ("Load reductions sheet").
* **Left: scenario.** Import… / Export of a bounds CSV (§3.5), in the same segment style as the header. Then **one box
  per problem** (§3.4; there are no preset buttons, because hypotheses are adjustable reference points rather than
  fixed truths: several fell in 2026). A box's head has a collapse toggle, the problem's name (click: select it,
  Shift-click: add it to the selection; its box is marked while selected), its bounds at the primary regime colored by
  status (a count of the knobs that are on while collapsed), ＋ (a form for a new lower bound or algorithm of that
  problem: kind, regime, exponent or formula; an empty regime means the main one, or every regime for an algorithm's
  formula) and × (removes the box and its knobs, after asking). Each knob shows an on/off checkbox, LB/UB, its Bound
  ID, a slider + number (any number of decimals) or a formula field (rendered preview while editing), ×, and below
  them its regime (an inline field) and note; a knob that is off collapses to a one-line summary and its note. An
  exponent knob of an exponential-time problem (CNF-SAT, MAX-CNF-SAT) has a third line with its **power** $a$ (§2.2):
  a slider (0.05–2) and a number, and the running time rendered, e.g. $= 2^{0.5\,n^{0.75}}$ (other knobs get it only
  if they have a power other than 1, e.g. from a bounds CSV). ƒ turns an exponent knob into the formula of its time
  at its regime ($n^{2}$, $2^{0.5\,n^{0.75}}$), and x a formula knob into its exponent and power there (an algorithm
  for every regime gets its problem's main regime, which an exponent needs). Above
  the boxes, a **bar** that stays at the top of the panel while it scrolls: the totals (or what the search above the
  graph found, with Clear) and Collapse all (Expand all when every box shown is collapsed; during a search both act on
  the boxes shown). The panel has no search field of its own (an earlier version had one, which only repeated the
  toolbar's). Below the boxes, **＋ Problem** (a new empty box
  with its form open, or the existing box shown). The inspector's "add here" opens a box's form prefilled with a regime.
  Constants (ω) with sliders. Settings. A link opens the initial-settings note. The boxes are rebuilt only when the
  knobs or boxes change, so typing and dragging never lose focus; switching a knob on or off, new analyses, the
  selection and the search update them in place (the bar is never rebuilt).
* **Search** (`src/ui/search.ts`; the text is a preference, empty after a reload): one text, typed in the field above
  the graph; case and spaces around it are ignored. A **problem** is
  found when its name contains the text (family or variation, as in the sheet or as shown: "k-clique" finds
  "$k$-Clique"); a **bound** when its Bound ID is the text (the whole ID: "U1" does not find U12) or its note or
  running-time formula contains it ("Williams", "omega"); a bound found finds its problem too. The graph outlines the
  problems found; the left panel shows only their boxes, marks the text found in names and notes and highlights the
  bounds found (a collapsed box with bounds found is shown open until the search changes or the box is closed; nothing
  found says so); the problems table lists only the problems found (the reductions table keeps its own filter: the
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
* **Center: graph** (Cytoscape). Nodes are problems, optionally boxed by family. Node color = status and label = name +
  `≥ LB ≤ UB`, both at the primary regime (§6.7; 14 px at 100 % zoom; e.g. `≥ 2^(0.5n^0.75)  ≤ 2^n`). Edges: solid if
  the row is `ok`, dashed grey otherwise, orange if it produced a listed LB, blue if used by a listed UB, purple if
  both. Search field (above), focus (below), zoom buttons (−, 100 %, +, Fit), re-layout; double-clicking a problem or
  reduction also shows the inspector.
  *Selection* (`src/ui/selection.ts`): a list of problems and reductions. A click selects the item alone; Shift-click
  (or Ctrl/⌘-click) adds it or, if selected, removes it, in the graph, the tables, the inspector's links and the boxes'
  names; a click on the background clears the selection. Selected problems get a teal border and halo, their reductions
  teal edges; a problem item also remembers the regime opened in the inspector.
  *Focus* (`src/ui/focus.ts`; a preference) dims everything except the selection and: **neighbours** (the reductions of
  selected problems and the problems at their other ends; a selected reduction's two problems); **ancestors &
  descendants** (every problem with a path of drawn reductions to a selected problem or from it, and the reductions on
  those paths; for a selected reduction, the ancestors of its source and the descendants of its target); or **bound
  derivations** (for each selected problem, at the regime opened in the inspector or else the primary one: the problems
  and reductions of its best lower bound's chain, drawn thick in orange, and of its best upper bound's tree, in blue;
  purple where they meet; a bound from a knob on the problem itself marks only the problem; a legend explains the
  colors). Derivation paths come from the analysis (`boundDerivations` in `src/engine/analysis.ts`).
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
  "Re-layout" starts over with a new seed. The family-boxes toggle only shows or hides the boxes.
* **Light / dark mode** (`src/ui/theme.ts`; a preference): `system` (the default) follows the operating system's
  setting, also when it changes; `light` and `dark` are explicit. The header's toggle (a moon, pressed in dark mode)
  always switches to the other look, and stores `system` when the new look is the system's, so it has no state that
  looks like another one. The CSS defines the dark colors for a dark system unless `data-theme="light"` is on `<html>`,
  and for `data-theme="dark"`; a small script in the page's head applies a saved choice before anything is drawn (no
  flash of the other look), and the graph takes up the colors it uses (family titles, highlights) on every change.
* **Panels** (`src/ui/panels.ts`): the left, right and bottom panels can be resized by dragging the bars between them
  (double-click: default size; arrow keys when focused) and shown or hidden with the buttons at the start of the header
  (the fourth hides all of them, or shows them all again). Default sizes depend on the window (narrower panels on
  laptops); sizes and visibility are saved with the other preferences. When the graph's container changes size, a view
  that showed the whole graph ("Fit") is fitted again, otherwise the point at the center stays in place. Below 1100 px
  the page becomes one scrolling column: graph first, then the panels.
* **Right: inspector.**
  *Nothing selected:* status counts, the edge legend, and an **Initial settings** side note: each group of
  `initialKnobs` with on/off, its description and (expandable) its knobs, and a **Restore initial settings** button
  (replaces knobs, constants and engine settings; sheet edits and pinned regimes are kept).
  *Problem:* signature, the limits of its parameters (§2.3, with their reasons as tooltips), the listed regimes of
  interest (§6.7: in their order, the first marked "in the graph", at most
  12; "show all" adds the others, faded, and "show fewer" goes back) with LB/UB/status, derivations as numbered lists (LB: the chain back to a
  knob; UB: the bound, then the bounds its calls use, depth first and each once, a row with several calls naming their
  items, e.g. "using 2 and 3"; the limit of a cycle shows the cycle, one lap marked ↻ and the bound it started from;
  the regime opened is the one the "bound derivations" focus draws), under each a collapsed list of what every
  reduction to or from the problem gives there or why it gives nothing (§6.10), "add a lower bound / an algorithm
  here"; the problem's **hypotheses & algorithms** (its box on the left: each knob's Bound ID, kind, value, "off" if
  switched off, and note; ＋ Lower bound / ＋ Algorithm open the box's form at the main regime); "pin regime" (a pinned
  spec that the limits clip shows the regime it stands for, "→ n=1, m=2"; a knob's regime likewise, under the knob: "at
  the limits of its parameters: …").
  *Bound IDs* (in that list, in derivations that end at a knob, and in the initial-settings note) are links: a click
  shows the bound in the left panel, which is shown if hidden; a search that hides the bound's box is cleared, the box
  is opened if collapsed, and the bound is scrolled to the middle of the panel and highlighted for a moment. Pointing at
  one highlights its problem in the graph (hover).
  *Several items:* a head with their number, expand all / collapse all / clear, then one collapsible section per item
  (title: name, status and bounds, or the reduction's ends; × removes it from the selection), in the order selected.
  *Reduction:* all sheet fields rendered with KaTeX (MathML output, no fonts needed), parsed formula, status and
  messages, free parameters and abbreviations, the LB facts it produced, a probe of the produced instances (target
  regime and scale of each call, RT as a time) at a chosen source regime, "disable in scenario", and an editor (modal) for
  the From/To problems, the To-time / From-time / RT / Restrictions / Randomized / Hypothesis cells and the row's config
  annotation as JSON (live parse checks; the header's Reductions "＋ Add" uses the same form).
* **Bottom tabs:** problems table (sortable, filtered by the search: the problems found by name or through their
  bounds), reductions table (status and first message of
  every row, filter for unused rows), diagnostics (the size of the analysis, knob problems, cycles that improve a bound
  and their limits, inconsistencies (one line per reduction, source and message, with the number of regimes), caps
  hit, CSV warnings, model diagnostics such as signature disagreements). Selected rows are highlighted; clicks select
  as in the graph.

State is kept in one immutable `AppState` (`src/ui/state.ts`); every change goes through `Store.update(fn)`
(`src/ui/store.ts`), which compares the old and new state: a new sheet or config rebuilds the model and the analysis
synchronously (views never see a mismatched pair); a new scenario or pinned regime schedules the analysis (debounced
while a slider moves, with a busy indicator); views subscribe to the kinds of change they display. The scenario, pinned
regimes and preferences, and the sheet/config when customized, are saved in `localStorage` (failures are ignored),
together with a format number (now 7). Data saved by the first version (no format number) keeps its sheet and config
edits, with the config's `presets` replaced by the current `initialKnobs`, but its knobs are replaced by the new initial
scenario (a message says so). Format 2, written before Bound IDs were numbers, named the initial knobs
`<group>/<variation>` (e.g. `seth/cnf-sat`) and knobs added in the app `k…`: such a knob gets the Bound ID of the
initial knob it was (if no other knob has it yet), any other id that is not a Bound ID the next free one of its kind
after those of the knobs and of the initial settings; values and order are kept. Formats 2 to 4 numbered Bound IDs
(1–33 for the initial knobs) and Reduction IDs without letters: the validators convert them (§3.1, §3.3: an initial
knob keeps its place, e.g. 10 → U1, a user's knob gets the same ID in the scenario and in a custom config, e.g. 34 →
U368; a disabled row 106 is R106) and regime specs of knobs and pins get their fractions back (§6.7). The initial
knob groups added since the saved format (format 5: `Trivial algorithms`, `Best known algorithms`, `Non-combinatorial
algorithms`; format 6: `Saha & Ye 2026`; `GROUPS_ADDED_IN`) are added to a saved custom config's `initialKnobs`
(unless the config has one of their knobs: the same Bound ID, read as the validator reads it, for the same problem, so
that a user's group of the same name does not hide them; a knob whose ID the config has for another knob comes without
it and gets the next free one), and their knobs, with the IDs the initial settings (built-in or that config) give them,
are appended to the saved knobs, unless a saved knob is one of them already (the same Bound ID for the same problem:
data of this version that an earlier one saved again gets nothing twice). A saved knob with one of their IDs (one the
user added, e.g. U368, or a format-4 number 34 → U368) moves to the next free ID of its kind, so that the initial
settings name the knobs of the left panel (whose links in the "Initial settings" note also check the problem); one
message says so, and which IDs moved (not before format 5, when every ID changed). A custom config saved before format
7 gets the built-in limits of parameters (§2.3) for every problem that has none in it (its own are kept); a config file
loaded later is used as it is. A saved custom config is used even when it has
problems (e.g. two row annotations whose IDs became one with the letter): its valid parts, with a message, as when a
config file is loaded; one that is not a JSON object is ignored. Formats 2 and 3 limited whole upper-bound derivations with `maxDepthUB` (default 4); a saved 4
becomes the new default 3 (other values are kept). Upgraded data is saved in the current format right away.

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
  a cycle that lowers an upper bound to its limit, cycles through `\min` of calls (limits approached lap by lap, not
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
  nicely unless none of its regimes does, and is listed; ST-Maximum Flow shows $n^2$ tight at one source (not
  $n^{2.909091}$ at $|S| = n^{10/11}$); the problems with more than 50 regimes list fewer than 30; an unusual regime is
  listed only as the first or when another problem's listed bound is derived through it. With the initial settings,
  every knob on, and the 2026 3SUM algorithm on: **no reduction gives a better bound than the one shown, and every
  bound inside a derivation is its problem's own bound at that regime** (a certificate that the fixpoint was reached,
  checked for every problem's primary regime with the explanations of §6.10); with the 2026 3SUM algorithm, 0-1
  Knapsack, Unbounded Knapsack and Superadditivity Testing have the same bound below $n^2$ (4, 5 and 6 reductions,
  each derivation through the next one's own); a 3SUM hypothesis of 2.05 makes 3SUM, Exact Weight 3-Star and
  3-Matching Detection ∞ through a cycle, without hitting any cap; SETH as $2^{0.5\,n^{0.75}}$ is kept by MAX-CNF-SAT,
  reaches no other problem, and is tight with an algorithm in $2^{0.5\,n^{0.75}}\,m$. **All hypotheses together with the
  trivial and best known algorithms (§3.6) produce no conflict** (ST-Maximum Flow, Set Intersection and CNF-SAT are
  tight; this caught the rounding of regimes to the nearest value, §6.7), and Valiant's parser conflicts with the
  combinatorial BMM hypothesis alone (at CFG Parsing, Combinatorial BMM and Combinatorial Triangle Detection).
* **limits of parameters** (`constraints.ts`): parsing (powers in every notation, products, sums, constant factors, a
  reason; malformed ones, unknown parameters, a parameter bounding itself); clipping (none; limits that hold; a
  polynomial clipped to a polynomial or a polylogarithm; a chain, $m \le n\alpha$ lowering the limit of $t$; the
  largest term of a sum; cyclic limits without a fixed cap, to their fixed point); TeX; regimes from specs clipped with
  their normalizer kept (a degenerate one when all is clipped; a spec whose normalizer the limits clip rejected),
  `clippedRegimes`; a formula knob's filter matching clipped sizes (and not smaller ones or other regimes), its bound
  given there; the model's limits and the warnings for bad ones; an engine derivation through a reduction that claims
  impossible instances (one fact at the clipped regime). Free parameters: two grid points with the same bound are
  refined (a better point between them is found). On the real sheet: every regime of every
  problem with limits is within them, with the initial settings and with every knob on; Triangle Listing's primary
  regime is $m = n^{1.5}$, $\alpha = n^{3/4}$, $t = n^2$, tight, among fewer than 100 regimes.
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
  added and other ids get new ones after them, a taken ID is not reused, Bound IDs are kept; format 3 keeps text ids;
  format 4's numbers become the initial knobs' IDs (10 → U1, 26 → U17) or the user's (34 → U368, 35 → L35); the
  knobs of the groups added since are appended with the IDs of the initial settings (once, with one message: before
  format 5 those of formats 5 and 6, and the user's 34, read as U368, moves to U370; format 5 those of format 6,
  switched off, and a knob the user added as U368 moves to U370, which the message says; saved again by the previous
  version, nothing is added twice) and so are the groups to a saved custom config (whose numbered IDs and row keys get
  letters; recognized by ID and problem, also next to a user's group of the same name; a knob whose ID the config has
  for another knob without it, the scenario's knob then with the config's ID; only the group of format 6 to a config
  of format 5, none to one of format 6, nor its knobs to the scenario when the config has the group already); rounded
  exponents of knob regimes and pins become fractions again before format 5; a saved config with problems is used for
  its valid parts (a non-object is ignored); a custom config before format 7 gets the built-in limits where it has none
  (its own kept); format 7 is loaded as is; a saved `maxDepthUB` of 4 from
  formats 2 and 3 becomes 3 (other values and format 4 are kept); the focus mode is restored when valid; the selection
  is not saved.
* **statuses**: conflicts as small as $10^{-4}$, and $n^{2-\Omega(1)}$ against $n^2$, are detected; floating-point noise
  is not a conflict; bounds of different powers (conflict or gap by the powers).
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
  `\Omega(...)`), regimes (given, invalid, foreign parameters, defaults), unknown and aliased problems, Bound IDs
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
* **e2e** (Playwright, optional): the built HTML loads without console errors and without any network request, renders
  all problems with their default bounds and the initial-settings note (with switched-off knobs as compact rows), one
  box per problem in the left panel, labels are larger than 3 px at "fit" and grow when the panels are hidden, a knob
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
  not distinguished.
* Grid search with refinement may miss the exact optimum of continuous free parameters (by about $10^{-7}$ with the
  default settings, more with fewer grid points or rounds, or when the objective has several separated maxima). Such
  misses only weaken bounds (§6.3); differences below $10^{-4}$ are reported as tight.
* Upper-bound derivations pass through at most `maxDepthUB` (default 3) regimes in a row that are not of interest
  (§6.6); pin a regime to make it one. Lower-bound derivations have no limit except the caps (20 000 steps, 400
  regimes per problem).
* Cycles through `\min` of calls are repeated lap by lap (200 laps per limit), so their limits may be approached only;
  cycles that move to another regime on every lap are followed step by step within the caps. A lower bound is taken to
  grow without limit when it still improves, without slowing down, at exponents beyond $10^6$.
* Rows whose formulas are prose, `??`, or use symbols that are not declared are skipped (listed in the Reductions tab).
* `O(x)` is read as $\Theta(x)$ except for constants and logarithmic coefficients (§4.2); a formula that divides by an
  $O(\cdot)$ bound would be read optimistically (none in the sheet does).
* The analysis runs on the page's main thread: changes that recompute upper bounds freeze the page for about a second.
* With ~210 problems the whole graph cannot be read at once in a small panel: at "Fit", labels are about 4 px tall with
  the default panels on a 1600 × 1000 window and about 7 px with the graph alone; zoom in (100 % = 14 px) to read.
* The sheet's prose restrictions (e.g. "only for $m = \Theta(n^{3/2})$") are not read; such conditions must be added as
  `requires` annotations (row R106 has one).
* The engine does not know which algorithms are combinatorial: an upper bound from fast matrix multiplication passes
  backward through reductions from the `Combinatorial …` problems as from any other, although their hypotheses only
  concern combinatorial algorithms (hence the switched-off `Non-combinatorial algorithms` group, §3.6).
* The built-in algorithms (§3.6) come from a quick pass over the literature (about 30 seconds per problem): notes mark
  the randomized ones and those of medium confidence, and a few problems have only a trivial algorithm.
