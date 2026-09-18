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

[History: why was it introduced.]


## Motivation

The scope check is estimated to increase script preparation time by about 25%
[roman's measurements, my reproducers]. Since this is part of phase-2
validation, it has a direct impact on transaction fees, which hinder developer
adoption (a core pillar of Cardano's 2030 strategy
[https://product.cardano.intersectmbo.org/vision/strategy-2030/]). This raises
the question: is the scope check worth it?


### No meaningful new script behaviour

A removal of well-scopedness requires a new failure mode in UPLC semantics: a
variable which is unbound cannot be evaluated, causing the CEK machine to
terminate with an error. This new behaviour does not meaningfully change
language semantics: replace each free variable by `error`, and you have a
well-scoped program that behaves identically. Put differently, any script logic
that can be written as an open term could already be written with well-scoped
UPLC.

Behaviour of well-scoped programs does not change: its evaluation can never
reach this new failure mode because all variables will be bound to a value
during execution.


### The implementation does not rely on the scope check

No part of the Haskell CEK machine implementation relies on a term being
well-scoped. There has always been logic in place that needed to deal with
variable lookup failure[]. The formalized Agda CEK machine would require
changes, which we discuss in Specification.

We are not aware of any other implementation of the CEK machine or tooling that
relies on the well-scopedness property. For example, the Amaru Rust
implementation does not perform the scope check in the first place, and it deals
with variable lookup failure in the CEK machine.


### The current scope check is unsound

The scope check has a known bug[], which causes it to accept some programs with
free variables. Therefore, open terms are de-facto part of the semantics
already. Moreover, a fix is more involved than the removal: a stricter scope
check could cause existing scripts to start failing, so a fix is classified as
backwards incompatible (see CIP-35 []) and requires a new ledger language. A
removal on the other hand is considered backwards compatible and can be released
with a hard fork for Plutus V1, V2 and V3.


## Developer tooling already performs the scope check off-chain

Languages such as Aiken, Plinth and Plutarch already check for scoping
indirectly by means of a type checker, which guarantees well-scopedness for the
generated UPLC. Other non-sensical programs that are typically rejected by a
compiler (e.g. `2 + true`) are also allowed to run and fail in the CEK machine.




## Specification

### The Plutus Core specification

The Scoping paragraph (Section 2.1.3) would need updating, mentioning that free
variables are allowed.

The reduction semantics would need a rule for a free variables:

```
---------------
x    ⟶    error
```

The CEK semantics needs to change accordingly with one rule:

```
s; ρ ▷ x    ↦    ◆    (if x is not bound in ρ)
```


The section on Term reduction (2.3.2) should mention that capture-avoiding
substitution also means that a term with a free variables can be safely
substituted without capture.

The specification also mentions "closed" in a few other places that would need
to be addressed.


### The Haskell CEK machine implementation

The Haskell CEK machine already deals with free variables: it throws a
`OpenTermEvaluatedMachineError` error [source line]. This is consistent with the
proposed change in the specification.

To preserve replayability of chain history, scope checking needs to be guarded
by protocol version, so that scope checking is still performed for historic
transactions before the PV that implements this CIP.


### The Agda metatheory

The current formalization in Agda[] uses intrinsically well-scoped syntax for
UPLC. As such, its abstract syntax cannot represent open terms. To fix this,
abstract syntax that corresponds to the syntax in the specification would have
to be added. The CEK semantics could be implemented very similarly to the
current ones, but Value environments should not be indexed by the scope anymore.
Since lookup in environments will be partial, the CEK case of free variables
needs to be implemented, following the rule outlined above.


### The conformance test suite

Currently, the conformance tests [] do not cover open terms. Tests should be
added that evaluate open terms which both:

- fail execution with an `OpenTermEvaluatedMachineError` error
- succeed by not evaluating the free variables


### Type of change and release

__Type of change__ This proposal is a change to Plutus Core that does not fall
into one of the specific Plutus changes mentioned in [CIP-0035], so it would
fall under "Other changes". The change is backwards-compatible because it allows
strictly more scripts to run, but should be PV guarded because it is not
forwards-compatible.

No changes to syntax, binary format and script-ledger interface are needed.


### Costing and fees

Incoming transactions that currently fail because of an ill-scoped script may
start to validate after a removal. There are two scenarios:

- Either the ill-scoped script still fails (a free variable evaluated). There is
  no observable consensus change: a phase 2 failure means collateral is
  forfeited. The node may have to do some extra work evaluating the program, but
  it is fairly compensated with the collateral.

- The script now now succeeds (no free variable was on the execution path). In
  the latter case, a new transaction may succeed and fees are paid in the usual
  way. Since this leads the ledger to accept more transaction, a PV check is in
  order to be able to validate the chain history.

Incoming transactions that currently succeed will keep succeeding after removal.
The node will save the overhead of performing the scope check on all (reference)
scripts used. This can eventually be reflected in lower fees.





## Rationale: How does this CIP achieve its goals?

By removing well-scopedness from the specification and the implementation of the
CEK machine, there is an immediate reduction of work performed by the node.
Eventually this can be reflected in in transaction costs by adjusted fee
parameters.





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


## Copyright

TODO

### Could well-scopedness improve run-time performance?

TODO

### Can the scope check move to phase 1?

TODO
