# Changelog

Normative changes to `SPECIFICATION.md`, one section per specification revision.
A document names the revision it is written against with `spec_version` (2.1).

This file records what changed and what a document written against the previous
revision has to do about it. Why a rule is the way it is belongs in the
specification itself.

## 0.1 (unreleased)

### Removed

- **14.4** transform kinds `array_uncons`, `array_cons`, and `array_reverse`,
  with the sections that defined them. A document using one of them is no longer
  valid.
- **14.4.1** the length-precondition rule. No transform kind has a precondition
  now, and the phase classification the rule pointed at is stated in 6.2.
- **4.4**, **6.2** the `array_uncons` empty-Array error examples, and its entry
  in the list of phase-dependent conditions.

### Added

- **14.4.2 `array_flatten`** (`Array<Array<T>> -> Array<T>`) and
  **14.4.3 `array_unflatten`** (`Array<T> -> Array<Array<T>>`). Neither has a
  precondition; `array_flatten` accepts inner collections of unequal length.
  Where a process needs a relation between the lengths, it declares an ordinary
  Pure Data input port and states the relation in `contracts`.
- **14.4.0** the criteria a transform kind is judged against, which is what the
  removals above are argued from. Marked **Non-normative**: it constrains future
  revisions of this specification rather than documents, and an implementation
  checks nothing in it.
- **1**, design goal 10: the shape of the Object-flow graph is statically
  determined, and run-time information may only index that shape as a finite
  scalar parameter.
- **1.1 Design principle: static shape.** States the principle every structured
  node kind is designed to, why `branch` is the one kind that does not have the
  property on its own, and restates the principle as a bound on the physical
  resources a workflow needs before it runs.

### Changed

- **14.4.1** the correspondence between a transform's input and output Object
  slots is now an order-preserving total bijection over traversal order, in
  place of the index variables and slice notation (`i`, `*`, `0`, `1..`). No
  transform prescribes which output position an input slot arrives at.
- **24.3** policy tracking through a transform is stated from that
  correspondence rather than from a worked example per kind.

### Migrating from 0.0

- A use of `array_reverse` has no replacement. Order is observable in v0 only
  through the correspondence between collections, the traversal order of `fold`,
  and the output order of `collect`, so a workflow that depended on reversal must
  express it in the process that produces the collection.
- A use of `array_uncons` or `array_cons` that was regrouping is written with
  `array_unflatten` or `array_flatten`. One that singled out the first element
  has no replacement, by criterion 3 of 14.4.0.

## 0.0 - 2026-08-19

The first tagged revision: the specification as it stood before the 0.1
amendment, and the text every implementation released so far was written
against (ofplang-validate 0.1.6, ofplang-schedule 0.2.4, ofplang-run 0.3.1,
ofplang-labcode 0.2.0).

Tagged retroactively, so this section records no changes against a predecessor.
The history before it is in the commit log.
