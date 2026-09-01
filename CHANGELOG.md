# Changelog

Normative changes to `SPECIFICATION.md`, one section per specification revision.
A document names the revision it is written against with `spec_version` (2.1).

This file records what changed and what a document written against the previous
revision has to do about it. Why a rule is the way it is belongs in the
specification itself.

## 0.1 (unreleased)

### Removed

- **18.1**, **19.1** output mode `last`, and **18.2** the section that related an
  empty `fold` traversal to it. Every output mode now applies uniformly to the
  whole traversal, and no mode requires an invocation to have happened, so an
  empty traversal is no longer a phase-dependent error. `carry` replaces every
  use of `last`; what it adds is that the initial value must be written.
- **4.4**, **6.2**, **25** the `mode: last` on an empty traversal error examples
  and its entry among the phase-dependent conditions. A zip-equal length
  mismatch is now the only one.

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
- **21.0** the list 2.3 promised: which binding and control sections are valid
  for each node kind. The rule existed and was enforced; what was missing was
  anywhere to look it up. Includes a non-normative note on why a repetition over
  a shared physical resource is a `fold` or a `do_while` and never a `map`.
- **17** which sections a `map` node has, and that a `bind` value is reused by
  every invocation while an `each` element goes to exactly one.
- **11** the binding entries of a node and the input ports of the process it
  invokes are in one-to-one correspondence, for every node kind. Until now the
  rule was stated for the ports (12.1, 12.2) without saying it covered a
  structured node, and nothing said a binding entry must name a port at all.
  For a `branch` the correspondence holds against each arm, which is what makes
  the two arms agree on their input ports through `args`.
- **17**, **18** a `map` node and a `fold` node must have at least one `each`
  source. Their shape is the body L times, and with no `each` source there is no
  L. `do_while` has no counterpart requirement, and 19 now says why.
- **16** an Object-bearing carry must be threaded: the carried input port's fate
  is the same-name output port, or that input is consumed and that output
  created. The section listed those two forms already but permissively, so a
  third arrangement -- the carried Object leaving through a collected output
  while the carry output is created -- was accounted for and yet had no
  expressible Object correspondence for the node, since it would have to name a
  position within a collection.
- **12.4 Object skeleton.** A process's skeleton is a partial injection from its
  input Object slots to its output Object slots, the input slots it consumes,
  and the output slots it creates, each created slot carrying the node at which
  it is created. Sections 13, 14.4.1, 15, 20.2 and 24.1 were each describing a
  part of this and are now stated in terms of it. The skeleton relates the slot
  families of two collections by a correspondence kind rather than by position,
  so it never enumerates a collection and never names an index, and equality is
  the comparison of two normal forms.
- **12.4.7** every process and node has exactly one skeleton. Where a construct
  has several descriptions of which one runs, they must all have equal
  skeletons; `branch` is the only such construct in v0. This is stronger than
  the principle of 1.1, which 1.1 now says.
- **19.3** a `do_while` node exposes a reserved Boolean output `exhausted`, true
  when it terminated by reaching `max_iterations` with the condition still true.
  Whether the limit was reached is a fact about the node rather than about the
  target process, which cannot know whether an invocation was its last, so no
  carry value could report it.
- **2.4** `exhausted` is a reserved name. It is the one entry in that list that
  is not a structural key: were a target process free to declare an output of
  that name, `<node>.exhausted` would name two different values.
- **11** what a `value` literal may be -- a primitive or a Pure Data `Array`,
  including an empty one -- and that a Pure Data carry may take its initial
  value from one while an Object-bearing carry may not.
- **1**, design goal 10: the shape of the Object-flow graph is statically
  determined, and run-time information may only index that shape as a finite
  scalar parameter.
- **1.1 Design principle: static shape.** States the principle every structured
  node kind is designed to, why `branch` is the one kind that does not have the
  property on its own, and restates the principle as a bound on the physical
  resources a workflow needs before it runs.

### Changed

- **15** the process-level marker `elidable_iso` is renamed
  `object_identity_map`, and the section it is declared in moves from a
  process's `traits` to its `behavior`. Two different things were called traits:
  a type trait, which is a predicate on a type declared in the top-level
  `traits` section, and a marker on a process. `elidable` also described the
  wrong thing -- what may be elided is the `objects` section, not the process,
  which may update views or consume time -- and `iso` was weaker than the
  requirement, since a cross-wired `objects.map` is an isomorphism and does not
  qualify. `object_identity_map` names what the marker actually asserts and
  matches the `objects.map` it infers.
- **2.4** `behavior` is reserved. `traits` stays reserved as the name of the
  top-level section that declares type traits.
- **15** the marker vocabulary is closed: `object_identity_map` is the only one
  v0 defines, and any other `behavior` entry is a validation error. Nothing
  checked a process-level marker's spelling before.


- **14.4.1** the correspondence between a transform's input and output Object
  slots is now an order-preserving total bijection over traversal order, in
  place of the index variables and slice notation (`i`, `*`, `0`, `1..`). No
  transform prescribes which output position an input slot arrives at.
- **24.3** policy tracking through a transform is stated from that
  correspondence rather than from a worked example per kind.

### Migrating from 0.0

- An output written with `mode: last` becomes a `carry`: the target process
  threads the value itself and the node exposes the final one. The initial value
  must now be written, which is the one thing `mode: last` did not require.
- A workflow that told bounded termination apart by reading a `mode: last`
  condition output reads `<node>.exhausted` instead.
- A port, process, node id, binding, return, type, trait, or type parameter
  named `exhausted` must be renamed.
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
