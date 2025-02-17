# Reduction Strategies

## Summary Table

| Strategy      | Performance    | Normalization | Arguments                 | Reduction Flags          | Refolding        | Effect on Qed                                                                          |
|---------------|----------------|---------------|---------------------------|--------------------------|------------------|----------------------------------------------------------------------------------------|
| vm_compute    | ✅✅✅         | Full          | ❌                         | ❌                       | ❌                | ✅✅✅ Bespoke Cast honored by Qed                                                        |
| lazy          | ✅✅           | Full / Head   | ❌                         | ✅✅                     | ❌                | ✅✅ Default cast does not store reduction flags                                          |
| cbv,compute   | ✅             | Full / Head   | ❌                         | ❌                       | ❌                | ✅ Default cast coincides but will use kernel reduction/conversion instead of cbv         |
| hnf           | ❔             | Head          |  ❔                        |  ❌                      | ✅                |  ❔                                                                                       |
| simpl         | ⛔⛔           | Full / Head   | ✅/⛔                      | ⛔ (only head)           | ✅                | ⛔ Uses default cast                                                                      |
| cbn           | ⛔⛔⛔         | Full / Head   | ✅✅                       | ✅✅                     | ✅✅              | ⛔ Uses default cast                                                                      |
| **Proposals** |
| [kred](#kred) | ✅             | Full / Head   | ✅✅                       | ✅✅                     | ✅                | ⛔ Uses default cast                                                                      |
| [MSP](#msp)   | ✅✅ (= lazy)  | Any (quoting) | ❌ (not necessary)         | ❌ (not necessary)       | ❌                | ✅✅✅ Fully integrated into kernel reduction/conversion                                  |

## Details

*   `vm_compute`
    *   Fully normalizes Rocq terms
    *   Translates Rocq terms to OCaml bytecode for efficiency
    *   Generally faster than all strategies listed below
    *   Translation does have a cost that is offset by automatic caching of the
        translated bytecode
    *   Does not support `Arguments`
    *   Does not support any reduction flags (not even `head`)
    *   Does not support fixpoint refolding
    *   Has a dedicated cast AST node that is honored by `Qed`
*   `lazy`
    *   Kernel reduction used in typechecking / conversion
    *   The most efficient Rocq reduction engine written in OCaml
    *   Full normalization by default
    *   Does not support `Arguments`
    *   Supports all reduction flags
    *   Does not support fixpoint refolding ([\[Experiment\] Hack a global fixpoint expansion in conversion. by ppedrot · Pull Request #18672 · coq/coq](https://github.com/coq/coq/pull/18672) )
    *   Largely coincides with the default cast except for reduction flags that
        will not be recorded in the cast.
*   `cbv`, `compute`
    *   ???
    *   Eager reduction
    *   Does not support `Arguments`
    *   Supports all reduction flags
    *   Does not support fixpoint refolding
    *   Default cast will coincide in terms of the result but will use kernel
        reduction/conversion to get there.
*   `hnf`
    *   Head normal form
    *   Does not support Arguments (❔)
    *   Does not support reduction flags (❔)
    *   Supports fixpoint refolding
*   `simpl`
    *   Rocq’s premier simplification engine
    *   Does not unfold aliases unless they uncover reduction steps
    *   Performance mixed
        *   Less efficient than `lazy` across the board but usually better than `cbn`
        *   Issues with primitive projections ([Primitive projections introduce slowdown in cbn and simpl · Issue #15720 · coq/coq](https://github.com/coq/coq/issues/15720) )
        *   Issues with nested fixpoints ([simpl is exponential in the number of nested fixpoints · Issue #13772 · coq/coq](https://github.com/coq/coq/issues/13772) )
        *   Issues without fixpoints or primitive projections ([!simpl scales exponentially even without nested fixpoints · Issue #16397 · coq/coq](https://github.com/coq/coq/issues/16397) )
    *   Partial support for `Arguments`
        *   `simpl` will ignore `Arguments : simpl never` in `match <HERE> with`
    *   Only supports `head` reduction flag
    *   Supports fixpoint refolding
    *   No dedicated cast, relies on default cast and thus kernel
        reduction/conversion at `Qed` time which could lead to performance
        issues
*   `cbn`
    *   An alternative to `simpl`
    *   Eagerly reduces aliases even when they uncover no reduction steps
    *   Performance is bad
        *   Slower than any other reduction mechanism often by orders of magnitude
        *   Similar issues as with `simpl`, just worse
    *   Full support for `Arguments`
    *   Supports fixpoint refolding
        *   Also refolds aliases (❔)
    *   No dedicated cast, relies on default cast and thus kernel
        reduction/conversion at `Qed` time which could lead to performance
        issues


# Problematic Reduction Examples

## <a id="reif"></a> Reification with User Terms
Iris Proof Mode and BlueRock’s automation use reflection and computation to
perform changes to the shallowly embedded iris goal.

### Requirements

*   Must only reduce computation introduced by the automation tactics
    *   Can be addressed with whitelist reduction flags, or
    *   by creating fresh copies of all functions used in the tactics.
    *   Both solutions are incredibly tedious and error-prone
*   Needs a reasonably performant strategy (lazy or better)
*   Reduction should ideally be captured by a bespoke cast so that it can be
    replayed or coincide with kernel reduction for Qed performance

## <a id="rep"></a> Avoiding Term Repetition in Proof Terms

BlueRock's automation avoids term repetition in the proof term by accumulating
intermediate results inside the context of an ever-growing function type. The
arguments of the function are newly injected terms that cannot be computed from
existing values that have been passed to the function.

This approach works wonders for term size but Qed performance depends entirely
on the ability to force terms across the forall boundary of this function type
of dynamic arity. For example, in the following type, `1 + 1` is reduced twice:
once in checking the type of `f` and once in checking the return type.

```coq
let n :=
1 + 1 in forall (f : Fin.t n), Fin.t n
```

Unfortunately not all values can be forced and types such as telescopes will
produce terms of cubic size without any way to share their reduction.

### Requirements

*   The solution must integrate into Qed, i.e. the kernel.
*   It must support whitelists or otherwise allow specific selection of terms to
    be reduced (see “Reification with User Terms” above)
    *   This eliminates vm_compute


## <a id="nice"></a> General “make goal look nice” Strategy

Any kind of longer-running automation that handles user terms probably needs to
integrate a generic simplification strategy to perform the same kind of
“normalization” that users rely on.

simpl and cbn are the obvious candidates but only cbn is safe to call on
arbitrary terms in arbitrary contexts without violating simpl never.
Unfortunately, this eliminates the faster of the two strategies and leaves us
with occasional catastrophic slowdowns in cbn.

### Requirements

*   Must honor simpl never
*   Must be quite fast so that it can be called after every goal change
*   Must support Arguments
*   Must support fixpoint refolding (as goals will otherwise become unreadable)
*   Should be somewhat principled so that users can predict its results

# Proposed Reduction Strategies

## <a href="kred"></a> [kred: A faster, somewhat compatible cbn alternative](https://github.com/coq/coq/pull/19309)

`kred` is a half finished re-implementation of `cbn` based on the kernel’s
reduction machine. It is essentially `lazy` with support for `Arguments` and
fixpoint refolding. It tries to be as faithful as possible to `cbn` but will
sometimes deviate, especially when refolding fixpoints opportunities are too
expensive to find.

Another way to look at it is to call it a slightly less ambitious and slightly
less agressively refolding version of cbn with the benefit of delayed
substitution.

### Advantages over cbn

kred can be arbitrarily faster than `cbn` and its performance will often
coincide with `lazy` when terms can be fully reduced. It is always faster than
`cbn` even when terms are not fully reduced.

These properties make `kred` a promising candidate for [a General “make goal look nice” Strategy](#nice).

### Disadvantages

As mentioned, `kred` is not as aggressive about refolding and will sometimes
generate slightly different goals.

`kred` is unfinished. The current code has at least one bug that introduces
incorrect de-Bruijn indices in some specific cases.

kred.ml is basically a copy of cClosure.ml. Its refolding mechanism cannot be
integrated into the kernel because it is simply not sound
(https://github.com/coq/coq/pull/18672). Handling Arguments require substantial
changes that are unlikely to be ported to cClosure.ml. So `kred` essentially
doubles the amount of kernel code with the added complexity of the copy not
being a verbatim copy.


## <a href="msp"></a> [MSP: Multi Stage Programming](https://github.com/coq/coq/pull/17839)

The MSP prototype provides primitives (currently implemented using actual
“primitives” but a next version will use dedicated AST nodes) that freeze or
force reduction of terms. Those primitives can be nested to perform fine-grained
reduction of some but not other terms.

The very basic example below force-appends two lists containing user terms
without touching those user terms which are guarded by the `block` primitive.

```coq
run ([block P; block Q] ++ [block R; block S])
```

blocked terms may contain unblocked sub-terms which are going to be reduced up
to blocked sub-sub-terms, etc.

`run` is what triggers MSP to work. `block` and `unblock` have no meaning
outside of run.

MSP addresses the use cases [Reification with User Terms](#reif) and [Avoiding Term
 Repitition in Proof Terms](#rep) completely and drastically improves on all existing
 reduction strategies in the precision it offers for reducing specific terms.

### Advantages

The current MSP prototype has already been tested on examples representing these
use cases and it is able to overcome existing shortfalls both in complexity of
use as well as in performance.

### Disadvantages

MSP must live in the kernel for it to be useful at Qed time. It might be
possible to make it opt-in but that hardly changes anything.

Automation-relevant theory must be rewritten to incorporate `block`, `unblock`,
and `run` in all the necessary places. This cannot be avoided with this
approach.

The current prototype introduces a sizeable performance regression for
developments that do not use the feature (+7% or so in developments that stress
the kernel) because of the way it has to find applications of the new primitives
before they are (mis-)handled in the usual way by the rest of the machine.

The next iteration will use dedicated `constr` nodes which is a way more invasive
change but should allow us to avoid any performance impact on developments that
do not use the feature.
