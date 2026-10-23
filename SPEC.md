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
the tool reports is an exponent, i.e. it ignores $N^{o(1)}$ factors (polylogs, $2^{O(\sqrt{\log n})}$, ...).

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

### 2.4 Bounds (facts)

* **LB fact** $(P, \rho, L)$: for every $\varepsilon > 0$, $P$ has no algorithm running in time $N^{L-\varepsilon}$ on
  instances of regime $\rho$.
* **UB fact** $(P, \rho, U)$: $P$ has an algorithm running in time $N^{U+o(1)}$ on instances of regime $\rho$.

Here $N$ is the normalizer of $\rho$ (or $2^{\text{param}}$ for exponential-scale regimes).

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

**Upper bounds (backward).**

$$UB(P,\rho) = \min\Big(\text{knobs},\ \min_{\text{rows } P \to Q,\ \theta}\ \max\big(g_\theta(UB(Q,\rho'_c)),\ R_\theta\big)\Big)$$

where $g_\theta(\cdot)$ substitutes each call's own target UB. *Soundness:* composing the algorithms.

The best LB for $(Q, \rho')$ is the maximum over all derivations, the best UB the minimum.

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
  So "OV with $d = \Theta(\log n)$" and "$d = \text{polylog}(n)$" are the same regime (the usual convention).
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
  row 171, "with $N = O(m + k\cdot 2^{n/(2k)})$". Spans that do not parse (e.g. `$n = \#\text{variables}$`) are ignored.
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
    }
  },
  "rows": {
    "1":   { "match": { "from": "Boolean Satisfiability / CNF-SAT", "to": "Orthogonal Vectors / k-OV" },
             "free": [ { "name": "k", "type": "int", "values": [2, 3, 4, 5, 6, 8] } ] },
    "103": { "free": [ { "name": "t", "type": "size", "base": "n", "min": 0, "max": 1 } ] },
    "303": { "free": [ { "name": "delta", "type": "real", "min": 0, "max": 1 },
                       { "name": "d", "type": "size", "base": "n", "min": "0", "max": "(1-\\delta)/2" } ] },
    "316": { "from": "O(c \\cdot T(n, m/t))", "free": [ ... ], "note": "a sound bound for the sheet's sum over color classes" },
    "90":  { "skip": "reason shown in the UI" }
  },
  "initialKnobs": [
    { "label": "SETH", "description": "CNF-SAT needs $2^{(1-o(1))n}$ time",
      "knobs": [ { "id": "1", /* + a scenario knob without id/enabled, §3.4 */ } ] },
    { "label": "OV hypothesis", "enabled": false, "knobs": [ ... ] }
  ]
}
```

`initialKnobs` is the scenario a first visit starts with (and **Restore initial settings** returns to): every knob of
every group, switched on unless its group has `"enabled": false`, with the group's label as its note unless the knob has
its own `note`. Each knob has a **Bound ID** `id`, a number like the sheet's Reduction IDs (the built-in ones are 1–33);
keep them when editing and give new knobs the next free number, since the bounds CSV (§3.5) matches rows to knobs by
them. A missing id gets the next free number; a repeated or invalid one is reported and replaced the same way. The
groups are only a description of where the starting knobs come from; the UI shows them in the "Initial settings" note
(§7) but has no per-group toggles, so each knob is edited on its own. (The first version called
this section `presets`; a config using that name is still read, with a message asking to rename it.) The built-in groups
are SETH, the 3SUM and APSP hypotheses and the textbook algorithms (on), and the OV, Min-Weight $k$-Clique, $k$-Clique
and combinatorial BMM hypotheses and the algorithms of Alman and Vassilevska Williams (arXiv:2610.06783, Oct 2026: 3SUM
in $n^{1.9992}$, APSP and (min,+)-product in $n^{2.9995}$, Exact Triangle in $n^{2.9983}$, their real-valued versions in
$n^{1.998}$ / $n^{2.998}$, Mono Convolution in $n^{1.4999}$) (off).

Row annotations are keyed by Reduction ID. The optional `match` records the problems the annotation was written for; if
the row with that ID connects other problems (a renumbered or different sheet), the annotation is ignored with a
warning instead of being misapplied. The in-app row editor adds `match` automatically.

Free-parameter types:

| type | meaning | grid |
|------|---------|------|
| `int` | integer constant | the listed `values` |
| `real` | real constant in `[min, max]` | `gridPoints` evenly spaced values incl. endpoints, then refinement |
| `size` | the quantity `base^τ`, `τ ∈ [min, max]` | as `real`, on `τ` |
| `fixed` | a constant with the given `value` (`null` = unknown constant) | one point |

`min`/`max` may be LaTeX expressions in previously listed free parameters (row 303). Names are canonical identifiers
(§4.3), e.g. `delta`, `B_med`, `c_epsilon`.

### 3.4 Scenario (importable/exportable; autosaved in the browser)

```jsonc
{
  "version": 1,
  "constants": { "omega": 2.371552 },
  "knobs": [
    { "id": "2", "enabled": true, "problem": "3SUM / 3SUM", "kind": "LB", "regime": "n=1, M=poly", "exponent": 2 },
    { "id": "12", "enabled": true, "problem": "Orthogonal Vectors / OV", "kind": "UB", "formula": "n^2 d" }
  ],
  "problems": ["Orthogonal Vectors / OV", "3SUM / 3SUM", "String Similarity / Edit Distance"],
  "settings": { "maxHopsLB": 8, "maxDepthUB": 4, "gridPoints": 9, "refineRounds": 20,
                "unknownRT": "assume-negligible", "useRandomized": true, "disabledRows": [] }
}
```

* `id` is the knob's Bound ID (§3.3): kept as given (a number, or other text), except that a missing or repeated one
  gets the next number after the largest one given (a repeated one is reported).
* `problems` (optional) orders the left panel's **boxes**, one per problem (`src/model/boxes.ts`): first the listed
  problems, then those of knobs not listed, in the order of their first knob; each box holds its problem's knobs in
  their order, and names of one problem (any case or spacing) share a box. A listed problem without knobs is an empty
  box (in the example, Edit Distance). Blank and repeated entries are dropped. Edits in the panel store the complete
  order, so a box stays when its last knob is removed; the initial scenario lists nothing (its boxes follow its knobs:
  first the hypotheses' problems, then the algorithms-only ones).
* A knob has exactly one of `exponent` (a number, slider in the UI) or `formula` (a LaTeX runtime in the problem's
  parameters and the constants, e.g. `n^{\omega k/3}`).
* LB knobs need a `regime`; UB knobs without a `regime` apply to every regime (the formula is evaluated there).
* Regime specs: `param=growth` pairs. Parameters not mentioned take the problem's first configured regime, else the
  defaults (`M=poly`, everything else `1`).

```
growth := number | number '/' number      polynomial exponent (n=1, m=2, m=4/3)
        | 'log' [':' number]              c·log2 N (default c = 1)
        | 'polylog'                       non-constant, (log N)^{O(1)}
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
| `Bound ID` | yes | the knob's id, a number like the sheet's Reduction IDs (other text is accepted): an import in "add" mode replaces the knob with the same id. A blank or repeated one gets the next free number after all numbers in the file and in the current knobs (warning) |
| `Upper or Lower` | yes | `Upper` (algorithm) or `Lower` (hypothesis); also `UB`/`LB`, `upper bound`/`lower bound`, `algorithm`/`hypothesis`, any case |
| `Runtime` | yes | a plain number is an **exponent** of the regime's main parameter N (a slider; e.g. `2`, or `1` for CNF-SAT, whose N is $2^n$); anything else is a LaTeX runtime in the problem's parameters and the constants, with or without `$...$` (`n^2 d`, `2^n \cdot m`, `\min(n^{\omega}, m^{2\omega/(\omega+1)})`). A lower bound may be written `\Omega(...)` or `n^{2-o(1)}` |
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
the same format (formulas in `$...$`, exponents as numbers, all eight columns); reading it back gives the same boxes and
knobs. The initial knobs keep their config ids 1–33, so their rows are grouped by problem rather than sorted by id;
exporting right after "Restore initial settings" gives exactly `data/initial-bounds.csv`, the sample file (regenerate it
with `npm run bounds-sample` after changing `initialKnobs`; a test fails while it is out of date). Knobs added in the
app also get the next free number, and each knob shows its Bound ID (`#7`).

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
| `O(x)`, `O_k(x)`, `\Theta(x)` | the growth of `x` up to a constant factor: invisible for polynomial growths, but `O(c)` is an *unknown* constant (so `n^{O(1)}` has an unknown exponent and `2^{O(k)}` is a constant) and `O(\log n)` has an unknown coefficient (so `2^{O(\log n)}` has an unknown exponent) |
| `\tilde{O}(x)` | `x` times a polylogarithmic factor |
| `o(1)` | the constant 0 (only meaningful in exponents: `n^{1+o(1)}`); `O(0)` is also 0 (e.g. `O(\gamma)` with $\gamma \to 0$) |
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

| kind | denotes | exponent |
|---|---|---|
| `const(v)` | $\Theta(1)$; `v` the value if known (`null`: unknown constant) | 0 |
| `log(p, c)` | $\Theta((\log N)^p)$, $p \in [0,\infty]$; $p=0$: $(\log N)^{o(1)}$ but $\omega(1)$ ($\log\log N$, $\alpha(N)$); $p=\infty$: $N^{o(1)}$ but $\omega(\text{polylog})$; `c` = coefficient of $\log_2 N$ when $p = 1$ (may be negative: used only as an exponent of 2) | 0 |
| `poly(e)` | $N^{e\pm o(1)}$, $e \ne 0$ | $e$ |
| `polyUnknown` | $N^{\Theta(1)}$ with unknown exponent (from `n^{O(1)}`, `\text{poly}(n)`) | error |
| `super` | $N^{\omega(1)}$ | $+\infty$ |

**Rep invariant.** `poly.e` finite and $\ne 0$; `log.p` in $[0, \infty]$; `log.c` is `null` unless `p == 1`;
negative `c` only if `p == 1`; `const.v` finite if non-null. Checked by every constructor (error messages are only built
on failure, because growths are created in the inner loops).

**Operations** (each total; errors are thrown, never silently mapped to 0):

| op | rule (summary) |
|---|---|
| `add(a,b)`, `max` | exact on two constants and on two $\Theta(\log N)$ quantities with known coefficients; otherwise the larger in the order (negative log quantity) < (negative constant) < 0 < poly(<0) < positive const < log (by p) < poly(>0) (by e) < super; `polyUnknown` absorbs anything ≤ poly |
| `min` | the smaller; `min(poly(e), polyUnknown) = poly(e)` |
| `sub(a,b)` | `a` if `b` is strictly smaller; consts and log coefficients subtract exactly; equal-order `poly` stays (assumes the difference is still $\Theta(a)$); `b > a` is an error |
| `mul` | exponents add (a sum of 0 gives `log(∞)`, i.e. $N^{o(1)}$); log powers add; constants multiply coefficients |
| `inv` | `poly(e) → poly(-e)`; `const(v) → const(1/v)`; `log → const(null)` (an $o(1)$ quantity is bounded by a constant) |
| `pow(a, k)` (k a known constant) | `poly(e·k)`, `log(p·k)`, `const(v^k)` |
| `pow(a, k)` (k unknown constant) | `poly → polyUnknown`, `log → log(p unknown)`, `const → const(null)` |
| `exp2(x)` | `const(v) → const(2^v)`; `log(1, c) → poly(c)`; `log(1, null) → polyUnknown`; `log(p<1) → log(∞)`; `log(p>1) → super`; `poly(e>0) → super`; `poly(e<0) → const` |
| `b^x` (b a known positive constant) | `exp2(x · log2 b)` (e.g. `4^i`, `(1/4+\gamma)^i`) |
| `log2(x)` | `poly(e) → log(1, e)`; `log(p) → log(0)`; `const(v) → const(log2 v)`; `polyUnknown → log(1, null)` |
| `ceil`, `floor`, `binom` | exact on known constants; $\lceil N^{-e} \rceil = 1$, $\lfloor N^{-e} \rfloor = 0$; otherwise identity / `a^b` |
| overflow | a constant that is not finite (e.g. `10^{400}`) is an EvalError, not a value |

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
growths, free parameters, then abbreviations in dependency order), evaluates the RT exponent (or `null` = unknown), and
chooses the first From-time alternative that evaluates. `evaluateF(prepared, value, onSite)` then evaluates the From-time
with every oracle call valued by `value(targetRegime, scale)` (the caller decides: $N^{s x}$ for the LB threshold
search, the target's UB for upper bounds) and reports each call site (target regime and scale) to `onSite`.
Normalizations of call arguments are cached per prepared row, since threshold searches re-evaluate the same calls.

A call is **degenerate** when no argument can be a normalizer (all constant or polylog). Its value is $N^{o(1)}$ if the
target has `timeType: "poly"`, and the evaluation fails otherwise (an exponential-time target on a polylog-size instance
may be superpolynomial).

### 6.3 Free parameters (`src/engine/freeparams.ts`)

The grid is the Cartesian product of each parameter's grid, capped at 400 points (the per-parameter resolution is
reduced when a row has several continuous parameters). After the grid search, the best point is **refined**
`refineRounds` times (default 20, at most 40) by a coordinate-wise zoom (two probes per parameter in a window that halves
each round) over the continuous parameters. For LBs the optimization is done separately for each target-regime key, and only
for keys reached by at least two grid points: when the free parameters move the produced regime (e.g. the trade-offs of
Kopelowitz–Pettie–Porat), every grid point is a separate fact and a perturbed point would land in another regime anyway.
For UBs, refinement likewise requires two grid points whose calls land in the same target regimes. When a refinement
step moves a parameter, later parameters whose ranges depend on it (e.g. $b \in [0, 1-a]$) are clamped into their new
ranges.

Optima at a *blocked boundary* are common (e.g. row 192: the LB tends to 2 as $B_{\text{med}} \to 1$, but at
$B_{\text{med}} = 1$ the reduction runtime equals the lower bound). Refinement approaches such suprema to about
$10^{-7}$ with the default settings (2.4 × 10⁻⁴ with the first version's 10 rounds).

The search is **one-sided**: any $\theta$ in the allowed ranges gives a valid implication, so a derived LB is the
supremum over $\theta$ approached from below, and a derived UB the infimum approached from above. Search errors can
therefore only weaken bounds, never produce a conflict; §6.7 relies on this.

### 6.4 Sums

`\sum_{i=a}^{b} f(i)`:
* `b` a known constant ≤ 64: exact sum over integers.
* `b` of class $\Theta(\log N)$ (known coefficient $c$, or unknown, which uses $c_{\max} = 8$): with $i = \theta\log_2 N$,
  the sum has $O(\log N)$ terms, so its exponent is $\max_{\theta \in [0, c]}$ of the term's exponent (grid of
  `gridPoints` values, endpoints included, then 30 halving rounds of local search, i.e. $\theta$ to about $2^{-30}c$).
  This sampled maximum is the one search that may err on the unsafe side (an underestimate), which is why it is made
  far more precise than the conflict tolerance of §6.7.
  With an unknown coefficient the evaluation fails if the term is still growing at $c_{\max}$.
* Anything else (unbounded `\sum_v`, polynomially many terms) fails, which forces the next alternative.

### 6.5 Lower-bound propagation (`src/engine/lower.ts`)

```
facts := best LB per (problem, key);  queue := instantiated LB knobs (hops = 0)
while queue not empty and pops < 20 000:
  f := pop (FIFO);  skip if superseded or f.hops = maxHopsLB
  for each ok row r with source f.problem:
    for each grid point θ (+ refinement per key):
      evaluate r at (f.regime, θ); require one shared non-degenerate target key and R < f.L
      x* := smallest x in [0, X] with g(x) ≥ f.L − 1e-9, X = max(64, 2(L+1)/s_min)
            (Illinois regula falsi with interleaved bisection, to 1e-9; s_min = smallest call scale)
      if g(X) < f.L: record an inconsistency (P solvable below its LB whatever Q costs)
    for each key: if x* > current + 1e-9: store fact (provenance: row, θ, parent f), enqueue
```

Termination: each fact is popped at most once per improvement; improvements are > 1e-9 and capped. Facts per problem are
capped at 400 keys (warning shown).

### 6.6 Upper-bound evaluation (`src/engine/upper.ts`)

`ub(P, ρ, depth)` is a memoized recursion (memo key: problem, regime key, depth) implementing §2.5 with `depth` hops at
most (`maxDepthUB`, default 4). Depth strictly decreases, so the recursion terminates even on cyclic graphs, and the
result for a key is a pure function of the knobs (independent of query order). UB knobs are evaluated by substituting the
regime's growths into the knob formula. *Branch and bound* keeps it fast without changing any result: a candidate
(row, θ) is skipped when its reduction runtime, or its From-time with every call worth $N^0$, already reaches the best
bound found so far. (Derivations may need no knob at all: e.g. row 103 with $t = n$ solves Mono Convolution by FFT and
only calls 3SUM on constant-size instances.)

### 6.7 Keys, regimes of interest, status

Key encoding per parameter: `p<e>` (polynomial exponent, rounded to 1e-6), `E` (the exponential-scale normalizer),
`L` (non-constant sub-polynomial), `c<v>` / `c?` (constant), and for log-dependent parameters only the class
`C`/`L`/`P`/`X`.

For each problem, the **regimes of interest** are its configured regimes, the regimes of its LB facts and of its knobs,
and regimes pinned in the inspector. The UB is computed at each of them, except at LB regimes beyond the first 12 per
problem (fewest hops first; some trade-off rows produce hundreds): those are computed when opened in the inspector.
Per regime:

| status | condition |
|---|---|
| conflict | UB < LB − 10⁻⁷ |
| tight | LB − 10⁻⁷ ≤ UB ≤ LB + 10⁻⁴ |
| gap | UB > LB + 10⁻⁴ |
| LB only / UB only / none | |

The tolerances are asymmetric because the searches are one-sided (§6.3): an UB below an LB cannot come from search
error, so the conflict tolerance only absorbs floating-point noise (about $10^{-8}$), and improvements as small as those
of the 2026 algorithms (3SUM in $n^{1.9992}$ against the 3SUM hypothesis, APSP in $n^{2.9995}$), or an
$n^{2-\Omega(1)}$ bound (§4.2), show up as conflicts. The tight tolerance absorbs the remaining search inaccuracy. (The
first version used $10^{-3}$ for both, which hid those conflicts.) Exponents are displayed with the fewest decimals, from four to eight, such that rounding never overstates a
bound by more than $10^{-6}$ (an upper bound is never shown smaller, a lower bound never larger) and a conflict or gap
never looks like equality: 3SUM through Exact Triangle shows as $n^{1.99915}$, not $n^{1.9991}$; integers stay short.

A node's color is its most severe status over all its regimes (conflict > tight > gap > LB only > UB only > none). Its
label shows the LB and UB at its **primary regime**: the pinned one if any, else the regime of that most severe status,
preferring fewer hops.

### 6.8 Recomputation

An analysis remembers fingerprints of its inputs. The next analysis of the same model reuses the previous lower-bound
facts when the LB knobs, constants and relevant settings are unchanged, and the previous upper-bound solver (with its
memo) when the UB knobs, constants and relevant settings are unchanged. Moving an LB slider therefore only redoes the
forward propagation (about 0.1–0.4 s on the full sheet); UB or constant changes redo both (about 1 s).

---

## 7. User interface (`src/ui`)

* **Header:** the status line (sheet, "built-in sheet" for the embedded one, reductions used, facts, time; in full as
  a tooltip), the panel toggles, then one segment per thing acted on, its name shown once and short verbs after it:
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
  them its regime (an inline field) and note; a knob that is off collapses to a one-line summary and its note. Above
  the boxes, counts and expand / collapse all; below them, **＋ Problem** (a new empty box with its form open, or the
  existing box shown). The inspector's "add here" opens a box's form prefilled with a regime. Constants (ω) with
  sliders. Settings. A link opens the initial-settings note. The panel is rebuilt only when the knobs or boxes change,
  so typing and dragging never lose focus; switching a knob on or off, new analyses and the selection update it in
  place.
* **Center: graph** (Cytoscape). Nodes are problems, optionally boxed by family. Node color = status, label = name +
  `≥ LB ≤ UB` at the primary regime (14 px at 100 % zoom). Edges: solid if the row is `ok`, dashed grey otherwise,
  orange if it produced a displayed LB, blue if used by a displayed UB, purple if both. Search box, focus (below), zoom
  buttons (−, 100 %, +, Fit), re-layout; double-clicking a problem or reduction also shows the inspector.
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
  *Layout* (`src/ui/graphLayout.ts`): a force-directed layout of ~200 problems spreads them thin (in the first version
  problems covered about 6 % of the drawing, so labels were about 1 px tall at "fit"). Instead, each family's problems
  are packed in rows inside its box (linked problems next to each other), the boxes and the family-less problems are
  placed by a seeded force layout of the box graph (fcose in a headless Cytoscape), overlaps are removed, the
  arrangement is stretched toward the panel's shape, and boxes slide toward the center until they touch. Problems now
  cover about a third of the drawing and no two ever overlap. Node sizes depend only on the names (text measured on a
  canvas; one line reserved for the bounds), so analyses never move anything; editing formulas keeps every position;
  adding problems keeps every unchanged box exactly in place and puts new or grown boxes at the nearest free spot;
  "Re-layout" starts over with a new seed. The family-boxes toggle only shows or hides the boxes.
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
  *Problem:* signature, regimes of interest with LB/UB/status, derivation trees (LB: chain back to a knob; UB: tree of
  rows and child UBs; the regime opened is the one the "bound derivations" focus draws), "pin regime", "add a lower
  bound / an algorithm here".
  *Several items:* a head with their number, expand all / collapse all / clear, then one collapsible section per item
  (title: name, status and bounds, or the reduction's ends; × removes it from the selection), in the order selected.
  *Reduction:* all sheet fields rendered with KaTeX (MathML output, no fonts needed), parsed formula, status and
  messages, free parameters and abbreviations, the LB facts it produced, a probe of the produced instances (target
  regime and scale of each call, RT exponent) at a chosen source regime, "disable in scenario", and an editor (modal) for
  the From/To problems, the To-time / From-time / RT / Restrictions / Randomized / Hypothesis cells and the row's config
  annotation as JSON (live parse checks; the header's Reductions "＋ Add" uses the same form).
* **Bottom tabs:** problems table (sortable, filtered by the search box), reductions table (status and first message of
  every row, filter for unused rows), diagnostics (knob problems, inconsistencies, caps hit, CSV warnings, model
  diagnostics such as signature disagreements). Selected rows are highlighted; clicks select as in the graph.

State is kept in one immutable `AppState` (`src/ui/state.ts`); every change goes through `Store.update(fn)`
(`src/ui/store.ts`), which compares the old and new state: a new sheet or config rebuilds the model and the analysis
synchronously (views never see a mismatched pair); a new scenario or pinned regime schedules the analysis (debounced
while a slider moves, with a busy indicator); views subscribe to the kinds of change they display. The scenario, pinned
regimes and preferences, and the sheet/config when customized, are saved in `localStorage` (failures are ignored),
together with a format number (now 3). Data saved by the first version (no format number) keeps its sheet and config
edits, with the config's `presets` replaced by the current `initialKnobs`, but its knobs are replaced by the new initial
scenario (a message says so). Format 2, written before Bound IDs were numbers, named the initial knobs
`<group>/<variation>` (e.g. `seth/cnf-sat`) and knobs added in the app `k…`: such a knob gets the number of the initial
knob it was (if no other knob has it yet), any other id that is not a number the next number after those of the knobs
and of the initial settings, and a message says so; values, order and numeric ids are kept. Upgraded data is saved in
the current format right away.

---

## 8. Architecture

```
src/util/csv.ts          RFC-4180 parse/serialize (pure)
src/math/lexer.ts        LaTeX → tokens
src/math/ast.ts          Expr ADT (immutable), free identifiers, T-call positions
src/math/parser.ts       tokens → Formula (alternatives of Expr); ParseError with position
src/math/cells.ts        math spans of a cell, primary formula, legend abbreviations
src/growth/growth.ts     Growth ADT + algebra (§5)
src/growth/evaluate.ts   Expr × Env → Growth (with T-call handler and sum handling)
src/model/regime.ts      Regime ADT, spec parsing, normalization, keys, display
src/model/config.ts      config + scenario schemas and validators (no dependencies)
src/model/model.ts       rows + config → Model {problems, rows (CompiledRow), diagnostics}
src/model/boxes.ts       a scenario's problem boxes (left panel; order of the bounds CSV)
src/model/boundsCsv.ts   bounds CSV ↔ knobs and boxes (§3.5)
src/engine/freeparams.ts grids and refinement
src/engine/rowEval.ts    evaluate one row at (regime, θ)
src/engine/lower.ts      LB propagation
src/engine/upper.ts      UB recursion
src/engine/analysis.ts   scenario + model → Analysis (facts, per-problem summaries, statuses); derivation paths
src/engine/explain.ts    per-row probes for the inspector
src/ui/selection.ts      the selection: items, clicks with Shift / Ctrl / ⌘ (pure)
src/ui/focus.ts          focus modes: what stays visible around the selection, derivation marks (pure)
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
* **growth**: each operation × each kind, including unknowns and error cases; exponent extraction.
* **evaluate**: formulas from the sheet with hand-computed exponents (e.g. `2^{n/2}` with $n = \log_2 N$ gives
  $N^{1/2}$; `\sum_{i=0}^{\log n} 4^i T(n/2^i)` gives $\max(x, 2)$).
* **regime**: spec parsing, normalization (poly, exponential scale, degenerate), keys.
* **engine**: synthetic graphs (chain, cycle, free parameters, sums, blocked RT, mixed regimes, conflicts, caps).
* **integration** on the real sheet: every formula cell parses; no `ok` row is lost; SETH ⇒ OV 2, $k$-OV $k$, Edit
  Distance 2, LCS 2, Diameter 2 vs 3 (sparse) 2, Subset Sum 1 (in $t$); 3SUM ⇒ GeomBase, Convolution 3SUM, geometry 2;
  APSP 3 ⇒ the dense subcubic-equivalence class 3 (Negative Triangle, Distance Product, Radius, Betweenness, Replacement
  Paths, Shortest Cycle); known algorithms make CNF-SAT, OV, 3SUM, Negative Triangle tight; an $n^{1.9}$ 3SUM algorithm
  flows backward to 3SUM′ and Convolution 3SUM; an $n^{2.9}$ Negative Triangle algorithm conflicts with the APSP
  hypothesis (also for APSP itself); **all classic hypotheses together with the known algorithms produce no conflict
  and no inconsistency** — a strong end-to-end check of the engine, the annotations and the sheet (it caught an unsound
  simplification in an early annotation of row 316, whose legend bound assumes $T$ superadditive in $m$); the initial
  scenario is consistent and every initial knob (also the switched-off ones) is valid; the 2026 algorithms of Alman and
  Vassilevska Williams conflict with the 3SUM and APSP hypotheses throughout their classes but not with SETH, OV,
  $k$-Clique or BMM, and give 3SUM in $n^{1.99915}$ through Exact Triangle (rows 53 and 60).
* **config / initial scenario**: `initialKnobs` groups (on/off, notes, invalid entries), the old name `presets`,
  `initialScenario` (numbered ids, the config's kept, round trip through the scenario validator), scenario ids (kept,
  missing or repeated ones numbered), box order (`problems`: kept, blanks and repeats dropped, invalid, missing).
* **boxes**: order (listed problems, then first knobs), empty boxes, knob order within a box, one box for names of one
  problem written differently, the box key of known and unknown problems.
* **derivation paths**: a lower bound back along its chain, an upper bound through its tree (only the best of
  alternative reductions; several calls), bounds from a knob on the problem itself (the problem alone), missing bounds,
  unknown problems, a given or unknown regime.
* **selection and focus**: plain and additive clicks (alone keeping the item's regime; added at the end; removed),
  modifier keys, regimes of items; neighbours and lineage (ancestors and descendants with cycles and dead ends;
  selected reductions; unknown reductions), derivations united over the selection.
* **saved data**: format-2 knob ids become the numbers of the initial knobs (all of the previous sample's 33 ids),
  added and other ids are numbered after them, a taken number is not reused, numeric ids are kept; format 3 is loaded
  as is; the focus mode is restored when valid; the selection is not saved.
* **statuses**: conflicts as small as $10^{-4}$, and $n^{2-\Omega(1)}$ against $n^2$, are detected; floating-point noise
  is not a conflict.
* **display precision** (`boundDecimals`): integers stay short, rounding never overstates a bound (1.99915, ω, 4/3),
  search noise below an integer shows as the integer, conflicts and gaps hidden by four decimals get more.
* **regressions** from an independent code review: vanishing ceilings, dependent free-parameter ranges during
  refinement, thresholds for targets normalized by slowly growing parameters, $n^{2-\Omega(1)}$ runtimes, constant
  overflow, order-independent upper bounds.
* **graph layout**: member ordering, row packing (aspect, gaps), overlap removal, nearest free spot, compaction, the
  whole layout (no overlaps, members inside their boxes, deterministic per seed, unchanged boxes stay put when problems
  are added, density above 25 % on the real sheet).
* **panels**: size clamping, validation of saved sizes, window-dependent defaults.
* **bounds CSV**: column presence and order, extra columns, Upper/Lower synonyms, exponents vs formulas (with `$...$`,
  `\Omega(...)`), regimes (given, invalid, foreign parameters, defaults), unknown and aliased problems, Bound IDs
  (blank, repeated), Enabled values, row errors and line numbers, rows that only list a problem, problems in order of
  first appearance, write/read round trip (box by box, empty boxes), the sample file being current and reading back as
  exactly the initial scenario, merge of knobs and boxes (replace / add).
* **display**: `timePlain` (polynomial, exponential with and without a factor, infinite).
* **e2e** (Playwright, optional): the built HTML loads without console errors and without any network request, renders
  all problems with their default bounds and the initial-settings note (with switched-off knobs as compact rows), one
  box per problem in the left panel, labels are larger than 3 px at "fit" and grow when the panels are hidden, a knob
  change through the UI updates the graph, the sample bounds CSV imports, the header's segments name every button
  after its verb and segment, the three exports download their files and an exported scenario loads back unchanged,
  a click and a Shift-click select two problems (two inspector sections), the focus modes dim fewer and fewer problems
  (off > ancestors & descendants > neighbours > bound derivations), 3SUM's own knobs make it stand alone in its
  derivations, the inspector renders a problem (with a derivation) and a reduction, and the help dialog opens.

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

* Exponents only; $N^{o(1)}$ factors and log-dependent parameters are invisible.
* Regime classes are coarse (§2.6). Results for sub-polynomial parameters (e.g. $d = c\log n$ for a specific $c$) are
  not distinguished.
* Grid search with refinement may miss the exact optimum of continuous free parameters (by about $10^{-7}$ with the
  default settings, more with fewer grid points or rounds, or when the objective has several separated maxima). Such
  misses only weaken bounds (§6.3); differences below $10^{-4}$ are reported as tight.
* UBs are found within `maxDepthUB` hops only.
* Rows whose formulas are prose, `??`, or use symbols that are not declared are skipped (listed in the Reductions tab).
* `O(x)` is read as $\Theta(x)$ except for constants and logarithmic coefficients (§4.2); a formula that divides by an
  $O(\cdot)$ bound would be read optimistically (none in the sheet does).
* The analysis runs on the page's main thread: changes that recompute upper bounds freeze the page for about a second.
* With ~210 problems the whole graph cannot be read at once in a small panel: at "Fit", labels are about 4 px tall with
  the default panels on a 1600 × 1000 window and about 7 px with the graph alone; zoom in (100 % = 14 px) to read.
* The sheet's prose restrictions (e.g. "only for $m = \Theta(n^{3/2})$") are not read; such conditions must be added as
  `requires` annotations (row 106 has one).
