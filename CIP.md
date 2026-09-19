# Removal of the UPLC scope condition and scope check


## Abstract

This CIP proposes the removal of A) the well-scopedness requirement from the
Plutus Core specification and B) the scope check from the implementation.
Removal of the check enables a reduction in fees for transaction validation with
a modest change to UPLC's semantics.


## Background

The Plutus Core Language Specification states [Section 2.1.3] that a UPLC
program must be _well-scoped_, meaning it has no free variables. It also states
that the property must be checked by before script execution in the CEK machine.
The _scope check_ is performed during phase 2 of transaction validation and
introduces overhead.

When a well-scoped ("closed") UPLC term is evaluated, we know that each variable
will be bound to a value by the time it is needed. This rules out
"unbound/undefined variable" errors at run-time. The purpose of the scope check
is to reject ill-scoped ("open") terms early.

<!--
[History: why was it introduced.]
-->


## Motivation

The scope check is estimated to increase script preparation time by about 25%
[roman's measurements, my reproducers]. As this is part of phase-2 validation,
that work is reflected in transaction fees and may hinder developer adoption (a
core pillar of Cardano's 2030 strategy
[https://product.cardano.intersectmbo.org/vision/strategy-2030/]). This raises
the question: is the scope check worth it?


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
variable lookup failure[]. The formalized Agda CEK machine would require
changes, which we discuss in the Specification section below.

We are not aware of any other implementation of the CEK machine or tooling that
relies on the well-scopedness property. For example, the Amaru Rust
implementation does not perform the scope check in the first place, and it deals
with variable lookup failure in the CEK machine.


### The current scope check is unsound

The scope check has a known bug[], which causes it to accept some programs with
free variables. Therefore, open terms are de-facto part of the semantics
already.

<!-- this argument applies if we were to fix existing language versions
retroactively

Moreover, a fix is more involved than the removal: a stricter scope
check could cause existing scripts to start failing, so a fix is classified as
backwards incompatible (see CIP-35 []) and requires a new ledger language. A
removal on the other hand is considered backwards compatible and can be released
with a hard fork for Plutus V1, V2 and V3.
-->


## Developer tooling already performs the scope check off-chain

Languages such as Aiken, Plinth and Plutarch already check for scoping
indirectly by means of a type checker, which guarantees well-scopedness for the
generated UPLC. Other non-sensical programs that are typically rejected by a
compiler (e.g. `2 + true`) are also allowed to run and fail in the CEK machine.




## Specification

This specification covers changes to the Plutus Core language specification[],
the implementation, the conformance test suite and the formalized metatheory.

### The Plutus Core specification

The reduction semantics for free variables is given by the following rule:

```
---------------
x    ⟶    error
```

The CEK semantics for free variables is given by the step:

```
s; ρ ▷ x    ↦    ◆    (if x is not bound in ρ)
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


### Type of change and release required

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

No changes to the binary format and script-ledger interface are needed.


## Rationale: How does this CIP achieve its goals?

By removing well-scopedness from the specification and the implementation of the
CEK machine, there is an immediate reduction of work for scripts that have no
free variables. Eventually this could be reflected in in transaction costs by
adjusted fee parameters.


## Path to Active

TODO


## Acceptance Criteria

TODO

<!--

MUST INCLUDE (CIP-35)
- external implementations are available
- plutus repo is updated with a specification of the proposal
- plutus repo is updated with an implementation of the proposal

-->

## Implementation Plan

The release type is a hard fork [CIP-0035]

TODO


## Considerations

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


### Could the well-scopedness property improve CEK performance?

TODO

### Can the scope check move to phase 1?

TODO


## Copyright

TODO

