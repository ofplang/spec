# FORMULATION

A mathematical formulation of the Object-flow Programming Language, v0.

This document accompanies `SPECIFICATION.md`. It defines a core calculus `ofp-core`, a
desugaring from v0 documents into it, a typing discipline that computes Object skeletons,
an operational semantics, and the results proved about them.

Section references of the form "spec N" point to `SPECIFICATION.md`.

---

## 0. Purpose and status

### 0.1 What this document is for

The specification defines validity in prose. This document restates the part of that
definition that concerns structure, types, and Object flow as a formal system, and proves
what that system guarantees.

The object of study is **the specification**, not any particular implementation. A claim
here has the form "a document valid by the rules of spec N has property P". Whether a
validator or a runner implements those rules correctly is a separate question.

### 0.2 Status of the results

| Result | Status |
|---|---|
| Lemma 1 (totality of D) | proved, modulo enumeration |
| Lemma 2 (soundness of D) | proved, modulo enumeration |
| Lemma 3 (completeness of D) | proved, modulo enumeration |
| Lemma 4 (skeleton agreement) | proved |
| Lemma 5 (exchange) | proved |
| Lemma 6 (termination of monomorphization) | proved |
| Lemma 7 (loop accounting closes) | proved |
| Lemma 8 (separation of creation points) | proved |
| Lemma 9 (lifting agrees) | proved |
| Proposition 2a, 2b (conservation) | proved |
| Proposition 2c (conservation on traces) | proved, assuming faithfulness |
| Proposition 3 (schedule independence) | proved, assuming faithfulness |
| Proposition 4 (orthogonality of annotations) | proved |
| Proposition 6 (conservativity of features) | proved |
| Proposition 7a (locality of partiality) | proved |
| Proposition 7b (termination) | proved, relative to atomic processes |
| Proposition 8a, 8b, 8c (cost) | proved |
| Proposition 1a (soundness of the free object) | provable; not written out |
| Proposition 1b (completeness of the free object) | **does not hold**; see 16.3 |
| Proposition 5 (stage erasure) | conjecture; outside the present scope (17) |

"Modulo enumeration" means the proof is a case analysis over a table of specification
conditions, and its reliability rests on that table being exhaustive. Section 13.4 states
what was done to establish that.

Auxiliary results carry letters, to keep them apart from the numbered ones above.

```
Lemma 0        skeletons are per port                        5.4
Lemma A        weakening and strengthening of arguments      12.1
Lemma B        the shape shared by the five LET rules        12.2
Lemma C1-C3    syntax direction, unique splitting, locality  12.3-12.5
Lemma S1-S4    complete morphisms form a subcategory of Slot 14.3
Lemma S5-S7    the auxiliary operations preserve completeness 14.4
Lemma S8       a correspondence induces a bijection          15.4
Lemma S9       flatten after unflatten is the identity       9.2
Lemma E1-E2    exchange and reordering during evaluation     15.7
Lemma G1       a skeleton does not depend on instantiation   15.10
Lemma 4a       the composition along a body is unique        13.5
Proposition I-1, Corollary I-2   the faithfulness triangle   11
```

### 0.3 What this document does not cover

`D1` (section 8) erases three layers of a v0 document. Nothing here says anything about
their validity.

```
contract expressions          spec 9, spec 27 rules 79-81
view schemas                  spec 7, spec 27 rules 50-58
scheduling policies           spec 23-24, spec 27 rules 44-48
portability and extensions    spec 26, spec 27 rules 82-84
runtime failures              spec 25
```

Proposition 4 shows the erasure does not change what a document denotes, so these layers
can be checked independently. Section 2.3 gives the order.

---

## 1. Setting

### 1.1 Linear and non-linear

A v0 type is either Pure Data or Object-bearing (spec 5.2). The two obey different
structural rules.

```
Pure Data       may be copied and discarded
Object-bearing  may be neither
```

This is the shape of Benton's mixed linear and non-linear logic, and of Barber's dual
intuitionistic linear logic: a cartesian part and a symmetric monoidal part, related by a
monoidal adjunction. A judgement carries two contexts.

```
Gamma ; Delta |- e        Gamma is Pure Data, Delta is Object-bearing
```

v0 is **strictly linear, not affine**. Implicit discard is forbidden (spec 12.2), so
`objects.consume` supplies a morphism `Cup -> I` at particular processes rather than a
natural family that would license weakening.

Pure Data wires carry a commutative comonoid structure -- copy and discard -- but not a
Frobenius structure: a Pure Data input port has indegree exactly one (spec 12.1), so wires
never merge.

### 1.2 Arrays are families, not lists

`Array<T>` in v0 is **a family over a finite index set**, not a list. The index carries the
correspondence between collections; it is not there to select elements.

Two consequences follow, and both are visible in the specification.

`zip: equal` (spec 17) is not a length check bolted onto a traversal. It is the domain of
the structure map that pairs two families over the same index set. What it requires is that
the two collections share an index.

`map` and `fold` are the functorial action and the traversal of that family. `array_uncons`
and `array_reverse` are absent (spec 14.4) because taking the head or reversing the order
addresses positions, which a family does not offer.

The categorical reading differs by domain.

```
T Pure Data       Array_I(T) = the I-fold product,      a representable functor (I -> T)
O Object-bearing  Array_I(O) = the I-fold tensor power, (x)_{i in I} O
```

Both are "the I-fold monoidal product", cartesian on the Pure Data side and tensor on the
Object side. Reading it as an exponential works only on the Pure Data side: `I -> O` has no
meaning for a linear `O`.

`zip` is then the laxator on the Pure Data side and an instance of symmetry and
associativity on the Object side, and in both cases it requires the same `I`.

### 1.3 Arrays are not initial algebras

`Array<T>` is not the initial algebra of `F(X) = 1 + T (x) X`. It has no `nil` constructor
and no total `case`.

`fold` is therefore not the unique morphism the universal property gives. It is an iteration
scheme the language provides. Equations that follow from uniqueness are not theorems.

```
not derivable   fold cons = id
                h . fold alpha = fold beta          (fold fusion)
                fold alpha . map f = fold (alpha . F(f))

derivable       map f . map g = map (f . g)         (functoriality; unaffected)
```

Section 16.3 states what this costs and why the cost is acceptable.

---

## 2. What is modelled

### 2.1 Three classes of condition

The validity conditions of the specification fall into three classes.

```
Class A   expressed by the typing rules of ofp-core
Class B   expressed by the domain of the desugaring D
Class C   erased by D1 and checked outside the core
```

Class B exists because `D` does not preserve everything. It does not keep binding section
names, so a Pure Data port bound under `state` -- which spec 11 forbids -- would still
produce a well-typed core term. The node translation of `D4` reads sections by name, so `D`
is simply undefined on such a document.

For that reason the adequacy lemmas carry "and `D(doc)` is defined" in their statements.

### 2.2 Which conditions fall where

Class A, keyed to the summary rules of spec 27 where one applies.

```
1, 3, 14, 16, 16a, 59, 61, 62    types, phases, Object-bearing-ness
15                               linearity; every Object-bearing value referred to once
18, 19, 21, 21a, 21b, 22, 23     bindings, acyclicity, skeleton, tracking completeness
18a, 19a, 19b, 19c, 19d          valid sections, binding coverage, each>=1,
                                 carry threading, max_iterations >= 1
20, 24, 25, 26, 27, 28           the objects section and the transform kinds
29, 30                           object_identity_map
31, 32, 33, 34, 34a, 36, 38      structured nodes and their output modes
39, 40                           branch skeleton equality
63-67, 88                        generics, rigid and flexible parameters
68, 69, 72, 73                   references
86, 87, 87a                      binding type match, literals
spec 12.1, 12.2, 12.4            degrees, skeleton
```

Class B.

```
2, 4, 5, 6, 8, 9, 10, 10a, 11, 60  document shape, imports, identifiers, type syntax
12, 13                           features
17                               state is Object-bearing, bind is Pure Data
41, 42                           script processes
77, 78                           import resolution
85                               description
spec 2.3                         each node kind fixes its valid sections
spec 2.4                         reserved names; no duplicate port names; unique node ids
spec 10.3                        entry
spec 22                          script outputs are phase: data
```

Class C.

```
7, 35, 43, 44-48, 50-58, 70, 71, 74, 75, 79-84, 89-91
```

Four of the summary rules state no validity condition and fall outside the classification.
Rule 37 describes what a `do_while` produces under bounded termination, which section 10
models as an evaluation rule rather than as a check. Rule 49 places runtime failure and
exception handling outside v0. Rule 76 is advice on where to draw an Object type boundary.
Rule 92 qualifies the other rules rather than adding one: an obligation classified by phase
holds only where the implementation determines the condition at that phase.

### 2.3 Class C splits in two

Class C is not uniform. Spec 2.2 places the scheduling check (step 8) after Object tracking
(step 7), because a policy target must be a body-visible Object-bearing value (spec 2.6.3)
and the set of identities it denotes follows the skeleton (spec 24.1).

```
C1   checkable independently        contracts, views, portability
C2   depends on class A             scheduling policy targets, temporal references
```

An implementation therefore proceeds in this order.

```
1. D0                    resolve imports
2. check C1              contracts, views, portability
3. D1 .. D5              desugar
4. check A and B         core typing
5. check C2              scheduling, using the skeletons from step 4
```

Proposition 4 is what makes the separation legitimate: erasing the annotations does not
change the skeleton, so C2 may read a result computed without it.

### 2.4 The trust boundary

Spec 14.1 states that an `objects` declaration is trusted. An implementation of an atomic
process is not proved to move Objects the way it says it does.

This is not only a convenience of formalization. **The IR cannot observe physical Object
identity.** No processor can confirm from the inside that the robot moved the cup it
reported moving. Verification therefore has two layers.

```
Layer 1   the driver's report agrees with the declaration     checkable at run time
Layer 2   the driver's report agrees with the physical world  needs external instrumentation
```

Layer 2 lies outside the language. Section 15.4 returns to this.

---

## 3. Notation

```
pi          a phase; pi in {graph, run, data}, ordered graph < run < data
tau         a type
A, R, O     an assignment: port name |-> type @ phase
Gamma       a Pure Data context
Delta       an Object-bearing context
sigma       a skeleton, written (mu, C, N)
mu          the partial injection of a skeleton
l           a node label
kappa       a correspondence kind; kappa in {id, op}
m           an output mode assignment
```

Restrictions and splittings of an assignment:

```
A|X    A restricted to the port names in X
A^p    the Pure Data part of A
A^o    the Object-bearing part of A
```

`A = A^p (+) A^o` always.

---

## 4. ofp-core: syntax

### 4.1 Types and phases

```
tau ::= Bool | Int | Float | String | N | Array<tau>
pi  ::= graph | run | data
```

`N` is a nominal type; the signature gives its `domain` (spec 5.1). There are no type
parameters: `D2` monomorphizes.

```
pure(tau)         tau contains no Object identity
depth(tau)        the number of Array nestings above the Object atom, for non-pure tau

depth(N)          = 0                for domain(N) = object
depth(Array<tau>) = depth(tau) + 1   for non-pure tau
```

A skeleton is stated **per port**, with depth as an attribute. Section 5.4 shows why no
finer representation is needed.

### 4.2 Terms

A body is a let sequence in a topological order of its dependency graph.

```
e ::= return(z... ; a... as p...)

    | let l : (d... ; o...) = f(z... ; y...) in e
    | let l : (d... ; o...) = map f (each w..., y...) (bind z...) in e
    | let l : (d... ; o...) = fold f (each w..., y...) (carry c..., c_o...) (bind z...) [m] in e
    | let l : (d... ; o... ; x) = dowhile f (carry c..., c_o...) (bind z...) (cond p) (max k) [m] in e
    | let l : (d... ; o...) = branch c (args z..., v...) (then f) (else g) [m] in e

z ::= x | v            a Pure Data argument: a variable or a literal
v ::= a literal        primitive, or a Pure Data Array
m ::= port |-> mode    mode in {carry, collect, drop, common}
```

`f` and `g` name signature entries. **Processes are not values**: ofp-core is first order.

The transform kinds are `array_flatten` and `array_unflatten` (spec 14.4).

### 4.3 Contexts

```
Gamma ::= . | Gamma, x : tau @ pi        requires pure(tau)
Delta ::= . | Delta, a : tau @ pi        requires non-pure(tau), pi /= graph
```

`Gamma` admits weakening and contraction. `Delta` admits neither.

```
Delta = Delta1 (+) Delta2      a disjoint splitting: the variable sets are disjoint
```

**The splitting is what makes duplication inexpressible.** Passing one variable to two
arguments needs a splitting that does not exist.

```
Delta_y <= A      the variables of Delta_y correspond bijectively to the ports of A,
                  types agree, and pi(variable) <= pi(port)
```

### 4.4 Signatures

```
Sigma ::= . | Sigma, N : domain(N)
        | Sigma, f : atomic(In_f) -> Out_f  |> sigma_f
        | Sigma, g : composite(In_g) -> Out_g = lambda(...). e
```

`In_f` and `Out_f` are assignments. Well-formedness of an atomic entry:

```
every type is well formed
(In_f)^o and (Out_f)^o have pi /= graph
sigma_f : Obj(In_f) -> Obj(Out_f) is complete (5.3)
```

A script process (spec 22) has `(In_f)^o = (Out_f)^o = empty`, `sigma_f = (empty, empty,
empty)`, and every output at `phase: data`.

A composite entry is well formed when its body types; its skeleton is read off that
derivation. **That a composite's skeleton is derived rather than declared is the content of
functoriality**, and is why composites are not inlined away.

A program is `(Sigma, entry)` with every entry well formed, an acyclic *process* dependency
graph (spec 10.2, rule 21b), and `entry` a composite. The node dependency graph of each body
is acyclic as well (spec 10.2, rule 21b); that is what `D4` relies on (8.6).

### 4.5 Phase conditions

Phase *order* appears in exactly five places. None is omitted from a rule.

```
1. ARG-VAR (7.1)            pi(variable) <= pi(port)
2. <= (4.3)                 pi(variable) <= pi(port), for Object-bearing arguments
3. RETURN (7.3)             pi(variable) <= pi(output port), for Object-bearing results
4. LET-DOWHILE (7.7)        pi(max) in {graph, run}, written as |= Int @ run
5. context well-formedness  Object-bearing implies pi /= graph (spec 6.1)
```

A literal is at `graph`, the least phase (spec 11.1.1), so `ARG-LIT` carries no phase condition.

Item 4 is written `Gamma |- k <= Int @ run` because `pi' <= run` iff `pi' in {graph, run}`.
Writing it that way puts it under the argument judgement, where weakening and strengthening
(Lemma A, 14.1) apply uniformly.

Phase also appears as an **equality** in three places, where two ports must agree. These use no
order, so Lemma A does not apply to them.

```
6. LET-FOLD (7.6), LET-DOWHILE (7.7)  carry compatibility: same type and phase   spec 16
7. valid mode assignment (6.3)        m(p) = common: same type and phase         spec 20.1 rules 2-4
8. implicit else arm (8.5)            arm outputs of the same names, types, and phases   spec 20
```

---

## 5. Skeletons

### 5.1 The category Slot

```
objects    finite sets of Object ports, each port carrying a depth
morphisms  triples (mu, C, N)
             mu : Obj(In) -> Obj(Out), a partial injection; each pair carries a kind kappa
             C  = Obj(In) minus dom(mu)      consumed
             N  = Obj(Out) minus im(mu)      created; each element carries a creation point
composition  relational composition
identity     the total identity injection
tensor       disjoint union
```

That `mu` is injective says no Object is duplicated. That it is partial allows one to be
consumed. **The linearity of v0 is in the definition of the morphisms.**

Assigning a fresh name at each creation amounts to taking the Kleisli category of a name
generation monad. The creation point (spec 12.4.3) is that name.

Composition, following spec 12.4.5:

```
sigma2 . sigma1 = (mu, C, N)
  mu(a) = mu2(mu1(a))                    where both are defined
  kappa composes by  id . id = id, otherwise op
  C = C1 union { a | mu1(a) in C2 }
  N = N2 union { mu2(b) | b in N1, b in dom(mu2) }
  creation points are carried from the element they came from
```

### 5.2 Correspondence kinds

Kinds are labels; their meaning is defined at the level of elements (section 9.2).

```
kappa = id    the element-level correspondence preserves position; depths agree
kappa = op    the element-level correspondence preserves traversal order; depths may differ
```

`id` is a special case of `op`, but the two are distinct labels for the purpose of equality
(spec 12.4.2). The set of kinds is open: adding one changes neither the normal form nor the
equality procedure.

### 5.3 Completeness

```
sigma is complete   iff  every element of Obj(In)  is in exactly one of dom(mu) and C
                    and  every element of Obj(Out) is in exactly one of im(mu) and N
```

The substantive part is that `mu` is a partial injection; the rest follows from the
definitions of `C` and `N`.

### 5.4 Skeletons are per port

**Lemma 0 (port granularity).** In v0, all Object identities contained in an Object-bearing
port share one fate, and all those contained in an Object-bearing output port share one
provenance.

*Proof.* The sources of a correspondence are `objects.map`, `array_flatten`,
`array_unflatten`, `consume`, and `create` (spec 14). Each relates whole ports. []

Hence a skeleton needs no index. A port is one entry, carrying its depth.

**A transform kind that split one port into two would invalidate this** and force a finer
representation. Criterion 3 of spec 14.4 rules such a kind out on the independent ground that
it would single out an index position, so the two constraints agree.

### 5.5 Skeleton of an atomic process

Read from the `objects` section (spec 12.4.4).

```
objects.map        adds (inputs.p |-> outputs.q) to mu with kind id
                   requires the resolved types to match (spec 14.1), hence equal depths
objects.transform  adds the correspondence the kind defines to mu with kind op
objects.consume    adds the named input port to C
objects.create     adds the named output port to N, creation point = the invoking node
```

The two transform kinds:

```
array_flatten     inputs.xss |-> outputs.xs     kind op, depth - 1
array_unflatten   inputs.xs  |-> outputs.xss    kind op, depth + 1
```

Both have `C = N = empty`. The change of depth is what distinguishes `op` from `id`.

The structure map is that of a monoidal functor: `Array_I . Array_J = Array_{I x J}`. Taking
the index category to be finite ordinals with order-preserving maps, `(FinOrd, x)` is
monoidal but **not symmetric** -- swapping `lex(m,n)` for `lex(n,m)` does not preserve order.
The absence of `transpose` from v0 is therefore structural, not a matter of taste. v0's
`Array` is a **semigroupal functor**: it has the multiplication and exposes neither the unit
(`singleton`) nor a symmetry (`transpose`).

### 5.6 Normal form

```
Part 1   the input port names in dictionary order, each with a destination:
         (output port name, kind), or `consumed`
Part 2   the created output port names in dictionary order, each with its creation point
```

Two skeletons are equal when their normal forms agree literally. Depth is determined by the
port type and is not part of the normal form.

Examples:

```
cup_inspect (object_identity_map)
  Part 1   inputs.cup -> (outputs.cup, id)
  Part 2   -

sample_replace (consume + create)
  Part 1   inputs.sample -> consumed
  Part 2   outputs.sample @ l

pool (array_flatten)
  Part 1   inputs.xss -> (outputs.xs, op)
  Part 2   -
```

### 5.7 Creation points are static allocation sites

A creation point is a **static allocation site**, not a run-time Object identity. It is the
allocation-site abstraction of pointer analysis.

Where a `do_while` carry replaces its Object, `n` iterations create `n` identities, but the
site is one. Intermediate identities do not appear in the skeleton (Lemma 7).

Nothing in v0 can name them. The four ways a value is named statically are policy targets
(spec 23.1 does not reach inside a loop body), skeleton equality (compared at node level),
contracts (`.view`, no identity), and downstream bindings (node output ports only).

`branch` shares one creation point across both arms. **Not because the two need not be
distinguished, but because they must not be**: separating them would make
`sigma_then /= sigma_else` and reject the both-arms-create case that spec 20.2 admits.

---

## 6. Auxiliary operations

These act on assignments, not on single types. The type constructor `Array` acts on a type;
`Fam` acts on an assignment. Different objects, hence different names.

### 6.1 Fam

```
Fam(A) = { p : Array<tau_p> @ pi_p  |  (p : tau_p @ pi_p) in A }
```

**Phase does not change.** It is `Array<tau> @ pi`, not `Array<tau @ pi>`; phase sits outside
the `Array`.

On skeletons, `Fam(sigma)` keeps the ports and the correspondence and raises every depth by
one. Kinds are preserved: `id` requires equal depths and both sides rise equally.

At the level of elements, `Fam(sigma)` sends the identity at position `i` of the input family
to position `i` of the output family, and in a structured node **position `i` is invocation
`i`**. This is what spec 17's output shape rule and the "ordered by invocation order" clauses
of spec 18.1 and 19.1 say.

**Lemma 9 (lifting agrees).** `Obj(Fam(A)) = Obj(A)`, with every depth raised by one.

*Proof.* `pure(Array<tau>)` iff `pure(tau)`, and `depth(Array<tau>) = depth(tau) + 1`. []

Lemma 9 is why `ModeS` is well defined relative to `Mode`. Per-port representation reduces it
to arithmetic on depths.

### 6.2 Mode and ModeS

```
Mode(m, A), per port p:
  m(p) = collect  ->  p : Array<tau_p> @ pi_p
  m(p) = carry    ->  p : tau_p @ pi_p
  m(p) = common   ->  p : tau_p @ pi_p
  m(p) = drop     ->  p is absent from the result

ModeS(m, sigma), per output port p:
  m(p) = collect  ->  depth + 1
  m(p) = carry    ->  depth unchanged
  m(p) = common   ->  depth unchanged
  m(p) = drop     ->  requires pure(tau_p); absent from the skeleton
```

The side condition on `drop` is spec 18.1 rule 6, spec 19.1 rule 4, and spec 20.1 rule 6 made
structural. Without
it, an Object-bearing port could leave the skeleton and completeness would fail (Lemma S6).

```
Mode(all-collect, A) = Fam(A)
ModeS(all-collect, sigma) = Fam(sigma)
```

so a `map` reads the same on both sides, which is functoriality stated syntactically.

### 6.3 Valid mode assignments

`m` is total on the target's output ports. Per spec 21.0 and the mode rules of spec 18-20:

| Node | Modes | Conditions |
|---|---|---|
| `map` | - | `m` is `all-collect`, fixed; there is no `outputs` |
| `fold` | carry, collect, drop | `m(p) = carry` iff `p` in `W`; `drop` needs `pure` |
| `dowhile` | carry, collect, drop | as above, and `(Out_f)^o` is a subset of `W` |
| `branch` | common, drop | see below |

For `branch`, `m` is total on the union of the two arms' output port names, and:

```
m(p) = common   requires p in both arms with the same type and phase   spec 20.1 rules 2-4
m(p) = drop     requires pure(tau_p)                                    spec 20.1 rule 6
non-pure(tau_p) implies m(p) = common                                   spec 20.1 rules 7, 9
```

The third condition is what forbids a one-sided Object-bearing output: such a port cannot be
`common`, so no valid `m` exists. A one-sided Pure Data output may be `drop` but not `common`
(spec 20.1 rule 10).

### 6.4 Fold and DoWhile

`W` is the set of carry port names, `E` the set of `each` port names, `B` the set of `bind`
port names; the node's syntax partitions `In_f` into the three.

**Carry restriction (spec 16).** For every carry port `w`:

```
mu_f(in.w) is either out.w or undefined
mu_f^{-1}(out.w) is either in.w or undefined
and the two agree: mu_f(in.w) = out.w iff mu_f^{-1}(out.w) = in.w
```

So each carry port is in exactly one of two cases.

```
(a) correspondence   mu_f(in.w) = out.w, with kind kappa_w
(b) replacement      in.w in C_f and out.w in N_f
```

Types agree across a carry, so depths agree and case (a) has kind `id`.

**Fold(sigma_f, W, m).**

```
input   Obj(In_f|W) (+) Fam(Obj(In_f|E))
output  Obj(Mode(m, Out_f))

carry port w:
  (a)  mu(in.w) = out.w, kind id
  (b)  in.w in C, out.w in N with creation point l

each port e, following the fate sigma_f gives in.e:
  mu_f(in.e) = out.q with m(q) = collect
       -> mu(in.e) = out.q, kind carried over; both sides at depth + 1 by Fam
  mu_f(in.e) = out.w with w in W
       -> not typeable; the carry restriction excludes it
  in.e in C_f
       -> in.e in C

other output port q not in W:
  out.q in N_f with m(q) = collect  -> out.q in N with creation point l, depth + 1
  m(q) = drop                       -> requires pure(tau_q); absent from the skeleton
```

**DoWhile(sigma_f, W, m)** is the carry part alone. There is no `each`, so no lifting, and
`(Out_f)^o` is a subset of `W` by spec 19 requirement 3.

---

## 7. Typing

The judgement computes the skeleton alongside the types.

```
Gamma ; Delta |- e => (R ; O) |> sigma        sigma : Obj(Delta) -> Obj(O)
```

Computing the skeleton in the rules rather than afterwards is what makes Proposition 2a an
induction on derivations.

### 7.1 Arguments

```
(x : tau @ pi') in Gamma      pi' <= pi
------------------------------------------  ARG-VAR
Gamma |- x <= tau @ pi

|- v <= tau      pure(tau)
--------------------------------  ARG-LIT
Gamma |- v <= tau @ pi
```

`ARG-LIT` carries no phase condition because a literal is a `graph` phase value, the least
phase (spec 11.1.1, rule 87a).

### 7.2 Literals

Literals are checked, never synthesized.

```
-----------------  LIT-BOOL       -----------------  LIT-INT
|- b <= Bool                      |- n <= Int

-----------------  LIT-FLOAT      -----------------  LIT-STRING
|- r <= Float                     |- s <= String

|- v1 <= tau  ...  |- vn <= tau
--------------------------------  LIT-ARRAY   n >= 1
|- [v1,...,vn] <= Array<tau>

--------------------  LIT-NIL
|- [] <= Array<tau>
```

`LIT-NIL` takes `tau` from the expected type. **This is why literals are checked rather than
synthesized**: the type of `[]` is not determined by the literal.

Conformance follows spec 11.1.1, which is the rule for static view values (spec 7.4). `NaN`,
infinity, `null`, and YAML custom tags do not conform.

An empty `Array` literal is the only way to obtain an empty Object-bearing collection: pass
it to the `each` of a `map` whose target creates, and the target runs zero times.

### 7.3 RETURN

```
Gamma |- z... <= R
Delta = a1:tau1@pi1, ..., an:taun@pin              exactly
O = p1:tau1@pi1', ..., pn:taun@pin'                pii <= pii'
rho : Obj(Delta) -> Obj(O) is the total bijection ai |-> pi, kind id
----------------------------------------------------------------------  RETURN
Gamma ; Delta |- return(z... ; a1 as p1, ..., an as pn) => (R ; O) |> (rho, empty, empty)
```

**That `Delta` is consumed exactly is the point of this rule.**

```
a leftover Object variable cannot be dropped   forbids disappearance
a variable cannot be returned twice            forbids duplication
O is determined as the image of Delta          forbids an output with no provenance
```

These are the three things spec 12.2 states as a degree rule and spec 13 forbids as
properties.

### 7.4 LET-ATOMIC

```
Sigma(f) = (In_f) -> Out_f |> sigma_f
Gamma |- z... <= (In_f)^p
Delta = Delta' (+) Delta_y        Delta_y <= (In_f)^o
d..., o... fresh
Gamma, d...:(Out_f)^p ; Delta', o...:(Out_f)^o |- e => (R ; O) |> sigma_e
------------------------------------------------------------------------  LET-ATOMIC
Gamma ; Delta |- let l : (d... ; o...) = f(z... ; y...) in e
    => (R ; O) |> sigma_e . (id_{Obj(Delta')} (x) sigma_f@l)
```

`f` may be atomic or composite; for a composite, `sigma_f` comes from its own derivation.

`Delta = Delta' (+) Delta_y` is where the wiring linearity lives. Spec 11 requires each input
port to be bound exactly once, and spec 12.2 requires each Object-bearing value to be referred
to exactly once; the splitting is both.

### 7.5 LET-MAP

```
Sigma(f) = (In_f) -> Out_f |> sigma_f
In_f = In_f|E (+) In_f|B
Gamma |- z... <= In_f|B                          bind
Gamma |- w... <= Fam((In_f|E)^p)                 Pure Data each
Delta = Delta' (+) Delta_y   Delta_y <= Fam((In_f|E)^o)
E is non-empty                                   spec 17
d..., o... fresh
Gamma, d...:Fam((Out_f)^p) ; Delta', o...:Fam((Out_f)^o) |- e => (R ; O) |> sigma_e
------------------------------------------------------------------------  LET-MAP
Gamma ; Delta |- let l : (d... ; o...) = map f (each w..., y...) (bind z...) in e
    => (R ; O) |> sigma_e . (id_{Obj(Delta')} (x) Fam(sigma_f)@l)
```

`bind` drawn from `Gamma` is spec 11's restriction that `bind` is Pure Data, so reuse across
invocations cannot touch Object identity.

`E` non-empty is spec 17: with no `each` source the traversal length `L` is not determined.

`zip: equal` is not expressed here. Spec 27 rule 92 states that v0 requires no static inference
of lengths, so it stays a run-time condition (section 15.2).

### 7.6 LET-FOLD

```
In_f = In_f|W (+) In_f|E (+) In_f|B

carry compatibility (spec 16)   every w in W is an output port of f with the same type and phase
bind (spec 11)                  (In_f|B)^o is empty
each (spec 18)                  E is non-empty

Gamma |- z... <= In_f|B
Gamma |- c... <= (In_f|W)^p                      Pure Data carry initials; literals allowed
Gamma |- w... <= Fam((In_f|E)^p)
Delta = Delta' (+) Delta_c (+) Delta_y
Delta_c <= (In_f|W)^o
Delta_y <= Fam((In_f|E)^o)

m is valid for fold (6.3)
sigma_f satisfies the carry restriction for W (6.4)
d..., o... fresh
Gamma, d...:Mode(m,Out_f)^p ; Delta', o...:Mode(m,Out_f)^o |- e => (R ; O) |> sigma_e
------------------------------------------------------------------------  LET-FOLD
Gamma ; Delta |- let l : (d... ; o...) = fold f (each w..., y...) (carry c..., c_o...) (bind z...) [m] in e
    => (R ; O) |> sigma_e . (id_{Obj(Delta')} (x) Fold(sigma_f, W, m)@l)
```

### 7.7 LET-DOWHILE

```
In_f = In_f|W (+) In_f|B                         no each

carry compatibility as for fold
(In_f|B)^o is empty
(Out_f)^o is a subset of W                       spec 19 requirement 3
p in ports(Out_f) with tau_p = Bool, pi_p = data  the condition output
Gamma |- k <= Int @ run                          spec 19 requirement 5
the value of k is at least 1                     spec 19 requirement 5

Gamma |- z... <= In_f|B
Gamma |- c... <= (In_f|W)^p
Delta = Delta' (+) Delta_c        Delta_c <= (In_f|W)^o
m is valid for dowhile
sigma_f satisfies the carry restriction for W
d..., o..., x fresh
Gamma, d...:Mode(m,Out_f)^p, x : Bool @ data ; Delta', o...:Mode(m,Out_f)^o |- e => (R ; O) |> sigma_e
------------------------------------------------------------------------  LET-DOWHILE
Gamma ; Delta |- let l : (d... ; o... ; x) = dowhile f (carry c..., c_o...) (bind z...) (cond p) (max k) [m] in e
    => (R ; O) |> sigma_e . (id_{Obj(Delta')} (x) DoWhile(sigma_f, W, m)@l)
```

`x` receives `exhausted` (spec 19.3). It is Pure Data, so it does not appear in the skeleton,
and it sits in `Gamma`, so it need not be used. It is a third component rather than part of
`d...` because it carries no output mode; keeping it outside `d...` makes that structural.

**The phase condition on `k` is what Proposition 7b rests on.** At `data` the bound would not
be fixed before the run and the shape would not be static.

### 7.8 LET-BRANCH

```
Sigma(f) = (In_a) -> Out_f |> .
Sigma(g) = (In_a) -> Out_g |> .
                                  the argument assignment agrees; the results need not

Gamma |- c <= Bool @ data         a variable, not a literal (spec 2.6.7)
Gamma |- z... <= (In_a)^p
Delta = Delta' (+) Delta_v        Delta_v <= (In_a)^o

m is total on ports(Out_f) union ports(Out_g) and valid for branch (6.3)
Com = { p | m(p) = common }
sigma_f@l = sigma_g@l             spec 20.2

d..., o... fresh
Gamma, d...:Mode(m, (Out_f|Com)^p) ; Delta', o...:Mode(m, (Out_f|Com)^o) |- e => (R ; O) |> sigma_e
------------------------------------------------------------------------  LET-BRANCH
Gamma ; Delta |- let l : (d... ; o...) = branch c (args z..., v...) (then f) (else g) [m] in e
    => (R ; O) |> sigma_e . (id_{Obj(Delta')} (x) ModeS(m, sigma_f)@l)
```

**Agreement is required on `Com` alone, not on the whole result assignment.** The implicit
`else` arm of spec 20 exposes no Data outputs, so a `then` arm with Data outputs would fail a
requirement of full agreement, while spec 20.1 rules 8 and 10 say those outputs are dropped and
the node stands.

Both arms receive the same `l`, which is how node-anchored creation points are realized. Where
both consume and create, the creation points agree and the skeletons can be equal, which spec
20.2 admits.

`Fam` is not applied. A generated port's depth does not rise, in contrast to `LET-MAP`
(Lemma 8).

### 7.9 One shape for four nodes

With `Fam`, `Mode`, and `ModeS`, the four structured rules take one shape.

```
result assignment   Mode(m, Out_f)
result skeleton     ModeS(m, .) of the node's skeleton
```

What differs is how `m` arises and which modes it may use (6.3). `map` has `m` fixed at
`all-collect` and no `outputs` section.

---

## 8. Desugaring

```
D : a valid v0 document  ->  an ofp-core program (Sigma, entry)
D = D5 . D4 . D3 . D2 . D1 . D0
```

`Sigma` is the document's processes read as signatures (4.4). `entry` is the document's `entry`
value, or the process named `main` where `entry` is omitted (spec 10.3). No stage transforms
either; the stages transform the process definitions that `Sigma` is read from.

**The order is prescribed.** The stages do not commute:

```
D2 before D3   synthesizing an implicit else arm needs the argument types resolved
D3 before D4   synthesizing one adds a signature entry
```

### 8.1 Variable names

**Core variables are the body dataflow references of spec 2.6.1, unchanged.**

```
inputs.p     an input port of the composite
l.p          output port p of node l
l.exhausted  the reserved output of a do_while node (spec 19.3)
```

No variables are generated and no references are resolved; a `from` in YAML is a variable
occurrence in core.

**This rests on the reserved-name list of spec 2.4.** The grammar

```
BodyRef ::= "inputs" "." Identifier | Identifier "." Identifier
```

is ambiguous where a node id is `inputs`. Spec 2.4 reserves `inputs` and forbids it as a node
id, so it is not. `outputs`, `self`, and `body` are reserved for the same kind of reason.
**If the reserved list changes, this convention loses its footing.**

### 8.2 D0: imports

Spec 3. `$import` leaves no trace in the expanded document and has no run-time meaning.

### 8.3 D1: erase the annotation layers

```
spec_version, features, scheduling, contracts, description, keys beginning with x-
```

Erasure touches nothing else. `features` restates what the body determines (spec 4.1);
`spec_version` and `description` are metadata; `scheduling` and `contracts` contribute to
neither the skeleton, the types, nor the phases.

**Proposition 4 follows from this stage alone** (section 15.1).

### 8.4 D2: monomorphize

Removes `type_params` and `where`, instantiating each generic process per call site. The
matching relation is spec 8.1's, written as inference rules.

```
                                    b primitive
--------------------------  M-PRIM
match(b, b, theta) = theta

--------------------------------  M-NOMINAL
match(N, N, theta) = theta

match(tau, tau', theta) = theta'
------------------------------------------------  M-ARRAY
match(Array<tau>, Array<tau'>, theta) = theta'

T not in dom(theta)   tau' atomic   domain(tau') = domain(T)
--------------------------------------------------------------  M-VAR-NEW
match(T, tau', theta) = theta[T |-> tau']

theta(T) = tau'
--------------------------------  M-VAR-OLD
match(T, tau', theta) = theta
```

`M-VAR-NEW` requires `tau'` atomic, which is spec 8's rule that a type parameter is never
instantiated with an `Array`. No subtyping, no implicit conversion, no relaxation by trait, no
inference of a common supertype (spec 8.1).

Spec 8.1 also distinguishes a **flexible** parameter (declared by the target) from a **rigid**
one (declared by the enclosing process), and requires that a constraint on a rigid parameter be
declared in the enclosing process's own `where`. That requirement is what makes checking
modular and is used by Proposition 8c.

An instantiated entry is named `f[T1=tau1,...,Tn=taun]`, the type variables in dictionary
order. **The name need not satisfy spec 2.4's identifier grammar**: core names are not YAML
identifiers.

### 8.5 D3: expand defaults

Three expansions, mutually independent.

**object_identity_map.** Where a process declares the marker and omits `objects`, add
`outputs.w: inputs.w` to `objects.map` for every Object-bearing input port `w`, and drop the
marker. A port whose type is an `Array` is included; the resulting `map` is read recursively
over Object slots (spec 14.1), giving `in.w |-> out.w` with kind `id`.

Where `objects` is present, no expansion happens; the marker is then an assertion the written
section must agree with (spec 15), which core does not check.

**Implicit else arm.** For a `branch` with no `else`, build

```
id_arm(l) : atomic(Delta_a) -> O_a  |> sigma_id

  Delta_a  the Object-bearing entries of the node's args
  O_a      output ports of the same names, types, and phases
  sigma_id in.w |-> out.w for each, kind id; C and N empty
```

and add it to `Sigma`. Pure Data args do not reach it, and it has no Data outputs (spec 20).

**Output modes.** Fill in `m` where `outputs` is omitted.

```
map        m = all-collect                                        spec 17

fold       m(p) = carry   for p in W
           m(p) = drop    for p not in W and pure
           a non-carry Object-bearing output requires explicit outputs   spec 18.2

dowhile    m(p) = carry   for p in W
           m(p) = drop    otherwise; non-carry Object-bearing outputs do not occur

branch     m(p) = common  for non-pure
           m(p) = drop    for pure                                spec 20.3
```

Where `outputs` is present it is authoritative and lists every output (spec 18.1 rule 9, 19.1
rule 15, 20.1 rule 1), so `m` is total.

### 8.6 D4: linearize the body

Dependency: `l1 -> l2` when a `from` in a binding or control section (spec 21.0) of `l2` has
the form `l1.p`. Both `branch.condition` and `do_while.max_iterations` are control sections
that may carry a `from`. An `inputs.p` reference creates no dependency; `scheduling` is gone,
so temporal references create none either. The graph is acyclic (spec 10.2, rule 21b).

Choose a topological order and emit the let sequence.

**No canonical order is prescribed.** `D` is a family indexed by the choice, and Lemma 5 shows
the choice does not matter. Fixing one would make `D` a function at the cost of an argument
that the chosen order is canonical, and nothing below needs `D` to be a function.

Node translations follow spec 21.0's section table. A `do_while` node introduces `l.exhausted`
and rewrites downstream references to it.

### 8.7 D5: returns

```
returns: { p1: {from: r1}, ..., pn: {from: rn} }   |->   return(z... ; a... as p...)
```

`RETURN` requires `Delta` to be consumed exactly, so an Object-bearing input port that appears
in no binding and no `returns` entry, and an Object-bearing output port with no `returns`
entry, both fail to type. These are the boundary cases spec 12.2 spells out as a degree rule.

---

## 9. Values and interpretation

### 9.1 Shapes and traversal

An Object-bearing value of depth `d` is a finite tree of depth `d` with identities at the
leaves. Identities are drawn from a countable set.

```
d = 0        sh(v) = a single identity
d >= 1       sh(v) = <sh(v1),...,sh(vL)>       the lengths need not agree at each level
```

`ids(v)` is the set of leaves, `tr(v)` the sequence of leaves left to right. Traversal is
first to last within a collection and outer-first through nesting (spec 14.4.1).

```
sh(v) = <<a, b>, <c>, <d, e, f>>
tr(v) = a, b, c, d, e, f
```

A value is **coherent** when `tr(v)` has no repetition. Coherence is linearity at run time: one
physical Object is not in two places.

### 9.2 What the correspondence kinds mean

```
kappa = id    requires sh(input) = sh(output)
kappa = op    requires tr(input) = tr(output); shapes may differ
```

Equal shapes imply equal traversal sequences, so `id` is a special case of `op`, agreeing with
the composition of kinds in 5.1.

**objects.map preserves shape.** Spec 14.1 says it preserves container structure, element
order, and contained identities, which is `sh(inputs.p) = sh(outputs.q)`.

**array_flatten preserves traversal order.** With input shape `<s1,...,sM>` and
`sk = <t_{k,1},...,t_{k,Lk}>`, the output shape is the concatenation. The `Lk` need not agree;
the traversal sequences are the same either way. **This is the formal ground for `array_flatten`
carrying no precondition** (spec 14.4.2).

**array_unflatten preserves traversal order.** The output shape partitions the input traversal
sequence into consecutive blocks. Which partition is chosen is the implementation's business
(spec 14.4.3), and **the traversal sequence is the same whichever is chosen** -- which is why
the divisor does not belong in `objects.transform`.

Composition: shape-preserving composes to shape-preserving, order-preserving to
order-preserving, and shape-preserving is order-preserving. Hence `id . id = id`, otherwise
`op`.

**Lemma S9.** `S(array_flatten) . S(array_unflatten) = id`. Unflattening then flattening
restores the shape whichever partition was chosen, so the composite preserves shape and the
kind refines to `id`. The rule `id . id = id, otherwise op` is an upper bound; particular
combinations can do better.

### 9.3 Interpretation

An interpretation `I` gives each atomic `f` a partial function.

```
I(f) : (Pure Data arguments, Object arguments, U)  ->  (Pure Data results, Object results, U')
```

`U` is the set of identities used so far. Undefined means the run is stuck.

### 9.4 Faithfulness

`I(f)` is **faithful to** `sigma_f` when, on every input where it is defined:

```
F1  correspondence   for p in dom(mu_f):
                       kappa = id  implies  sh(out.mu_f(p)) = sh(in.p)
                       kappa = op  implies  tr(out.mu_f(p)) = tr(in.p)
F2  consumption      for p in C_f: no identity of ids(in.p) occurs in any output
F3  creation         for q in N_f: ids(out.q) is a subset of U' minus U
F4  monotonicity     U is a subset of U', and U' minus U holds only identities of N_f
F5  coherence        the outputs are coherent
```

**F1 is exactly the element-level meaning of the kinds (9.2).** Faithfulness says the
implementation honours the meaning of what it declared.

A script process has `sigma_f = (empty, empty, empty)` and is faithful vacuously.

This is spec 14.1's trust boundary, written down. It is assumed, never proved.

---

## 10. Operational semantics

Big-step, one node at a time, node execution atomic.

```
<rho_d, rho_o, U> |- e ==> <R, O, U', T>
```

`rho_d` maps Pure Data variables to values, `rho_o` maps Object variables to Object-bearing
values. **Reading an Object variable removes it from `rho_o`**: a read is a consumption. A state
is coherent when distinct variables share no identity and each value is coherent.

`T` is a trace: a sequence of node labels, each carrying the identities that execution consumed
and created. A structured node contributes one entry, the union over its invocations.

### 10.1 Rules

```
E-RETURN     <rho_d, rho_o, U> |- return(z... ; a... as p...)
                 ==> <[[z...]]rho_d, {pi |-> rho_o(ai)}, U, empty>

             Typing gives dom(rho_o) = {a...}, so the environment empties exactly.

E-ATOMIC     apply I(f) to the arguments; bind the results; continue

E-MAP        L is the common traversal length; a mismatch is stuck (15.2)
             invoke f once per element, collect the outputs
             bind values are shared; L = 0 invokes f zero times

E-FOLD       thread the carry through L invocations in order
             L = 0 leaves the carry at its initial value

E-DOWHILE    k = max_iterations, at least 1
             invoke; stop when the condition output is false, setting x := false
             stop when n = k, setting x := true                    spec 19

E-BRANCH     evaluate the condition; run the selected arm
             args are supplied to that arm only, read once
```

### 10.2 Observation

Which identities are created depends on the choice of fresh names, so observation is taken up
to renaming.

```
<R, O> ~ <R', O'>   iff  R = R' and some bijection of identities fixing the initial ones
                         carries O to O'
```

This is the usual treatment for name generation. The observation of a run is the equivalence
class of its final result. **A trace is not observable**; it is an instrument of proof.

---

## 11. The concrete model and the faithfulness triangle

The operational semantics is a model. Call it `M`.

```
objects     assignments
morphisms   partial functions from values to values, with name generation, modulo ~
composition sequential execution
tensor      disjoint union
```

Extracting the flow of identity from a morphism of `M` gives a functor `S_M : M -> Slot`,
defined on those morphisms that act per port (Lemma 0).

Let `S` be the syntactic category of core terms and `[[-]]_I : S -> M` the interpretation
extended to terms.

**Proposition I-1 (faithfulness triangle).** If `I` is faithful at every atomic process, then

```
S_M . [[-]]_I = S
```

*Proof.* Both sides are functors from `S` to `Slot`, and `S` is generated by the atomic
processes. Agreement on generators is `S_M(I(f)) = sigma_f`, which is F1, F2, F3. []

**Corollary I-2 (faithfulness is local).** Faithfulness need only be checked at atomic
processes. Composites and structured nodes inherit it.

This is what licenses a verification effort to stop at the atomic level. It is also the
conceptual content of Proposition 2c: once freeness is established, the induction of section
15.5 collapses to "agree on generators, hence agree".

---

## 12. Structural lemmas

### 12.1 Lemma A: weakening and strengthening

```
(i)  Gamma |- z <= tau @ pi  and  Gamma is a subset of Gamma''   implies  Gamma'' |- z <= tau @ pi
(ii) Gamma, x : tau' @ pi' |- z <= tau @ pi  and  z /= x         implies  Gamma |- z <= tau @ pi
```

*Proof.* Two rules. `ARG-LIT` ignores `Gamma`. `ARG-VAR` is a lookup. []

### 12.2 Lemma B: the LET schema

The five `LET` rules share one shape. For a node `N` at label `l`:

```
Args_d(N)   a requirement on the Pure Data arguments, checked against Gamma
Args_o(N)   a requirement on the Object arguments, of the form Delta_N <= A_N
Res_d(N)    the Pure Data results
Res_o(N)    the Object-bearing results
sigma_N     the skeleton
Side(N)     the remaining side conditions
```

```
Args_d(N) holds under Gamma
Delta = Delta' (+) Delta_N        Args_o(N)
Side(N)
Gamma, d...:Res_d(N) ; Delta', o...:Res_o(N) |- e => (R ; O) |> sigma_e
----------------------------------------------------------------------
Gamma ; Delta |- let l : (d... ; o...) = N in e
    => (R ; O) |> sigma_e . (id_{Obj(Delta')} (x) sigma_N@l)
```

with

```
B1   every component is determined by N -- its signature, its mode assignment, its label --
     and by nothing in Gamma, Delta', or e
B2   Delta' does not occur in sigma_N
```

*Verification.*

| Rule | Args_d | Args_o | Res | sigma_N | Side |
|---|---|---|---|---|---|
| LET-ATOMIC | `(In_f)^p` | `(In_f)^o` | `Out_f` | `sigma_f@l` | - |
| LET-MAP | `In_f\|B`, `Fam((In_f\|E)^p)` | `Fam((In_f\|E)^o)` | `Fam(Out_f)` | `Fam(sigma_f)@l` | `E` non-empty |
| LET-FOLD | as above plus carry initials | `(In_f\|W)^o`, `Fam((In_f\|E)^o)` | `Mode(m,Out_f)` | `Fold(sigma_f,W,m)@l` | carry restriction, `m` valid, `E` non-empty |
| LET-DOWHILE | as above plus `k <= Int @ run` | `(In_f\|W)^o` | as above plus `x` | `DoWhile(sigma_f,W,m)@l` | carry restriction, `m` valid, `(Out_f)^o` in `W` |
| LET-BRANCH | `c <= Bool`, `(In_a)^p` | `(In_a)^o` | `Mode(m, .\|Com)` | `ModeS(m,sigma_f)@l` | `sigma_f@l = sigma_g@l`, `m` valid |

**The phase condition on `max_iterations` sits in `Args_d`.** The phase of `k` is read from
`Gamma`, so it is not determined by `N` alone and cannot be a side condition under B1. Written
`Gamma |- k <= Int @ run` it is an ordinary argument judgement and Lemma A applies to it.

### 12.3 Lemma C1: rules are syntax directed

Exactly one rule applies to each term form and to each argument form. Six term forms, six
rules; `LET-ATOMIC` covers both atomic and composite targets. `ARG-VAR` and `ARG-LIT` are
distinguished by the shape of `z`, and the literal rules by the shape of the value.

### 12.4 Lemma C2: the splitting is unique

In `Delta = Delta' (+) Delta_N`, the set `Delta_N` is the Object variables written as arguments
of the node, so `Delta'` is `Delta` minus it.

With Lemma C1, **a typing derivation is unique when it exists**, which is also why the
algorithm of Proposition 8a needs no backtracking.

### 12.5 Lemma C3: rules are local

Every premise mentions only sub-derivations, the signature entry of the called process, the
mode assignment, the label, and the contexts. No premise refers to the presence or absence of
any other construct.

The two premises worth checking are `sigma_f@l = sigma_g@l`, which reads two signature entries
and a label, and the carry restriction, which reads a skeleton and a set of port names.
`Fold`, `DoWhile`, `Fam`, `Mode`, and `ModeS` depend only on their arguments.

---

## 13. Adequacy

### 13.1 Statements

> **Lemma 1 (totality).** If `doc` satisfies every class A and class B condition, `D(doc)` is
> defined.

> **Lemma 2 (soundness).** If `D(doc)` is defined and types, then `doc` satisfies every class
> A and class B condition.

> **Lemma 3 (completeness).** If `doc` satisfies every class A and class B condition, then
> `D(doc)` is defined and types.

> **Corollary.** If `doc` is valid by the specification, `D(doc)` is defined and types. The
> class C conditions are additional and do not obstruct A or B.

### 13.2 Proof of Lemma 3

Class B conditions are what the stages of `D` require (2.2), so `D(doc)` is defined.

```
D0   spec 3
D1   spec 2.1, 2.3, 4.1, 2.7
D2   spec 8; termination is Lemma 6
D3   spec 15, 20, 18.3, 19.2, 20.3
D4   spec 2.4, 10.2, 12, 21.0, 22, rule 21b
D5   spec 12.3
```

One class B condition is not required by a stage: `spec 10.3`, the entry process. It is
resolved when the program `(Sigma, entry)` is formed rather than by a stage (8).

Class A conditions give the premises of the typing rules. The proof is a case analysis over
the tables of 2.2; representative cases:

**Signatures.** Spec 27 rules 1, 3, 59, 61, 62 make the types well formed; rules 14 and 16
give the Object-bearing and phase constraints; rules 22 and 23 with spec 13.1 make `sigma_f`
complete.

**Arguments.** Spec 11 requires every input port to be bound exactly once across the sections
valid for the node kind, and spec 12.1 and 12.2 give the port degrees. Spec 27 rule 86 gives
the type match, rule 87 the literal conformance.

**Splitting.** Spec 12.2 says every Object-bearing value in a body is referred to exactly once,
which is `Delta = Delta' (+) Delta_N`.

**RETURN.** Spec 12.2's list of what an Object-bearing output port may connect to, together
with spec 13's prohibition of an unknown fate and an unknown provenance, gives exactly the
condition that `Delta` be consumed exactly.

**Structured nodes.** Spec 17-21 give `Side(N)` for each kind, including `E` non-empty
(spec 17, spec 18 requirement 3), the carry restriction (spec 16), and `max_iterations >= 1`
(spec 19 requirement 5).

**branch.** Spec 20.2 gives `sigma_f@l = sigma_g@l`; spec 20.1 gives the validity of `m`.

**Generics.** Spec 27 rules 63-67 and 88 let `D2` succeed. By Lemma G1 the skeleton does not
depend on instantiation, so typing succeeds before and after equally. []

### 13.3 Proof of Lemma 2

That `D(doc)` is defined gives the class B conditions.

For class A the tables are read the other way: each typing premise must be implied by a
specification condition. The correspondence table of 2.2 is that reading.

Two premises are **stronger than the specification**.

```
1. equality by normal form (5.6)
   Spec 20.2 requires identity-equivalence and does not give a procedure.
   Section 13.5 examines whether the two agree.

2. the element-level definition of the kinds (9.2)
   Spec 14.1 describes map in prose. Section 9.2 checks the two agree.
```

**No premise is weaker.** Nothing typeable is invalid by the specification. []

### 13.4 On the exhaustiveness of the tables

The proofs are case analyses, so they are only as good as the enumeration behind them.

```
enumerated   the summary rules of spec 27, each classified
             every occurrence of "validation error", by section
read in full spec 2, 5, 7, 8, 11, 12, 13, 14, 16-22, 24-27
scanned      spec 3 (imports), 9 (contracts), 23 (scheduling), the view half of spec 7
```

The four sections that were only scanned are class B or class C and do not reach the typing
rules. **Every section that can bear on class A was read in full.**

Reading in full rather than searching for the phrase "validation error" matters: spec 5.3
states that an `each` source collection must not also be connected elsewhere or returned, and
spec 2.6.7 fixes the output modes available to a condition output, and neither uses that phrase.

Completeness of the enumeration is not claimed formally.

### 13.5 Lemma 4: the skeleton agrees

> The skeleton the typing derivation computes agrees with the fate and provenance the
> specification assigns.

Spec 12.4 states the skeleton, and sections 5 to 7 above restate it as typing rules, so the
two agree by construction at that level. What has to be checked is that both agree with the
rules that determine fate and provenance in the first place: spec 13, 14, 16, and 17-20.

**Atomic processes.** Section 5.5 against spec 14. The valid input fates and output
provenances of spec 13.1 are exactly the split into `dom(mu)`/`C` and `im(mu)`/`N`. Container
`map` is spec 14.1 read recursively, which is kind `id`; a structural change requires
`transform`, which is kind `op`.

**Composites.** Spec 13 says only "derived from the body graph and `returns`", giving no
algorithm.

> **Lemma 4a (uniqueness of the composition).** Given the body graph and `returns`, the
> skeleton satisfying spec 13 is unique.

*Proof.* By spec 12.1 and 12.2 the Object-bearing part of a body graph is a matching: each
output port has outdegree one, each input port indegree one. The fate of a composite input
port slot is either a node binding or a `returns` entry, and in the first case the node's own
skeleton determines what follows. The graph is acyclic (spec 10.2, rule 21b), so following it
terminates, and at no
step is there a choice. Provenance of an output port slot is likewise fixed by its `returns`
entry. []

Core computes that unique correspondence. That the let order does not matter is Lemma 5.

**Structured nodes.** Spec 17's output shape rule is `Fam`; spec 18.1, 19.1, 20.1's mode rules
are `Mode` and `ModeS`; "ordered by invocation order" is the element-level meaning of `Fam`
(6.1). Spec 16 with the carry restriction is 6.4.

**Inference and instantiation.** Spec 15's inference is `D3`; spec 8.1 requires Object tracking
to be checked after instantiation, and `D2` precedes `D3` and typing accordingly. Lemma G1
says the order does not change the answer.

### 13.6 Where core adds precision

Where the specification states a requirement without a procedure, core supplies one.

```
spec 20.2   identity-equivalence, as equality of normal forms (5.6)
spec 14.1   container semantics, as equality of sh(v) (9.2)
spec 13     derivation along the body, as composition along the let sequence (7)
spec 17     output shape, as Fam (6.1)
```

Section 13.7 checks the first of these against the specification's own wording; sections 9.2
and 13.5 cover the rest.

Three things the specification states are not modelled here.

```
spec 13.2   policy and .view do not transfer across replacement    class C
spec 19     diagnostics for bounded termination                    implementation's discretion
spec 14.1   that an implementation honours its declaration         trust boundary
```

### 13.7 Branch skeleton equality against spec 20.2

Spec 20.2 requires the two arms to have equal skeletons and gives the cases that follow. Core
decides equality by normal form (5.6). The two decide the same set.

| Both arms | core | spec 20.2 |
|---|---|---|
| correspond from the same argument slot | equal | valid |
| both consume and both create | equal, the creation point being the node | valid |
| one corresponds, the other creates | `mu` and `N` differ | invalid |
| correspond from different argument slots | `mu` differs | invalid |
| only one has the Object-bearing output | `N` or `mu` differs in size | invalid |
| use different correspondence kinds | the kind differs | invalid |
| both consume, no Object-bearing output | equal | no common output; nothing to check |
| one applies two transforms, the other none | equal, by Lemma S9 | valid |

The second row is what node-anchored creation points decide (5.7). Whichever arm runs, one new
Object appears at the node's output, so where its identity came from does not depend on the arm.

On the last row. Unflattening and then flattening gives the identity
correspondence (Lemma S9), so an arm that does both and an arm that does nothing have the same
correspondence, and the two are equal. Spec 20.2 speaks of arms deriving an output "through
identity-preserving `map` declarations or the same standard identity-preserving structural
transform relation", and reading *relation* as the slot correspondence rather than as the
syntax written gives the same answer.

The sixth row is unreachable in v0, as spec 20.2 notes: the only `op` correspondence comes from
a transform, both transform kinds change the type of the value, and spec 20.1 rule 4 requires a
common output to have the same type in both arms, so the type check reports first. It is listed
because the set of kinds is open (5.2).

---

## 14. Structural results

### 14.1 Lemma 5: exchange

> Let
>
> ```
> t  = let l1 : (d1 ; o1) = N1 in let l2 : (d2 ; o2) = N2 in e
> t' = let l2 : (d2 ; o2) = N2 in let l1 : (d1 ; o1) = N1 in e
> ```
>
> If `N2` refers to none of `d1, o1` and `N1` to none of `d2, o2` -- call them **independent** --
> and `t` types, then `t'` types with the same result assignment and the same skeleton.

*Proof.* Write the derivation of `t` using Lemma B. From `Delta = Delta' (+) Delta_{N1}` and
`Delta' union o1 = Delta'' (+) Delta_{N2}` with `Delta_{N2}` disjoint from `o1`, we get
`Delta_{N2}` inside `Delta'`. Put `Delta0 = Delta'` minus `Delta_{N2}`; then

```
Delta = Delta0 (+) Delta_{N2} (+) Delta_{N1}
```

Build the derivation of `t'` with `l2` outside. `Args_d(N2)` held under `Gamma, d1`; by
independence it does not mention `d1`, so Lemma A(ii) gives it under `Gamma`. `Args_o(N2)`
constrains only `Delta_{N2}`, unchanged. `Side(N2)` depends only on `N2` (B1). Inside,
`Args_d(N1)` held under `Gamma` and Lemma A(i) gives it under `Gamma, d2`.

The final contexts agree, since a context is a finite map and its order carries no meaning.

For the skeleton, apply the interchange law of `Slot`:

```
(id (x) id_{B1} (x) sigma2) . (id (x) sigma1 (x) id_{A2})
  = id (x) sigma1 (x) sigma2
  = (id (x) sigma1 (x) id_{B2}) . (id (x) id_{A1} (x) sigma2)
```

on `A0 = Obj(Delta0)`, `A1 = Obj(Delta_{N1})`, `A2 = Obj(Delta_{N2})`. Disjointness comes from
the splitting; `B1` and `B2` are disjoint because `o1` and `o2` are results of different nodes,
and both are disjoint from `A0` because they are fresh.

Checking the interchange law on the three components: `mu` by cases on which part an element
lies in, using that `id` is the unit for kind composition; `C` because the image of the first
stage avoids the second's consumptions; `N` because the first stage's creations pass through the
second's identity part.

**The creation points are unchanged.** The elements of `N` carry the labels of `sigma1` and
`sigma2`, which the swap does not touch. **A position-based definition of creation point would
break here**, which is why they are node-anchored (spec 12.4.3). []

**Corollary 5.1.** Any two topological orders of a body give the same result assignment and
skeleton.

*Proof.* Two linear extensions of a partial order are joined by a finite sequence of
transpositions of adjacent incomparable elements: take the first position at which they differ,
find that element in the other order, and note that everything strictly between is incomparable
to it, since it precedes them in one order and follows them in the other. Bubble it into place;
the inversion count strictly decreases. Adjacent and incomparable implies independent. []

**Corollary 5.2.** `D` is well defined up to the choice `D4` makes, which justifies not
prescribing a canonical order (8.6).

### 14.2 Lemma 6: monomorphization terminates

A type parameter is instantiated only with an atomic type (`M-VAR-NEW`), so the possible images
of `theta` lie in the finite set of atomic types the document names. Each process therefore has
finitely many instantiations, and the dependency graph is acyclic (spec 10.2), so the recursion
ends. []

### 14.3 Complete morphisms form a subcategory

```
Lemma S1   id_S is complete
Lemma S2   (x) preserves completeness on disjoint parts
Lemma S3   . preserves completeness
```

*Proof of 2.4.* Injectivity: a composite of partial injections is a partial injection. Domain
cover: for `a` in `A`, either `a` is in `C1`, or `mu1(a) = b`, and then either `b` is in `C2`
(so `a` is in `C`) or `mu2(b)` is defined (so `a` is in `dom(mu)`) -- exactly one. Codomain
cover: symmetric. []

**Corollary S4.** The complete morphisms of `Slot` contain `id_S` and are closed under
composition and tensor.

### 14.4 The auxiliary operations preserve completeness

```
Lemma S5   Fam        ports and correspondence unchanged; only depths rise
Lemma S6   ModeS      drop requires pure, so no Object port ever leaves
Lemma S7   Fold, DoWhile
```

**The side condition on `drop` is what Lemma S6 needs.** Allowing an Object-bearing output to
be dropped would remove a port from both `im(mu)` and `N`, and the codomain cover would fail.
That is the role of spec 18.1 rule 6.

*Proof of 2.8.* Input is `Obj(In_f|W) (+) Fam(Obj(In_f|E))`, output `Obj(Mode(m, Out_f))`.

A carry port is in case (a) or (b) by the carry restriction, exactly one, and each covers its
domain and codomain element once.

An `each` port `e` is in `C_f` or in `dom(mu_f)` by completeness of `sigma_f`. In the second
case `mu_f(in.e) = out.q`; `q` in `W` contradicts the carry restriction, and `m(q) = drop`
would need `pure(tau_q)` while `in.e` is Object-bearing. So `m(q) = collect`.

An output `q` outside `W` is in `N_f` or in `im(mu_f)` by completeness of `sigma_f`, and the
preimage is an `each` port -- a carry port is excluded by the restriction, a `bind` port by
being Pure Data. []

**The carry restriction is what this proof needs.** Without it a carry input could escape to a
non-carry output while the carry output is created independently, and the node's skeleton would
misdescribe what happens.

### 14.5 Lemma 7: loop accounting closes

Write `sigma_b` for `sigma_f` restricted to a carry port `w`, and `sigma_b^(n)` for `n`
iterations.

```
case (a)   sigma_b^(n) = ({in.w |-> out.w}, empty, empty)     for all n >= 0
case (b)   sigma_b^(n) = (empty, {in.w}, {out.w})             for all n >= 1
           sigma_b^(0) = id_S
```

*Proof.* Induction on `n`. In case (b), composing `(empty, {in.w}, {out.w})` with itself gives
`mu = empty`, `C = {in.w} union empty = {in.w}`, `N = {out.w} union empty = {out.w}`. The
identity created by iteration `k` is consumed as the input of iteration `k+1` and vanishes from
the composite. []

**The `n = 0` case is an over-approximation.** With an empty `each`, a `fold` does not invoke
its target, so the carry output is the carry input: nothing is consumed and nothing is created,
while the node's skeleton says otherwise (spec 18.2 against spec 12.4.5).

It is sound: the Object count is unchanged either way, and a consumption paired with a creation
loses and duplicates nothing. It is also conservative in the direction that matters -- a policy
on the carried Object is treated as having ended at the node, so it stops earlier than needed
rather than being applied to an Object it was not meant for.

`do_while` is unaffected: it invokes its target at least once (spec 19), so `n >= 1`.

The `each` and collect sides are exact for every `n`, including zero: element `i` is handled by
invocation `i`, and `Fam` of an empty family is the empty family.

### 14.6 Lemma 8: creation points separate

```
8a (per port)     N is a set of ports; a port is never created twice
8b (per identity) an identity created at l is determined by (port, position of the leaf)
```

*Proof.* 8a: `N` is a set and port names are unique within a process. 8b: identities within a
port sit at distinct leaves of `sh(q)`.

Where several identities share a creation point, a lifting by `Fam` was involved; `Fam` gives
each component a distinct position. `LET-BRANCH` applies no lifting, so it creates one port's
worth, and only one arm runs. []

**Lemma 8b does not say the IR can tell them apart.** Spec 2.6.2 gives no way to address a
collection element. A creation point is a static site, not a run-time identity.

---

## 15. Propositions

### 15.1 Proposition 4: annotations are orthogonal

> With `erase` the removal of `spec_version`, `features`, `scheduling`, `contracts`,
> `description`, and `x-` keys, for a valid `doc`,
>
> ```
> D(doc) = D(erase(doc))
> ```

*Proof.* `D1` removes exactly what `erase` does, so `D1 . erase = D1`, and `D2` onwards read
only `D1`'s output. []

**Validity is preserved in one direction only.** A statically false contract makes a document
invalid (spec 9), and an undefined feature name likewise (spec 4.1); erasing either can make an
invalid document valid. The proposition speaks only about valid documents.

It does not say contracts may be ignored. A contract is a hard condition at run time
(spec 9.3). What is asserted is that the denotation of a successful run does not depend on the
annotations.

The proof is one line because core carries no annotation layer at all (8.3).

### 15.2 Proposition 7a: partiality is local

> Among the constructs of core, only the zip condition of a structured node with more than one
> `each` source is partial.

| Construct | Total | Why |
|---|---|---|
| `RETURN`, literals | yes | reading variables; static checks |
| `LET-ATOMIC` | yes | left to the interpretation |
| `array_flatten` | yes | ragged input accepted; no precondition |
| `array_unflatten` | yes | no precondition; the division is the implementation's |
| `map`, `fold` with one `each` | yes | the length is that source's; zero is allowed |
| `map`, `fold` with several | **no** | the sources must be of equal length |
| `LET-DOWHILE` | yes | no `each`; bounded by `max` |
| `LET-BRANCH` | yes | the condition is `Bool` |

**The zip condition is unlike a condition on a single value.** A condition such as "this
collection is non-empty" constrains the shape of one value, and no amount of type structure
short of tracking lengths removes it. The zip condition instead constrains **the sharing of an
index set** between two collections, which is the domain of the laxator (1.2).

So a version that tracks the index in the type -- `Array<T> @ s`, with the several `each`
sources required to carry the same `s` -- **absorbs the condition into type checking and the
partiality disappears**. The one remaining partiality is precisely the one such a version would
remove.

Spec 27 rule 92 confirms it cannot be removed otherwise: v0 requires no static inference of
lengths, so an implementation may establish a mismatch at run or data phase and report it there.

### 15.3 Proposition 7b: the control skeleton terminates

> If every invocation of an atomic process terminates, a core program terminates.

*Proof.* The dependency graph is acyclic (spec 10.2), so induct on height. An atomic process is
one invocation. A composite has finitely many `let`s, and each contributes finitely many: `map`
and `fold` invoke `L` times with `L` the length of a finite collection, `do_while` at most `k`
times with `k` fixed before the run, `branch` once, an ordinary node once. []

**The phase condition on `max_iterations` is what the bound rests on.** At `data` phase `k`
would not be fixed before the run.

Data computation sits outside: `python_script_processes` may run arbitrary Python. The claim is
relative -- the control skeleton terminates; the leaves are a black box -- matching the
treatment of `Sigma`'s interpretation throughout.

### 15.4 Proposition 2a and 2b: conservation

> **2a.** If `Gamma ; Delta |- e => (R ; O) |> sigma` then `sigma` is complete.
>
> **2b.** For a complete skeleton,
>
> ```
> |ids(output)| = |ids(input)| - |consumed| + |created|
> ```
>
> and the induced map on identities is a partial injection.

*Proof of 2a.* Induction on the derivation.

`RETURN`: `rho` is a total bijection with `C` and `N` empty. **That the rule consumes `Delta`
exactly is what the base case needs**; leaving a remainder would put an element of `Obj(Delta)`
in neither `dom(mu)` nor `C`.

`LET`: by Lemma B, `sigma = sigma_e . (id (x) sigma_N@l)`. The induction hypothesis gives
`sigma_e`; `sigma_N` is complete by

```
LET-ATOMIC (atomic)     required of the signature
LET-ATOMIC (composite)  the induction hypothesis on its own derivation
LET-MAP                 Lemma S5
LET-FOLD, LET-DOWHILE   Lemma S7
LET-BRANCH              Lemma S6
```

`id` is complete (2.2), the parts are disjoint by the splitting and freshness, so 2.3 and 2.4
apply. Acyclicity makes the induction well founded. []

For `LET-BRANCH`, restricting to `Com` loses nothing: `m` is valid only if every Object-bearing
port is `common` (6.3), so no Object-bearing output is dropped.

*Proof of 2b.* Every correspondence induces a bijection on the identities it relates
(Lemma S8, below), so `|ids(p)| = |ids(mu(p))|`. Completeness splits `Obj(Out)` into `im(mu)`
and `N`, and `Obj(In)` into `dom(mu)` and `C`.

```
|ids(out)| = sum over im(mu) + sum over N
           = sum over dom(mu) + |created|
           = |ids(in)| - |consumed| + |created|
```

No duplication, since `mu` is injective and Lemma S8 gives bijections within a port. No
disappearance, since every input port is in `dom(mu)` or `C`, and membership of `C` is a
declared consumption. []

> **Lemma S8.** If `mu(p) = q` then `ids(p)` and `ids(q)` are in bijection.

*Proof.* By 9.2, `tr(p) = tr(q)` whatever the kind. Equal sequences give a bijection by
position. []

**Both kinds inducing bijections is what makes conservation work**, and it holds because no v0
correspondence splits a port (Lemma 0).

### 15.5 Proposition 2c: conservation on traces

> If `I` is faithful and `<rho_d, rho_o, U> |- e ==> <R, O, U', T>`, then
>
> ```
> (i)   O is coherent
> (ii)  |Ids(O)| = |Ids(rho_o)| - |cons(T)| + |crea(T)|
> (iii) every identity in cons(T) belonged to a slot some node's skeleton declares consumed
> (iv)  every identity in crea(T) belongs to a slot some node's skeleton declares created
> ```

*Proof.* Induction on the derivation. `E-RETURN` reads variables; coherence of `rho_o` gives
coherence of `O`, and typing gives the exact correspondence. `E-ATOMIC` uses F1 for the counts
(Lemma S8 is a consequence of F1), F2 and F3 for (iii) and (iv), F5 and F3 for coherence.
Structured nodes follow their own semantics; the iteration case is 15.6. `E-BRANCH` reads its
arguments once, and equal skeletons make `C` and `N` the same slots either way. []

**Conceptually this is Proposition I-1** (section 11): conservation is a property of `Slot`
(2b), transported along `S_M . [[-]]_I = S`. The induction is that argument unfolded by hand.
Once freeness is available the proof is one line.

### 15.6 Intermediate identities

A replacing carry over `n` iterations creates `n` identities and consumes `n-1` internally. The
trace records per node (10), so an intermediate identity appears in both sets.

```
cons(l) = { the input identity } union { the n-1 intermediates }
crea(l) = { the n-1 intermediates } union { the output identity }
```

The count still holds, the intermediates cancelling:

```
|Ids(out)| = |Ids(in)| - (1 + (n-1)) + ((n-1) + 1) = |Ids(in)|
```

**(iii) and (iv) are containments, not equalities.** Where a `fold` has an empty `each` and a
replacing carry, the skeleton declares a consumption and a creation that do not occur
(Lemma 7). (iii) reads "an identity that was consumed belonged to a declared slot", which holds
vacuously when nothing is consumed. **The skeleton is an upper bound on what happens**, and the
asymmetry is built into the statement.

### 15.7 Proposition 3: schedule independence

> If `I` is faithful, any two topological orders of a body give observations equal up to
> renaming.

**A schedule is a choice of topological order.** Core terms are already sequenced, so the
freedom a scheduler has is the freedom `D4` has.

> **Lemma E1 (exchange for evaluation).** For `t` and `t'` related as in Lemma 5, evaluation
> from a coherent state gives `<R, O> ~ <R', O'>`, and `T'` is `T` with two entries
> transposed.

*Proof.* The environment splits as `rho_0 (+) rho_{N2} (+) rho_{N1}` along the splitting of
`Delta`.

**Object reads are disjoint.** `N1` reads only `rho_{N1}` and `N2` only `rho_{N2}`, so neither
alters what the other reads.

**Data is read-only.** `rho_d` is never consumed, and by independence neither node reads the
other's results, so the arguments are the same either way.

Hence `I(N1)` and `I(N2)` receive the same arguments in both orders and return the same results.
Only the choice of fresh names differs; F3 and F4 make both choices "not used so far", and a
bijection fixing the initial identities relates them. The final environments agree, an
environment being a finite map. []

*Proof of Proposition 3.* Corollary 5.1 joins any two orders by transpositions of independent
adjacent elements; Lemma E1 applies to each; `~` is transitive. []

> **Lemma E2.** The `L` invocations of a `map` may run in any order.

*Proof.* Invocation `i` reads position `i` of each `each` source, and a coherent value has
distinct identities at distinct positions, so the invocations read disjoint identities. `bind`
is Pure Data. []

**This does not extend to `fold` or `do_while`**: the carry creates a dependency between
iterations. That is the run-time face of the observation that a shared physical resource forces
sequencing.

**On concurrency.** Node execution was assumed atomic. Under that assumption, running
independent nodes concurrently agrees with some sequential interleaving, and Lemma E1 makes
all interleavings agree. The assumption is defensible because v0 observes only final results;
a device's motion is not instantaneous, but no intermediate state is observed.

**On the tension with physics.** Proposition 3 is about denotation. Physically, time matters:
samples degrade, devices are occupied. `scheduling` describes that gap, and Proposition 3 is
the statement with policies set aside. Proposition 4 is what makes setting them aside legitimate.

### 15.8 Proposition 6: features are conservative

For a feature set `F`, let `Doc(F)` be the documents whose derived features lie in `F`. Feature
derivation is syntactic (spec 4.1), so `Doc(F)` is a syntactic class.

> For `doc` in `Doc(F)`, validity under the `F` fragment and validity under all of v0 agree, and
> the resulting type and skeleton agree.

*Proof.* By Lemma C1 the derivation of `D(doc)` uses only rules for constructs occurring in it,
so no rule outside `F` is ever used. By Lemma C3 the premises of the rules used refer to no
other construct. Removing or adding rules outside `F` therefore leaves the derivation intact,
and by Lemma C2 it is unique, so type and skeleton agree.

The global conditions are feature-independent: node id uniqueness (spec 2.4, rule 10a),
acyclicity of both dependency graphs (spec 10.2, rule 21b), `entry` (spec 10.3), reference
resolution (spec 2.6.8), imports (spec 3). The
`features` section itself (spec 4.1) is checked within `F`, since `feat(doc)` is a subset of
`F`. []

> **Corollary 6a (partial implementations are sound).** An implementation supporting only `F`
> decides `Doc(F)` exactly as a complete one does.

**This is what the feature mechanism is worth.** Spec 4 aims at letting an implementation tell
whether it supports what a document needs; Proposition 6 says more -- within its range it is
indistinguishable from a complete implementation.

### 15.9 The scope of conservativity

Changes to the language run on two axes.

```
feature axis   adding or removing constructs within one revision   must be conservative
version axis   changing the rules of a construct                   need not be
```

**Conservativity is required on the feature axis only.** Removing a construct, or changing what
an existing one means, is not conservative by nature, and belongs on the version axis.

A feature that breaks Lemma C3 breaks Proposition 6. Four ways it could:

```
a global side condition, such as "at most one X per document"
relaxing an existing rule in the presence of the feature
changing a default value
giving meaning to a key previously ignored, such as an x- key promoted
```

**A guideline follows.** A new feature adds the rules for its own constructs and changes no
existing rule, default, or global condition. A change that cannot meet this is a version change,
not a feature -- the guideline decides which axis a change belongs on, it does not forbid the
change.

**A refinement the guideline needs.** `generic_processes` looks like a violation: spec 11.1 defines binding type
match by the matching relation of spec 8.1, which the feature introduces. The specification
notes that the relation reduces to identity of type expressions when no type parameter is
involved, so a document not using the feature is judged as before. Hence:

> Where a feature generalizes a relation an existing rule refers to, conservativity is kept if
> the generalization reduces to the original relation in the feature's absence.

The seven features of spec 4.2 all satisfy the guideline, so Proposition 6 holds unconditionally.

### 15.10 Proposition 8: cost

> **8a.** Type checking, skeleton computation, and skeleton equality are linear in the size of
> the core term. Equality needs the ports ordered, so `O(n log n)` unless the declaration order
> is taken as canonical.
>
> **8b.** Monomorphization can enlarge a program by a factor of `m^k`, with `m` the number of
> atomic types and `k` the greatest number of type parameters on one process.
>
> **8c.** Deciding validity does not require materializing the monomorphized program, and is
> linear in the size of the document.

*Proof of 8a.* A skeleton is per port (Lemma 0), so its size is the number of Object-bearing
ports. Composition walks `dom(mu1)` once. A composite composes once per `let`, and intermediate
skeletons are no larger than the body's port count. Type checking is a lookup per argument, a
splitting per node, and the side conditions. Equality compares two ordered lists (5.6). Both
arms of a `branch` carry the same port set, so the declaration order matches. []

*Proof of 8b.* With `k` parameters and `m` atomic types there are at most `m^k`
instantiations, and a chain of generic processes each calling the next realizes the bound. []

*Proof of 8c.* **A skeleton does not depend on instantiation.**

```
whether a port is Object-bearing   from the type parameter's declared domain (spec 8)
the depth of a port                from the Array nesting of the type expression
objects.map, transform             paths, syntactic
consume, create                    likewise
transform role typing              checkable symbolically
```

> **Lemma G1.** For a generic `f` and any instantiation `theta`, `sigma_{f[theta]} = sigma_f`
> as port-and-depth data.

So Object tracking completeness, the inference of `object_identity_map`, transform validity, and
branch skeleton equality are all checked once, before instantiation. What needs an instantiation
is type argument inference and the `where` check, both local to a call site, and there are
linearly many. []

**The `where` check is modular because spec 8.1 says so.** Where a flexible parameter is
inferred to a rigid one, the constraint is discharged from what the enclosing process declares.
Without that rule the check would need the outer `theta` and 8c would fail.

**Producing `D(doc)` and deciding validity are different problems.** Lemmas 2 and 3 concern the
typability of `D`'s image, not the cost of computing it.

---

## 16. Freeness and its limit

### 16.1 What Sigma and Free_OFP(Sigma) are

```
Sigma            the declarations of the atomic processes
                 extracted from YAML: ports and the objects section
                 no Python, no device driver

I                the implementations
                 outside the language; trusted to honour Sigma

Free_OFP(Sigma)  every IR graph buildable from those declarations
                 by composition, tensor, and the four structured node kinds
```

`Sigma` is a declaration, not an implementation. `I` is the implementation.

The free object needs the right structure. **It is not `Free_LNL(Sigma)`**: core is first order,
with no exponential and no closure, and the structured nodes are not generated by composition
and tensor. What is needed is

```
1. a symmetric monoidal category whose Pure Data objects carry a commutative comonoid
   structure; no closure, no exponential
2. Array_I, the I-fold monoidal power: cartesian on the Pure Data side, tensor on the
   Object side (1.2)
3. Array as a semigroupal functor from (FinOrd, x): the multiplication only, no unit,
   no symmetry (5.5)
4. map, fold, do_while, branch as operations
5. Sigma as generators
```

### 16.2 The two halves

```
1a (soundness)     morphisms equal in Free_OFP(Sigma) give the same observation under every
                   faithful interpretation

1b (completeness)  morphisms giving the same observation under every faithful interpretation
                   are equal in Free_OFP(Sigma)
```

In implementation terms:

```
1a   equal graphs behave alike     a rewrite is justified without inspecting any driver
1b   alike graphs are equal        two protocols can be proved equivalent by calculation
```

**1a holds**, directly from the universal property: `[[-]]_I` is determined by its action on
generators. Proposition 3 is an instance.

### 16.3 1b does not hold

**Because `Array<T>` is not an initial algebra, fold fusion is not in the equational theory.**

```
h . fold alpha = fold beta        for h a homomorphism
```

is true semantically and not derivable. So there are pairs of morphisms that `Free_OFP(Sigma)`
distinguishes and observation does not, and 1b fails.

An example in v0 syntax:

```yaml
# 1: measure each, then total
- id: m
  kind: map
  process: measure_one
  each:
    plate:
      from: inputs.plates
- id: s
  process: sum_values
  bind:
    values:
      from: m.value

# 2: total while measuring
- id: f
  kind: fold
  process: measure_and_add
  carry:
    total:
      value: 0.0
  each:
    plate:
      from: inputs.plates
```

The two differ in whether an intermediate collection is built and agree in what they produce.
**The equational theory of v0 does not identify them**, so an optimizer rewriting the first into
the second cannot be justified from the theory.

Quotienting by observational equivalence would make 1b hold trivially and say nothing. A
meaningful statement needs a recognizable finite axiomatization proved to coincide with
observation, and no candidate is at hand.

### 16.4 What the limit costs

**Nothing at present.** A v0 scheduler chooses an execution order and does not rewrite the graph
(spec 23). The limit bites where these are wanted:

```
optimizing the IR by rewriting        eliminating intermediate collections, fusing folds
proving two protocols equivalent      "this revision produces what the original did"
detecting duplicates by normal form   identifying protocols that mean the same thing
```

None is a goal of v0, so 1b's failure agrees with the design rather than contradicting it.

### 16.5 Where the limit comes from

```
Array<T> is not an initial algebra
  no nil as a constructor (an empty literal derives one, but not the constructor)
  no total case
  hence fold has no universal property
  hence fusion is not a theorem
```

This follows from reading `Array` as a family rather than as a list (1.2). A family has no
constructors to induct over, so the absence of an initial algebra structure is what that
reading amounts to, not an oversight.

Adding an `array_case` node would make it an initial algebra and give fusion **as a
meta-theorem**. Stating the universal property *inside the language* needs higher order: without
processes as values there is no way to quantify over algebras. So fusion belongs at the
meta-level, and the justification of a rewrite belongs outside the language.

### 16.6 What a later version might do

The size-index direction removes the partiality of `zip` (15.2), admits `transpose` by adding a
symmetry as a generator (5.5), and lets index correspondence be tracked across composite
boundaries. **None of these bears on the equational theory**: fold fusion has nothing to do with
indices.

The likeliest place a completeness result could be had is the skeleton level -- externalize the
Data computation and ask whether the equational theory of `Slot` is complete. Equality there is
decided by normal form (5.6), so the question is well posed. It would be completeness about the
flow of Objects, not about protocols.

---

## 17. Open problems

```
1  the exhaustiveness of the classification tables (13.4)
   the enumeration is the summary rules, every "validation error", and a full reading of
   every section bearing on class A; formal completeness is not claimed

2  Proposition 5, stage erasure
   phase is a side condition here and not a structure of terms, so partial evaluation is not
   directly definable; separating terms by phase would triple the rules for a goal outside
   the present scope

3  observational equivalence including scheduling (15.7)
   Proposition 3 sets policies aside; an equivalence modulo policy satisfaction is where
   the physical side of this language would be stated

4  Proposition 1b (16.3)
   a recognizable axiomatization coinciding with observation

5  layer 2 of the trust boundary (2.4)
   whether an identity oracle -- barcodes, RFID, positional tracking -- belongs in the
   language, and what a static check can mean for a language about physical objects
```

Item 5 is the one that reaches furthest. **That the IR cannot observe physical identity is not a
convenience of formalization but a property of the subject**, and spec 14.1's trust boundary can
be read as its reflection.

---

## References

- P. N. Benton, *A mixed linear and non-linear logic: proofs, terms and models*, CSL 1994.
- A. Barber, *Dual intuitionistic linear logic*, LFCS technical report, 1996.
- P. Wadler, *Linear types can change the world!*, Programming Concepts and Methods, 1990.
- P. Selinger, *A survey of graphical languages for monoidal categories*, New Structures for Physics, 2010.
- J. Meseguer and U. Montanari, *Petri nets are monoids*, Information and Computation, 1990.
- K. Cho and B. Jacobs, *Disintegration and Bayesian inversion via string diagrams*, MSCS, 2019.
- T. Fritz, *A synthetic approach to Markov kernels, conditional independence and theorems on sufficient statistics*, Advances in Mathematics, 2020.
- R. Cockett and S. Lack, *Restriction categories I*, Theoretical Computer Science, 2002.
- J. Gibbons, *APLicative programming with Naperian functors*, ESOP 2017.
- S. Mac Lane, *Categories for the working mathematician*, 2nd edition, chapter VII.
- The Dex language, index types.
