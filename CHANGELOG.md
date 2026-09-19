# Changelog

Normative changes to `SPECIFICATION.md`, one section per specification revision.
A document names the revision it is written against with `spec_version` (2.1).

This file records what changed and what a document written against the previous
revision has to do about it. Why a rule is the way it is belongs in the
specification itself.

## 0.3 - 2026-09-19

### Added

- **4.5 Experimental features**, the category a feature belongs to when its
  specification may be changed or removed in a later revision of v0 without a
  migration path. An experimental feature is an ordinary feature for the
  purposes of 4.1 through 4.4 -- declared in `features`, derived from the body,
  subject to 4.4 where an implementation lacks it -- and the marking adds two
  things: that stability disclaimer, and a recommendation that an implementation
  report the requirement so an author notices one introduced through `$import`.
  A feature's own section may define a diagnostic validation mode in which
  checking for it is disabled; what disabling means is that section's to say.
  It changes no rule of 4.1 through 4.4.

  This is what the revision is for. A feature still being worked out has
  nowhere to live under the stability a v0 feature otherwise carries, and the
  choice without the category is between shipping nothing and promising what
  will not hold.
- **28 Feature: `units`**, the first experimental feature (4.5), annotating `Int`
  and `Float` with a unit. A unit is part of the type, so every rule that
  requires two types to be the same requires their units to agree, and 11.1,
  16, 8.1 and the structured node output rules need no addition of their own.
  Unit atoms are opaque and must be declared in a top-level `units` section;
  v0 defines no registry, no dimensions and no conversion, so a conversion is
  written as an ordinary atomic process (28.6) and is trusted as `objects.map`
  is. Every unit condition is decided at graph phase, and a unit has no runtime
  representation, so an implementation may discard every annotation once
  validation has succeeded. Also **4.2**, **4.3**, **4.4**, **2**, **2.2**,
  **2.3**, **2.4**, **2.5**, **7.1**, **7.3**, **7.4**, **8.1**, **9.2**,
  **11.1**, **11.1.1**, and **27 rules 3a, 50a, 54a, 81a, 86a**.

### Changed

- **2.3**, **26**, **27 rule 6** and **27 rule 45** name the `unit` string of a
  scheduling preference payload explicitly, and say it is unrelated to the unit
  expressions of 28. The two are different things and the word was doing double
  duty in four places.
- **2**, **2.3** and **27 rule 9** list `units` among the sections whose
  omission is an empty mapping.

### Migrating from 0.2

- `units` is now a reserved name (2.4). A document that uses it as a process
  name, port name, node id, binding name, return name, type name, trait name or
  type parameter name must rename it. Every document in this organization's
  repositories was scanned before the name was reserved -- 1984 files, 1956
  YAML documents -- and none used it in any of those positions.


## 0.2 - 2026-09-17

### Added

- **14** every path in an `objects` section must name an Object-bearing port,
  in `map`, `consume`, and `create` as well as in `transform`. Only 14.4.1 said
  it, and only of `transform`, so a `create: [outputs.n]` on an `Int` port, a
  `consume` of a Pure Data input, and a `map` between two Pure Data ports were
  all well-formed. They are not no-ops that happen to be harmless: the four
  declarations are defined over Object slots, a Pure Data port has none (5.2),
  and 13.1 quantifies over slots, so nothing downstream ever looked at such an
  entry -- including the type match 14.1 requires of a `map`. The entry named a
  port other than the one meant, and said so nowhere. Also **27 rule 20**;
  **4.4**, whose example is now an `objects` declaration rather than
  `objects.transform`; and **27 rule 28**, which stops repeating the transform
  case of a rule now stated for all four.
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
- **1.1** the resource guarantee is stated with the condition it needs. It read
  that the upper bound on the physical resources a workflow needs is fixed
  before the run, without qualification, and that was not so: an atomic process
  may declare an `Array` output whose length nothing relates to its inputs, and
  a `create` on such a port, or a `map` traversing such a value, then introduces
  a number of Objects that no length in the document reaches. The condition
  names what has to hold of an atomic's `Array` output ports -- derivable from
  `objects.map` / `objects.transform` / `object_identity_map`, or bounded by
  something v0 does not check -- and says what is still true without it: the
  workflow creates finitely many Objects, but no bound is fixed before the run.
  **Finiteness is not boundedness.** The symbolic estimate is corrected with it:
  `n x cost` holds for a scalar carry port, and a collection carry port may
  change width per iteration, the change following from the target's body. No
  rule, declaration, or validation requirement is added.

  The `branch` paragraph is corrected with it. It said that arms free to differ
  would leave "the number of plates consumed" unable to be "stated statically",
  which reads as though a count were statically stated as things are. It is
  not, and was not before: arms already create and consume different numbers,
  since a skeleton records provenance and not count (12.4.3). What differing
  arms would cost is that the shape is one family -- the disjoint union the
  same sentence names -- so the estimate is what can no longer be stated as
  one family, and the count does not enter it.
- **12.4.3**, **14.3**, **20.2**, **27 rule 26**, and `FORMULATION.md` **13.7**
  and **15.6** apply
  12.4.1, which already says that a collection port's Object slots are a family
  indexed at run time rather than one slot. Each was written as though an
  Object-bearing port carried exactly one identity: `create` introduced "a new
  Object identity", a `branch` whose arms both create had "one new Object"
  appear at its output, and a replacing carry over `n` iterations created `n`
  identities. All three hold at depth 0 only. Skeleton equality is agreement on
  provenance and not on count (12.4.3), so a `branch` whose arms create
  collections may create a different number of Objects in each; what does not
  depend on the arm is where those identities came from, which is what 24.1
  needs. No proposition or proof of `FORMULATION.md` changes: 15.6 discharges
  the iteration case of Proposition 2c, and what it gives that proof -- the
  intermediates cancel -- is unaffected by writing the widths as `w_i`.

  `FORMULATION.md` **14.6** has the same correction one step further back. The
  remark after Lemma 8 read that several identities sharing a creation point
  means a lifting by `Fam` was involved, which a `create` naming a port of
  positive depth does without any lifting. Lemma 8b is unaffected -- its proof
  is that identities within a port sit at distinct leaves of `sh(q)`, which
  holds however they got there -- but the remark is what a reader arriving at
  that section alone would take the lemma to rest on.

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
