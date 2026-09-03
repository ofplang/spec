# Changelog

Normative changes to `SPECIFICATION.md`, one section per specification revision.
A document names the revision it is written against with `spec_version` (2.1).

This file records what changed and what a document written against the previous
revision has to do about it. Why a rule is the way it is belongs in the
specification itself.

## 0.2 (unreleased)

### Added

- **2.4** node ids are unique within one composite body. The condition was never
  stated, and without it `<node_id>.<output>` does not name one value. Also
  **27 rule 10a**.
- **10.2** the node dependency graph of a composite body must be acyclic. Only
  the *process* dependency graph was required to be, which is a different graph:
  it concerns which process invokes which, not the order of nodes within one
  body. Nothing forbade a body whose nodes referred to each other in a cycle,
  and a Pure Data cycle satisfied every degree rule of 12.1. Also
  **27 rule 21b**, which states both graphs together.
- **11.1.1** a literal is a `graph` phase value. `graph` is the least phase, so
  this is what makes a literal admissible wherever a Pure Data value of any
  phase is expected, and it is why no phase condition is ever stated for one.
  Also **27 rule 87a**.
- **27 rule 16a** the general phase-flow rule. 6 stated the order and the
  permitted flow, but no summary rule did; rule 16 covers only the prohibition
  on Object-bearing values at `graph` phase.
- **4.4** three validation error examples: a value whose phase is later than the
  port it fills, a duplicate node id, and a cycle in a body's node dependency
  graph.
- **11.2 Constant slots**, a document position that expects a Pure Data value
  fixed no later than a declared phase, filled by a source entry (2.6.6) and
  governed by the rules that govern a binding to an input port: the phase flow
  of 6, the type match of 11.1, the literal conformance of 11.1.1. v0 has one,
  `do_while.max_iterations`, whose type and phase conditions 19 stated for
  itself because 11.1's table is indexed by binding section and a slot is not
  one. Also **2.6.6**, which says a slot takes a source entry of the same shape.
- **2.6.8** a subsection for reference resolution. The rule applies to every
  reference form of 2.6 and was the closing paragraph of 2.6.7, whose subject is
  structured condition references.

### Changed

- **21** the restriction of `mode: drop` to Pure Data outputs is cited from
  18.1, 19.1, and 20.1. The sentence draws a conclusion about `common`, which is
  a `branch` mode, so 20.1 belongs among its grounds.
- **19** requirement 5 states `max_iterations` as a constant slot (11.2) rather
  than as a phase and type condition of its own, and **27 rule 19d** with it.
  **27 rule 16a** covers a constant slot alongside a binding, and **4.4** gains
  an Object-bearing value filling one. What is decided at which phase is
  unchanged; it is now said in one place.

### Migrating from 0.1

Conditions now stated that were not stated before. A 0.1 document that meets
them is unaffected; one that does not was never meaningful:

- Two nodes of one composite body with the same `id`. A reference
  `<node_id>.<output>` did not name one value, so an implementation resolved it
  to one of them or rejected it, with nothing in the specification to say which.
- A body whose node dependency graph has a cycle. Evaluation cannot start, and
  the composition of the body's node skeletons (12.4.5) is not defined.


## 0.1 - 2026-09-01

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
- **15** the inference the marker permits is stated directly instead of being
  qualified as applying to "top-level" ports, which read two ways. It applies to
  every Object-bearing input port, an `Array` port included, and pairs with the
  output port of the same name, **type, and phase**; a port with no such
  counterpart is left unexplained, which is an incompleteness error. The
  matching of types was implied by the definition above it and missing from the
  inference rule, so an inferred map could relate ports that an explicitly
  written one may not (14.1).
- **2.1** `spec_version` is checked. It still does not select an interpretation
  -- there is one set of rules and every document is read by them -- but an
  implementation now refuses a document declaring a later MINOR of the same
  MAJOR, or any other MAJOR, rather than answer for a revision it does not
  implement. An earlier MINOR is accepted: a revision within one MAJOR is an
  edit of the same language, and where such a document uses something a later
  revision removed, the error naming that construct says more than a version
  mismatch would. The current revision is `0.1`.
- **14.1** the resolved types of an `objects.map` entry's source and target must
  match. Every comparable rule carried such a condition already -- a transform's
  role typing, the identity-map marker, carry compatibility, a branch's common
  outputs, binding type compatibility -- and this was the one place without one,
  so `outputs.plate: inputs.cup` between unrelated Object types was accepted and
  the claim to preserve container structure was made between slot structures
  that do not correspond.
- **19** `max_iterations` must be at least 1. A bound of zero contradicts the
  at-least-once invocation the section opens with.
- **12.2** the degree rule for an Object-bearing value read as a *source* of a
  composite body, which 13 forbade breaking as a property while 12 never stated
  it operationally. A composite's own input port needed it most: the rule above
  it governs an output port, and a composite input port read as a source is not
  one.
- **19.1** rule 2 takes precedence where the condition output is also a carry
  binding, which rules 2 and 5 disagreed about.
- **12.4 Object skeleton.** A process's skeleton is a partial injection from its
  input Object slots to its output Object slots, the input slots it consumes,
  and the output slots it creates, each created slot carrying the node at which
  it is created. Sections 13, 14.4.1, 15, 20.2 and 24.1 were each describing a
  part of this and are now stated in terms of it. The skeleton relates the slot
  families of two collections by a correspondence kind rather than by position,
  so it never enumerates a collection and never names an index, and equality is
  the comparison of two normal forms.
- **20.2** a `branch`'s two arms must have equal Object skeletons, replacing the
  list of four forbidden shapes. **Two arms that both consume and both create
  are now valid**: the creation point is the branch node in each case, so
  whichever arm runs, one new Object appears at that node's output and where its
  identity came from does not depend on the arm. A composite arm can also be
  judged for the first time, since a skeleton is derived from a body graph as
  well as from an `objects` section -- previously the comparison read `objects`
  declarations, which a composite does not have, so a composite arm could never
  satisfy it.
- **16** the threading requirement is checked for a composite target process as
  well as an atomic one, for the same reason.
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

- **2.6.7** no longer calls the `do_while` condition output a non-carry output,
  which contradicted 19.1 rule 5; how the node exposes it is 19.1's to say.


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

Renames, which are mechanical:

- The process-level marker `elidable_iso` becomes `object_identity_map`, and a
  process declares it under `behavior` rather than `traits`. The top-level
  `traits` section, and a type's `implements`, are unchanged. A marker v0 does
  not define is now an error rather than silently ignored, so a document that
  keeps the old spelling is rejected rather than read as declaring nothing.
- A port, process, node id, binding, return, type, trait, or type parameter
  named `exhausted` must be renamed: the name is reserved (2.4).

Removals, where a document has to be rewritten:

- `mode: last` becomes a `carry`: the target process threads the value itself
  and the node exposes the final one. The initial value must now be written,
  which is the one thing `mode: last` did not require.
- A workflow that told bounded termination apart by reading a `mode: last`
  condition output reads `<node>.exhausted` instead.
- `array_uncons` or `array_cons` used to regroup becomes `array_unflatten` or
  `array_flatten`. One that singled out the first element has no replacement, by
  criterion 3 of 14.4.0.
- `array_reverse` has no replacement. Order is observable in v0 only through the
  correspondence between collections, the traversal order of `fold`, and the
  output order of `collect`, so a workflow that depended on reversal must express
  it in the process that produces the collection.

Requirements a document may not have satisfied before, none of which existed to
be broken deliberately:

- Every input port of a node's target process is bound exactly once, and every
  binding entry names an input port (11). The rule was stated for an ordinary
  node's ports and is now stated for every node kind and in both directions.
- A `map` node and a `fold` node need at least one `each` source (17, 18).
- An Object-bearing carry is threaded through its target: the carried input's
  fate is the same-name output, or that input is consumed and that output
  created (16).
- The two ports of an `objects.map` entry must have the same resolved type
  (14.1). The `object_identity_map` inference pairs on name, type and phase for
  the same reason (15), so a process whose same-name ports differ in type no
  longer has a mapping inferred for them.
- `max_iterations` must be an integer of at least 1 (19).
- A `branch`'s two arms must have equal Object skeletons (20.2). This accepts
  more than the rule it replaces -- two arms that both replace the Object are
  valid, and an arm may be a composite -- and rejects the same arrangements the
  old list of four prohibitions did.

Two things a 0.0 document could have been written with that no revision made
invalid: an Object-bearing value referred to twice or not at all inside a
composite body, and a composite Object output port with no `returns` entry. 13
forbade both as properties of an incomplete skeleton in 0.0 as it does now. What
0.1 adds is the operational rule they follow from (12.2), so an implementation
that checked only 12's degree rules may begin reporting documents it used to
accept.

Written for 0.1 and read by 0.0: a document declaring `spec_version: "0.1"` is
refused by an implementation of 0.0 only if that implementation checks the
declaration, which no released one does (2.1 made it decide nothing until this
revision). Such an implementation reads the document by 0.0's rules and does not
say so.

## 0.0 - 2026-08-19

The first tagged revision: the specification as it stood before the 0.1
amendment, and the text every implementation released so far was written
against (ofplang-validate 0.1.6, ofplang-schedule 0.2.4, ofplang-run 0.3.1,
ofplang-labcode 0.2.0).

Tagged retroactively, so this section records no changes against a predecessor.
The history before it is in the commit log.
