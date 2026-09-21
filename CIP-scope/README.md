---
CIP: "?"
Title: Removal of the scope check in Plutus Core
Category: Plutus
Status: Proposed
Authors:
  - Jacco Krijnen <jacco.krijnen@midgardlabs.io>
Implementors: []
Discussions:
  - Plutus-issue: https://github.com/IntersectMBO/plutus/issues/7368
  - Original-PR: https://github.com/cardano-foundation/CIPs/pull/?
Created: 2026-09-21
License: CC-BY-4.0
---


## Abstract

This CIP proposes the removal of A) the well-scopedness requirement from the
Plutus Core specification and B) the scope check from the implementation.
Removal of the check enables a reduction in fees for transaction validation with
a modest change to UPLC's semantics.



## Motivation: Why is this CIP necessary?

### Background

The Plutus Core Language Specification[^plutus-spec] states in Section 2.1.3
that a UPLC program must be _well-scoped_, meaning it has no free variables. It
also states that the property must be checked by before script execution in the
CEK machine. The _scope check_ is performed during phase 2 of transaction
validation and introduces overhead.

When a well-scoped ("closed") UPLC term is evaluated, we know that each variable
will be bound to a value by the time it is needed. This rules out
"unbound/undefined variable" errors at run-time. The purpose of the scope check
is to reject ill-scoped ("open") terms early.

<!--
TODO: History: why was it introduced? See e.g. https://github.com/IntersectMBO/plutus/issues/7368
-->

[^plutus-spec]: https://plutus.cardano.intersectmbo.org/resources/plutus-core-spec.pdf

### The scope check is costly


The scope check is estimated to increase script preparation time by about 25%
[^bench]. As this is part of phase-2 validation, that work is reflected in
transaction fees and may hinder developer adoption (a core pillar of Cardano's
2030 strategy[^2030-strategy]). This raises the question: is the scope check
worth it?

[^bench]: https://github.com/IntersectMBO/plutus/issues/7368
[^2030-strategy]: https://product.cardano.intersectmbo.org/vision/strategy-2030/



### No meaningful new script behaviour

A removal of well-scopedness introduces a new failure mode in UPLC semantics: a
variable which is unbound cannot be evaluated, causing the CEK machine to
terminate with an error. This new behaviour does not meaningfully change
language semantics: replace each free variable by `error`, and you have a
well-scoped program that behaves identically. Put differently, any script logic
that can be written as an open term could already be written with well-scoped
UPLC.

Behaviour of well-scoped programs does not change: its evaluation can never
reach the new failure mode because all variables will be bound to a value during
execution.


### The implementation does not rely on the scope check

No part of the Haskell CEK machine implementation relies on a term being
well-scoped. There has always been logic in place that needed to deal with
variable lookup failure[^cek-open-error]. The formalized Agda CEK machine would require
changes, which we discuss in the Specification section below.

We are not aware of any other implementation of the CEK machine or tooling that
relies on the well-scopedness property. For example, the Amaru Rust
implementation does not perform the scope check in the first place, and it deals
with variable lookup failure in the same way as the Haskell node.


[^cek-open-error]: https://github.com/IntersectMBO/plutus/blob/57d6d00c307c802d8a5c0f92253205438a9180f4/plutus-core/untyped-plutus-core/src/UntypedPlutusCore/Evaluation/Machine/Cek/Internal.hs#L1085


### The current scope check is unsound

The scope check has a known bug[^scope-bug], which causes it to accept some programs with
free variables. Therefore, open terms are de-facto part of the semantics
already.

<!-- this argument only applies if we were to fix existing language versions
retroactively with only PV guarding

Moreover, a fix is more involved than the removal: a stricter scope
check could cause existing scripts to start failing, so a fix is classified as
backwards incompatible (see CIP-35 []) and requires a new ledger language. A
removal on the other hand is considered backwards compatible and can be released
with a hard fork for Plutus V1, V2 and V3.
-->

[^scope-bug]: https://github.com/IntersectMBO/plutus-private/issues/2374


## Developer tooling already performs the scope check off-chain

Languages such as Aiken, Plinth and Plutarch already check for scoping
indirectly by means of a type checker, which guarantees well-scopedness for the
generated UPLC. Other non-sensical programs that are typically rejected by a
compiler (e.g. `2 + true`) are already allowed to run and fail in the CEK machine.






## Specification

This specification covers changes to the Plutus Core language specification[],
the implementation, the conformance test suite and the formalized metatheory.


### Type of change

CIP-0035 lists typical changes to Plutus Core and how those affect the Plutus
language version (LV). Which change applies here is not directly obvious.
Consider for example:

> Changing the behaviour of a construct in the language

This sounds applicable, because evaluating variables can now fail. However, free
variables have never been a valid construct: the combination of concrete syntax
and the well-scopedness requirement ruled out terms with free variables.

Therefore, the following type of change is more appropriate:

> Adding a construct to the language

The proposed change therefore requires bumping the language version from 1.1.0
to 1.2.0, which in turn requires a hard fork.

No changes to the binary format or script-ledger interface are needed.


### The Plutus Core specification

The reduction semantics for free variables is given by the following rule:

```

---------
x ⟶ error
```

The CEK semantics for free variables is given by the step:

```
s; ρ ▷ x ↦ ◆    (if x is not bound in ρ)
```

Text in the specification is updated accordingly. For example:

- The paragraph about "Scoping" (Section 2.1.3) does not require terms to be
  well-scoped, but mentions that free variables are allowed.
- The definition of substitution should state that capture-avoidance is
  necessary because values may contain free variables.

### The Plutus Core implementation

The CEK machine should throw an `OpenTermEvaluatedMachineError` error when
trying to evaluate a free variable, after charging a `BVar` step.

Only for Plutus language version ≤ 1.1.0, `mkTermToEvaluate` performs the scope
check. This applies to Plutus V1, V2, V3.

### The Agda metatheory

The plutus-metatheory formalization in Agda [] should include abstract syntax
without scoping restrictions and a corresponding CEK machine, in addition to the
scoped and typed formalisations (which cannot represent open terms). The CEK
machine will implement the semantics for free variables as outlined above.

### The conformance test suite

The plutus conformance test suite should be extended with tests for open terms.
In particular it should test the following two behaviours:

- failure with an `OpenTermEvaluatedMachineError` error
- succesful termination with free variables.


### Versioning

TODO


## Rationale: How does this CIP achieve its goals?

By removing well-scopedness from the specification and the implementation of the
CEK machine, there is an immediate reduction of work for scripts that have no
free variables. Eventually this could be reflected in in transaction costs by
adjusted fee parameters.


### How does this affect transaction validation?

Incoming transactions that were invalid due to a script failing the scope check
can be valid after the scope check removal. There are two scenarios:

- Either the ill-scoped script still fails (because it evaluated a free
  variable). There is no observable consensus change: a phase 2 failure means
  collateral is forfeited. The node may have to do some extra work evaluating
  the program, but it is compensated for that with the collateral.

- The script now now succeeds (no free variable was on the execution path), so
  fees are paid in the usual way. Since this leads the ledger to accept more
  transaction, a PV check is in order to be able to validate the chain history.

Crucially, incoming transactions that currently succeed will keep succeeding
after removal. The node will save the overhead of performing the scope check on
all (reference) scripts used. This can eventually be reflected in lower fees.


### Alternatives considered

TODO


#### Fixing the scope check instead of removing it

TODO

#### Moving the scope check to phase 1

TODO

#### Keeping well-scopedness for CEK performance


## Path to Active

TODO


### Acceptance Criteria

TODO

<!--

MUST INCLUDE (CIP-0035)
- external implementations are available
- plutus repo is updated with a specification of the proposal
- plutus repo is updated with an implementation of the proposal

-->

### Implementation Plan

The release type is a hard fork [CIP-0035]

TODO


## Considerations





## Copyright

This CIP is licensed under
[CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).

