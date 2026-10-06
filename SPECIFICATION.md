# Object-flow Programming Language v0 Specification

Revision: 0.5 (draft)  
Date: unreleased  
Supersedes: revision 0.4 (2026-10-03), released and tagged `v0.4`. CHANGELOG.md records what changed and what a document written against 0.4 has to do about it.

A document names the revision it is written against with `spec_version` (2.1). This revision makes the returns of a composite correspond one to one with its output ports (12.3), the counterpart of the correspondence 11 states between a node's bindings and its target's input ports. 0.4 required a `returns` entry only for an Object-bearing output, through Object tracking completeness (13), so a declared Pure Data output could go unreturned -- and a node that bound it read a value nothing gave -- and a `returns` entry naming no output was not an error at all.

This revision also removes the `scheduling_policies` feature: the `scheduling` section of a composite (23) and Object policy targets (24). v0 does not treat time. When an operation runs, how long an Object waits between operations, and the conditions it is kept in are left to the environment and the plan, and the section stated preferences about them that no validation outcome depended on. A `scheduling` section and the feature name are now validation errors. Sections 23 and 24 and rules 44-48 and 70 remain as stubs so that the numbering of the rest is unchanged, and the names the section used stay reserved (2.4).

Revision 0.4 made the pairing of binding sections with port kinds a rule in both directions (11). 0.3 forbade an Object-bearing value under `bind` but stated the converse only as usage, so a Pure Data value under `state` was valid, and a reader that took the section at its word treated information as material. The section a port takes is now decided by the port's declared type, and for a type parameter by its declared domain.

Revision 0.3 introduced the category of **experimental features** (4.5): a v0 feature whose specification may be changed or removed in a later revision without a migration path, where a feature outside the category would be given one. Stability is what an author relies on when they write against a revision, so a feature still being worked out needs somewhere to live that says so. Without such a place the choice is between shipping nothing and making a promise about to be broken.

The category had its first member in the same revision. `units` (28) annotates `Int` and `Float` with a unit, which is part of the type: every rule that requires two types to be the same then requires their units to agree, so the binding match, carry compatibility, structured node outputs and generic instantiation need no rule of their own. Unit atoms are opaque names declared in a top-level `units` section; v0 defines no registry, no dimensions and no conversion, so a conversion is an ordinary atomic process whose arithmetic the IR trusts. Every condition the feature states is decided at graph phase, and a unit has no runtime representation, so an implementation may discard every annotation once validation has succeeded.

Revision 0.2 stated four conditions that 0.1 relied on without stating: node ids are unique within a body, a body's node dependency graph is acyclic, a value flows only into an equal-or-later phase, and a literal is a `graph` phase value. It also named the kind of position `max_iterations` occupies, a constant slot (11.2), and corrected 1.1's upper bound on the physical resources a workflow needs, which holds only under a condition 1.1 did not state.

Revision 0.1 removed the `array_uncons`, `array_cons` and `array_reverse` transform kinds and the `last` output mode, renamed the `elidable_iso` marker to `object_identity_map` and moved it to a process's `behavior` section, added the `array_flatten` and `array_unflatten` transform kinds and the `do_while` node's reserved `exhausted` output, and introduced the Object skeleton (12.4), in terms of which Object tracking completeness, the identity-map marker, a transform's correspondence, a branch's two arms, and a scheduling policy's target are all stated.

This document is a self-contained specification for a dataflow-oriented workflow IR with linear Object tracking. It focuses on successful workflow semantics, Object/data flow, structured control, and type modeling. Timing, runtime failures, exceptions, retries, cancellation, compensation, and recovery are intentionally outside the scope of v0.

v0 uses a **Core + Features** model. The canonical v0 form contains a `features` section listing all features required by the document body. If `features` is omitted, the required features are derived from the document body and the document is interpreted as if the derived feature set had been written explicitly.

---

## 1. Design Goals

The language is intended to describe workflows involving both ordinary data and physical or logical Objects that must be tracked linearly.

The key goals of v0 are:

1. Keep the core language small.
2. Support structural `$import` for splitting descriptions across files without introducing a module system.
3. Track Object-bearing values without implicit creation, loss, duplication, discard, or consumption.
4. Support structured dataflow patterns through explicit feature-gated node kinds.
5. Leave when operations run to the implementation; v0 states no timing preference or constraint.
6. Avoid deep subtyping, inheritance, refinement types, Optional, Result, union branch outputs, and dependent typing in v0.
7. Express common operational grouping through nominal type traits rather than subtyping or parameterized Object types.
8. Keep contract-visible views and trait membership explicit in YAML, except for v0 built-in primitive and Array views.
9. Keep feature derivation mostly syntactic and easy to validate from YAML structure.
10. Keep the shape of the Object-flow graph statically determined. Information that becomes known only at run time is admitted solely as a finite scalar parameter that indexes that shape; it may not change the shape itself.

### 1.1 Design principle: static shape

The central design decision of v0 is that the shape of the Object-flow graph is statically determined. Information that becomes known only at run time is admitted solely as a finite scalar parameter that indexes that shape.

Every structured node kind is designed to that principle:

| Node | Shape | Run-time parameter | Source of finiteness |
|---|---|---|---|
| `map` | the body L times, in parallel | collection length L | length of `each` |
| `fold` | the body L times, in series | collection length L | length of `each` |
| `do_while` | the body n times, in series | iteration count n | `max_iterations` |
| `branch` | a single shape | none | - |

The shape of `map`, `fold`, and `do_while` is one family indexed by a single scalar. A traversal's Object consumption can therefore be estimated symbolically as `L x cost`, and for `do_while` the iteration count is bounded statically, given by `max_iterations`.

Where a loop's carry port is a scalar Object port, the series of Objects it holds is one series whatever n is: n stretches its length and does not change its form, and the loop's cost is `n x cost`. Where the carry port is a collection, n stretches the series and the target may also change its width from one iteration to the next. The per-iteration change follows from the target process's body and needs no declaration; the loop's cost is that change compounded n times. Both are bounded before the run under the condition below. Only the first is linear in n.

`branch` is the only node kind without that property. If the two arms were allowed to differ in how they route Object identity, the shape would not be one family but the disjoint union of two shapes, and that is multiplicative under composition: a composite containing k branches would have up to 2^k shapes. Resource estimates and the tracking of an individual Object could then no longer be stated as one family.

Section 20 requires the two arms to agree on the Object identities they expose for that reason. **`branch` is a branch in the Data dimension, and must be an identity in the Object dimension.** A conditional repetition that consumes and creates Objects is written with `do_while`, which satisfies this principle because it pushes its shape into the scalar n and bounds it with `max_iterations`.

For the same reason v0 does not admit a `branch` whose arms declare different port names, an Object-bearing output produced by one arm only, or a conditional Object creation or consumption.

**Restated in terms of resources**

Under the condition below, the upper bound on the physical resources a workflow needs is fixed before the workflow runs. Just as `max_iterations` bounds the iteration count, the static shape of the Object-flow graph bounds the number of Objects consumed. A description whose consumption depends on a branch taken at run time is not admitted for that reason.

**The condition.** Every `Array` output port of every atomic process has a length that is either derivable or bounded.

```text
derivable  the length follows from the process's own Object declarations:
           objects.map (14.1), objects.transform (14.4), or the
           object_identity_map inference (15). Each relates the output's slot
           family to an input's, so the length comes with it.
bounded    the length is bounded by something outside those declarations -- a
           contract, or knowledge of the process. v0 does not check this. A
           statement of atomic process behavior is trusted, as 14.1 trusts one.
```

A Pure Data `Array` output has no derivation available: the `objects` section describes Object behavior, and a Pure Data port has no Object slots for it to relate. Such a port meets the condition only by the second clause.

Where the condition holds and every `run` phase argument is given, an upper bound on the number of Objects the workflow creates is determined, and is constructed by one traversal of the body. Every other length in a document is reached from those: a literal (11.1.1), an argument, a collected output (17, 18), a transform (14.4), the arms of a `branch` (20), or a loop bounded by `max_iterations` (19). The process dependency graph and each body's node dependency graph are acyclic (10.2), so the traversal terminates.

Where the condition does not hold, the workflow still creates finitely many Objects -- every traversal is over a finite collection and every loop is bounded -- but no upper bound is fixed before the run. **Finiteness is not boundedness**, and the table above says finiteness.

The condition is a property of atomic processes, which v0 cannot see into. It is stated here rather than as a validation rule for that reason. An implementation may report an atomic `Array` output port that meets neither clause; a document with no such port is one for which this section's guarantee holds.

How many Objects a declaration introduces at a collection port is 14.3.

This principle is weaker than the uniqueness of the Object skeleton (12.4.7). Two descriptions may agree on how many Objects they consume and create and still have different skeletons, if what an output Object's identity comes from is not the same; 12.4.7 forbids that separately.

An operation whose consumed quantity genuinely varies at run time is expressed by pushing the variation inside an atomic process and passing that process the Objects corresponding to the bound. Where a reagent container must be replaced depending on the number of dispenses, for example, the process is written to take an `Array<Reagent>` of the estimated upper bound and to return the remainder. Which container is dispensed from is the implementation's internal business and is not visible in the Object flow the document describes.

---

## 2. Document Structure and Processing Order

A v0 document may contain:

```yaml
spec_version: "0.5"
features: []
units: {}
traits: {}
types: {}
processes: {}
entry: main
```

### 2.1 Specification version metadata

A v0 document may declare the revision of this specification it is written against, using the top-level `spec_version` field.

```yaml
spec_version: "0.5"
```

If present, the value must be a string using a two-number version format:

```text
MAJOR.MINOR
```

A malformed `spec_version` value is a validation error. Omission of `spec_version` is allowed, and a document that omits it is read as being written against the revision the implementation implements.

**What the declaration decides, and what it does not**

`spec_version` does not select how a document is interpreted. There is one set of rules -- the revision the implementation implements -- and every document is read by them. Declaring an earlier revision does not ask for that revision's rules, and no implementation is required to keep more than one set.

What it decides is whether the implementation is willing to answer at all. An implementation implements one revision, and:

```text
same MAJOR, MINOR <= the implemented MINOR   accepted
same MAJOR, MINOR >  the implemented MINOR   validation error
different MAJOR                              validation error
```

A document declaring a later revision is refused because the implementation cannot know what it was written to mean: the constructs it uses may be ones this revision does not define, and reading it by these rules would answer a question that was not asked. A different MAJOR is a different generation of the language.

An earlier MINOR is accepted rather than refused because a revision within one MAJOR is an edit of the same language, and where such a document uses something the current revision removed, the error naming that construct says more than a version mismatch would. What the acceptance does not promise is that the document is still valid: it is read by the current rules, and CHANGELOG.md records what changed between revisions.

**The current revision**

```text
0.5
```

An implementation states the revision it implements. Two implementations of different revisions may therefore disagree about one document, and the declaration is what makes that disagreement legible rather than silent.

### 2.2 Processing order

The processing order is:

1. Resolve `$import` structurally.
2. Validate document shape, reserved keys, and reserved metadata field formats.
3. Resolve types, unit atoms, type traits, view schemas, phases, and process references.
4. Derive required features from the expanded document body.
5. Validate the declared `features` section, if present.
6. Type-check node bindings and structured node outputs.
7. Check Object tracking completeness and linearity.
8. Resolve the entry process.

`$import` has no runtime meaning after import resolution.


### 2.3 YAML shape strictness

v0 portable YAML is closed by default. At every defined mapping position, only keys explicitly defined by the v0 specification are allowed. Unknown keys are validation errors.

Implementation extension keys are allowed only when they use the reserved extension-key prefix `x-`. A document containing `x-` extension keys is not strict portable v0 unless the validator is explicitly run in an extension-tolerant mode.

The v0-defined optional `description` key (2.7) is allowed at the document root and at trait, type, and process definition mappings. It is a v0-defined key, not an extension key, so a document using `description` remains strict portable v0.

For every mapping position, the specification defines a shape schema consisting of allowed keys, required keys, value kinds, and conditional requirements. A value-kind mismatch, missing required key, unexpected sequence item shape, or unexpected `null` value is a validation error unless explicitly allowed by the relevant schema rule.

`null` values are not valid in v0 portable YAML. Future feature extensions may define nullable values explicitly, but v0 core does not.

Omitted `units`, `traits`, `types`, `inputs`, and `outputs` sections are interpreted as empty mappings. The `features` section may be omitted and is then interpreted as the feature set derived from the expanded document body. The `processes` section is required.

A process that has no input ports may omit `inputs`. A process that has no output ports may omit `outputs`. Omitted `inputs` and `outputs` are equivalent to empty mappings.

Each node kind defines the binding and output-control sections that are valid for that kind. A section that is not defined for the node kind is a validation error.

### 2.4 Identifier syntax and reserved names

v0 portable YAML uses case-sensitive ASCII identifiers.

Unless a more specific rule is defined for a syntactic position, user-defined identifiers must match:

```text
[A-Za-z_][A-Za-z0-9_]*
```

The period character `.` is not allowed in v0 identifiers. It is reserved for possible future use by namespaces, modules, qualified names, or other path-like constructs. This restriction applies to process names as well as type names, trait names, type parameter names, port names, node ids, binding names, return names, and view field names.

The recommended style is:

```text
Type names: PascalCase or UpperCamelCase
Trait names: PascalCase or UpperCamelCase
Type parameter names: short uppercase names such as T, P, A, or B
Process names: lower_snake_case
Port names, node ids, binding names, return names, and view field names: snake_case or another ASCII identifier style chosen by the author
```

These style recommendations are not validation requirements unless an implementation chooses to enforce them as non-portability diagnostics.

Names are case-sensitive. For example, `Sample`, `sample`, and `SAMPLE` are distinct names.

Unicode characters are not allowed in identifiers in portable v0. UTF-8 text is allowed in YAML string values and comments, including Japanese text.

The built-in names `Bool`, `Int`, `Float`, `String`, `Array`, and `Numeric` are reserved and must not be redeclared as user-defined type names, trait names, or type parameter names.

Input port names and output port names are separate namespaces. Therefore, an input port and an output port of the same process may have the same name. Duplicate names within `inputs` are validation errors. Duplicate names within `outputs` are validation errors.

Within one composite body, node ids are unique. Two nodes with the same `id` are a validation error, since `<node_id>.<output>` would then not name one value.

The key `$import` is the only `$`-prefixed reserved key defined by v0. Any other `$`-prefixed key is a validation error in portable v0.

The following names are reserved and must not be used as process names, port names, node ids, binding names, return names, type names, trait names, or type parameter names:

```text
inputs
outputs
self
view
objects
features
traits
types
processes
entry
body
nodes
returns
state
bind
carry
each
args
then
else
condition
scheduling
policies
during
object
from
to
kind
process
phase
type
value
script
contracts
requires
ensures
behavior
exhausted
units
```

Every name in that list but `exhausted` is a structural key of the document, or was one. `scheduling`, `policies`, `during`, `object` and `to` were keys of the `scheduling` section, which revision 0.5 removed (23); they stay reserved, so that a name the language may use again does not become a process, port or node name in the meantime. `traits` stays reserved as the name of the top-level section that declares type traits (7.3); a process declares its behavior markers under `behavior` (15), which is a different vocabulary. `exhausted` is there for the other reason a name is reserved: a `do_while` node exposes a reserved output of that name (19.3), so were a target process free to declare an output called `exhausted`, a reference to `<node>.exhausted` would name two different values and there would be no way to say which.

Unit atom names occupy a namespace of their own, declared in the top-level `units` section (28.1). The reserved names listed above do not apply to unit atom names, and a unit atom name may coincide with a type name, a trait name, a process name, or a port name.

View field names are deliberately absent from that list: they must match the identifier grammar and must not contain `.`, but a reserved name is allowed. A view field name occurs in only two places, and in neither can it be confused with a structural key. In a view schema the field name is the outer key and the declaration keys `type` and `value` are inside it, so `value: {type: Int}` declares a field named `value`. In a contract reference the field is the single segment after `.view`, so `inputs.x.view.value` reads that field. Reserving names here would cost expressiveness without removing an ambiguity.

A validator may report a more specific error when a name fails the grammar for its syntactic position or conflicts with a reserved name.

The name `description` is a v0-defined metadata key at certain definition mappings (2.7) but is not a reserved identifier name. It may be used as a user-defined process name, port name, node id, binding name, return name, view field name, type name, trait name, or type parameter name.


### 2.5 Type expression syntax

In v0 portable YAML, every `type` field value must be a YAML string scalar containing a v0 type expression.

The v0 type expression grammar is:

```text
TypeExpr      ::= TypeAtom UnitSuffix? | ArrayType
ArrayType     ::= "Array" "<" S? TypeExpr S? ">"
TypeAtom      ::= Identifier
Identifier    ::= [A-Za-z_][A-Za-z0-9_]*
S             ::= one or more ASCII space or tab characters

UnitSuffix    ::= "[" UnitExpr "]"
UnitExpr      ::= "1" | UnitTerm (("*" | "/") UnitTerm)*
UnitTerm      ::= UnitIdent Exponent?
Exponent      ::= "^" "-"? [1-9][0-9]*
UnitIdent     ::= [A-Za-z_][A-Za-z0-9_]*
```

Whitespace is allowed only immediately inside the angle brackets of `Array<T>`. Therefore the following type expressions are valid and have the same meaning:

```text
Array<T>
Array< T >
Array<   T   >
Array<Array< Sample >>
```

Whitespace between `Array` and `<` is not allowed. Therefore the following are validation errors:

```text
Array <T>
Array < T >
```

The recommended canonical style is to write type expressions without whitespace:

```text
Array<T>
Array<Array<Sample>>
```

A unit suffix is defined by the `units` feature (28). Whitespace is not allowed inside a unit suffix, nor between a type atom and its suffix.

The only built-in primitive Data types are:

```text
Bool
Int
Float
String
```

The only built-in type constructor is:

```text
Array<T>
```

`Array` requires exactly one type argument. Nested Arrays such as `Array<Array<Sample>>` are valid.

The names `Bool`, `Int`, `Float`, `String`, `Array`, and `Numeric` are reserved and must not be redeclared as user-defined type names, trait names, or type parameter names.

A type atom must resolve to exactly one of:

```text
a built-in primitive type
a top-level user-defined type
a type parameter declared by the current process
```

A type parameter must not shadow a top-level user-defined type name or a reserved built-in name.

Unknown type names are validation errors. Malformed type expressions are validation errors. A `type` field whose value is not a YAML string scalar is a shape validation error.

No other type constructors or type syntax are defined by v0. In particular, `Optional<T>`, `Result<T,E>`, union types, nullable suffixes such as `T?`, map types, tuple types, function types, and multiple type arguments are not valid v0 type expressions.

A unit suffix is valid only where the `units` feature is in effect, and only on `Int` and `Float` (28.2).

Names such as `Optional`, `Result`, and `Map` are not reserved by v0 merely because future versions may define additional type constructors.

Implementations may normalize parsed type expressions internally, but such normalization has no effect on document semantics and must not silently repair, case-normalize, or otherwise rewrite malformed type expressions.

### 2.6 Reference syntax

In v0 portable YAML, every `from` field value must be a YAML string scalar containing a v0 reference expression.

The following reference syntaxes are distinct and are valid only in their specified contexts.

#### 2.6.1 Body dataflow references

Node bindings, `branch.condition.from`, and `body.returns` use body dataflow references:

```text
BodyRef ::= "inputs" "." Identifier | Identifier "." Identifier
```

The first form refers to an input port of the current composite process. The second form refers to an output of a direct child node in the same composite body.

Body dataflow references must not include `.view`, nested node paths, process names, or `outputs.*`.

#### 2.6.2 Atomic Object paths

Atomic `objects` declarations use Object paths:

```text
ObjectInputPath  ::= "inputs" "." Identifier
ObjectOutputPath ::= "outputs" "." Identifier
```

`objects.map` maps Object output paths to Object input paths. `objects.consume` lists Object input paths. `objects.create` lists Object output paths. `objects.transform` role values use Object input paths under `inputs` and Object output paths under `outputs`.

v0 Object paths refer to process ports. They do not directly address Array elements, Object slots, `.view` fields, or implementation-specific internal fields. Object slots are derived from the resolved port type.

#### 2.6.3 Scheduling Object target references

Removed in revision 0.5 with the `scheduling` section (23).

#### 2.6.4 Temporal references

Removed in revision 0.5 with the `scheduling` section (23).

#### 2.6.5 Contract view references

Contract expressions use contract view references.

In `requires`, references may use:

```text
inputs.<port>.view
inputs.<port>.view.<field>
```

In `ensures`, references may use:

```text
inputs.<port>.view
inputs.<port>.view.<field>
outputs.<port>.view
outputs.<port>.view.<field>
```

Contract expressions do not directly reference ports without `.view`. Although every valid contract reference is a view reference, v0 requires `.view` to be written explicitly.

In v0, contract view field references address one view field at a time. Nested field paths below a view field are not defined in v0.

#### 2.6.6 Binding source entries

A binding source entry must contain exactly one of:

```text
from
value
```

A source entry containing both `from` and `value`, or neither, is a validation error.

A constant slot (11.2) is filled by a source entry of the same shape.

#### 2.6.7 Structured condition references

`branch.condition.from` uses a body dataflow reference.

`do_while.condition.output` is an output name of the target process, not a body dataflow reference. The named target process output must exist and must be a Boolean Data output. It is evaluated after each invocation of the target process. The `do_while` node repeats while this output value is `true` and exits when it is `false`, subject to `max_iterations`.

The condition output is an ordinary Data output of the target process, and how the node exposes it follows the output-mode rules of 19.1. It may be exposed using `collect` or `drop`, and where it is also a carry binding it is exposed with `mode: carry` (19.1 rule 5). If `do_while.outputs` is omitted and the condition output is not a carry binding, it is dropped by default.

#### 2.6.8 Reference resolution

This applies to every reference form of 2.6, not only to the preceding one.

Reference parsing and reference resolution are separate validation steps. A malformed reference is a validation error. A syntactically valid reference whose target does not exist is an unknown reference validation error. A reference that is not valid in its syntactic context is an invalid reference scope validation error. A reference whose resolved type or phase does not satisfy the target requirement is a type or phase validation error.

### 2.7 Descriptive metadata

A v0 document may attach optional human-readable descriptive metadata using the reserved `description` field.

`description` is allowed only at the following definition mappings:

```text
the document root mapping
a trait definition under traits.<TraitName>
a type definition under types.<TypeName>
a process definition under processes.<name>
```

At each of these positions, `description` is optional. If present, its value must be a YAML string scalar. UTF-8 text is allowed, including Japanese. A `null`, sequence, mapping, or non-string scalar value is a validation error, consistent with 2.3.

`description` is metadata only. It does not affect document interpretation, validation semantics, feature derivation, type checking, Object tracking, or runtime behavior. Unlike `spec_version` (2.1), which is checked although it does not select an interpretation, nothing at all is decided by it. Two documents that differ only in `description` values are semantically identical.

v0 does not define `description` on input ports, output ports, type parameters, view fields, nodes, bindings, contracts, or any mapping position not listed above. The descriptive meaning of a port is expected to be documented in the enclosing process `description`, and the descriptive meaning of a view field in the enclosing type `description`. A `description` key at an undefined position is an unknown-key validation error under 2.3.

Unlike the metadata keys `type`, `value`, and `phase`, the name `description` is intentionally not added to the reserved identifier list (2.4). Because `description` is meaningful only at the definition mappings listed above, it does not conflict with user-defined identifiers used elsewhere, such as a port named `description` or a view field named `description`.

When `$import` resolution merges mappings, a `description` key is treated like any other key. A duplicate `description` at the same expanded mapping level after import resolution is a duplicate-key validation error (3.2, 3.3).

---

## 3. Structural Imports

v0 supports structural imports using the reserved key `$import`.

```yaml
$import: <path>
```

The value of `$import` may also be a sequence of paths. In that case, the files are imported in list order.

```yaml
$import:
  - <path>
  - <path>
```

`$import` is resolved before validation, type checking, Object tracking completeness checks, feature derivation, and entry process resolution.

`$import` is structural inclusion, not a module system. It does not introduce namespaces, aliases, selective imports, visibility rules, or re-export semantics.

Import cycles are validation errors.

`$import` is a reserved key in v0. Top-level `imports` and `$imports` are not part of v0.

### 3.1 Import paths

In v0, the value of `$import` is a path string, or a sequence of path strings.

Relative `$import` paths are resolved relative to the file containing the `$import`.

```yaml
$import: ./types.yaml
```

```yaml
$import:
  - ./base.yaml
  - ../shared/labware.yml
```

The v0 core language does not forbid `..` path segments.

Absolute paths without a URI scheme are allowed in v0, but they are discouraged because they reduce portability across machines, directory layouts, and packaging contexts. Authors should prefer relative paths for portable documents.

URI-scheme references such as `file:`, `http:`, and `https:` are implementation extensions in v0 and are not portable v0.

A `$import` target denotes a whole YAML document. URI fragments in `$import` values are not defined in v0 and are validation errors for portable v0 documents.

v0 does not require a specific file extension. Authors should prefer `.yaml` or `.yml` for ofplang documents and imported fragments.

### 3.2 Import insertion rules

When `$import` appears in a mapping, each imported document must be a mapping. Imported mappings are merged into the surrounding mapping.

Example:

```yaml
types:
  $import:
    - ./base_types.yaml
    - ./labware_types.yaml

  LocalType:
    domain: data
```

Import resolution is equivalent to replacing the surrounding mapping with the merge of:

1. imported mappings in list order, and
2. local keys in the surrounding mapping.

Duplicate keys are validation errors. Duplicate keys are checked at the same mapping level in the final expanded mapping. v0 uses shallow structural merge at the insertion location only; it does not define deep merge, override, or conflict-resolution semantics.

When `$import` appears as an item of a sequence, each imported document may be either a single item or a sequence.

If the imported document is a sequence, its items are spliced into the surrounding sequence at the import position.

If the imported document is not a sequence, it is inserted as one sequence item.

Example:

```yaml
processes:
  main:
    kind: composite
    body:
      nodes:
        - id: prep
          process: prep
        - $import: ./process_nodes.yaml
        - id: finish
          process: finish
```

If `process_nodes.yaml` contains a sequence of nodes, those nodes are spliced between `prep` and `finish`.

### 3.3 Imports and reserved metadata

`$import` is direct structural inclusion. Imported fragments normally do not contribute top-level reserved metadata such as `spec_version`.

If an imported fragment contributes a reserved metadata key into a mapping that already contains the same key, the result is a duplicate key after import resolution and is a validation error. v0 does not define override, version negotiation, or multi-version import semantics.

After import resolution, the expanded document is validated as ordinary ofplang.

### 3.4 Import resolution boundary conditions

The value of `$import` must be either a non-empty YAML string scalar or a non-empty sequence of non-empty YAML string scalars.

An empty `$import` sequence, an empty import path string, `null`, or any non-string import path value is a validation error.

Relative `$import` paths are resolved relative to the file containing the `$import`.

Import resolution is recursive. If an imported document itself contains `$import`, those imports are resolved before the importing document is validated as an expanded document.

`$import` is an import-resolution construct only. It has no runtime meaning and does not remain as a key in the expanded document.

An import target must be a single YAML document. YAML multi-document streams are not portable v0 import targets.

When `$import` appears in a mapping, each imported document must expand to a mapping. Imported mappings are merged into the surrounding mapping using v0 shallow structural merge rules.

When `$import` appears as an item of a sequence, each imported document may expand to either a sequence or a single item. If the imported document expands to a sequence, its items are spliced into the surrounding sequence at the import position. Otherwise, it is inserted as one sequence item.

Import cycles are validation errors. Implementations must detect cycles after resolving relative paths against the importing file. Implementations should use a stable canonical file identity when available.

The same import target may be imported more than once, but the expanded result must still satisfy all duplicate-key, namespace uniqueness, shape, and semantic validation rules.

If an import target cannot be found, cannot be read, cannot be parsed as YAML, contains more than one YAML document, has an invalid root shape for its insertion location, or otherwise cannot be expanded, import resolution fails and the document is invalid.

URI-scheme references such as `file:`, `http:`, and `https:` are implementation extensions in v0 and are not portable v0. URI fragments in `$import` values are not defined in v0 and are validation errors for portable v0 documents.

Extension keys imported from another document are treated like extension keys written locally. Duplicate-key detection applies to extension keys as well.

---

## 4. Feature Model

### 4.1 Canonical feature declaration

`features` declares the feature set required by the document.

The canonical v0 form contains a `features` section listing all features required by the document body.

If `features` is omitted, the required feature set is derived from the document body, and the document is interpreted as if that derived set had been written explicitly.

If `features` is present, it must include every feature required by the document body. Missing required features are validation errors.

A `features` section may include features that are not required by the document body. Such extra features are allowed and do not change the semantics of the document.

Every feature name listed in `features` must be a feature name defined by v0. Unknown feature names are validation errors.

If a document requires a feature that is defined by v0 but not supported by a particular implementation, the document is valid v0 but unsupported by that implementation.

### 4.2 v0 feature names

Defined v0 feature names are:

```text
node_map
node_fold
node_do_while
node_branch
generic_processes
python_script_processes
units
```

`units` is an **experimental** feature (4.5).

`scheduling_policies` was removed in revision 0.5 (23). It is not a v0 feature name, so a `features` section naming it is a validation error (rule 13).

### 4.3 Feature derivation

Required features are derived primarily from explicit YAML syntax.

Structured node features are derived from node `kind` values:

```text
kind: map       -> node_map
kind: fold      -> node_fold
kind: do_while  -> node_do_while
kind: branch    -> node_branch
```

A process with a `type_params` section requires:

```text
generic_processes
```

Feature derivation is performed on the expanded document body, so `generic_processes` is derived from a `type_params` section present after `$import` resolution. `$import` itself is a structural mechanism resolved before feature derivation (spec 2.2) and is not a feature.

A `script` section with `script.language: python` requires:

```text
python_script_processes
```

A type expression containing a unit suffix (28.2) requires:

```text
units
```

A non-empty `units` section requires:

```text
units
```

A unit suffix requires the feature even where the unit expression is dimensionless. `Float[1]` is the same type as `Float` (28.5), but writing it is use of the syntax.

### 4.4 Unsupported features vs validation errors

A validation error means the IR does not satisfy v0 well-formedness rules.

An unsupported feature means the IR is valid v0 but requires a v0-defined feature not supported by a particular implementation.

Examples of validation errors:

```text
unknown feature name
required feature missing from an explicit features section
malformed spec_version metadata value
spec_version names a revision the implementation does not implement
Object-bearing output is unused
Object-bearing output fans out
Object-bearing input has no source
Object-bearing input has multiple sources
objects path does not exist
objects section is incomplete
objects section appears on a composite process
import cycle
duplicate key after import resolution
invalid import shape for insertion location
unknown type name
unknown trait name
unknown implemented trait
malformed view schema
view field has Object-bearing type
view field static value does not conform to its declared type
contract references an unknown view field
contract contains a statically known basic type error
script process has Object-bearing input or output
script process contains an objects section
script process declares an unsupported script language
fold or do_while carry output is missing
branch arm has one-sided Object-bearing output
branch arms have unequal Object skeletons
Object-bearing value has graph phase
value phase is later than the port or constant slot it fills
Object-bearing value fills a constant slot
duplicate node id within a composite body
cycle in a composite body's node dependency graph
Object slot has multiple incompatible fates
Object slot has multiple incompatible provenances
unknown v0 transform kind
invalid transform role set
transform role type mismatch
Pure Data path appears in an objects declaration
transform path participates in multiple incompatible Object fates or provenances
node carries a section that is not valid for its node kind
process declares a behavior marker v0 does not define
process Object skeleton is not complete
objects.map source and target port types do not match
do_while max_iterations is less than 1
map or fold node has no each source
Object-bearing carry is neither preserved nor replaced by the target process
node binding names no input port of the target process
target process input port is not bound, or is bound more than once
composite output port has no returns entry
returns entry names no output port of the composite
do_while node outputs section lists the reserved output exhausted
zip-equal traversal length mismatch known at graph phase
unit mismatch between a binding source and its target port
unit mismatch in a contract expression
unit suffix on a type that is not Int or Float
malformed unit expression
undeclared unit atom
duplicate unit atom name in the units section
units entry whose value is not an empty mapping
```

The last entry is conditional on what the implementation determines rather than an
unconditional obligation. It names a condition that v0 classifies by the earliest phase
at which it is determined, and v0 requires no static inference of Array or traversal
lengths, so an implementation that never establishes a length mismatch at graph phase
reports the condition at run or data phase instead, as the entry below says. Its absence
from such an implementation's validation errors is correct behavior rather than a missing
check.

Examples of run-start or runtime data errors:

```text
zip-equal traversal length mismatch, when the mismatch is first determined at run or data phase
```

Examples of unsupported features:

```text
workflow uses kind: map but implementation lacks node_map
workflow uses kind: fold but implementation lacks node_fold
workflow uses kind: do_while but implementation lacks node_do_while
workflow uses kind: branch but implementation lacks node_branch
workflow uses generic processes but implementation lacks generic_processes
workflow uses script.language: python but implementation lacks python_script_processes
workflow uses unit suffixes but implementation lacks units
```

### 4.5 Experimental features

A v0-defined feature may be marked **experimental**. An experimental feature is an ordinary v0 feature for the purposes of 4.1 through 4.4: it is declared in `features`, it is derived from the document body by the rules of 4.3, and an implementation that does not support it is subject to 4.4.

The marking adds two things. **Stability**: the specification of an experimental feature may be changed or removed in a later revision of v0 without a migration path, where a non-experimental feature would be given one. **Diagnostics**: an implementation should report that a document requires an experimental feature, whether the requirement was declared or derived, so that an author can notice a dependency introduced through `$import`.

A document that depends on an experimental feature should list that feature explicitly in `features` even though 4.1 allows the set to be derived. This is a recommendation, not a validation requirement.

An experimental feature's own section may define a diagnostic validation mode in which checking for that feature is disabled. Such a mode is not a conformance criterion, and the conditions it must satisfy are stated by that section.

---

## 5. Data, Objects, and Object-Bearing Values

### 5.1 Data vs Object

A type has a domain:

```yaml
types:
  Image:
    domain: data

  Robot:
    domain: object
```

- `domain: data` values are unrestricted Pure Data.
- `domain: object` values are Object identities tracked linearly.

An Object is not merely a record of attributes. It represents a workflow token for something whose identity, use, and lifecycle must be tracked.

v0 tracks physical Object identity and explicit Object creation/consumption. It does not model automatic identity transfer across physical replacement.

### 5.2 Object-bearing values

A value is **Object-bearing** if its type contains one or more Object slots.

v0 has only one built-in type constructor:

```text
Array<T>
```

Object slots are defined recursively:

```text
object_slots(Atomic Data)   = []
object_slots(Atomic Object) = [self]
object_slots(Array<T>)      = elements[*].object_slots(T)
```

Examples:

```text
Image                  => []
Cup                    => [self]
Array<Cup>             => [elements[*]]
Array<Array<Cup>>      => [elements[*].elements[*]]
Array<Image>           => []
```

If `object_slots(T)` is empty, values of type `T` are Pure Data. If not, they are Object-bearing and subject to linear Object tracking.

Object identity belongs to Object slots, not to an enclosing Object-bearing value merely because that value contains Object slots. For an Atomic Object type, the `self` slot carries the Object identity. For `Array<T>`, the Array container is not itself a separate Object identity; the contained Object identities are the identities of the Object slots in the elements.

Linearity for Object-bearing values is an accounting rule for all contained Object slots. It must not be interpreted as assigning a distinct physical Object identity to the container value itself.

### 5.3 Object-bearing containers

An Object-bearing value may be bound as an ordinary node `state` input, a loop `carry` value, a branch `args` value, or a node output. Object-bearing containers such as `Array<Cup>` are treated as Object-bearing values.

When an Object-bearing container is passed through a binding, the container value is treated as one linear value at the binding level. The binding does not by itself destructure or traverse the contained Object slots.

Structured `each` bindings are an explicit exception: an Object-bearing collection used as an `each` source is linearly used by the structured node, and its contained Object slots are distributed to per-element invocations according to the structured node semantics. The source collection value must not also be connected elsewhere in the same body or returned directly.

Object tracking completeness is still checked at Object slot level. A container-level `objects.map` or `objects.transform` must account for every contained Object slot according to its declared meaning.

---

## 6. Phases

Every process port has a `type` and a `phase`. Process input ports are declared under `inputs`, and process output ports are declared under `outputs`.

```yaml
inputs:
  x:
    type: T
    phase: data
```

Phase order:

```text
graph < run < data
```

Meaning:

- `graph`: available during graph construction, validation, and type or shape determination.
- `run`: fixed at workflow run start and fixed for one run.
- `data`: ordinary runtime dataflow values, including most Object-bearing values.

Allowed phase flow:

```text
graph -> graph/run/data
run   -> run/data
data  -> data only
```

Invalid phase flow:

```text
data -> run/graph
run  -> graph
```

A value may flow from an earlier phase to a later phase, but not from a later phase to an earlier phase.

### 6.1 Phase of Object-bearing values

Object-bearing values are normally `data` phase values.

A `run` phase Object-bearing value is allowed only for rare cases where the Object identity is fixed at workflow run start, such as an initial resource or externally supplied Object token.

Object-bearing values must not have `graph` phase in v0.

When a `run` phase Object-bearing value flows into a `data` phase port, it is treated as the same linear Object token entering ordinary runtime dataflow. This phase lowering does not create, duplicate, consume, or otherwise alter the Object identity.

Most Object-producing process outputs should be `data` phase.

### 6.2 Phase-dependent error classification

Some requirements depend on values such as Array lengths or traversal emptiness. These facts may become known at different phases. v0 classifies such errors by the earliest phase at which the violation is determined.

If a violation is determined at `graph` phase, it is a graph-time validation error. The IR is not a valid portable v0 document.

If a violation is not known at `graph` phase but is determined at `run` phase, it is a run-start validation error or preflight error for that run. The IR may still be structurally valid v0, but the run must not proceed under those run-phase values.

If a violation is determined only at `data` phase, it is a runtime data error. Runtime data errors are outside the core validation semantics of v0, but this specification may define standard execution behavior for such cases where useful.

This principle applies to phase-dependent conditions such as a zip-equal length mismatch, and similar checks whose truth may depend on graph, run, or data phase values.

---

## 7. Types, Traits, and View Metadata

### 7.1 Built-in primitive Data types

v0 defines the following built-in primitive Data types:

```text
Bool
Int
Float
String
```

These primitive types are Pure Data types. They have no Object slots.

Under the `units` feature, `Int` and `Float` may carry a unit suffix, which is part of the type (28). A unit suffix does not change the set of built-in primitive types and does not give a primitive type an Object slot.

v0 defines one built-in type constructor:

```text
Array<T>
```

`Array<T>` may be Pure Data or Object-bearing depending on `T`. If `object_slots(T)` is empty, `Array<T>` is Pure Data. Otherwise, `Array<T>` is Object-bearing.

No other primitive types or type constructors are defined by v0. Additional primitive-like types or constructors are implementation extensions unless represented as user-defined nominal Data types.

### 7.2 Nominal Data and Object types

User-defined atomic types are nominal and are declared in the top-level `types` section. Each user-defined type has a `domain`:

```yaml
types:
  Image:
    domain: data

  Robot:
    domain: object
```

Object types should represent meaningful workflow state classes.

Guideline:

```text
If two physical Objects are not safely interchangeable as workflow state values,
they should normally be represented as different Object types.
```

Examples of appropriate Object types:

```text
Robot
Camera
Cup
Plate96
Plate384
Tube15ml
Sample
```

User-defined Data types are also nominal. Their internal representation is not specified by v0 unless exposed through their view schema.

### 7.3 Type traits

v0 defines one built-in type trait:

```text
Numeric
```

The built-in primitive Data types that satisfy `Numeric` are:

```text
Int
Float
```

Under the `units` feature, `Numeric` is satisfied only by a numeric primitive type whose unit is dimensionless.

```text
Numeric<Float>      satisfied
Numeric<Float[1]>   satisfied, since Float[1] is Float (28.5)
Numeric<Float[s]>   not satisfied
Numeric<Int[s]>     not satisfied
```

A generic process constrained by `Numeric` therefore does not accept a unit-annotated value. v0 defines no way to abstract over a unit (28.12), and a process that computed on an arbitrary unit could not state what unit its result has.

`Numeric` is a closed built-in trait in v0. It must not be redeclared in the document's top-level `traits` section, and user-defined types must not implement `Numeric`. `Numeric` may be used only as a generic type constraint over v0 primitive numeric types.

Process-level behavior markers are declared under a process's `behavior` section (15) and are a separate vocabulary; they are not type traits and this section does not govern them.

All non-built-in type trait names used by user-defined types or generic constraints must be declared in the document's top-level `traits` section.

```yaml
traits:
  PlateLike: {}
  HasWells: {}
  Labware: {}
```

Document-defined traits are nominal membership markers only. A document-defined trait does not imply fields, operators, conversions, subtyping, inheritance, Object behavior, or view structure. The closed built-in primitive trait `Numeric` is satisfied only by `Int` and `Float`; it does not define user-extensible operators, implicit conversions, subtyping, or numeric behavior for user-defined types.

User-defined nominal Data and Object types may implement declared non-built-in traits using `implements`:

```yaml
traits:
  PlateLike: {}
  HasWells: {}
  Labware: {}

types:
  Plate96:
    domain: object
    implements:
      - PlateLike
      - HasWells
      - Labware
    view:
      manufacturer:
        type: String
      catalog_number:
        type: String
      well_count:
        type: Int

  Plate384:
    domain: object
    implements:
      - PlateLike
      - HasWells
      - Labware
    view:
      manufacturer:
        type: String
      catalog_number:
        type: String
      well_count:
        type: Int
```

An unknown type trait name in top-level `traits`, `implements`, or `where` is a validation error. A type must not implement a trait that is not declared in the document's top-level `traits` section, except that `Numeric` is built in and cannot be implemented by user-defined types. Redeclaring `Numeric` in top-level `traits` or listing `Numeric` in a user-defined type's `implements` section is a validation error.

Common process grouping should be represented by traits, not by subtyping or parameterized Object types in v0. However, because traits are nominal membership markers only, any concrete process, field, view structure, conversion, or Object behavior must still be declared explicitly elsewhere in the document.

A generic process can accept any type that implements a declared trait:

```yaml
traits:
  PlateLike: {}

processes:
  plate_seal:
    kind: atomic
    type_params:
      P:
        domain: object
    where:
      - PlateLike<P>
    inputs:
      plate:
        type: P
        phase: data
    outputs:
      plate:
        type: P
        phase: data
```

The concrete type is preserved through the process. The constraint `PlateLike<P>` means only that the concrete type substituted for `P` implements the nominal trait `PlateLike`.

### 7.4 View metadata

`.view` denotes the contract-visible projection of a value. Physical Object identity is not expressed through `.view`; it is tracked by Object flow.

Primitive Data types have specification-defined views. For `Bool`, `Int`, `Float`, and `String`, `.view` is the scalar value itself.

For `Array<T>`, `.view` exposes the standard field:

```text
Array<T>.view.length: Int
```

`Array<T>.view.length` is the number of top-level elements in the Array value. This applies uniformly to Pure Data Arrays and Object-bearing Arrays. For example, both `Array<Float>` and `Array<Sample>` expose `.view.length`.

User-defined nominal Data and Object types define their contract-visible view schema in the document's `types` section. A view schema is a mapping from field names to field declarations. Each field declaration has a `type` and may optionally have a statically known `value`.

```yaml
types:
  Image:
    domain: data
    view:
      width:
        type: Int
      height:
        type: Int
      channels:
        type: Int

  Plate96:
    domain: object
    implements:
      - PlateLike
    view:
      manufacturer:
        type: String
      catalog_number:
        type: String
      well_count:
        type: Int
        value: 96

  Plate384:
    domain: object
    implements:
      - PlateLike
    view:
      well_count:
        type: Int
        value: 384
```

View fields are required, read-only Pure Data projections. In v0, a user-defined view field type must be a v0 primitive Pure Data type or an Array whose element type recursively satisfies the same restriction. Under the `units` feature the restriction is extended to admit a unit suffix where the base type is `Int` or `Float`, and a static view value is then checked against the base type alone (28.9).

Therefore, the following view field types are valid:

```text
Bool
Int
Float
String
Array<Bool>
Array<Int>
Array<Float>
Array<String>
Array<Array<Int>>
```

The following view field types are not valid in v0:

```text
user-defined nominal Data types
Object types
Array<user-defined nominal Data type>
Array<Object type>
```

Object-bearing view fields are validation errors. User-defined nominal Data types are not valid view field types in v0, even when they define their own view schemas. If structured contract-visible metadata is needed in v0, authors should represent it using separate primitive view fields or Arrays of primitive view fields.

If a view field declaration contains `value`, that value is a type-level static view value. It is the same for every value of that nominal type. The static value must conform to the declared view field type.

`null` is not a valid static view value in v0.

Static view values are checked against their declared types as follows:

```text
Bool:
  the YAML value must be a boolean scalar.

Int:
  the YAML value must be an integer scalar.

Float:
  the YAML value must be a finite numeric scalar.
  YAML integer, floating-point, and exponent numeric forms are accepted when
  the YAML processor represents them as numeric values.
  NaN and infinity values are not valid portable v0 static values.

String:
  the YAML value must be a string scalar.
  UTF-8 string contents are allowed.

Array<T>:
  the YAML value must be a sequence.
  Each element must recursively conform to T.
```

A static value must not rely on YAML custom tags or implementation-specific scalar types for portable v0 conformance.

The finiteness requirement above constrains the type-level static value only. It does not extend to the Float values a workflow produces at run time. A non-finite runtime view value is a data-phase concern (6.2), not a violation of this section, and v0 does not require an implementation to reject one. The asymmetry is deliberate: a static value is a constant written into the document and exchanged with every consumer of it, whereas a runtime value is computed and never appears in the IR.

If a view field declaration omits `value`, the field is an ordinary required runtime or instance-level view projection. Static and non-static view fields use the same contract reference syntax.

Examples:

```yaml
contracts:
  requires:
    - expr: "inputs.plate.view.well_count == 96"
```

When `inputs.plate` has type `Plate96`, `inputs.plate.view.well_count` is statically known to be `96` because the `Plate96` type declares that static view value.

A malformed static view value, or a static view value that does not conform to the declared field type, is a validation error.

A contract expression may reference only fields present in the resolved type's view schema, or specification-defined primitive and Array views. An unknown view field reference is a validation error. If a referenced view field has a static value, an implementation may use that value during graph-time contract checking and constant folding.

Object-bearing values may have `.view` metadata, but that metadata is associated with the Object token and is Pure Data only. `.view` must not expose Object identity or Object-bearing values. A static view value is type-level metadata and does not create, consume, duplicate, map, transform, or otherwise affect Object identity.

Static view values are graph-time type-level constants. They do not create workflow values, do not create Object identities, do not consume or map Objects, and do not affect Object tracking except by providing Pure Data metadata for validation and contract checking.

v0 view schemas do not support Optional, Result, union, nullable, dependent, computed, effectful, nested nominal record, or runtime-shape-dependent fields. Static view values are explicit constants, not computed fields. View fields are always present if the value exists.

Contract expressions may refer to Array length through the standard Array view field:

```yaml
contracts:
  requires:
    - expr: "inputs.samples.view.length > 0"
```

v0 contract expressions do not include a built-in `len` function. Array length is accessed through the standard Array view field instead.

### 7.5 When to split Object types

Use a separate Object type when at least one of the following is true:

1. The accepted process set is substantially different.
2. The Objects are not safely interchangeable as workflow state values.
3. The Objects imply different address spaces, data shapes, or operational geometry.

For example, `Plate96` and `Plate384` should be separate Object types because they have different well layouts and are not generally state-compatible.

Do not introduce `Plate<96WellPlate>` or other parameterized Object types in v0. Use nominal types plus traits.

---

## 8. Generic Constraints

Generic processes require the `generic_processes` feature (spec 4.2). A document requires the feature when any process declares a `type_params` section. An implementation that lacks `generic_processes` treats such a document as valid v0 but unsupported (spec 4.4) rather than instantiating it.

Generic type parameters are written using `type_params`. Each type parameter declaration must specify a domain:

```yaml
type_params:
  T:
    domain: data
  O:
    domain: object
```

The valid type parameter domains are:

```text
data
object
```

A type parameter is instantiated with an atomic type whose domain matches the declared type parameter domain.

A `domain: data` type parameter may be instantiated with a built-in primitive Data type or a user-defined nominal Data type.

A `domain: object` type parameter may be instantiated with a user-defined atomic Object type.

A type parameter is not instantiated with an Array type. Object-bearing collection types are expressed by writing `Array<O>`, where `O` is an Object-domain type parameter. Data collection types are expressed by writing `Array<T>`, where `T` is a Data-domain type parameter.

Type-level constraints are written using `where`. In v0, constraints are trait membership constraints. Document-defined traits are nominal membership constraints. The built-in `Numeric` trait is a closed primitive membership constraint satisfied only by `Int` and `Float`.

```yaml
traits:
  Washable: {}

processes:
  wash_any:
    kind: atomic
    type_params:
      O:
        domain: object
    where:
      - Washable<O>
    inputs:
      item:
        type: O
        phase: data
    outputs:
      item:
        type: O
        phase: data
```

A constraint `TraitName<T>` is satisfied when the concrete type substituted for `T` implements the declared trait `TraitName`.

A constraint `Numeric<T>` is satisfied only when `T` is instantiated as `Int` or `Float`. This is not a union type and does not allow a single port or value to have type `Int | Float`. It restricts generic instantiation to one concrete primitive numeric type. Because `Numeric` is satisfied only by primitive Data types, it may be applied only to a `domain: data` type parameter.

Traits do not imply operators, fields, conversions, subtyping, inheritance, Object behavior, or view structure. The built-in `Numeric` trait does not make user-defined numeric types possible in v0.

Unknown trait names, malformed constraints, constraints applied to non-type-parameter references, attempts to redeclare `Numeric`, and attempts by user-defined types to implement `Numeric` are validation errors.

v0 has no implicit conversions in workflow dataflow, process input port binding, process output port typing, or generic instantiation. Limited numeric promotion exists only inside contract expressions.

### 8.1 Generic process instantiation

v0 does not define explicit type argument syntax at node invocation sites. A generic process invocation is instantiated by inferring type arguments from the values bound to the target process input ports.

Every type parameter declared by a process must appear in at least one input port type of that process. A type parameter that appears only in output port types, only in `where`, or not at all is a validation error.

Type inference uses structural matching between target process input port types and the resolved types of the bound source values.

Two kinds of type parameter can occur in one match, and they behave differently. A type parameter declared by the **target** process is **flexible**: instantiating the invocation is what determines it. A type parameter declared by the **enclosing** process — the one whose body contains this invocation — is **rigid**: inside that body it stands for a type that is unknown but already fixed by whoever will instantiate the enclosing process, so nothing here may choose it. A concrete atomic type is a built-in primitive type or a top-level user-defined type.

```text
A built-in primitive type matches only the same built-in primitive type, which under the
  units feature means the same base type with an equal unit normal form (28.4).
A user-defined nominal type matches only the same nominal type.
Array<X> matches Array<Y> by recursively matching X and Y.
An unbound flexible parameter matches a concrete atomic type whose domain matches the declared flexible parameter domain.
An unbound flexible parameter matches a rigid parameter whose domain matches the declared flexible parameter domain.
An already-bound flexible parameter matches only what it is already bound to.
A rigid parameter matches only that same rigid parameter.
All other matches fail.
```

No subtyping, implicit conversion, trait-based widening, union matching, common-supertype inference, or Array-to-type-parameter binding is performed.

A unit-annotated numeric type is a concrete atomic type of `domain: data`. A flexible parameter of that domain may therefore be instantiated with one, and an already-bound parameter matches only the same base type and unit.

A rigid parameter does not match a concrete atomic type. Inside a generic process whose parameter is `U`, a port of type `U` may hold a value of any type the caller eventually chooses, so binding it to a port that requires one particular concrete type is not sound and is a validation error.

A literal binding source (11.1.1) determines no type argument. An integer literal conforms to a `Float` port as well as an `Int` one, so it does not say which of them the parameter is; inference reads only the resolved types of values that come from elsewhere. A type parameter whose every occurrence is bound by a literal therefore cannot be inferred.

If a type parameter cannot be inferred, or if matching constraints infer incompatible concrete types for the same type parameter, the invocation is a validation error.

Type argument inference is performed during graph validation. It does not depend on runtime values. Source value types are resolved from process signatures, node binding rules, structured node output rules, and previously validated graph structure.

After type arguments are inferred, all `where` constraints are checked during graph validation. A document-defined trait constraint `Trait<T>` is satisfied only if the inferred concrete type for `T` implements `Trait`. The built-in constraint `Numeric<T>` is satisfied only when `T` is `Int` or `Float`.

When a flexible parameter is inferred to be a **rigid** parameter, there is no concrete type to look a trait up on, so the constraint is instead discharged from what the enclosing process already promises. A constraint `Trait<T>` whose `T` was inferred to be the rigid parameter `U` is satisfied only if the enclosing process declares `Trait<U>` in its own `where`. `Numeric<T>` inferred to a rigid `U` is satisfied only if the enclosing process declares `Numeric<U>`. This is what makes a generic process usable from inside another generic process: the caller states the constraints its own parameter satisfies, and those are what the callee's constraints are checked against.

The `where` section, if present, must be a sequence of string scalar constraints. Each constraint must have the form:

```text
Constraint ::= TraitName "<" S? TypeParamName S? ">"
S          ::= one or more ASCII space or tab characters
```

Whitespace is allowed only immediately inside the angle brackets. Therefore `PlateLike<P>` and `PlateLike< P >` are valid and have the same meaning. `PlateLike <P>` is a validation error.

The trait name must be either a declared document-defined trait or the built-in trait `Numeric`. The type parameter name must name a type parameter declared by the same process. Constraints over concrete types, multiple type arguments, unknown traits, unknown type parameters, and malformed constraints are validation errors.

Generic atomic process Object tracking is validated after instantiation. The process definition is first checked for syntactic correctness of paths, roles, and declarations. After concrete types are inferred, Object-bearing slots are computed and `objects` completeness, `object_identity_map` inference, transform role typing, and Object linearity are checked on the instantiated process signature.

A type parameter has no phase. Phase is a property of process ports, not of types or type parameters.

---

## 9. Contracts

Contracts express value-level hard conditions over type-defined `.view` projections.

```yaml
contracts:
  requires:
    - expr: "inputs.x.view >= 0"
  ensures:
    - expr: "outputs.y.view >= 0"
```

### 9.1 Reference scope

Reference scope:

```text
requires: inputs.*.view
ensures:  inputs.*.view and outputs.*.view
```

`requires` expressions may reference only `inputs.*.view`.

`ensures` expressions may reference both `inputs.*.view` and `outputs.*.view`.

The structure and meaning of `.view` are defined by the referenced value's type. A contract expression may reference only fields present in the resolved type's view schema, or specification-defined primitive and Array views.

Where a port's declared type is a type parameter of the enclosing process, there is no resolved type at the process definition: the type argument is chosen independently at each instantiation (8.1), and a document may instantiate the same generic process differently at different invocations. Such a reference is therefore not resolved against any view schema when the definition is validated, and neither the presence of the referenced field nor the type of the reference is a validation error there. Everything about the expression that does not depend on the referenced type — its grammar, its reference scope, the ports it names, and the operands that are not themselves such references — is still validated at the definition.

Such a reference is resolved at each invocation that instantiates the process. The type arguments inferred there (8.1) are substituted into the target's port types, and the contract expression is checked against the resulting resolved types, exactly as it would be checked on a process whose ports were declared with those types. A view field absent from the resolved type's view schema, a field whose type the expression cannot use, and a bare `.view` on a type argument that turns out to be nominal are validation errors of that invocation. A `where` constraint does not make any of these decidable earlier: a trait is a nominal membership marker and declares no field (7.3).

An invocation resolves only the type arguments it infers itself. Where a type argument is a type parameter of the enclosing process, or where inference determined none, there is still no resolved type and the contract is not checked at that invocation.

The domain of the type parameter is known at the definition even though the type argument is not, and what it decides is decided there. A parameter declared `domain: object` is instantiated only by a nominal Object type, and a nominal type's `.view` is a view schema rather than a scalar (7.4), so a bare `.view` on such a port is a validation error at the definition. A parameter declared `domain: data` may be instantiated by a primitive type, whose `.view` is the scalar itself, so a bare `.view` on that port is not.

### 9.2 Contract expression language

Contracts use a small, side-effect-free expression language. In v0 portable YAML, every contract `expr` value must be a YAML string scalar containing a v0 contract expression.

A contract expression may contain only:

```text
Boolean literals: true, false
integer literals
floating-point literals
double-quoted string literals
contract view references
the operators: and, or, not, ==, !=, <, <=, >, >=, +, -, *, /
parentheses
ASCII whitespace
```

Boolean literals are lowercase `true` and `false`.

Integer literals use decimal notation:

```text
IntLiteral ::= 0 | [1-9][0-9]*
```

Floating-point literals use decimal notation with digits on both sides of the decimal point, optionally followed by an exponent:

```text
FloatLiteral ::= [0-9]+ "." [0-9]+ Exponent?
Exponent     ::= ("e" | "E") ("+" | "-")? [0-9]+
```

Negative numbers are parsed as unary `-` applied to a positive numeric literal. Therefore `-1.0e+023` is parsed as unary `-` applied to the floating-point literal `1.0e+023`.

Leading-zero integer literals, floating-point literals without digits on both sides of the decimal point, `NaN`, and infinity literals are not valid v0 contract literals. For example, `1e3`, `1.`, `.5`, `NaN`, and `Infinity` are invalid, while `1.0e3`, `1.0e+23`, and `1.0E-23` are valid floating-point literals.

String literals use double quotes and JSON-style escapes. UTF-8 text is allowed in string literal contents.

Contract references must explicitly include `.view`.

In `requires`, references may use:

```text
inputs.<port>.view
inputs.<port>.view.<field>
```

In `ensures`, references may use:

```text
inputs.<port>.view
inputs.<port>.view.<field>
outputs.<port>.view
outputs.<port>.view.<field>
```

v0 contract expressions do not support direct port references, omitted `.view`, function calls, method calls, indexing, slicing, quantifiers, assignment, mutation, I/O, host-language expressions, implicit conversions, or Object creation/consumption. Array length is accessed through `Array<T>.view.length`.

Operator precedence, from highest to lowest, is:

```text
parentheses
unary not and unary -
* and /
+ and -
==, !=, <, <=, >, >=
and
or
```

Arithmetic and Boolean binary operators are left-associative. Unary operators are right-associative. Comparison operators are non-associative, so expressions such as `1 < x < 10` are validation errors. Write such expressions explicitly with Boolean operators, such as `1 < x and x < 10`.

v0 performs no implicit type conversions in workflow dataflow, process input port binding, process output port typing, or generic instantiation. Contract expressions have a limited numeric promotion rule for primitive numeric operands only. This rule is local to contract expressions and does not imply implicit conversion anywhere else in v0.

The following operator category checks are defined by v0:

| Operator | Allowed operands | Result type | Notes |
|---|---|---|---|
| `and`, `or` | `Bool`, `Bool` | `Bool` | Boolean operands only. |
| `not` | `Bool` | `Bool` | Boolean operand only. |
| `==`, `!=` | Same primitive type, or any numeric pair | `Bool` | Primitive types are `Bool`, `Int`, `Float`, and `String`; numeric pairs are any combination of `Int` and `Float`. |
| `<`, `<=`, `>`, `>=` | Any numeric pair | `Bool` | String ordering is not supported in v0. |
| binary `+`, `-`, `*` | Any numeric pair | `Int` if both operands are `Int`; otherwise `Float` | Numeric pairs are any combination of `Int` and `Float`. |
| binary `/` | Any numeric pair | `Float` | `Int / Int` produces `Float` in contract expressions. |
| unary `-` | `Int` or `Float` | Same type as operand | |

When the `units` feature is in effect, operands may carry unit annotations. The unit rules are given in 28.11 and are applied independently of the base-type rules in this table. They do not change which operand pairs this table allows, and they do not change any result base type.

String values support only `==` and `!=`. String ordering and string concatenation are not supported in v0 contract expressions. Mixed numeric operations and comparisons are allowed only by the local numeric promotion rule above.

A contract expression must type-check to `Bool`.

All subexpressions are parsed and type-checked. Validation must not depend on Boolean short-circuiting to ignore malformed, invalid, or statically erroneous subexpressions.

A statically determinable basic evaluation error, such as division by zero in a constant subexpression, is a validation error. Runtime contract evaluation errors that depend on runtime view values are runtime verification concerns unless they can be determined statically.

If a contract expression can be fully evaluated from static view values and literals at graph validation time, and it evaluates to `false`, the document is invalid. If the expression depends on runtime or instance-level view values, false evaluation is a runtime contract violation rather than an IR validation error.

Numeric precision, overflow, division behavior beyond statically determinable errors, and Boolean short-circuit behavior during runtime evaluation are implementation concerns unless otherwise specified by a future feature.

### 9.3 Validation and runtime verification

A malformed contract expression, an invalid reference, an unknown view field, or a statically known basic type error is a validation error.

If a contract expression evaluates to false during execution, the invocation violates the contract. Runtime contract violations and failures to obtain required view projections are runtime verification concerns, not IR validation errors unless they can be determined statically.

Contracts are hard conditions for implementations that support them. A contract violation is not a validation error in the IR; it is a runtime or verification concern.

---

## 10. Processes

Every executable unit is a `process`.

```yaml
processes:
  some_process:
    kind: atomic
    inputs: {}
    outputs: {}
```

The input and output endpoints of a process are collectively called **ports**.
Process input ports are declared under `inputs`, and process output ports are declared under `outputs`.
The YAML keys are `inputs` and `outputs`; `ports` is the generic term used when rules apply to both input and output endpoints.

Process kinds:

```text
atomic
composite
```

### 10.1 Atomic processes

Atomic processes declare their Object behavior using `objects`.

### 10.2 Composite processes

Composite processes define a body graph and returns.

```yaml
processes:
  main:
    kind: composite
    inputs: {}
    outputs: {}
    body:
      nodes:
        - id: step
          process: some_process
      returns:
        result:
          from: step.result
```

Composite processes do not have an `objects` section in v0. Their Object behavior is derived from the body graph and `returns`.

The process dependency graph must be acyclic. Recursive composite dependencies are not allowed in v0.

Within a composite body, the node dependency graph must likewise be acyclic. It has an edge from one node to another where a `from` in a binding or control section (21.0) of the second names an output of the first. This is a separate requirement: the process dependency graph concerns which process invokes which, and the node dependency graph concerns the order of nodes within one body. A cycle in either is a validation error.

### 10.3 Entry process

`entry` names the entry process. If `entry` is omitted and a process named `main` exists, `main` may be used as the entry. If `entry` is omitted and no process named `main` exists, the document has no entry process and this is a validation error.

---

## 11. Node Invocation Bindings

Node bindings connect values to process input ports. Node-side inputs are classified by binding section, and the valid binding sections depend on the node kind.

```text
state: Object-bearing linear input for ordinary non-structured node invocations.
       `state` is not used for loop-carried structured values in v0.

carry: Loop-carried value for `fold` and `do_while`.
       A carry value may be Pure Data or Object-bearing.
       For Object-bearing carry, linearity and Object tracking rules apply.
       For Pure Data carry, ordinary Data rules apply, but the value is threaded
       through structured control by same-name, same-type, same-phase outputs.

args: Branch arm arguments.
       These values are made available only to the selected branch arm at runtime.
       An Object-bearing branch argument is not duplicated across arms.
       Branch arguments may be Pure Data or Object-bearing.

each:  Array input.
       Traversed element-wise by structured nodes such as `map` and `fold`.

bind:  Pure Data input.
       Ordinary data or literal; unrestricted and not threaded.
```

Ordinary non-structured node invocations use `state` to bind Object-bearing linear input ports and `bind` to bind Pure Data input ports.

Example ordinary node invocation:

```yaml
- id: move_once
  process: robot_move_to
  state:
    robot:
      from: inputs.robot
  bind:
    pose:
      from: inputs.pose
```

`bind` is Pure Data only, and `state` is Object-bearing only. An Object-bearing value must not be passed through `bind`, and a Pure Data input port must not be bound under `state`, whether by a reference or by a literal. Both are validation errors.

Which of the two sections a port takes is decided by the port's declared type: a port whose type has an Object slot (5.2) is bound under `state`, any other port under `bind`. Where the port's type is a type parameter, the parameter's declared domain (8) decides, so the section is known in a generic body before any instantiation.

The rule holds in both directions because the section is the one place a node says what a binding does. A value bound under `state` is used linearly: it is consumed by this invocation and nothing else refers to it. A value bound under `bind` is read, and the same value may be read by any number of invocations. A Pure Data value written under `state` would claim a linearity it does not have, and a reader that takes the section at its word would treat information as material. Nothing is lost by the rule, since `bind` accepts a reference or a literal of any phase.

The binding entries of a node and the input ports of the process it invokes are in one-to-one correspondence. Every input port of the target process must be bound exactly once, across all binding sections valid for the node's kind (21.0), and every binding entry must name an input port of the target process. Both directions are validation errors when they fail: an unbound input port, an input port bound in two sections or twice in one section, and a binding entry that names no input port of the target.

This applies to every node kind. v0 defines no default value for an input port, so no port is exempt from being bound, and it holds for Pure Data ports as well as Object-bearing ones: 12.1 gives a Pure Data input port an indegree of exactly one, not of at least zero.

For `branch`, the target is each arm, and `args` is the only binding section (21.0). The correspondence therefore holds against each arm separately: an input port of either arm that no argument names, and an argument that names no input port of one of the arms, are both validation errors. Together with the same-type requirement of section 20, this makes the two arms agree on their input ports through `args`.

Literal values are written under `value`:

```yaml
bind:
  intensity:
    value: 3
```

A `value` is a Pure Data literal. What may be written is a v0 primitive type -- `Bool`, `Int`, `Float`, or `String` -- or a Pure Data `Array`, written as a YAML sequence whose elements satisfy the same restriction recursively.

```yaml
bind:
  intensity:
    value: 3
  labels:
    value: ["a", "b", "c"]
  empty:
    value: []
```

Conformance of a literal to its declared type follows the same rule as a static view value (7.4). `null`, YAML custom tags, `NaN`, and infinities are not portable v0 values. A value of an Object-bearing type cannot be written as a literal: v0 has no literal notation for an Object, and a literal introduces no Object identity (13).

An empty `Array` literal is how an Object-bearing empty collection is obtained. An `each` source of length zero makes a `map` perform zero invocations and collect empty outputs (17), so a `map` over an empty literal produces an empty Array of the target's output type, Object-bearing or not.

```yaml
- id: none
  kind: map
  process: int_to_cup
  each:
    n:
      value: []
```

A Pure Data carry binding may take its initial value from a literal in the same way. An Object-bearing carry binding must use `from`, since no literal can denote an Object.

```yaml
carry:
  reached:
    value: false
  score:
    from: inputs.initial_score
```

In `fold` and `do_while`, `carry` means a loop-carried value. The output of one iteration becomes the carry input of the next iteration. `carry` does not imply physical Object identity preservation; physical identity behavior is determined independently by Object tracking declarations.

In `branch`, `args` are supplied only to the selected arm. They are not fan-out, even when Object-bearing. Branch output ports are selected-arm output ports exposed as common outputs.

### 11.1 Binding type compatibility

Every binding connects a value to a port. The resolved type of the bound value must **match** the resolved type of the port it is bound to. A binding whose value type does not match its port type is a validation error.

Matching is the structural matching relation defined for generic instantiation (8.1). When neither type contains a type parameter, that relation reduces to identity of type expressions: `Cup` matches only `Cup`, `Array<Cup>` matches only `Array<Cup>`, and `Array<Cup>` does not match `Cup`. v0 performs no subtyping, widening, or implicit conversion at a binding, exactly as it performs none in generic instantiation. Under the `units` feature, identity of type expressions is identity of base type and unit normal form (28.4), so `Float[mg/mL]` and `Float[mg*mL^-1]` are the same type.

What is compared depends on the binding section:

```text
state    the bound value               against the target process input port
bind     the bound value               against the target process input port
carry    the bound value               against the target process input port
each     the bound value's element type against the target process input port
args     the bound value               against the same-named input port of each arm (20)
returns  the bound value               against the composite output port it returns
```

An `each` source must be an Array. Its element type is what the target process input port is matched against, because the source is traversed element-wise (11, 17, 18): a `map` or `fold` over `each: {label: {from: inputs.labels}}` binds one element of `labels` to the port `label` per invocation. A source that is not an Array cannot be traversed and is a validation error.

The requirement that a branch argument correspond to a same-name, same-type, same-phase input port of each arm is stated in 20; the requirement that a `fold` or `do_while` carry output be same-name, same-type, same-phase is stated in 16. This section adds the complementary rule for the value bound *into* those sections.

`branch.condition.from` must resolve to a `Bool` Pure Data value (20).

When the bound source is a structured node output, its resolved type is the type that structured node exposes for it (21).

#### 11.1.1 Literal values

A binding source entry that carries `value` rather than `from` (2.6.6) supplies a literal. The literal must conform to the port's declared type, checked exactly as a static view value is checked against its declared view field type (7.4):

```text
Bool     the YAML value must be a boolean scalar.
Int      the YAML value must be an integer scalar.
Float    the YAML value must be a finite numeric scalar; a YAML integer is accepted.
String   the YAML value must be a string scalar.
Array<T> the YAML value must be a sequence whose every element conforms to T.
```

The acceptance of an integer literal for a `Float` port is the same latitude 7.4 gives a static `Float` view value, and for the same reason: YAML integer, floating-point, and exponent forms all denote a number, and a document should not have to write `3.0` where `3` is meant. It is not a subtyping rule and does not extend to `from` bindings, where an `Int`-typed value does not match a `Float` port.

A literal is interpreted according to the declared type of its target, including its unit annotation when the `units` feature is in effect (28.8). The conformance check above is performed against the base type, so an integer literal fills a `Float[s]` port exactly as it fills a `Float` port.

A literal is a `graph` phase value. `graph` is the least phase (6), so a literal satisfies the phase requirement of every Pure Data position it may be written in, and no phase condition is ever stated for a literal.

A literal is Pure Data. A literal bound to an Object-bearing port is a validation error, since a literal cannot introduce an Object identity (13).

### 11.2 Constant slots

A **constant slot** is a document position that expects a Pure Data value fixed no later than a declared phase. A constant slot declares two things: a **slot type**, which is a Pure Data v0 type expression, and a **phase upper bound**, which is `graph` or `run`.

A constant slot is filled by a source entry (2.6.6): exactly one of `from` or `value`.

```yaml
max_iterations:
  value: 100
```

```yaml
max_iterations:
  from: inputs.attempt_budget
```

A constant slot is treated as an input port whose declared type is the slot type and whose declared phase is the phase upper bound. The rules that govern a binding to an input port therefore govern a constant slot without addition:

```text
phase   the phase of the value must be no later than the upper bound (6, rule 16a).
        A value of a later phase is a phase flow validation error.
type    the resolved type of the value must match the slot type (11.1).
literal a literal must conform to the slot type as 11.1.1 requires, and is a
        graph phase value, so it satisfies every upper bound.
```

A constant slot is Pure Data. An Object-bearing value must not fill a constant slot, whether by `value` -- no literal denotes an Object (13) -- or by `from`.

A `from` in a constant slot is a body dataflow reference (2.6.1) and is resolved in the scope of the composite body that contains the slot. Like any other body dataflow reference, it contributes an edge to that body's node dependency graph, which must remain acyclic (10.2, rule 21b).

Where a slot's value is supplied by `value`, any further condition on the slot is decided at `graph` phase. Where it is supplied by `from`, it is decided at the earliest phase at which the value is determined, as a validation error or a preflight error (6.2).

v0 defines one constant slot: `do_while.max_iterations` (19), of slot type `Int` with upper bound `run`.

---

## 12. Port Degree and Linearity Rules

### 12.1 Data ports

Pure Data input port:

```text
indegree = 1
```

Pure Data output port:

```text
outdegree >= 0
fan-out allowed
unused output allowed
```

### 12.2 Object-bearing ports

Object-bearing input port:

```text
indegree = 1
```

Object-bearing output port:

```text
outdegree = 1
fan-out forbidden
implicit discard forbidden
```

An Object-bearing output port must be connected either to:

1. A downstream node input in the same body, or
2. `body.returns` as a composite output, or
3. A structured node output explicitly exposed by that structured node.

If an Object-bearing output port is not connected, it is a validation error.

Read from the other end, within a composite body every Object-bearing value is referred to exactly once. The values a body has are the composite's own input ports, the output ports of each ordinary node, and the outputs a structured node exposes (21.0). A reference is a `from` in any binding section of any node, or a `from` in a `body.returns` entry.

```text
referred to twice or more   duplication
referred to zero times      an Object with no fate
```

Both are validation errors. 13 forbids them already, as properties an incomplete Object skeleton has; this states the degree rule they follow from, so that the rule can be applied to a value directly. The composite's own input ports need it most: 12.2 above governs an *output* port, and a composite input port read as a source of the body is not one.

### 12.3 Composite returns

`body.returns` is a connection from an internal graph output port to the composite boundary.

```yaml
body:
  nodes:
    - id: step
      process: cup_wash
      state:
        cup:
          from: inputs.cup
  returns:
    cup:
      from: step.cup
```

Here `step.cup` is connected to the composite output port `cup`; it is not unused.

The entries of `body.returns` and the output ports of the composite are in one-to-one correspondence. Every output port must have exactly one `returns` entry, and every `returns` entry must name an output port of the composite. Both directions are validation errors when they fail: an output port with no entry, and an entry that names no output port.

This holds for Pure Data output ports as well as Object-bearing ones, and for the entry process as for any other composite. A declared output is a value the composite says it gives; one with no entry gives nothing, so a node that binds it reads no value and a run of the entry process has no final value to return for it. For an Object-bearing output the requirement also follows from Object tracking completeness (13), which gives every output Object slot a provenance; for a Pure Data output this is the rule that says so. An entry that names no output port is a value returned nowhere, the counterpart of a binding entry that names no input port (11).

Because every composite returns every output it declares, a reference `<node>.<output>` to a node that invokes a composite names a value whenever `<output>` is an output port of that composite (2.6.8).

### 12.4 Object skeleton

The **Object skeleton** of a process is the description of where that process moves Object identity to, and nothing else. Types, Pure Data, view metadata, execution content, and time are not part of it.

#### 12.4.1 Components of a skeleton

For a process `p`, after generic instantiation:

```text
In(p)  = the union of the object_slots of p's input ports
Out(p) = the union of the object_slots of p's output ports
```

The skeleton `S(p)` is a triple:

```text
phi : a partial injection from In(p) to Out(p)   the correspondence
C   : the slots of In(p) outside the domain of phi   consumed
N   : the slots of Out(p) outside the image of phi   created
      each slot in N carries a creation-point identifier (12.4.3)
```

That `phi` is injective is what says no Object is duplicated. That it is partial is what allows one to be consumed.

For a scalar Object-bearing port the port is one slot. For a collection port the slots are a family indexed at run time, and v0 relates two such families by a correspondence kind (12.4.2) rather than by position. A skeleton therefore never enumerates the slots of a collection and never names an index.

#### 12.4.2 Correspondence between collection slots

A **correspondence kind** classifies how two slot families correspond. v0 defines two:

```text
identity          nesting structure, order, and position all agree
order_preserving  traversal order is preserved; nesting structure may differ
```

`identity` is a special case of `order_preserving`, but the two are distinct labels for the purpose of skeleton equality (12.4.3).

A later revision may add kinds. Because a kind is a label drawn from a finite set, adding one changes neither the shape of the normal form nor the equality procedure; it adds a label the first part of the normal form may carry.

#### 12.4.3 Normal form and equality

The **normal form** of a skeleton is:

```text
Part 1  the input slot names in dictionary order, each carrying a row value:
        (output slot name, correspondence kind), or `consumed`

Part 2  the created output slot names in dictionary order, each carrying its
        creation-point identifier
```

Two skeletons are **equal** when their normal forms agree. The normal form is determined by the skeleton, so equality is a comparison of two pieces of finite data and needs no reasoning about the structure of the correspondence.

A **creation-point identifier** names the place at which an Object identity first becomes observable from the rest of the graph. In v0 that place is a node in a body.

```text
non-structured node       that node
map / fold / do_while     that node
branch                    that node, not the arm
inside a composite        the interior node's identifier, carried outward by returns
```

The creation point is a node rather than a process definition because the same process invoked from two nodes creates two different physical Objects.

```yaml
nodes:
  - id: a
    process: cup_create     # a.cup
  - id: b
    process: cup_create     # b.cup, a different cup
```

Where a `branch` creates an Object in both arms, the creation point is the `branch` node in both cases. Whichever arm runs, the new Objects appear at that node's output, and where their identity came from does not depend on the arm.

A skeleton computed for a process *definition* has undetermined creation points, since the node that will invoke it is not known there. Equality is decided between skeletons placed at nodes.

#### 12.4.4 Skeleton of an atomic process

An atomic process's skeleton is read directly from its `objects` section:

```text
objects.map        adds the named pair to phi, with correspondence kind identity
objects.transform  adds the correspondence the kind defines (14.4) to phi,
                   with correspondence kind order_preserving
objects.consume    adds the named input slot to C
objects.create     adds the named output slot to N, its creation point being the
                   node that invokes the process
```

A process that omits `objects` entirely and declares `object_identity_map` has the skeleton that 15 infers.

`objects.map` and `objects.transform` both add to `phi` and differ only in the correspondence kind. They are separate sections because a correspondence between whole ports is most readably written as a pair of names, and one that changes nesting is most readably written as a kind; the distinction is one of notation, not of structure.

#### 12.4.5 Composition of skeletons

A composite's skeleton is composed along its body. Every Object-bearing value in a body is referred to exactly once (12.2), so each Object slot has one path through the body and the composition is determined.

**Non-structured node.** The target process's skeleton, placed at this node.

**`map` node.** The target's skeleton lifted element-wise: the input side is the slot family of the `each` source, the output side the slot family of the collected output. The lift does not change the correspondence kind, since it lifts the relation between elements to a relation between the collections holding them. A slot the target creates is created at the `map` node.

**`fold` and `do_while` carry slot.** The target's skeleton restricted to the carried port. By 16 the restriction is one of two things, and the node's skeleton follows:

```text
a correspondence   the node's carry input slot corresponds to its carry output
                   slot, with the correspondence kind the target gives it

consume + create   the node consumes its carry input slot and creates its carry
                   output slot, the creation point being the node
```

The Objects created by intermediate iterations are consumed within the same node, so the accounting closes there whatever the iteration count is. This is why a skeleton is determined for `fold` and `do_while` although their iteration count is not known until run time.

**`fold` collect output.** The target's output slot lifted to the slot family of the collected output, as for `map`.

**`branch` node.** Both arms' skeletons must be equal, and the node's skeleton is that common skeleton (20.2).

**Composite process.** Composed along the body's bindings and `returns`, starting from the composite's own input port slots and ending at its output port slots. An input port slot that appears in no node binding and in no `returns` entry is in neither the domain of `phi` nor `C`, and an output port slot with no `returns` entry is in neither the image of `phi` nor `N`; 12.4.6 fails in both cases. A composite has no `objects` section to declare a consumption or a creation with (10.2), so its body must account for every boundary slot.

#### 12.4.6 Completeness of a skeleton

A skeleton is **complete** when:

```text
every slot of In(p)  is in exactly one of dom(phi) and C
every slot of Out(p) is in exactly one of im(phi) and N
```

#### 12.4.7 Uniqueness of a skeleton

Every process and every node must have exactly one skeleton.

Where a construct has several descriptions of which one runs, the skeletons of those descriptions must be equal in the sense of 12.4.3. If they are not, the skeleton is not determined, and the origin of the identity of an output Object slot depends on a choice made at run time. Every place downstream that refers to that Object -- a binding on a later node or a `body.returns` entry -- then has no determined answer.

In v0, `branch` is the only construct with several descriptions. Every other has one, so its skeleton is determined by construction. `fold` and `do_while` decide their iteration count at run time, but by 12.4.5 their skeleton does not depend on it.

This requirement is stronger than the principle of 1.1. The number of Objects consumed and created may agree between two descriptions and their skeletons still differ, if what an output Object's identity comes from is not the same.

#### 12.4.8 Computational cost

Computing a skeleton takes one traversal of the body graph. A correspondence kind is drawn from a finite set, and equality compares slot names, kinds, and creation-point identifiers. Computing and comparing skeletons is therefore linear in the size of the document.

Nothing in it requires index arithmetic, polynomial normalization, or constraint solving. An implementation that finds it needs any of those has diverged from this section.

---

## 13. Object Tracking Completeness

Object tracking completeness is the central well-formedness property for Object behavior.

```text
Object tracking completeness:
  A process's Object skeleton (12.4) must be complete (12.4.6).
```

Equivalently: every Object-bearing input port slot and output port slot of a process has exactly one explanation, in terms of `map`, `consume`, `create`, `transform`, body graph flow, or `returns`.

It forbids:

```text
implicit Object creation
implicit Object disappearance
implicit Object duplication
implicit Object discard
unknown Object output provenance
unknown Object input fate
```

All atomic and composite processes must satisfy Object tracking completeness.

The skeleton is read from the `objects` section for an atomic process, or from the `object_identity_map` inference rule (15) where `objects` is omitted, and composed along the body graph and `returns` for a composite (12.4.4, 12.4.5).

Object tracking completeness describes successful invocation behavior. Runtime failures and exceptions are out of scope for v0.

### 13.1 Object slot fate and provenance

For every atomic process after generic instantiation:

1. Compute `object_slots` for every input port and output port.
2. Every input port Object slot must have exactly one declared fate.
3. Every output port Object slot must have exactly one declared provenance.
4. No Object slot may be duplicated across outputs.
5. No Object slot may disappear unless explicitly consumed.
6. No Object slot may be implicitly created.

Valid input fates include:

```text
map source
transform input
consume
```

Valid output provenances include:

```text
map target
transform output
create
```

No input port Object slot may have multiple incompatible fates, and no output port Object slot may have multiple incompatible provenances.

Examples of invalid Object behavior:

```yaml
objects:
  map:
    outputs.sample: inputs.sample
  consume:
    - inputs.sample
```

This is invalid because `inputs.sample` has two incompatible fates: mapped and consumed.

### 13.2 Object replacement

A process may consume an Object-bearing input and create a same-name, same-type Object-bearing output.

```yaml
objects:
  consume:
    - inputs.sample
  create:
    - outputs.sample
```

This is Object replacement. The created Object is a new physical Object identity. v0 does not treat the created Object as a continuation of the consumed Object for identity or metadata purposes.

Object replacement by `consume + create` does not imply `.view` metadata inheritance. The created output Object has its own Object identity and its own type-defined view metadata. Any relationship between the consumed input Object's view metadata and the created output Object's view metadata must be defined by the process's semantics or expressed by contracts.

Structured carry compatibility may allow such replacement when the output has the required same name, type, and phase.

---

## 14. Atomic `objects` Section

The `objects` section is allowed only on atomic processes.

All paths in `objects` use explicit namespaces in v0:

```text
inputs.<path>
outputs.<path>
```

Short forms such as `cup: cup` are not canonical v0 syntax.

Every path in `objects` must name an Object-bearing port. A path naming a Pure Data port is a validation error, in `map`, `consume`, `create`, and `transform` alike.

All four declarations are defined over Object slots: `map` relates the slots of two ports, `consume` ends those of an input port, `create` introduces one at every slot of an output port, and `transform` fixes a correspondence between the slots on each side. A Pure Data port has no slots -- `object_slots` is empty for it (5.2) -- so such an entry declares nothing, and the completeness check of 13.1, which quantifies over Object slots, never reaches it. It can only be a mistake about which port was meant, and is reported rather than passed over in silence.

For a generic process the condition is decided without instantiation, unlike the slot accounting of 13.1: a type parameter declares its domain and is instantiated only with a type of that domain (8), so a port whose type names one is already known to be Object-bearing or not.

### 14.1 `map`

`map` preserves physical Object identity.

```yaml
objects:
  map:
    outputs.cup: inputs.cup
```

Meaning:

```text
outputs.cup is the same physical Object identity as inputs.cup.
```

`map` may be cross-wired. For example, physical switching can be expressed as:

```yaml
objects:
  map:
    outputs.a: inputs.b
    outputs.b: inputs.a
```

For atomic processes, `objects.map` is a declarative Object behavior claim made by the process definition.

The resolved types of the target path and the source path of each `objects.map` entry must match. A mapping between ports of different types is a validation error. Matching is the structural relation of 11.1, which reduces to identity of type expressions where no type parameter is involved.

```yaml
# valid
objects:
  map:
    outputs.cup: inputs.cup          # Cup and Cup
    outputs.cups: inputs.cups        # Array<Cup> and Array<Cup>
    outputs.a: inputs.b              # cross-wired, and both are Cup

# validation error
objects:
  map:
    outputs.plate: inputs.cup        # Plate96 and Cup
    outputs.cup: inputs.cups         # Cup and Array<Cup>
```

The requirement is what gives the container-structure claim below its meaning. `object_slots` has the same structure on both sides only where the types match, so a claim to preserve length, nesting, and order says nothing at all between ports whose slot structures differ. Every comparable rule already carries such a condition -- a transform's role typing (14.4.1), the `object_identity_map` marker (15), carry compatibility (16), a branch's common outputs (20), and binding type compatibility (11.1) -- and `objects.map` was the one place without one.

For Object-bearing container values, a `map` from `inputs.xs` to `outputs.ys` is interpreted recursively over Object slots. The IR processor treats the source and target value structures as corresponding identity-preservingly. For `Array<T>`, this declared behavior preserves top-level length, nesting structure, element order, and contained Object identities at corresponding slots. Processes that declare Object-bearing container structure changes, such as changing order, length, grouping, or nesting, must use an explicit `objects.transform` rather than `map`.

This declaration is trusted by the IR processor. v0 validation checks that the declaration is well-formed and complete, but it does not prove that an implementation of the atomic process actually preserves the runtime container structure. If an implementation changes order, drops elements, duplicates elements, or otherwise violates the declared mapping, that is an implementation correctness error or runtime verification concern, not an IR validation error unless it can be determined statically.

### 14.2 `consume`

`consume` terminates an input Object identity in the current workflow.

```yaml
objects:
  consume:
    - inputs.sample
```

### 14.3 `create`

`create` introduces a new Object identity at every Object slot of the named output port.

```yaml
objects:
  create:
    - outputs.sample
```

For a scalar Object port that is one identity. For a collection port it is the whole slot family, and this declaration does not fix its size: a skeleton names the port and never an index (12.4.1). The condition under which that number is bounded before the run is 1.1.

### 14.4 `transform`

`transform` describes standard structural transformations of Object slots for Arrays.

The canonical syntax is a list of transform entries:

```yaml
objects:
  transform:
    - kind: <transform-kind>
      inputs:
        <role>: <inputs.* path>
      outputs:
        <role>: <outputs.* path>
```

Valid transform kinds in v0 are:

```text
array_flatten
array_unflatten
```

All transform paths use explicit namespaces such as `inputs.*` and `outputs.*`.

Transforms preserve physical Object identities while changing container structure. A transform must account for all Object slots in its input and output paths exactly once. No Object slot may be duplicated, lost, implicitly created, or implicitly discarded by a transform.

#### 14.4.0 Acceptance criteria for set transforms

**Non-normative.** This section constrains future revisions of this specification rather than documents. It states what a proposed transform kind is judged against; nothing in it is checked by an implementation.

The set transforms v0 admits are **regroupings** only. **Reindexing** is not admitted.

A new transform kind is considered only where it meets all three of the following criteria.

**Criterion 1: value independence.** The correspondence of indices must follow from the shape of the collections alone -- their lengths and their nesting structure. It must not depend on element values, on the result of a comparison, or on the evaluation of a predicate.

**Criterion 2: detectability of a mis-connection.** Where the output of the transform is passed to the `each` of one `map` or `fold` alongside a different collection that the input corresponds to, the mistake must be detectable by the `zip: equal` length check. The transform must therefore change the collection length, or must carry another means of detection where it does not.

**Criterion 3: no privileged position.** The operation must not single out a particular index position, such as the first or the k-th. Order exists for the correspondence between collections and for traversal order, not for the selection of an element.

What each criterion rejects: a value-dependent rearrangement such as sorting or partitioning by a predicate (criterion 1); reversal of order (criterion 2); separating the first element, or splitting at a given position (criterion 3).

#### 14.4.1 Transform validation

A transform entry is validated from its declared `kind`, its role names, and the resolved types of its input and output paths.

Each transform kind defines an exact set of required input and output roles. Missing roles, extra roles, invalid namespaces, paths that do not exist, or role type mismatches are validation errors.

For v0 transform kinds, role typing is:

```text
array_flatten:
  inputs.xss:  Array<Array<T>>
  outputs.xs:  Array<T>

array_unflatten:
  inputs.xs:   Array<T>
  outputs.xss: Array<Array<T>>
```

In each case, the same `T` must be used consistently within that transform entry.

Each transform kind contributes to the process's Object skeleton a correspondence of kind `order_preserving` (12.4.2). It fixes the correspondence between its input Object slots and its output Object slots as an **order-preserving total bijection**: the Object slots contained in the input, listed in traversal order, and those contained in the output, listed in traversal order, correspond one to one in that order.

Traversal order is from first to last within a collection, and outer-first dictionary order through nested collections.

Defining the correspondence this way allows validation to check that each input Object slot is accounted for exactly once and that each output Object slot has exactly one provenance, even when Array lengths are not statically known. A transform does not prescribe which position of the output a given input slot arrives at. Since v0 does not address collection elements through an Object path (2.6.2), that information is not needed.

Multiple transform entries may appear in one `objects.transform` list, but each entry declares a direct Object slot relation from `inputs.*` paths to `outputs.*` paths for the atomic process. v0 does not define transform chaining, intermediate transform values, or references from one transform entry to another.

An Object slot must not be accounted for by more than one Object behavior declaration. If a slot appears in `map`, `consume`, `create`, or `transform` in a way that gives it multiple fates or multiple provenances, it is a validation error.

#### 14.4.2 `array_flatten`

```text
Array<Array<T>> -> Array<T>
```

Canonical syntax:

```yaml
objects:
  transform:
    - kind: array_flatten
      inputs:
        xss: inputs.xss
      outputs:
        xs: outputs.xs
```

Role typing:

```text
inputs.xss:  Array<Array<T>>
outputs.xs:  Array<T>
```

The correspondence is an order-preserving total bijection. The inner collections of the input need not be of equal length. This kind has no precondition.

A process that requires the inner collections to be of equal length states that condition in `contracts`:

```yaml
flatten_samples:
  kind: atomic
  inputs:
    xss:
      type: Array<Array<Sample>>
      phase: data
    inner_len:
      type: Int
      phase: run
  outputs:
    xs:
      type: Array<Sample>
      phase: data
  objects:
    transform:
      - kind: array_flatten
        inputs:
          xss: inputs.xss
        outputs:
          xs: outputs.xs
  contracts:
    ensures:
      - expr: "inputs.xss.view.length * inputs.inner_len.view == outputs.xs.view.length"
```

#### 14.4.3 `array_unflatten`

```text
Array<T> -> Array<Array<T>>
```

Canonical syntax:

```yaml
objects:
  transform:
    - kind: array_unflatten
      inputs:
        xs: inputs.xs
      outputs:
        xss: outputs.xss
```

Role typing:

```text
inputs.xs:   Array<T>
outputs.xss: Array<Array<T>>
```

The correspondence is an order-preserving total bijection. This kind has no precondition.

How the input is divided is not prescribed by this kind. Where the number of groups or the size of each group must be specified by the process's user, it is supplied through an ordinary Pure Data input port and the required relation is stated in `contracts`. It does not appear in `objects.transform`.

```yaml
regroup_samples:
  kind: atomic
  inputs:
    xs:
      type: Array<Sample>
      phase: data
    inner_len:
      type: Int
      phase: run
  outputs:
    xss:
      type: Array<Array<Sample>>
      phase: data
  objects:
    transform:
      - kind: array_unflatten
        inputs:
          xs: inputs.xs
        outputs:
          xss: outputs.xss
  contracts:
    ensures:
      - expr: "outputs.xss.view.length * inputs.inner_len.view == inputs.xs.view.length"
```

That the division is the same for the same input across invocations is a matter of the process's semantics, guaranteed by the implementation. The IR does not verify it. This is the principle of 14.1, that a transform declaration is trusted by the IR processor, applied here.

**Non-normative.**

Where the input of `array_unflatten` contains no Object slots, the outer collection of the output being empty is the natural reading, and the same holds for `array_flatten` where the input contains none. In both cases the slot count is 0 and Object tracking is unaffected, so this specification does not prescribe the behavior.

The output collection of `array_flatten`, and that of `array_unflatten`, has a different index system from the input collection. Passing the output alongside the input to the `each` of one `map` or `fold`, as collections that correspond, is therefore not an intended use. v0 does not check for it, but since the length changes, the `zip: equal` check detects it in most cases.

---

## 15. Object Identity Map

`object_identity_map` is a process-level behavior marker and inference permission. It is not a type trait, and it is not the general condition for structured control.

```text
object_identity_map:
  A same-name, same-type, same value-structure, physical identity-preserving
  mapping from Object-bearing inputs to Object-bearing outputs.
```

For an Object-bearing input and output with the same name and the same type, `object_identity_map` means that the output is the same logical value structure as the input and that all contained Object identities are preserved at corresponding Object slots.

For Object-bearing Arrays, this implies that Array length, nesting structure, element order, and contained Object identities are preserved. A process that changes Array length, order, grouping, or nesting does not have this marker.

Implementations may approximate this check by verifying that corresponding `object_slots` are mapped identity-preservingly and that no Object-bearing structural transform is declared.

Example:

```yaml
processes:
  cup_inspect:
    kind: atomic
    behavior:
      - object_identity_map
```

For atomic processes, if `objects` is completely omitted and the process declares `object_identity_map`, v0 infers, for every Object-bearing input port of that process, a `map` to the Object-bearing output port of the same name, type, and phase.

A port whose type is an `Array` is included. The inferred `map` is interpreted recursively over Object slots as 14.1 defines, so Array length, nesting structure, element order, and contained Object identities are preserved.

Where an Object-bearing input port has no output port of the same name, type, and phase, the inference produces nothing for it, and the resulting `objects` behavior does not satisfy Object tracking completeness (13): the port has no fate, which is a validation error. The same holds for an Object-bearing output port with no corresponding input port. The marker asserts an identity map, and a port it cannot be asserted of is a port the marker does not explain.

Canonical inferred form:

```yaml
objects:
  map:
    outputs.cup: inputs.cup
```

If an `objects` section is present, no implicit completion is performed. The written `objects` section must account for all Object-bearing input and output slots. The marker is then a statement about how the process behaves, which the written `objects` section must agree with.

`object_identity_map` is the only behavior marker v0 defines. A `behavior` entry naming anything else is a validation error. The section is named for what it describes rather than for this one marker, so a later revision can add a marker about a property other than Object behavior -- execution time, idempotence, reversibility -- without the section name having to change.

**This marker does not say the process is a no-op.** A process may update `.view` metadata, consume time, or have any other effect that Object behavior does not describe. What it grants is a licence to infer the Object mapping and to reason about Object behavior; it says nothing about whether the process must be executed.

---

## 16. Structured Carry Compatibility

Structured carry compatibility is required for `fold` and `do_while` carry outputs.

```text
For each carry binding name `c`, the target process must provide an output
named `c` with the same type and phase.
```

A carry value may be Pure Data or Object-bearing.

Structured carry compatibility does not imply physical Object identity preservation. It guarantees that a same-name, same-type, same-phase value is threaded across loop iterations.

For Object-bearing carry, the target process must relate the carried input port to the same-name output port in one of exactly two ways:

```text
the fate of the carried input port is the same-name output port
  (the carry transition preserves physical identity), or
the carried input port is consumed and the same-name output port is created
  (the carry transition replaces the Object)
```

Any other arrangement is a validation error. In particular it is an error for the fate of the carried input port to be a different output port, or for the same-name output port to have a provenance other than the carried input port or a `create`.

The reason is that the carried Object would otherwise leave the loop, or enter it, through a collected output or an `each` element, and the node's own Object correspondence would then have to name a position within a collection -- the first collected element, or the last element traversed. v0 does not address collection elements through an Object path (2.6.2), and 1.1 admits a shape indexed by a scalar rather than one that varies by position, so such a node has no expressible Object correspondence. Both permitted forms avoid this: identity preservation carries the same slot through, and replacement creates and consumes its intermediate Objects within the node, so the accounting closes there.

For Pure Data carry, the target process's same-name output becomes the next iteration's carry value. Pure Data carry is not subject to Object tracking or linearity restrictions, and the requirement above does not apply to it: Data has no identity, so the same-name output is the next carry value whatever it was computed from, which is the ordinary case of an accumulator.

`branch` does not use `carry` in v0. It uses `args` and explicit or implicit `common` outputs instead.

---

## 17. Feature: `node_map`

`node_map` enables structured nodes with `kind: map`.

`kind: map` performs independent element-wise invocations.

```yaml
- id: create_cups
  kind: map
  process: cup_create_from_label
  each:
    label:
      from: inputs.labels
```

Output shape:

```text
process output p: T
map output p: Array<T>
```

If the target process output is already an Array, nested Arrays are produced:

```text
process output groups: Array<Cup>
map output groups: Array<Array<Cup>>
```

A `map` node has the binding sections `each` and `bind`. `state`, `carry` and `args` are not valid for `map` (21.0).

An `each` binding is traversed element-wise: each element is passed to exactly one invocation. A `bind` binding is the opposite -- the same value is reused by every invocation and is not updated between them. As 11 requires, `bind` carries Pure Data only, so reuse never affects Object identity. Where the `each` length is zero the target process is not invoked and no `bind` value is used.

`map` requires only Object tracking completeness of the target process. It does not require structured carry compatibility.

In v0, `map` does not define an `outputs` section. A `map` node exposes all declared outputs of the target process, with each output collected into an Array according to the output shape rules below.

Multiple `each` inputs use zip-equal semantics:

```text
zip: equal
```

All `each` inputs of the same `map` must have equal top-level Array length. If a length mismatch is determined at `graph` phase, it is a graph-time validation error. If it is first determined at `run` phase, it is a run-start validation error or preflight error. If it is first determined at `data` phase, it is a runtime data error.

A `map` node must have at least one `each` source. The shape of a `map` is its body L times, indexed by the traversal length L (1.1), and L is the common length of the `each` sources; with no `each` source there is no L and nothing determines how many invocations the node performs. An absent `each` section and an empty one are the same error.

If the `each` length is zero, the map performs zero invocations and produces empty collected outputs.

---

## 18. Feature: `node_fold`

`node_fold` enables structured nodes with `kind: fold`.

`kind: fold` threads loop-carried values through element-wise invocations while traversing one or more Arrays.

A fold carry value may be Pure Data or Object-bearing.

Example:

```yaml
- id: process
  kind: fold
  process: sample_process_once
  carry:
    sample:
      from: inputs.sample
    score:
      from: inputs.initial_score
  each:
    reagent:
      from: inputs.reagents
  outputs:
    sample:
      mode: carry
    score:
      mode: carry
    measurement:
      mode: collect
```

Requirements:

1. Target process is Object tracking complete.
2. Each `carry` binding has a same-name, same-type, same-phase output on the target process, and an Object-bearing carry satisfies the threading requirement of section 16.
3. At least one `each` source is present. As for `map`, the shape of a `fold` is its body L times and L is the common length of the `each` sources, so with no `each` source nothing determines how many invocations the node performs. An absent `each` section and an empty one are the same error.
4. Carry outputs may be Pure Data or Object-bearing and must be exposed with `mode: carry` when `outputs` is present.
5. Non-carry Pure Data outputs may be exposed only using explicit output modes.
6. Non-carry Object-bearing outputs are allowed only with explicit `mode: collect`.
7. `bind` inputs are Pure Data only.
8. Multiple `each` inputs use zip-equal semantics, with phase-dependent error classification for length mismatches.

Object-bearing `each` values are not carry values. They follow ordinary Object tracking rules for the target process.

An Object-bearing collection used as an `each` source is linearly used by the `fold` node. The collection value itself is not treated as having a separate Object identity; its contained Object slots are passed to per-element invocations in traversal order and must be fully accounted for by the target process behavior and the fold output modes.

`bind` inputs are re-used for each invocation and are not updated between iterations. `carry` values are updated by the corresponding same-name carry output after each invocation.

### 18.1 Fold output modes

For a `fold` node, valid output modes are:

```text
carry
collect
drop
```

Rules:

1. `mode: carry` may be used only for outputs corresponding to fold carry bindings. The carry value may be Pure Data or Object-bearing.
2. When `outputs` is present, every carry binding must be listed with `mode: carry`.
3. `mode: collect` may be used for non-carry outputs of the target process. The collected output may be Pure Data or Object-bearing.
4. For target process output `p: T`, a fold output `p` with `mode: collect` has type `Array<T>` and contains per-invocation output values in invocation order.
5. `mode: drop` may be used only for non-carry Pure Data outputs.
6. Object-bearing outputs must not use `mode: drop`.
7. Every Object-bearing output of the target process must be exposed either as `mode: carry` or as `mode: collect`.
8. In v0, all collected per-invocation outputs of the same `fold` have the same Array length.
9. If `outputs` is present, it is fully explicit: every target process output must be listed with `mode: carry`, `collect`, or `drop`, subject to the Object-bearing restrictions above.

### 18.2 Default fold outputs

An empty `each` traversal is not an error. The node performs zero element-wise invocations, `mode: collect` outputs are empty Arrays -- including an Object-bearing collect output, which then contains zero Object slots -- and `mode: carry` outputs expose the initial carry value unchanged. No fold output mode requires an invocation to have happened.

**Non-normative.**

Where a `fold` carry transition replaces the Object by `consume` + `create` (16) and the `each` length turns out to be zero, the target process is not invoked, so the carry output is the same Object as the carry input: nothing is consumed and nothing is created. The node's Object skeleton (12.4.5) describes the transition as a replacement whatever the traversal length is, and is therefore an over-approximation of what happens in that case.

The over-approximation is sound. The number of Objects is unchanged either way, and a consumption paired with a creation duplicates nothing and loses nothing, so neither linearity nor completeness can be broken by the difference. Skeleton equality is unaffected too, since the skeleton does not depend on the traversal length.

`do_while` is not affected: it invokes its target at least once (19), so its traversal length is never zero.

If `outputs` is omitted:

```text
carry outputs are exposed as carry
non-carry Pure Data outputs are dropped
if the target process has any non-carry Object-bearing output, the fold node must declare an explicit `outputs` section
```

Thus, `fold` does not collect outputs by default. Use explicit `mode: collect` when non-carry outputs are needed. If the target process has any non-carry Object-bearing output, the fold node must have an explicit `outputs` section listing each such output with `mode: collect`. If `outputs` is present, the default behavior is disabled and the `outputs` section is fully explicit.

---

## 19. Feature: `node_do_while`

`node_do_while` enables structured nodes with `kind: do_while`.

`kind: do_while` invokes a process at least once and repeats while a Boolean condition output is true.

A do-while carry value may be Pure Data or Object-bearing.

Example:

```yaml
- id: loop
  kind: do_while
  process: sample_passage_once
  carry:
    sample:
      from: inputs.sample
    score:
      from: inputs.initial_score
  bind:
    medium:
      from: inputs.medium
  condition:
    output: continue
  max_iterations:
    value: 100
  outputs:
    sample:
      mode: carry
    score:
      mode: carry
```

Requirements:

1. Target process is Object tracking complete.
2. Each `carry` binding has a same-name, same-type, same-phase output on the target process, and an Object-bearing carry satisfies the threading requirement of section 16.
3. Object-bearing outputs of the target process must be carry outputs. Non-carry Object-bearing outputs are forbidden in `do_while`.
4. `condition.output` names a Boolean Data output of the target process.
5. `max_iterations` is required. It is a constant slot (11.2) of slot type `Int` with phase upper bound `run`, and its value must be greater than or equal to 1. A value of zero or a negative value is a validation error. This follows from do-while semantics: the target process is invoked at least once, and a bound of zero would contradict that guarantee. The phase at which the bound of at least 1 is decided follows 11.2.
6. Non-carry Data outputs may be exposed only using explicit output modes.

`do_while` has no `each` section and requires no `carry` binding. Its iteration count comes from the condition output and `max_iterations` rather than from a traversal length, which is why the requirement `map` and `fold` carry -- at least one `each` source -- has no counterpart here. A `do_while` with no carry binding is meaningful: its target is a physical operation, and the condition output it returns may differ between invocations even though the node binds the same values each time.

`condition.output` is an output name, not a body-scope dataflow reference. The condition output is evaluated after each invocation of the target process. The `do_while` node repeats while this output value is `true` and exits when it is `false`, subject to `max_iterations`.

If `max_iterations` is reached while the condition remains true, the `do_while` node terminates by reaching its iteration limit. This is **bounded termination**. Bounded termination is not an IR validation error, and v0 does not define it as a runtime failure.

Under bounded termination, the standard v0 execution behavior is that the `do_while` node still produces outputs from the invocations that actually ran. Carry outputs expose the final executed invocation's carry outputs. Collected outputs contain all executed invocation outputs in invocation order. If the condition output is collected, the final collected condition value is `true`. The node's reserved `exhausted` output (19.3) is `true`.

An implementation may report bounded termination as a diagnostic, warning, runtime status, or runtime concern.

### 19.1 Do-while output modes

For a `do_while` node, valid output modes are:

```text
carry
collect
drop
```

Rules:

1. `mode: carry` may be used only for outputs corresponding to do-while carry bindings. The carry value may be Pure Data or Object-bearing.
2. When `outputs` is present, every carry binding must be listed with `mode: carry`.
3. `mode: collect` may be used only for non-carry Data outputs of the target process.
4. `mode: drop` may be used only for non-carry Data outputs.
5. The condition output is an ordinary Boolean Data output of the target process and may be exposed using `collect` or `drop`. Where the condition output is also a carry binding, rule 2 takes precedence and it is exposed with `mode: carry`: the value threaded across iterations is what the node exposes, and the final one is the value the condition was last read from.
6. Non-carry Object-bearing outputs remain forbidden in `do_while`.
7. Collected Data outputs are ordered by invocation order.
8. The condition output includes the final `false` value when the loop exits normally because the condition became false, if the condition output is collected.
9. If `max_iterations` is reached while the condition remains true, the node terminates by reaching its iteration limit. This is bounded termination.
10. Under bounded termination, exposed outputs are produced from the invocations that actually ran.
11. Under bounded termination, `mode: carry` exposes the final executed invocation's carry output.
12. Under bounded termination, `mode: collect` contains all executed invocation outputs in invocation order.
13. If the condition output is collected under bounded termination, the collected values include the final `true` value.
14. In v0, all collected per-invocation Data outputs of the same `do_while` have the same Array length.
15. If the target process has any non-carry Object-bearing output, the `do_while` node is invalid. Otherwise, when `outputs` is present, it is fully explicit: every target process output must be listed with `mode: carry`, `collect`, or `drop`.
16. The reserved output `exhausted` (19.3) must not be listed in `outputs`.

### 19.2 Default do-while outputs

If `outputs` is omitted:

```text
carry outputs are exposed as carry
non-carry Data outputs are dropped, including the condition output where it is not a carry binding
non-carry Object-bearing outputs are forbidden
```

Thus, `do_while` does not collect Data outputs by default. Use explicit `mode: collect` when those outputs are needed. The reserved `exhausted` output (19.3) is exposed either way; it is not part of this default and is never listed. If `outputs` is present, the default behavior is disabled and the `outputs` section is fully explicit.

### 19.3 Iteration limit output

A `do_while` node has a reserved Boolean output named `exhausted`, in addition to the outputs it exposes from its target process.

```text
exhausted = true   the node terminated by bounded termination: `max_iterations`
                   was reached while the condition output was still true
exhausted = false  the node terminated because the condition output became false
```

`exhausted` is a Pure Data output with `phase: data`. It is always defined, so it is not listed in the `outputs` section; listing it there is a validation error, whether or not `outputs` is present for other reasons.

```yaml
- id: passage
  kind: do_while
  process: passage_once
  carry:
    sample:
      from: inputs.sample
  condition:
    output: continue
  max_iterations:
    value: 10
  outputs:
    sample:
      mode: carry
    continue:
      mode: drop

# passage.exhausted is available as a Bool
```

`exhausted` is the only way a workflow can tell the two terminations apart. Whether the iteration limit was reached is a fact about the node -- `max_iterations` is written on the node -- and not about the target process, which cannot know whether an invocation was its last. A carry value cannot report it for that reason.

`exhausted` reports reaching the limit, not the number of iterations. Where the iteration count or the history of the condition output is needed, the target process updates a Pure Data carry value, or the condition output is exposed with `mode: collect`.

No target process declares an output named `exhausted`: the name is reserved (2.4), so a reference to `<node>.exhausted` always names this output.

---

## 20. Feature: `node_branch`

`node_branch` enables structured nodes with `kind: branch`.

`kind: branch` selects one of two arms based on a Boolean Data condition.

A branch has `args`, not `state`. Branch arguments may be Pure Data or Object-bearing. They are supplied only to the selected arm at runtime. An Object-bearing branch argument is not duplicated across arms.

Example:

```yaml
- id: handle
  kind: branch
  condition:
    from: check.is_dirty
  args:
    cup:
      from: check.cup
  then:
    process: cup_wash
  else:
    process: cup_polish
```

Requirements:

1. Each arm process is Object tracking complete.
2. Each branch argument must correspond to an input port of the same name, type, and phase in each explicit arm process.
3. An Object-bearing branch argument is made available only to the selected arm and is not fan-out.
4. Branch outputs are common outputs selected from the executed arm.
5. One-sided Object-bearing outputs are forbidden in v0.
6. The two arms must have equal Object skeletons (20.2).
7. v0 does not provide Optional, Result, union-like branch outputs, or conditional Object provenance.

If `else` is omitted, it acts as an implicit identity arm for branch arguments for the purpose of Object-bearing common outputs. The implicit else arm returns each Object-bearing branch argument as a same-name Object-bearing output with the same type, phase, and physical identity. It does not implicitly expose Data outputs. Therefore, Data outputs from the `then` arm cannot be exposed as common outputs unless an explicit `else` arm is provided and the corresponding outputs are valid as common Data outputs.

### 20.1 Branch output modes

For a `branch` node, valid output modes are:

```text
common
drop
```

Rules when `outputs` is present:

1. `outputs` is authoritative. Only listed outputs are exposed.
2. `mode: common` may be used for outputs present in both arms with the same name, type, and phase. The output may be Pure Data or Object-bearing.
3. If an output listed as `common` is missing from either arm, it is a validation error.
4. If an output listed as `common` has different type or phase across arms, it is a validation error.
5. If the two arms' Object skeletons are unequal, it is a validation error (20.2).
6. `mode: drop` may be used only for Data outputs.
7. Any Object-bearing output produced by either arm must be listed with `mode: common`.
8. Unlisted Data outputs are dropped.
9. One-sided Object-bearing outputs are forbidden in v0.
10. One-sided Data outputs may be dropped, but cannot be exposed as `common`.

Example with explicit outputs:

```yaml
- id: choose_process
  kind: branch
  condition:
    from: inputs.use_a
  args:
    sample:
      from: inputs.sample
  then:
    process: sample_process_a
  else:
    process: sample_process_b
  outputs:
    sample:
      mode: common
```

For this branch to be valid, both `sample_process_a` and `sample_process_b` must expose `outputs.sample` as the same Object identity as the branch argument `args.sample`.

### 20.2 Branch Object skeleton

A `branch` node has no `objects` section. Its skeleton is derived from the two arms: from the `objects` section where an arm is atomic, and from the body graph where it is composite (12.4.4, 12.4.5).

**The two arms' skeletons must be equal in the sense of 12.4.3, with both placed at the `branch` node.** If they are not, it is a validation error. If they are, the node's skeleton is that common skeleton, and the node's skeleton is therefore determined whichever arm runs.

**Where the requirement comes from**

This is 12.4.7 applied to `branch`, which is the only construct in v0 with two descriptions of which one runs. Where the arms' skeletons differ, the node has no determined skeleton, and the origin of the identity of an output Object slot depends on a choice made at run time. Every place downstream that refers to that Object -- a binding on a later node or a `body.returns` entry -- then has no determined answer.

It is also stronger than the principle of 1.1. The two arms may agree on how many Objects they consume and create and still have unequal skeletons, if what an output Object's identity comes from is not the same.

**What follows**

```text
both arms correspond from the same argument slot     skeletons agree            valid
both arms consume and both create                    creation point is the node in
                                                     each case, so they agree    valid
one arm corresponds, the other creates               phi and N disagree          error
the arms correspond from different argument slots    phi disagrees               error
only one arm has the Object-bearing output           N or phi differs in size    error
the arms use different correspondence kinds for
  the same output slot                               the kind disagrees          error
```

The second row is what the creation point being a node rather than a process definition decides (12.4.3). Where both arms create, the creation point is this `branch` node in both cases: whichever arm runs, the new Objects appear at this node's output, so where their identity came from does not depend on the arm and the skeletons are equal.

For a collection output the arms need not create the same *number* of Objects. A skeleton names the port and not the size of its slot family (12.4.1), and the normal form carries the port name and the creation point and not a count (12.4.3). Skeleton equality is agreement on provenance, not on count. For the resource consequence see 1.1.

Where `else` is omitted, the implicit arm's skeleton is the identity correspondence from each Object-bearing entry of `args` to the same-name output (12.4.5, 20.3). The `then` arm must therefore have that same skeleton: an arm that consumes or creates an Object needs an explicit `else` that does the same.

The last row is not reachable in v0. The only correspondence of kind `order_preserving` comes from a transform, both v0 transform kinds change the type of the value, and rule 4 of 20.1 requires a common output to have the same type in both arms, so the type check reports first. It is stated because the set of correspondence kinds is open (12.4.2).

**Who checks it**

Skeleton equality is decided statically and the validator checks it. An execution engine assumes it and needs no run-time check.

### 20.3 Default branch outputs

If `outputs` is omitted, default branch output derivation applies only to Object-bearing outputs.

All Object-bearing outputs of the `then` and `else` arm processes are treated as implicit `common` output candidates.

For each Object-bearing output produced by either arm, both arms must produce an output with the same name, type, and phase, and the two arms' skeletons must be equal (20.2). If these conditions are satisfied, the branch node exposes that output as a common Object-bearing output. An Object-bearing output present in only one arm, corresponding outputs with different types or phases, and unequal arm skeletons are each a validation error.

Data outputs are not exposed by default. If `outputs` is omitted, Data outputs produced by branch arms are dropped.

If `outputs` is present, the default rules in this section do not apply.

---

## 21. Structured Node Output Summary

### 21.0 Valid sections per node kind

2.3 says that each node kind defines the binding and output-control sections valid for it, and that a section not defined for the kind is a validation error. This is that list.

| Node kind | Binding sections | Control sections |
|---|---|---|
| non-structured | `state`, `bind` | - |
| `map` | `each`, `bind` | - |
| `fold` | `each`, `bind`, `carry` | `outputs` |
| `do_while` | `bind`, `carry` | `condition`, `max_iterations`, `outputs` |
| `branch` | `args` | `condition`, `then`, `else`, `outputs` |

Besides these, every node carries `id`, and every node but a non-structured one carries `kind`. A node names the process it invokes with `process`, except a `branch`, which names one per arm inside `then` and `else`.

Writing a section against a node kind the table does not give it is a validation error. `map` has no `outputs`: its output shape is fixed, every target output `p: T` being exposed as `Array<T>` (17), so v0 defines nothing to shape it with. `condition` means different things by kind: for a `branch` it is a body dataflow reference, and for a `do_while` it names an output of the target process (2.6.7).

**Non-normative.**

An Object-bearing value that every invocation of a structured node shares is bound with `carry`. It cannot be `bind`, which is Pure Data, and it cannot be `each`, which gives each element to exactly one invocation. So a repetition over a shared physical resource is written with `fold` or `do_while`, never with `map`: the Object-bearing inputs of a `map` target are supplied from `each` alone, and `map` therefore expresses only independent operations over Objects that are pairwise distinct.

That follows from the physical fact rather than restricting it. One reagent container cannot be dispensed from by several invocations at once, so an operation over a shared resource is inherently sequential, and which structured node a workflow can be written with says whether its invocations may run in parallel.

```yaml
# One reagent container used across M plates, in turn
- id: run
  kind: fold
  process: dispense_to_one_plate
  carry:
    reagent:
      from: inputs.reagent        # the container is threaded
  each:
    plate:
      from: inputs.plates         # the plates are traversed
  outputs:
    reagent:
      mode: carry
    plate:
      mode: collect
```

Structured nodes may declare an `outputs` section to control which outputs are exposed by the structured node and how those outputs are shaped.

If `outputs` is omitted, the node uses the default output behavior defined for its structured node kind.

If `outputs` is present, only outputs explicitly listed in `outputs` are exposed by the structured node.

An exposed output is an ordinary body-visible value: another node may bind it, and `body.returns` may return it. Its resolved type — which binding type compatibility (11.1) is checked against — follows from the mode, given a target process output `p: T`:

```text
map       every output p          Array<T>
collect   fold / do_while         Array<T>
carry     fold / do_while         the carry binding's type, which is T (16)
common    branch                  T, the same type in both arms (20.1)
drop      any                     not exposed; not a body-visible value
```

`map` output types follow the shape rule of 17, so a target output that is already an Array becomes a nested Array. Because `mode: drop` is restricted to Pure Data outputs (18.1, 19.1, 20.1), an Object-bearing output is only ever exposed as `carry`, `collect`, or `common`.

A `do_while` node also exposes the reserved Boolean output `exhausted` (19.3), which is not shaped by a mode and is never listed in `outputs`.

Common modes:

```text
carry   exposes the final loop-carried value for `fold` or `do_while`
collect  collects per-invocation outputs into an Array; Object-bearing collect is allowed only where explicitly defined
common   exposes a branch output common to both arms
drop     explicitly suppresses a Data output
```

The valid modes depend on the structured node kind.

Summary:

```text
map:
  process output p: T -> map output p: Array<T>

fold:
  carry output: final carry value, same name as carry binding
  Pure Data or Object-bearing output with mode collect: Array of per-iteration values
  Data output with mode drop: not exposed
  default: expose carry outputs only; drop non-carry Pure Data outputs; require explicit outputs for non-carry Object-bearing outputs
  Object-bearing outputs: carry or explicit collect only; never drop
  empty traversal: allowed; collect outputs are empty Arrays and carry outputs are the initial values

do_while:
  carry output: final carry value, same name as carry binding
  Data output with mode collect: Array of per-iteration values
  Data output with mode drop: not exposed
  condition output: ordinary Boolean Data output; may use collect or drop, or carry
    where it is also a carry binding (19.1 rule 5)
  exhausted: reserved Bool output; always exposed; never listed in outputs
  default: expose carry outputs only; drop non-carry Data outputs including condition
  Object-bearing outputs: carry outputs only

branch:
  common output: selected arm output, same name/type/phase in both arms
  Data output with mode common: common Data output from both arms
  Object-bearing output with mode common: common Object-bearing output from both arms, the arms having equal skeletons
  Data output with mode drop: not exposed
  default: expose all same-name/same-type/same-phase Object-bearing outputs as common, the arms having equal skeletons; drop Data outputs
  one-sided Object-bearing outputs: invalid
  arm-dependent Object identity for common Object-bearing outputs: invalid
```

---

## 22. Feature: `python_script_processes`

`python_script_processes` supports inline script execution for Pure Data processes using Python.

A script process is an `atomic` process with a `script` section.

```yaml
processes:
  math_add:
    kind: atomic
    inputs:
      x:
        type: Float
        phase: data
      y:
        type: Float
        phase: data
    outputs:
      z:
        type: Float
        phase: data
    script:
      language: python
      code: |
        return {
          "z": x + y
        }
```

In portable v0, the only defined script language is:

```text
python
```

A script process that declares any other `script.language` is a validation error for portable v0. Implementations may provide additional script languages as extensions, but documents using those extensions are not portable v0 documents.

No `script.returns` field is defined in v0.

### 22.1 Pure Data restriction

All script process input ports and output ports must be Pure Data.

Object-bearing input ports and Object-bearing output ports are validation errors for script processes.

A script process must not contain an `objects` section.

Script processes do not create, consume, map, or transform Objects.

### 22.2 Execution model

The script `code` is evaluated as the body of an implementation-provided Python function.

Input port names are bound as local variables using the corresponding process input port names.

The function must return a mapping from declared output names to output values.

The returned mapping must contain exactly the declared output names.

Returned values must conform to the declared output types.

In v0, script process outputs must have `phase: data`.

If a script process's declared interface violates v0 script restrictions, it is a validation error. This includes Object-bearing input ports or output ports, an `objects` section on a script process, or an unsupported script language.

If the script executes but returns a mapping that does not exactly match the declared output names, or returns values that do not conform to declared output types, the invocation fails runtime verification. Such failures are not IR validation errors unless they can be determined statically.

### 22.3 Python imports and dependencies

Portable v0 Python scripts should rely only on Python built-ins.

Imports of Python standard library modules are implementation-defined. An implementation may allow selected standard library modules for convenience, but portable v0 does not require any standard library module to be available.

Imports of third-party packages, local project modules, network modules, or implementation-specific modules are not portable v0 and must be treated as implementation extensions.

An implementation may restrict all imports, including standard library imports, for sandboxing, security, determinism, or reproducibility.

If a script imports a module that is not allowed by the implementation, the IR may still be syntactically valid, but the workflow requires an unsupported script dependency for that implementation.

---

## 23. Removed: `scheduling_policies`

Revision 0.5 removed the `scheduling_policies` feature and the `scheduling` section of a composite process, which attached best-effort preferences on the gap between operations and on the temperature an Object is kept at. The section number is kept so that the sections after it keep theirs.

v0 states nothing about when an operation runs. A `scheduling` key in a process is a validation error, an unknown key (2.3), and a `features` section naming `scheduling_policies` is one too, an unknown feature name (4.2). A document written against revision 0.4 that used the section is migrated by deleting the section and removing `scheduling_policies` from `features`. No other part of a document depended on it.

The names the section used remain reserved (2.4).

---

## 24. Removed: Object Policy Targets

Revision 0.5 removed Object policy targets with the `scheduling` section (23). The section number is kept.

---

## 25. Runtime Failures and Exceptions

Runtime failures, exceptions, retries, cancellation, compensation, cleanup handlers, and recovery are outside the scope of v0.

v0 defines well-formed workflow structure and successful Object/Data flow semantics.

Recoverable domain-level failures may be modeled explicitly as ordinary Data outputs using user-defined Data types, if needed.

Example:

```yaml
types:
  AnalyzeOutcome:
    domain: data
```

No `Result<T, E>`, `Optional<T>`, exception edge, or try/catch construct is included in v0.

Runtime data errors and runtime verification errors may occur when invocation-time data violates requirements that could not be determined statically, such as a zip-equal traversal length mismatch when the relevant lengths are only known at data phase.

For phase-dependent requirements, v0 classifies the error by the earliest phase at which the violation is determined: graph-time validation error, run-start validation or preflight error, or runtime data error. Runtime data errors are outside the core validation semantics of v0, but this specification may still define standard execution behavior for particular runtime data errors.

`do_while` bounded termination is not an IR validation error and is not defined by v0 as a runtime failure. It is a runtime execution condition in which the node terminates by reaching `max_iterations` while the condition output remains `true`. v0 defines the outputs for bounded termination from the invocations that actually ran; implementations may report the condition as a diagnostic, warning, runtime status, or runtime concern.

---

## 26. Implementation Extensions and Portability

v0 distinguishes portable v0 validation from implementation extensions.

A strict portable v0 document uses only v0-defined syntax, keys, feature names, type syntax, process kinds, node kinds, script languages, and validation semantics.

A validator may provide an extension-tolerant mode. Extension-tolerant mode may accept implementation-defined extension keys using the reserved `x-` prefix, but those documents are not strict portable v0 documents unless the extensions are removed.

Implementation extension keys must use the reserved prefix `x-`. Unknown keys that do not use the `x-` prefix are validation errors even in extension-tolerant mode.

The v0 core validator does not define the schema or semantics of `x-` extension values. YAML well-formedness, string-key restrictions, duplicate-key restrictions, and import expansion still apply to extension keys and their values.

A feature name listed in `features` must be either a v0-defined feature name or, in extension-tolerant mode, an implementation-defined extension feature name using the form:

```text
x-[A-Za-z_][A-Za-z0-9_]*
```

Unknown non-extension feature names are validation errors. A v0-defined feature that is required by the document but not supported by a particular implementation is not a v0 validation error; it is an unsupported feature condition for that implementation.

In strict portable v0, the only portable script language is `python`. Other script languages are not portable v0. An implementation may support additional script languages only as extensions.

v0 core validation does not accept implementation-defined type constructors, process kinds, node kinds, Object transform kinds, binding sections, output modes, or alternate generic inference semantics. Extensions that change these areas define an extended dialect rather than strict portable v0.

Implementations may report validation, portability, unsupported-feature, and extension-use diagnostics separately.

---

## 27. Summary of v0 Core Rules

1. A process's input and output endpoints are collectively called ports; input ports are declared under `inputs`, and output ports are declared under `outputs`.
2. A v0 document may include optional reserved `spec_version` metadata naming the revision it is written against. If present, it must use the two-number string format `MAJOR.MINOR`; omission is allowed. It does not select an interpretation -- a document is read by the rules of the revision the implementation implements -- but an implementation refuses a later MINOR of the same MAJOR, and any other MAJOR, rather than answer for a revision it does not implement (2.1).
3. Types are nominal; built-in primitive Data types are `Bool`, `Int`, `Float`, and `String`, and the only built-in type constructor is `Array<T>`.
3a. Under the `units` feature (28), `Int` and `Float` may carry a unit suffix, which is part of the type: two numeric types with different unit normal forms are different types. Unit atoms are opaque names in their own namespace and must be declared in the top-level `units` section; `Float[1]` is `Float`. No other type may carry a unit suffix.
4. `$import` provides structural inclusion before validation; it is not a module system and introduces no namespace or aliasing. Import paths should be relative for portability; URI-scheme references are implementation extensions, and URI fragments are not defined in v0.
5. Imported fragments normally omit reserved metadata; duplicate keys at the same expanded mapping level after import resolution are validation errors.
6. v0 portable YAML is closed by default; unknown keys are validation errors.
7. Implementation extension keys must use the `x-` prefix and are not strict portable v0 unless explicitly accepted by an extension-tolerant validator mode.
8. `null` values are not valid in v0 portable YAML.
9. Omitted `units`, `traits`, `types`, `inputs`, and `outputs` are interpreted as empty mappings; `features` may be omitted and derived; `processes` is required.
10. v0 identifiers are case-sensitive ASCII identifiers matching `[A-Za-z_][A-Za-z0-9_]*`; `.` is not allowed in identifiers, including process names.
10a. Within one composite body, node ids are unique. Two nodes with the same `id` are a validation error, since `<node_id>.<output>` would then not name one value.
11. UTF-8 text is allowed in YAML string values and comments, but not in identifiers.
12. `features` is canonical when present and must include all required features; if omitted, required features are derived from the body.
13. Unknown non-extension feature names are validation errors.
14. Object-bearing values are determined by recursive `object_slots`; Object identity belongs to Object slots, not to the enclosing Object-bearing value as a whole.
15. Object-bearing values are linear: no fan-out, no implicit discard of contained Object slots. Within a composite body, every Object-bearing value -- a composite input port, an ordinary node's output port, or an output a structured node exposes -- is referred to exactly once (12.2).
16. Object-bearing values are normally `data` phase; rare `run` phase is allowed; `graph` phase is invalid.
16a. A value may flow from an earlier phase to a later phase but not the reverse; the order is `graph < run < data`. A binding or constant slot (11.2) whose value has a phase later than the port or slot it fills is a validation error.
17. Ordinary node invocations use `state` for Object-bearing linear inputs and `bind` for Pure Data inputs. The port's declared type decides which (for a type parameter, its domain); an Object-bearing value under `bind` and a Pure Data port under `state` are both validation errors.
18. In `fold` and `do_while`, `carry` is loop-carried and may be Pure Data or Object-bearing.
19. `branch` uses `args`, not `state`; Object-bearing branch arguments are supplied only to the selected arm. For a `branch`, the correspondence of rule 19a holds against each arm separately.
18a. Each node kind has a fixed set of valid binding and control sections (21.0); writing another against it is a validation error. `map` has no `outputs` section.
19a. The binding entries of a node and the input ports of the process it invokes are in one-to-one correspondence: every input port is bound exactly once across the binding sections valid for the node kind, and every binding entry names an input port. v0 defines no default value for an input port.
19b. A `map` node and a `fold` node must each have at least one `each` source; their traversal length is otherwise undetermined. `do_while` has no such requirement.
19c. For an Object-bearing carry, the target process either has the carried input port's fate be the same-name output port, or consumes that input port and creates that output port. No other arrangement is valid.
19d. `do_while` requires `max_iterations`, a constant slot (11.2) of slot type `Int` with phase upper bound `run`, whose value is at least 1. A bound of zero or a negative value contradicts the guarantee that the target process is invoked at least once, and is a validation error.
20. Atomic Object behavior is declared using explicit `inputs.*` / `outputs.*` paths, and every such path must name an Object-bearing port (14).
21. Composite Object behavior is derived from body graph flow and `returns`, as the composition of the body's node skeletons (12.4.5).
21c. The `returns` entries of a composite and its output ports are in one-to-one correspondence: every output port, Pure Data or Object-bearing, is returned exactly once, and every `returns` entry names an output port (12.3).
21a. A process's Object skeleton is the triple of a partial injection from its input Object slots to its output Object slots, the input slots it consumes, and the output slots it creates, each created slot carrying the node at which it is created. Object tracking completeness is completeness of that skeleton (12.4.6), and every process and node must have exactly one (12.4.7).
21b. Two dependency graphs must be acyclic, and they are different graphs. The process dependency graph is acyclic: a composite must not depend on itself, directly or through others (10.2). Within one composite body, the node dependency graph must also be acyclic; it has an edge from one node to another where a `from` in a binding or control section (21.0) of the second names an output of the first. A cycle in either is a validation error.
22. All processes must satisfy Object tracking completeness.
23. Every Object slot must have exactly one fate or provenance.
24. `map` preserves physical identity, and the resolved types of an entry's source and target must match (14.1).
25. `consume` ends an input Object identity.
26. `create` introduces a new Object identity at every Object slot of the named output port; for a collection port the declaration does not fix how many (14.3, 12.4.1).
27. `consume + create` is Object replacement; `.view` metadata does not automatically transfer across replacement.
28. v0 standard transforms are `array_flatten` and `array_unflatten`; they have no `params`, require strict role typing, and fix the correspondence between input and output Object slots as an order-preserving total bijection.
29. `object_identity_map` is a process-level behavior marker and inference permission, declared under a process's `behavior` section; it is not a type trait or a structured-control requirement, and it does not say the process may be skipped.
30. For Object-bearing Arrays, `object_identity_map` preserves length, nesting, order, and contained Object identities.
31. Structured node features are `node_map`, `node_fold`, `node_do_while`, and `node_branch`.
32. `map` requires only Object tracking completeness and exposes all target process outputs as Array outputs; v0 does not define `map.outputs`.
33. `fold` and `do_while` require structured carry compatibility for carry outputs; `fold` also allows non-carry Object-bearing outputs only with explicit `mode: collect`.
34. Empty `fold` traversal is allowed: no fold output mode requires an invocation to have happened.
34a. A `do_while` node exposes a reserved Boolean output `exhausted`, true when it terminated by bounded termination. It is never listed in `outputs`, and `exhausted` is a reserved name.
35. Zip-equal length mismatches are classified by the earliest phase at which the mismatch is determined.
36. `fold` and `do_while` expose carry outputs by default and drop non-carry Pure Data outputs by default; in `fold`, non-carry Object-bearing outputs must be explicitly listed with `mode: collect`.
37. If `do_while` reaches `max_iterations` while its condition output remains `true`, the node terminates by bounded termination; this is not an IR validation error or a v0-defined runtime failure, and outputs are produced from the invocations that actually ran.
38. `branch` exposes Object-bearing outputs by default only when they are common to both arms with the same name, type, and phase, and the two arms have equal Object skeletons (12.4.3); Data outputs are dropped by default.
39. A branch whose two arms have unequal Object skeletons is invalid in v0: the origin of an output Object's identity would depend on the selected arm. Two arms that both consume and both create are equal, since the creation point is the branch node in each case.
40. If branch `outputs` is present, it is authoritative and the default branch output derivation does not apply.
41. `python_script_processes` supports inline `language: python` script processes for Pure Data only; other script languages are not portable v0.
42. Python script code returns an output-name mapping; `script.returns` is not defined in v0.
43. Script return mismatches are runtime verification errors unless statically determined.
44. Removed in revision 0.5 (23).
45. Removed in revision 0.5 (23).
46. Removed in revision 0.5 (23).
47. Removed in revision 0.5 (23).
48. Removed in revision 0.5 (23).
49. Runtime failure and exception handling are outside v0.
50. v0 defines one closed built-in type trait, `Numeric`, satisfied only by `Int` and `Float`; document-defined type traits are declared in top-level `traits`.
50a. `Numeric` is satisfied only by a numeric primitive type whose unit is dimensionless, so a generic process constrained by `Numeric` does not accept a unit-annotated value.
51. Document-defined type traits are nominal membership markers only and do not imply fields, operators, conversions, subtyping, inheritance, Object behavior, or view structure; user-defined types cannot implement `Numeric`.
52. User-defined types implement traits using `implements`.
53. User-defined type views are declared in `types.*.view`; view fields are required, read-only Pure Data projections.
54. In v0, user-defined view field types must be primitive Pure Data types or Arrays recursively containing only primitive Pure Data element types.
54a. A view field type may carry a unit suffix where its base type is `Int` or `Float`. A static view value is checked against the base type; the declared unit is the unit of that value.
55. User-defined nominal Data types, Object types, Arrays of user-defined nominal Data types, and Arrays of Object types are not valid user-defined view field types.
56. A user-defined view field may declare an optional static `value`; if present, it is a graph-time type-level constant that must conform to the field's declared view field type.
57. Static view values must not be `null`; Float static values may use YAML integer, floating-point, or exponent numeric forms, but NaN and infinity are not valid portable v0 static values. This constrains the type-level constant only, not the Float values a workflow produces at run time.
58. Static view values do not create workflow values, Object identities, Object mappings, or Object tracking effects.
59. Every `type` field value is a YAML string scalar containing a v0 type expression. `Array<T>` is the only built-in type constructor, requires exactly one type argument, and may be nested.
60. Whitespace is allowed only immediately inside `Array` angle brackets, such as `Array< T >`; whitespace between `Array` and `<` is invalid.
61. Type atoms must resolve to a built-in primitive type, a top-level user-defined type, or a type parameter declared by the current process. Type parameters must not shadow top-level user-defined type names or reserved built-in names.
62. `Optional<T>`, `Result<T,E>`, union syntax, nullable suffixes such as `T?`, map types, tuple types, function types, and multiple type arguments are not valid v0 type expressions.
63. Generic type parameters are declared with a required `domain` of either `data` or `object`. A document that declares any `type_params` requires the `generic_processes` feature; `$import` is a pre-derivation structural mechanism and is not a feature.
64. A `domain: data` type parameter may be instantiated only with a built-in primitive Data type or user-defined nominal Data type; a `domain: object` type parameter may be instantiated only with a user-defined atomic Object type.
65. Type parameters are not instantiated with Array types; collection genericity is written as `Array<T>` or `Array<O>`.
66. Generic type argument inference and `where` constraint validation are performed during graph validation and do not depend on runtime values.
67. `where` constraints are string scalar trait constraints of the form `TraitName<T>` or `TraitName< T >`; whitespace before `<` is invalid.
68. Body dataflow references use only `inputs.<port>` or `<node_id>.<output>`. They do not use `.view`, `outputs.*`, nested node paths, or process names.
69. Atomic Object paths use only `inputs.<port>` and `outputs.<port>`. Object slots and Array elements are derived from the resolved port type, not directly addressed by path syntax.
70. Removed in revision 0.5 (23).
71. Contract expressions may reference only explicit `.view` references. v0 does not omit `.view` even though contracts only refer to views.
72. A binding source entry must contain exactly one of `from` or `value`.
73. `branch.condition.from` is a body dataflow reference. `do_while.condition.output` is a target process output name, not a body dataflow reference.
74. Primitive Data type views and `Array<T>.view.length` are defined by v0.
75. Contract expressions are checked against resolved view schemas and defined operator categories; limited `Int` / `Float` numeric promotion is local to contract expressions and does not imply implicit conversion elsewhere in v0.
76. Object type boundaries should reflect carry compatibility, replacement compatibility, and operational geometry.
77. `$import` values must be non-empty string scalars or non-empty sequences of non-empty string scalars. Import resolution is recursive, import targets must be single YAML documents, and `$import` does not remain in the expanded document.
78. Import cycles, unreadable or unparsable import targets, invalid import root shapes, YAML multi-document import targets, and URI fragments in portable v0 imports are validation errors.
79. Contract `expr` values are YAML string scalars using the v0 contract expression language. Contract references must explicitly include `.view`; direct port references and omitted `.view` are not valid v0 contract references.
80. Contract expression Float literals may use decimal exponent notation such as `1.0e+23`; integer exponent notation such as `1e3` is not a v0 Float literal.
81. Contract expressions must type-check to `Bool`; all subexpressions are parsed and type-checked without relying on Boolean short-circuiting to ignore invalid subexpressions.
81a. Contract expression unit rules are applied to each subexpression independently of base-type promotion: `*` and `/` compose units, while binary `+`, `-` and the comparisons require equal unit normal forms. A subexpression whose every leaf is a numeric literal carries no unit and is read in the unit of the operand it is compared or added to. A unit mismatch in a contract expression is a validation error, not a runtime contract violation.
82. Strict portable v0 uses only v0-defined syntax, keys, feature names, type syntax, process kinds, node kinds, script languages, and validation semantics.
83. Extension-tolerant mode may accept `x-` extension keys and `x-` extension feature names, but unknown non-`x-` keys remain validation errors.
84. v0 core validation does not accept implementation-defined type constructors, process kinds, node kinds, Object transform kinds, binding sections, output modes, or alternate generic inference semantics; such changes define an extended dialect.
85. A v0 document may include optional human-readable `description` metadata at the document root and at trait, type, and process definitions. `description` must be a YAML string scalar, is not a reserved identifier name, and does not affect document interpretation, validation semantics, feature derivation, type checking, Object tracking, or runtime behavior.
86. The resolved type of a bound value must match the resolved type of the port it is bound to, in every binding section and in `body.returns`. Matching is the structural matching relation of generic instantiation, which reduces to identity of type expressions when no type parameter is involved. An `each` source must be an Array, and it is its element type that is matched against the target input port.
86a. Binding type match includes the unit: a unit is never added, removed, or converted implicitly, at a binding, a constant slot, a carry, a structured node output, or a generic instantiation. Unit conversion is written as an ordinary atomic process, whose numeric correctness the IR does not check.
87. A literal binding source (`value`) must conform to its port's declared type, checked exactly as a static view value is checked against a view field type; an integer literal is accepted for a `Float` port. A literal must not be bound to an Object-bearing port.
87a. A literal is a `graph` phase value, the least phase, so it satisfies the phase requirement of every Pure Data position it may be written in.
88. Structural matching distinguishes a flexible type parameter (declared by the target process, determined by instantiation) from a rigid one (declared by the enclosing process, already fixed). A flexible parameter may be inferred to be a rigid parameter of matching domain; a rigid parameter matches only itself and never a concrete type. A `where` constraint whose parameter was inferred to a rigid parameter is satisfied only if the enclosing process declares that same constraint over it.
89. A contract view reference through a port whose declared type is a type parameter is not resolved against a view schema at the process definition, since the type argument is chosen at each instantiation; the grammar, reference scope, named ports, and other operands of the expression are still validated there. The parameter's domain is known at the definition: a bare `.view` on an `object`-domain parameter is a validation error, and on a `data`-domain parameter it is not.
90. Such a reference is instead resolved at each invocation that instantiates the process, by substituting that invocation's inferred type arguments into the target's port types and checking the expression against the resolved types. An invocation resolves only the type arguments it infers itself: where a type argument is a type parameter of the enclosing process, or where inference determined none, the contract is not checked at that invocation. A `where` constraint does not make a view field decidable, because a trait declares no field.
91. View field names must match the v0 identifier grammar and must not contain `.`, but they are not subject to the reserved-name list: a view field name is only ever an outer key in a view schema or the single segment after `.view` in a contract reference, so it cannot be confused with a structural key.
92. Where a validation error names a condition classified by the phase at which it is determined, the obligation is conditional on the implementation determining it at that phase. v0 requires no static inference of Array or traversal lengths, so an implementation that establishes emptiness or a length mismatch only at run or data phase reports it there, and not reporting it as a validation error is correct.

---

## 28. Feature: `units`

`units` is an **experimental** feature (4.5). It enables unit annotations on the built-in numeric primitive types.

A unit annotation is part of the type. Two values whose types differ only in their unit annotation have different types, and every rule in this specification that requires two types to be the same therefore requires their unit annotations to agree.

v0 defines no unit registry, no physical dimensions, and no scale relationships between units. A unit atom is an opaque name. The specification does not know that `s` denotes a second, that `min` and `s` measure the same physical quantity, or that one is sixty times the other.

All checks defined by this feature are performed at graph phase. No unit condition is classified as a run-start or runtime data error under 6.2.

A unit has no runtime representation. Two values whose types differ only in their unit annotation are represented identically, and no operation of a workflow reads a unit, so an implementation may discard every unit annotation once graph validation has succeeded and hand the execution layer the same document it would have received with no unit written in it. What a unit constrains is which documents are valid, not what a run does.

### 28.1 Unit declarations

Every unit atom used in a document must be declared in the top-level `units` section.

```yaml
units:
  s: {}
  min: {}
  uL: {}
  mL: {}
  mg: {}
```

A `units` declaration is a mapping from unit atom names to declaration bodies. In v0 the only valid declaration body is an empty mapping. A unit atom is a nominal name and nothing more: a declaration carries no dimension, no scale, no relationship to any other unit atom, and no conversion behaviour. A `units` entry whose value is not an empty mapping is a validation error.

A unit atom name must match `UnitIdent` (28.2). A name that does not, including a name a YAML processor represents as a scalar other than a string, is a validation error. Duplicate names within `units` are validation errors.

A unit atom appearing in a unit expression that is not declared in `units` is a validation error. This applies to unit expressions in port types, in constant slot types, in view field types, and anywhere else a type expression may appear.

This requirement exists so that a misspelled unit is reported where it is written, rather than only when the misspelled value is connected to something. `Float[sec]` in a document that declares `s` but not `sec` is a validation error at the point of use.

The dimensionless unit is written `1` (28.2) and is not a unit atom. It is not declared, and `1` is not a valid unit atom name.

Unit atom names occupy their own namespace. A unit atom name may coincide with a type name, a trait name, a process name, or a port name without conflict, and the reserved names of 2.4 do not apply to unit atom names.

An omitted `units` section is interpreted as an empty mapping (2.3), in which case no unit expression other than `1` can be written. A `units` section may be assembled through `$import` under the ordinary shallow structural merge rules of 3.2, which is the intended way to share a unit vocabulary across documents. 28.13 states what that costs.

### 28.2 Unit suffix syntax

A type expression may carry a unit suffix. The grammar of 2.5 becomes:

```text
TypeExpr      ::= TypeAtom UnitSuffix? | ArrayType
ArrayType     ::= "Array" "<" S? TypeExpr S? ">"
TypeAtom      ::= Identifier
Identifier    ::= [A-Za-z_][A-Za-z0-9_]*
S             ::= one or more ASCII space or tab characters

UnitSuffix    ::= "[" UnitExpr "]"
UnitExpr      ::= "1" | UnitTerm (("*" | "/") UnitTerm)*
UnitTerm      ::= UnitIdent Exponent?
Exponent      ::= "^" "-"? [1-9][0-9]*
UnitIdent     ::= [A-Za-z_][A-Za-z0-9_]*
```

Whitespace is not allowed anywhere inside a unit suffix, including immediately inside the brackets, and none is allowed between a type atom and its suffix. The whitespace rule of 2.5 is unchanged elsewhere, so `Array< Float[uL] >` is valid and `Float[mg / mL]` is not.

`*` and `/` are left-associative and have equal precedence, so `a/b*c` is `(a/b)*c`. Parentheses are not part of v0 unit expression syntax. An expression that cannot be written without parentheses must be written using explicit exponents.

`1` denotes the dimensionless unit and may appear only as the whole unit expression.

Valid unit suffixes:

```text
Float[s]
Float[uL]
Float[mg/mL]
Float[m*s^-2]
Float[1]
Int[count_]
Array<Float[mg/mL]>
```

Invalid unit suffixes:

```text
Float[mg / mL]     whitespace
Float [s]          whitespace before the suffix
Float[2s]          a unit atom must not begin with a digit
Float[s^0]         an exponent is a nonzero decimal without a leading zero
Float[1*s]         1 may appear only as the whole unit expression
Float[%]           % is not an identifier character
```

`Float[v/v]` is well-formed but denotes `v` divided by `v`, which normalizes to the dimensionless unit (28.3). See 28.13.

A unit suffix may be attached only to the built-in numeric primitive types `Int` and `Float`. A unit suffix on `Bool`, on `String`, on a user-defined nominal type, on `Array`, or on a type parameter is a validation error.

A unit suffix is written on the element type, not on the Array: `Array<Float[uL]>` is an Array of microlitre values. `Array[uL]<Float>` is not v0 syntax.

### 28.3 Normal form

A unit expression denotes a mapping from unit atoms to nonzero integer exponents. The normal form is computed as follows.

1. Expand the expression into atom/exponent pairs. A `UnitTerm` without an explicit exponent has exponent 1. A term on the right of `/` has its exponent negated. The expression `1` expands to no pairs.
2. Sum the exponents of equal atom names.
3. Remove every atom whose summed exponent is zero.
4. Sort the remaining atoms by atom name in ASCII order.

If the result is empty, the unit expression is **dimensionless**.

The canonical written form of a normal form joins the sorted atoms with `*`, writes each exponent with `^` unless it is 1, and writes `1` for the empty result. Implementations should use this form in diagnostics. It is not required in documents.

```text
mg/mL          normalizes to   mL^-1*mg
mg*mL^-1       normalizes to   mL^-1*mg
m*s^-2         normalizes to   m*s^-2
s/s            normalizes to   1
uL/mL          normalizes to   mL^-1*uL
```

Note that `uL/mL` is **not** dimensionless in v0. Because unit atoms are opaque, the specification does not know that `uL` and `mL` measure the same quantity, and the exponents of two distinct atoms do not cancel.

### 28.4 Type identity

Two type expressions denote the same type when their base types are the same and their unit normal forms are equal.

```text
Float[mg/mL]    ==  Float[mg*mL^-1]     same normal form
Float[uL]       !=  Float[mL]           distinct atoms
Float[mg/mL]    !=  Float[g/L]          distinct atoms
Float[s]        !=  Int[s]              distinct base types
Float[s]        !=  Float[s^2]          distinct exponents
```

### 28.5 Absence of a unit suffix

A numeric type written without a unit suffix is the same type as the same numeric type annotated with a dimensionless unit expression.

```text
Float    ==  Float[1]    ==  Float[s/s]
Int      ==  Int[1]
```

`Float` is not "a Float whose unit is unknown". It is a Float whose unit is dimensionless. Consequently:

```text
Float[s]  !=  Float
```

and neither direction connects. Binding a `Float[s]` value to a `Float` input port is a validation error, and binding a `Float` value to a `Float[s]` input port is a validation error. A unit is never added or removed implicitly.

### 28.6 No implicit conversion

v0 defines no conversion between units. `Float[min]` and `Float[s]` are unrelated types, as are `Float[uL]` and `Float[mL]`.

Unit conversion, adding a unit to an unannotated value, and removing a unit are written as ordinary atomic processes:

```yaml
processes:
  min_to_s:
    kind: atomic
    inputs:
      x:
        type: Float[min]
        phase: data
    outputs:
      y:
        type: Float[s]
        phase: data

  as_seconds:
    kind: atomic
    inputs:
      x:
        type: Float
        phase: data
    outputs:
      y:
        type: Float[s]
        phase: data
```

The numeric correctness of such a process is not checked by the IR. As with `objects.map` (14.1), the declaration is trusted. An implementation that multiplies by the wrong factor has an implementation correctness problem, not an IR validation error.

These processes are the intended way to connect a unit-annotated document to an `$import`ed library whose ports are unannotated. Because they appear as nodes, every point at which a unit is introduced or discarded is visible in the document.

Such a process is an ordinary atomic process at run time. Because a unit has no runtime representation, a conversion process is the one place where a unit change has a numeric consequence, and that consequence is the process's own implementation rather than anything the IR expresses.

### 28.7 Ports, bindings, and structured nodes

The unit annotation participates in type identity (28.4) and therefore requires no additional rules in the following places. The listed rules apply unchanged.

- Binding type compatibility (11.1): a `bind`, `state`, `carry`, `args`, `each`, or `returns` source must have the type of what it is bound to, including its unit. For an `each` source it is the element type that is matched, so traversing `Array<Float[s]>` binds `Float[s]`.
- Constant slots (11.2): a slot type is a type expression, so a slot of type `Int` is a slot of `Int[1]`. `do_while.max_iterations` has slot type `Int` (19), so a unit-annotated value does not fill it.
- Structured carry compatibility (16): a carry output must have the same name, type, and phase, and the type includes the unit.
- Structured node outputs (17, 18.1, 19.1, 20.1, summarized in 21): a `map` target output `p: Float[s]` is exposed as `p: Array<Float[s]>`, a collected `fold` or `do_while` output likewise, and a `branch` common output must have the same type in both arms, including its unit.
- Generic instantiation (8.1): structural matching compares base type and unit normal form. A unit-annotated numeric type is a concrete atomic type of `domain: data`, so a `domain: data` type parameter may be instantiated with one. `Float[s]` and `Float[m]` matched against the same type parameter is a validation error, as any two distinct concrete types would be.

The feature does not interact with Object tracking. A unit suffix may be attached only to the numeric primitive types, which have no Object slots (5.2), so an Object skeleton (12.4) never mentions a unit. Linearity (12.2), Object tracking completeness (13), the `objects` section (14) -- whose every path must name an Object-bearing port -- and the `object_identity_map` marker (15) are all unaffected.

### 28.8 Literals

A numeric literal written under `value` has no unit of its own. It is interpreted according to the type of the port or constant slot it fills.

```yaml
bind:
  duration:
    value: 30        # 30 seconds, if the target port is Float[s]
```

This is the existing rule that a literal is checked against the declared type of its target (11.1.1), extended to cover the unit. A literal is checked against the **base type**: the latitude 11.1.1 gives an integer literal filling a `Float` port is unchanged by the presence of a unit, so `value: 30` fills a `Float[s]` port.

A literal determines no type argument (8.1), so a literal bound to a port whose type is a bare type parameter remains a validation error in v0.

### 28.9 View fields and static view values

A view field type may carry a unit suffix. The restriction of 7.4 is extended: a view field type must be a v0 primitive Pure Data type, optionally unit-annotated where the base type is `Int` or `Float`, or an Array whose element type recursively satisfies the same restriction.

```yaml
types:
  Plate96:
    domain: object
    view:
      well_count:
        type: Int
        value: 96
      well_volume:
        type: Float[uL]
        value: 200
```

A static view value is checked against the **base type** by the rules of 7.4. The unit suffix does not affect that check. The declared unit is the unit of the static value.

### 28.10 Disabling unit checking

An implementation may offer a diagnostic validation mode in which unit checking is disabled. In that mode the document is validated as if every unit suffix had been removed from every type expression, as if the `units` section were empty, and as if the contract expression rules of 28.11 were not in effect.

The transformation only merges type distinctions; it never introduces one. Every document rejected with unit checking disabled is also rejected with it enabled. The mode is therefore useful for determining whether a validation failure has a cause other than units.

Four conditions apply.

1. The implementation must report that the mode was used.
2. Acceptance in that mode does not mean the document is a valid v0 document. The mode is not a conformance criterion.
3. The meaning of a document is always the meaning it has when the feature is enabled.
4. The implementation must not emit, as a canonical or normalized form, a document from which unit suffixes or the `units` section have been removed.

### 28.11 Contract expressions

Unit checking in contract expressions is performed on the type of each subexpression. Base-type rules are unchanged from 9.2; unit rules are applied independently and do not affect base-type promotion.

Every subexpression is one of two kinds. A **literal expression** is a subexpression whose every leaf is a numeric literal: a numeric literal itself, a unary `-` applied to a literal expression, and any `+`, `-`, `*`, or `/` applied to two literal expressions. A literal expression carries no unit. Every other subexpression is **unit-bearing** and has a unit normal form; a `Bool` or `String` subexpression is unit-bearing and dimensionless.

| Operator | Unit rule | Unit of result |
|---|---|---|
| `*` | none | sum of the operand exponents, a literal expression contributing none |
| `/` | none | left exponents minus right exponents, a literal expression contributing none |
| `+`, `-` (binary) | if both operands are unit-bearing, their normal forms must be equal | the unit-bearing operand's unit |
| `==`, `!=`, `<`, `<=`, `>`, `>=` | if both operands are unit-bearing, their normal forms must be equal | `Bool` |
| unary `-` | none | same as operand |
| `and`, `or`, `not` | operands are `Bool`, which is dimensionless | `Bool` |

Where one operand of a binary `+`, `-`, or comparison is a literal expression, no unit condition is imposed: the literal expression is interpreted in the unit of the other operand, exactly as a literal filling a port is interpreted in the unit of that port (28.8). Where both operands are literal expressions, the result is a literal expression.

Base-type promotion and unit composition are orthogonal:

```text
Float[uL] * Int          base Float, unit uL          -> Float[uL]
Int[s]    + Int[s]       base Int,   unit s           -> Int[s]
Float[uL] / Float[uL]    base Float, unit 1           -> Float
Int[mL]   / Int[s]       base Float, unit mL/s        -> Float[mL/s]
Float[s]  >= -30.0       the literal expression is read in s
Float[s]  >  1.0 + 2.0   the literal expression is read in s
```

```yaml
contracts:
  requires:
    - expr: "inputs.volume.view >= 200.0"
    - expr: "inputs.volume.view * inputs.concentration.view <= inputs.max_mass.view"
    - expr: "inputs.volume.view <= inputs.plate.view.well_volume"
```

In the second expression, if `volume` is `Float[mL]` and `concentration` is `Float[mg/mL]`, the product has unit `mg`, and `max_mass` must be `Float[mg]`. If `volume` were `Float[uL]`, the product would have unit `mL^-1*mg*uL` and the comparison would be a validation error, because v0 does not know that `uL` and `mL` are related.

A unit mismatch in a contract expression is a validation error, not a runtime contract violation.

Where a contract reference resolves through a port whose declared type is a type parameter, the expression is resolved at each invocation that instantiates the process (9.1), and the unit rules of this section are applied to the resolved types there.

### 28.12 Unit suffixes and type parameters

A unit suffix contains a unit expression, never a type expression. A type parameter name appearing inside a unit suffix is resolved as a unit atom name, and is a validation error unless an atom of that name is declared in `units`. A type parameter is never given a unit suffix (28.2), and v0 defines no way to abstract over a unit.

### 28.13 Guidance (non-normative)

Because unit atoms are opaque and no registry exists, two spellings of the same unit are two different units. A document that declares both `s` and `sec` will accept both, and `Float[s]` and `Float[sec]` will not connect. Authors should adopt a single spelling convention and place it in a file imported by every document in a project.

Suggested ASCII spellings, chosen to be forward-compatible with a possible future registry:

```text
time            s  min  h  d  ms  us
volume          L  mL  uL  nL
mass            g  mg  ug  ng  kg
amount          mol  mmol  umol  nmol
concentration   mg/mL  ug/mL  mol/L  mmol/L
temperature     degC  K
length          m  mm  um  nm
rotation        rpm
centrifugation  xg
ratio           pct  vv  wv
```

A ratio must not be written as a quotient of one atom by itself: it parses as a quotient and normalizes to `1`. Write `vv`.

A shared vocabulary file pairs naturally with the conversion processes of 28.6, since both are project-wide and neither is checked by the specification:

```yaml
# units_common.yaml
units:
  s: {}
  min: {}
  h: {}
  uL: {}
  mL: {}
  L: {}

processes:
  min_to_s:
    kind: atomic
    inputs:
      x: { type: Float[min], phase: data }
    outputs:
      y: { type: Float[s], phase: data }
```

Because a unit conversion process is not checked by the IR, such processes should be written once in a file of this kind and imported, rather than repeated.

**A vocabulary is imported once.** `$import` is structural inclusion, not a module system (3), and duplicate keys after import resolution are validation errors even where the two declarations are identical (3.2). A unit vocabulary must therefore reach a document by exactly one import path: a document that imports the vocabulary directly must not also import a library that imports it. This is a property of `$import` that `traits` and `types` share, and 28.1's declaration requirement makes it unavoidable for `units`. The practical arrangement is a single vocabulary file imported only by the top document, with library files declaring unit-annotated ports and leaving the atoms to their importer.
