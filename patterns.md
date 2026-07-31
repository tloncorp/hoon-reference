# Hoon Patterns & Idioms

How to write idiomatic Hoon: composition patterns, error handling, and common pitfalls. All examples are drawn from real production code.

This file leans heavily toward production defaults for Gall agents. Some patterns here are everyday Hoon technique; others are only worth the complexity in larger agents with state, subscribers, compatibility requirements, or wrapper libraries.

---

## Common Idioms

### Return cards-first with :_

Since agents return `(quip card _this)` = `[(list card) agent]`, and the card list is often built from logic while `this` is simple, use `:_` to write the agent state first:

```hoon
:_  this
:~  [%pass /wire %agent [ship app] %poke cage]
    [%give %fact [/path]~ cage]
==
```

### Conditional with =? for optional mutation

```hoon
=?  receipts  yow                ::  only update receipts if yow is true
  (~(del by receipts) ship)
```

### =* for Aliasing Existing Wings

Use `=*` (tistar) when you just want a shorter name for an existing wing or expression that you will be invoking repeatedly. Unlike `=/`, it does not add a new value to the subject, but it does support an optional type annotation. The expression is re-evaluated each time the alias is used:

```hoon
::  GOOD: alias a deep wing for readability
=*  sender  p.id.u.mkey
?.  =(sender who)  cor

::  BAD: =/ copies and adds a redundant type annotation
=/  sender=ship  p.id.u.mkey
?.  =(sender who)  cor
```

Use `=/` in place of `=*` only when you will be modifying the value but need to track the original, or when you want to freeze the value at bind time for some other reason.

**Audit pass criteria.** When reviewing a freshly written arm, scan every `=/` and decide:

- **Source is a pure wing access (`a.b.c`)? → `=*`** if you reference it 2+ times and the path is verbose enough to obscure the call sites; **inline directly** if used once or if the source is short (roughly 14 characters or less).
- **Source is a computed value** (function call, literal, constructed cell)? → keep `=/`. These aren't aliases; they freeze a result.
- **`=/  =face:type  source`?** → keep. The `=face` form is a face-mold cast, not aliasing.
- **Used zero times?** → delete.

The rule of thumb: `=/` for results, `=*` for aliases, inline for once-and-short. Mixing them up isn't wrong, just noisy — `=/  fid=@ud  id.c-notebook.cmd` adds a redundant `@ud` annotation and a copy where `=*  fid  id.c-notebook.cmd` would do.

### Faces Default to the Type Name

When binding a single value of some type, the default is the `=face:type` form — Hoon auto-creates a face matching the type name, so you don't write the name twice:

```hoon
::  GOOD: =face:type — face is the type's name
=/  =flag:n        [(slav %p ship.pole) `@tas`name.pole]
=/  =notebook:n    [nid title.act [our now now our]:bowl]
=/  =note:n        (~(got by notes.notebook-state) nid)

::  REDUNDANT: writing the face name explicitly when it matches the type
=/  flag=flag:n    [(slav %p ship.pole) `@tas`name.pole]

::  BAD: ad-hoc abbreviation with no semantic motivation
=/  nb=notebook:n  [nid title.act [our now now our]:bowl]
```

The slight stutter in lines like `se-core(flag flag, ...)` (a wing-replace where the new value's face matches the wing being replaced) is acceptable. Wing resolution is right-to-left dot lookup, so a local face named `notebook` doesn't shadow `.notebook` of `notebook-state` in expressions like `notebook.notebook-state` — the dotted path still resolves correctly.

#### When to use a semantic face instead

Reach for a non-default face when the **role** the value plays is more informative than its type — typically when multiple values of the same type coexist in scope and you need to tell them apart. The face describes *which one*, the type tag still carries *what kind*:

```hoon
::  GOOD: two notebooks in scope; face conveys role, type tag stays explicit
=/  old-nb=notebook:n  notebook.notebook-state
=.  notebook.notebook-state  notebook(title 'renamed', updated-by src.bowl)
=/  new-nb=notebook:n  notebook.notebook-state
(diff-notebooks old-nb new-nb)

::  GOOD: source vs destination of a move
=/  src-folder=folder:n  (~(got by folders.notebook-state) from-fid)
=/  dst-folder=folder:n  (~(got by folders.notebook-state) to-fid)

::  GOOD: a placeholder vs the real thing
=/  placeholder-net=net:n  [%sub *@da |]
::  ...later when the snapshot arrives, real-net is the one we save
```

The rule of thumb: if the face would just restate the type, use `=type` and let the type name speak. If the face would carry information the type alone doesn't (`old`, `new`, `src`, `dst`, `placeholder`, `target`), use a semantic name and keep the type tag explicit.

Same logic for shadowing collisions — if a local `flag` would clash with `flag.act` you're already destructuring, pick a face that disambiguates (`book-flag`, `target-flag`) rather than dropping back to a vowel-stripped abbreviation.

### Cast Above Assertions

In a gate arm, the cast (`^+ se-core`, `^- type`) goes immediately after the gate sample, *above* any `?>`/`?<` assertions:

```hoon
::  GOOD: cast first, then narrow
++  se-rename-notebook
  |=  cmd=c-cmd:n
  ^+  se-core                         ::  cast: tells the type system what we produce
  ?>  ?=(%rename -.c-notebook.cmd)    ::  narrow inside that cast's scope
  ?>  (se-is-owner src.bowl)
  ...

::  BAD: assertions before the cast
++  se-rename-notebook
  |=  cmd=c-cmd:n
  ?>  ?=(%rename -.c-notebook.cmd)
  ^+  se-core                         ::  cast comes too late
  ...
```

The cast is a contract about the arm's output. Assertions narrow the *input* types — they should run inside the cast's scope, not before it. This ordering also matches how the rest of the body reads: cast on top, then guards, then logic.

### Tuple Types with `*` for Don't-Care Parts

When binding a value whose type is a tuple but you only access part of it, replace the unused fields with `*`:

```hoon
::  GOOD: only -.net.entry is checked, so the second half is *
=/  entry=[=net:n *]
  (~(got by books) flag)
?:  ?=(%pub -.net.entry)
  ...

::  GOOD: only the notebook-state half is read
=/  entry=[* =notebook-state:n]
  (~(got by books) flag)
=*  title  title.notebook.notebook-state.entry

::  BAD: full type when half is unused
=/  entry=[=net:n =notebook-state:n]
  (~(got by books) flag)
?:  ?=(%pub -.net.entry)              ::  notebook-state.entry never used
  ...
```

`*` (the bunt of any noun) is the hoon idiom for "I don't care about this part". The compiler still type-checks the parts you *do* annotate. This applies inside gate samples too — `|= [f=flag-v9:n [* =notebook-state:n]]` for a turn body that only reads the notebook-state half.

### Same-Subject Cell Collapse `[a b c]:subject`

When constructing a cell whose elements are all wings of the same subject, **and** that cell is the *last* sub-expression of the surrounding form, collapse to `[a b c]:subject`:

```hoon
::  GOOD: gate args are wings of bowl; the cell is the last (only) expression
(slugify [title id]:notebook.notebook-state)

::  GOOD: bowl-tail of the notebook tuple. The cell is the LAST element
::  of the outer tuple, so the splice extends through.
=/  =notebook:n
  [nid title.act [our now now our]:bowl]

::  GOOD: list-item construction — last sub-expression is :notebook-state
`[flag [notebook visibility]:notebook-state]
```

The "last sub-expression" caveat matters because `[a b c]:subject` is structurally `=>(subject [a b c])` — the right-associative noun shape splices the subject into the trailing slot. If the cell isn't last, the splice doesn't apply and you have to write the wings out:

```hoon
::  WRONG to apply collapse here: revision=@ud follows the bowl-cell,
::  so [src now now src]:bowl is NOT the last element of the outer tuple.
::  The note construction must spell every wing:
=/  =note:n
  :*  nid
      id.notebook.notebook-state
      fid
      title.c-notebook.cmd
      ~
      body-md.c-notebook.cmd
      src.bowl  now.bowl  now.bowl  src.bowl    ::  inline; collapse not available
      revision=0
  ==
```

Reach for this when an arm has multiple `[our now now our]:bowl` or `[title id]:foo`-shape constructions. It's micro-optimization — three keystrokes saved per use — but it reduces visual repetition in dense type-construction code.

### Bare Arguments Instead of Literal Cells

Call gates with bare arguments and let the call auto-cons; a literal cell
adds brackets without adding readability.

```hoon
(poke-a %create-notebook 'NB')        ::  not (poke-a [%create-notebook 'NB'])
```

This flattens through nesting, but only on the *trailing* element — the same
right-associativity that makes `[a b c]:subject` work. If the gate takes
`[dap=term f=flag cmd=command]` and `command` is head-tagged, every leaf can
go bare:

```hoon
(poke-a %notebook f %create-note 2 'T' 'B')
::  not (poke-a %notebook f [%create-note 2 'T' 'B'])
```

A nested cell that *isn't* last keeps its brackets — the splice only reaches
the tail.

### Bare Computed Arms for Predicates

When a predicate depends only on state and needs no arguments, write it as a bare arm — no gate:

```hoon
++  has-owner  ?=(^ owner)
++  is-gateway-live  =(status %up)
```

This is cleaner than a gate that takes no sample. Use it for boolean predicates that guard multiple arms:

```hoon
?>  has-owner
```

### Pattern: check-then-crash for permissions

```hoon
?>  =(our src):bowl              ::  must be local
?<  =(our src):bowl              ::  must be foreign
```

### Pattern: `?>` with predicates, not `need` as a guard

When you need to crash if a precondition is unmet but don't use the unwrapped value, use `?>` with a predicate — not `(need my-unit)` with a throwaway binding:

```hoon
::  GOOD: crash if owner not configured
?>  ?=(^ owner)

::  BAD: need crashes on ~, but we throw away the result
=/  owner-guard  (need owner)
```

Reserve `(need my-unit)` for when you actually use the unwrapped value.

### Pattern: murn for filter-map

When you want to both filter and transform:

```hoon
%+  murn  ~(tap by map)
|=  [key val]
^-  (unit result)
?.  condition  ~
`(transform val)
```

### Pattern: weld for combining card lists

```hoon
:(weld cards1 cards2 cards3)     ::  3+ lists
(weld cards1 cards2)             ::  2 lists
```

### Pattern: log and continue

```hoon
~&  [dap.bowl %unexpected wire]  ::  printf-style debug
[~ this]

~?  condition                    ::  conditional printf
  [dap.bowl 'message']
[~ this]

((slog 'error message' tang) [~ this])  ::  structured logging
```

### Anonymous recursion with |-

For loops without naming a gate:

```hoon
|-
?~  items  result
=.  state  (process i.items)
$(items t.items)
```

### Tisgar (=;) for complex values

When the expression being bound is a long gate or complex form, `=;` lets you write the type annotation and body first:

```hoon
=;  out=(quip card _+.state)
  [-.out this(+.state +.out)]
%.  [bowl request state]
%-  (steer:rudder _+.state command)
...
```

---

## Composition Patterns

This section shows how Hoon's primitives combine in practice. Each example is annotated with *why* the author made specific choices.

### When to Name vs When to Inline

**Name a value when** it's used more than once, when naming clarifies intent, or when the expression is complex enough that reading it inline would obscure the surrounding logic.

**Inline when** the expression is short, used once, and the surrounding context makes its purpose obvious.

```hoon
::  NAMED: used twice (in the assertion and the response)
=/  known=?  (~(has by outgoing) ship.cmd)
=;  [yow=? =_outgoing]
  ...
  ?.  yow  ~
  ...
?-  -.cmd
    %meet  :-  !known  ...
    %part  [known (~(del by outgoing) ship.cmd)]
==

::  INLINED: used once, meaning is obvious from context
?>  =(our src):bowl
=+  !<(cmd=command vase)

::  INLINED: short gate, used only as an argument to turn
%+  turn  ~(tap in out)
|=  o=ship
[%pass /hey %agent [o dap.bowl] %poke %pals-gesture !>([%hey ~])]
```

Hoon syntax is approximately tree-shaped and highly structural. Runes have some number of sub-hoons, and a sub-hoon can be any hoon expression. This lets you inline many expressions inside of each other. Idiomatic Hoon leans into this, actively avoiding binding values and computations to names when they can be inlined inside of the only rune/expression that makes use of them.

**Inline conditions in `=?`** when they're used once. Don't pre-bind a flag just to feed it to `=?`:

```hoon
::  GOOD: condition inlined directly
=?  cor  ?&(pending-restart (is-owner-recently-active now.bowl))
  (send-dm 'Your bot is back online. ✅')

::  BAD: unnecessary intermediate binding
=/  should-notify  ?&(pending-restart (is-owner-recently-active now.bowl))
=?  cor  should-notify
  (send-dm 'Your bot is back online. ✅')
```

**Name intermediate card-building values** when constructing a card involves multiple steps:

```hoon
::  YES: each step builds on the last, names clarify the role
=/  =gesture  ?-(-.cmd %meet [%hey ~], %part [%bye ~])
=/  =cage     [%pals-gesture !>(gesture)]
[%pass /[-.gesture] %agent [ship.cmd dap.bowl] %poke cage]

::  NO: don't name trivially obvious single-use values
[%pass /eyre/connect %arvo %e %connect [~ /[dap.bowl]] dap.bowl]
```

### =; and =- for "Result First, Computation Below"

Use `=;` (tishep) when you want to write how you'll use a value *before* the complex expression that produces it. This is extremely common for HTTP handlers, state transformations, and any case where the "what do I do with the result" is simpler than "how do I get the result."

```hoon
::  =; names and types the result, body uses it, THEN the complex value follows
::  Reads as: "given an `out` of this type, do this... here's how to get `out`:"
=;  out=(quip card _+.state)
  [-.out this(+.state +.out)]       ::  simple: destructure and apply
%.  [bowl !<(order:rudder vase) +.state]  ::  complex: the actual computation
%-  (steer:rudder _+.state command)
:^  pages
    (point:rudder /[dap.bowl] & ~(key by pages))
  (fours:rudder +.state)
|=  cmd=command
...
```

Use `=-` (tishep without a name) when you don't need to name the value — you're just reordering for readability:

```hoon
::  =-  puts the value computation below the expression that uses it
::  "apply this to state, here's what 'this' is:"
=-  this(grid -)       ::  use the result
?-  -.action           ::  compute the result
    %buy   ...
    %set   ...
    %giv   ...
==

::  another example: transform then assign
=-  old(- %4, tokes -, avoid (turn avoid.old :(cork trip cass crip)))
%+  roll  fresh.old    ::  complex fold that produces the value
|=  [[w=@da r=@t] =_tokes.old]
...
```

### =^ Chains for State Threading

When a sequence of operations each needs the current state and may update it, chain `=^` calls. Each `=^` destructures a `[value new-state]` pair:

```hoon
::  Simple chain: two operations, combine their cards
=^  cards  state   (some-operation args)
=^  more   state   (another-operation)
[(weld cards more) this]

::  Longer chain from groups agent: each core operation may emit cards
::  and update state. abet collects accumulated cards.
=.  cor  se-abet:(se-c-create:se-core flag create-group.c-groups)
fi-abet:(fi-join:(fi-abed:fi-core flag) ~)

::  Real example: handle gossip + state together
=.  memory        (~(put in memory) hash)
=/  mage=cage     (en-cage:up data.rumor)
=^  cards  inner  (on-agent:og /~/gossip/gossip %fact mage)
=^  caz1   state  (play-cards:up cards)
=^  caz2   state  (jump-rumor:up rumor)
[(weld caz1 caz2) this]
```

### The emit/abet Pattern (Card Accumulation)

For complex agents with many nested cores, instead of threading card lists everywhere, accumulate cards in the subject and flush them at the end:

```hoon
::  the door carries a card accumulator alongside state
|_  [=bowl:gall cards=(list card)]
++  abet  [(flop cards) state]            ::  flush: reverse + return with state
++  cor   .                               ::  self-reference for chaining
++  emit  |=(=card cor(cards [card cards]))  ::  add one card
++  emil  |=(caz=(list card) cor(cards (welp (flop caz) cards)))  ::  add many

::  usage in the agent door: just =^ into abet
++  on-poke
  |=  [=mark =vase]
  ^-  (quip card _this)
  =^  cards  state
    abet:(poke:cor mark vase)     ::  poke does emit/emil internally, abet flushes
  [cards this]
```

This lets helper arms just call `(emit card)` instead of threading card lists through every return.

This is a good pattern when you already have nested helper doors or wrappers. It is overkill for small agents where plain `(quip card _state)` threading stays readable.

Note: abed-abet is a specific instance of a more generic pattern. Old base desk code (dojo, clay) applies this pattern for state/change accumulation on specific parts of state. That kind of factoring lets you capture logic relating to specific parts of state in dedicated "engines" which maintain all the invariants, and — because they all produce the modified engine core — can be chained very easily.

### Per-Entity Engines (abed/abet)

When state contains a map of entities (groups, channels, notebooks, threads) and many operations are *scoped to one entity*, factor each entity's logic into a sub-core that loads-by-id, mutates, and writes-back. The pattern has two named arms, with n number of operation arms:

- `++ se-abed` — load: take the entity id, look it up in state, return a sub-core with that entity's data hoisted into the immediate subject
- entity-mutating arms — `++ se-rename`, `++ se-update-note`, etc. — operate on the loaded subject, accumulating cards
- `++ se-abet` — write back: stash the (possibly mutated) entity into state, hand control back to the parent core

This collapses the "look up by flag → mutate → write back" boilerplate that would otherwise repeat at every call site:

```hoon
::  the sub-core: a door over [identifier, entity-data, accumulator]
++  se-core
  |_  [=flag =net =notebook-state gone=_|]
  ++  se-core  .
  ++  emit  |=(=card se-core(cor cor(cards [card cards])))   ::  accumulate via parent
  ::
  ++  se-abed                     ::  load: flag -> populated se-core
    |=  f=flag
    ^+  se-core
    ?>  =(ship.f our.bowl)        ::  host-side assertion
    ?~  entry=(~(get by books) f)  ~|(not-found+f !!)
    se-core(flag f, net net.u.entry, notebook-state notebook-state.u.entry)
  ::
  ++  se-abet                     ::  write back: persist + return parent core
    ^+  cor
    ?:  gone                                              ::  marked deleted
      cor(books (~(del by books) flag))
    cor(books (~(put by books) flag [net notebook-state]))
  ::
  ++  se-rename                   ::  example mutating arm
    |=  title=@t
    ^+  se-core
    =.  title.notebook.notebook-state  title
    (se-update [%updated notebook.notebook-state])        ::  fat update
  --

::  call site: the chain reads top-to-bottom
::  (load flag) -> (apply mutation) -> (write back)
=.  cor  se-abet:(se-rename:(se-abed:se-core flag) title)
```

Three things this pattern unlocks:

1. **Top-level dispatch arms collapse.** Instead of `+peek` having seven arms — one per resource — that each parse the flag, look it up, check permissions, and encode JSON, `+peek` becomes a single delegate to `++ no-peek` inside the sub-core, and the sub-core handles all per-notebook reads with the entity already loaded.
2. **Permission and structural checks live in one place.** `se-abed` enforces "this is host-side" once; downstream arms don't re-check. A parallel `no-core` (subscriber-side) does the same for `?=(%sub -.net)`-only operations — though typically `no-abed` accepts any net and the `%sub` guard moves into the specific arms that need it, so peek/watch can use the same loaded core regardless of host/subscriber mode.
3. **`abet` makes deletion symmetric.** A `gone=_|` flag on the sub-core lets a delete arm signal "remove this entity" without the call site knowing the difference; `abet` reads the flag and calls `del` instead of `put`.

Use this pattern when state contains `(map id entity)` and at least three or four operations route to that entity.

### Cascading Guards (Early Return)

Hoon doesn't have `return`. Instead, stack conditional checks that handle edge cases first, falling through to the main logic:

```hoon
++  on-poke
  |=  [=mark =vase]
  ^-  (quip card _this)
  ?>  =(our src):bowl                    ::  crash if not local
  =+  !<(cmd=command vase)              ::  extract typed value
  ?:  (~(has in in.cmd) ~.)             ::  guard: illegal input
    ~|  [%illegal-empty-list-name]
    !!
  ?:  =(our.bowl ship.cmd)              ::  guard: self-reference is no-op
    [~ this]
  ::
  ::  main logic follows...
  ::
```

For `on-agent`, cascade on wire first, then sign type. Separate switch cases with `::` on its own line for visual rhythm:

```hoon
++  on-agent
  |=  [=wire =sign:agent:gall]
  ^-  (quip card _this)
  ?+  wire  (on-agent:def wire sign)    ::  unknown wire -> default
      [%hey ~]
    ?+    -.sign  (on-agent:def wire sign)
        %poke-ack
      ?~  p.sign  [~ this]              ::  ack: success, no-op
      ((slog u.p.sign) [~ this])        ::  nack: log and continue
    ::
        %fact
      ?.  ?=(%expected-mark p.cage.sign)  cor
      =+  !<(=update q.cage.sign)
      (handle-update update)
    ::
        %kick
      (emit %pass /path %agent [our.bowl %agent] %watch /path)
    ==
  ==
```

### ?. (Inverted If) for the Short Branch First

When the "true" branch is short (often a no-op or early exit) and the "false" branch is the main logic, use `?.` to put the short branch first:

```hoon
::  ?.  reads as "unless test, do main-logic, otherwise short-thing"
?.  yow  ~                               ::  unless yow, return empty
%+  weld  (update-widget bowl +.state)   ::  otherwise, build cards
^-  (list card)
:~  ...  ==

::  common in on-agent for "if not a fact, delegate"
?.  ?=(%fact -.sign)
  (on-agent:def wire sign)
::  handle fact...

::  contrast with ?: which puts the true branch first
?:  =(~ in.cmd)                          ::  if empty set
  [known (~(del by outgoing) ship.cmd)]  ::  do removal
::  otherwise, do the more complex thing...
```

### ?- and ?+ for Dispatch

Use `?-` (exhaustive switch) when you must handle every case. Use `?+` (switch with default) when most cases are handled the same way.

```hoon
::  ?- for tagged unions: every tag must be handled
?-  -.gesture
  %hey  :-  !has  (~(put in incoming) ship)
  %bye  :-   has  (~(del in incoming) ship)
==

::  ?+ for marks or wires: most are delegated to default
?+  mark  (on-poke:def mark vase)        ::  default: pass to default-agent
    %pals-command   ...                   ::  handle specific marks
    %pals-gesture   ...
    %handle-http-request  ...
==

::  ?+ with !! default: crash on anything unexpected
?+  wire  ~|(%unexpected-wire !!)
    [%grid ~]    ...
    [%action *]  ...
==
```

### =? for Conditional State Updates

When you might or might not need to update a value:

```hoon
::  only clear receipts if we're actually sending something new
=?  receipts  yow
  (~(del by receipts) ship.cmd)

::  only track guests if not localhost
=?  guests  !=(.127.0.0.1 address.inbound-request)
  (~(put ju guests) name.session address.inbound-request)

::  chained: migrate through versions conditionally
=?  old  ?=(%0 -.old)  (state-0-to-1 old)
=?  old  ?=(%1 -.old)  (state-1-to-2 old)
?>  ?=(%2 -.old)
```

### %_ for Multi-Field Mutation

When you need to update multiple fields at once, `%_` is cleaner than nested `=.`:

```hoon
::  update three fields of state at once
%_  state
  streams  (~(del in streams) source)
  viewers  (~(del by viewers) source)
==

::  update state inline (equivalent to multiple =. calls)
:_  %_  this
      incoming  (~(del in incoming) u.who)
      receipts  (~(del by receipts) u.who)
    ==
```

### Building Cards Inside List Literals

You can bind intermediate values inside `:~` list elements. Each `=/` is scoped to its element:

```hoon
:~  =/  =gesture  ?-(-.cmd %meet [%hey ~], %part [%bye ~])
    =/  =cage     [%pals-gesture !>(gesture)]
    [%pass /[-.gesture] %agent [ship.cmd dap.bowl] %poke cage]
  ::
    =/  =effect   ?-(-.cmd %meet [- ship]:cmd, %part [- ship]:cmd)
    =/  =cage     [%pals-effect !>(effect)]
    [%give %fact [/targets]~ cage]
==
```

### Anonymous Loops with |- and $

`|-` creates an anonymous trap (a core with one arm named `$`). Recurse by calling `$` with updated bindings:

```hoon
::  simple iteration
=/  out=(list ship)  ~(tap in targets)
|-
?~  out  receipts                        ::  base case
=.  receipts  (~(del by receipts) i.out) ::  process head
$(out t.out)                             ::  recurse on tail

::  accumulator pattern
=|  out=(list card)                      ::  initialize accumulator
|-
?~  cards  [out state]                   ::  done: return accumulated
=^  caz  state  (play-card i.cards)      ::  process one
$(out (weld out caz), cards t.cards)     ::  recurse with updated acc

::  insert-in-order (not just iterating, but finding position)
|-  ^+  fresh
?~  fresh  [rumor ~]
?:  (gte when.rumor when.i.fresh)
  [rumor fresh]
[i.fresh $(fresh t.fresh)]
```

### roll for Stateful Folds

When iterating with an accumulator that isn't just a list, use `roll`:

```hoon
::  fold over spots, accumulating both a diff-grid and the full grid
%+  roll  spol
|=  [=spot new=^grid =_grid]
?>  &((lte x.spot size) (lte y.spot size))
=/  =tile  [%pending ship.action]
:-  (~(put by new) spot tile)
(~(put by grid) spot [%pending ship.action])
```

### Helpers Below Agent with =<

Use `=<` to place helper arms below the agent door. The agent can call into the helper core; the helper core has access to the full subject including state:

```hoon
::  =< means "evaluate the first thing in the context of the second"
::  so the agent door sees the helper core below it
=<
|_  =bowl:gall                           ::  agent door (above)
+*  this  .
    def   ~(. (default-agent this %|) bowl)
++  on-init  ...
++  on-poke  ...
--
::                                       ::  helper core (below)
|%
++  update-widget
  |=  [=bowl:gall records]
  ^-  (list card)
  ...
--
```

For simple apps, the helper core is just `|%`/`--`. For complex apps (like groups), it's a door with its own state (the abet/emit pattern).

### The +* Alias Pattern

`+*` creates computed aliases evaluated fresh on each arm entry. Use it for cores you'll call repeatedly:

```hoon
::  in a simple agent:
+*  this  .                              ::  self-reference
    def   ~(. (default-agent this %|) bowl)  ::  default handler

::  in a wrapper library:
+*  this    .
    og      ~(. inner bowl)              ::  inner agent
    up      ~(. helper bowl state)       ::  wrapper helper
    pals    ~(. lp bowl)                 ::  pals library

::  in a helper core:
+*  state   +<+                          ::  the state portion of the sample
    pals    ~(. lp bowl)                 ::  pals scry helper
```

### Self-Poke for Code Reuse

When HTTP handler or admin logic should follow the same path as a typed poke, recurse into `on-poke`:

```hoon
::  HTTP handler reuses the poke handler
|=  cmd=command
=^  caz  this
  (on-poke %pals-command !>(cmd))
['Processed successfully.' caz +.state]

::  %noun handler redirects to the typed mark handler
?+  q.vase  $(mark %pals-command)        ::  re-enter on-poke with correct mark
    %resend  ...                          ::  handle special noun commands
==
```

### Deduplicate Three-Repeat Idioms Behind a Helper Arm

Once a small **stateless** transform — a URL parse, a permission check on an explicit argument, a JSON envelope shape — appears three or more times in the same file, name it. The classic case is a one-liner like `++ strip-query`, which drops the query string from a URL tape and is called from every URL-matching branch in an HTTP handler:

```hoon
::  GOOD: a one-line helper, defined once
++  strip-query
  |=  url=tape
  ^-  tape
  =/  qi=(unit @ud)  (find "?" url)
  ?~  qi  url
  (scag u.qi url)

::  call sites — three branches in the same +serve-http arm:
=/  url-path=tape  (strip-query (trip url.request.inbound-request))
...
=/  pub-path=tape  (strip-query (slag 11 url-tape))   ::  /notes/pub/...
...
=/  share-path=tape  (strip-query (slag 13 url-tape)) ::  /notes/share/...

::  BAD: open-coding the same find/scag dance at every site
=/  url-tape=tape   (trip url.request.inbound-request)
=/  qi=(unit @ud)   (find "?" url-tape)
=/  url-path=tape   ?~(qi url-tape (scag u.qi url-tape))
...
=/  rest=tape       (slag 11 url-tape)
=/  qi=(unit @ud)   (find "?" rest)
=/  pub-path=tape   ?~(qi rest (scag u.qi rest))
...
```

Good targets: URL/path parsing, permission predicates that take an explicit subject (`++ can-edit |= who=ship`), JSON envelope construction, small format conversions. A one-line helper that eliminates 8 open-coded copies is worth more than declaring the same arm inline 8 times.

Pair this with the inline `?~ name=expr` form so a `(get …)`-style helper's call sites stay a single line:

```hoon
?~  qi=(find "?" url)  url   ::  bind + null-check inline

::  vs the unrolled form
=/  qi=(unit @ud)  (find "?" url)
?~  qi  url
```

#### When restructuring beats a helper arm

The previous example is the right move because `strip-query` is a pure function of its argument — there is no entity it operates on, no `?>` it would do for you, no "next call" it would route to. When the duplication is *shaped differently* — every call site loads the same entity by id and then operates on its fields — a helper arm is a half-measure. A `++ get-book` that looks up a notebook is one call site's worth of cleanup, but every call site still has to:

```hoon
?~  entry=(get-book flag)  ``json+!>(~)        ::  null-check
?>  (can-view-flag flag src.bowl)              ::  permission check
=/  fld-list  (turn ~(val by folders.notebook-state.u.entry) ...)  ::  reach through .u.entry
```

The repeated work isn't the lookup — it's the *load entity → check → reach into fields*. A helper arm only collapses the first line. The real fix is to push all of that into a per-entity engine (see "Per-Entity Engines (abed/abet)" above), where `abed` does the load + permission check once and the inner arms see the entity already in subject.

Rule of thumb: if your candidate helper takes only an id (`flag`, `nest`, `note-id`) and every caller follows up by reading the looked-up value's fields, the right answer is probably an `abed`-shaped sub-core, not a helper arm. If the candidate is a pure transform with no entity in sight, a helper arm is the right call.

### `|^` (kelt) for Arm-Scoped Helpers

When a handler accumulates four or five small helpers that nobody else needs to call, wrap the arm in `|^` and nest them inside. The helpers become inaccessible from the rest of the core, which is what you want — they were never meant to be public:

```hoon
++  poke
  |=  [=mark =vase]
  ^+  cor
  |^                                   ::  scoped helpers nested below
  ?+  mark  ~|(bad-mark+mark !!)
      %notes-action
    ::  ...dispatch into one of the local handlers below...
  ::
      %notes-command
    ::  ...
  ==
  ::
  ++  join-remote                      ::  nested: only callable from inside +poke
    |=  =flag
    ^+  cor
    ...
  ::
  ++  handle-send-invite
    |=  [=flag who=ship]
    ^+  cor
    ...
  --
```

This is the same `|^` you use inside `on-load` to scope migration arms — same purpose, different host arm. Reach for it when arm-private helpers start cluttering the surrounding core's namespace.

### Helper Arm Design: Args vs State Readers

When a helper always operates on the current state values, don't pass them as arguments — just read from state. When a helper needs a value that may differ from current state, take it as an argument:

```hoon
::  GOOD: always uses current status and lease-until, no args needed
++  give-status-update
  ^+  cor
  (give %fact ~[/v1] %status-update-1 !>(`update`[%status status lease-until]))

::  GOOD: takes explicit arg because it needs the OLD lease before state changes
++  cancel-lease-timer
  |=  lease=(unit @da)
  ^+  cor
  ?~  lease  cor
  (emit %pass /lease-check %arvo %b %rest u.lease)
```

The caller can then do state mutations and pass the old value explicitly:

```hoon
=.  status  %up
=.  cor  (cancel-lease-timer lease-until)  ::  cancel OLD lease
=.  lease-until  `new-lut                  ::  then update
```

### State Mutations Before Side Effects

Order operations so pure state changes come first, then card emissions. This makes the data flow clearer and avoids subtle bugs where a side effect reads stale state:

```hoon
::  GOOD: state first, then effects
=.  status  %up
=.  boot-id  `bid
=.  cor  (cancel-lease-timer lease-until)
=.  lease-until  `lut
=.  cor  (emit %pass /lease-check %arvo %b %wait lut)
give-status-update

::  BAD: interleaving state and effects makes ordering bugs likely
=.  cor  cancel-lease-timer
=.  status  %up
=.  lease-until  `lut
(status-update status lease-until)
```

### Configuration as Constants

Define tunable parameters as arms in the helper core:

```hoon
|_  =bowl:gall
++  identity-duration   ~d7
++  initial-messages    25
++  max-message-length  280
++  heartbeat-timer     ~s30
```

### Checking Agent Availability Before Poking

Before poking an agent that may not be installed, check with a `%gu` scry:

```hoon
?.  .^(? %gu /(scot %p our.bowl)/hark/(scot %da now.bowl)/$)
  ~                                      ::  hark not running, skip
=/  =cage  [%hark-action !>(action)]
[%pass /hark %agent [our.bowl %hark] %poke cage]~
```

### How Much to Put in One Expression

Hoon style favors a vertical, one-operation-per-line flow. However, short expressions can be inlined when the meaning is obvious:

```hoon
::  GOOD: inline when it's a simple one-liner
[~ this]                                 ::  no-op return
``noun+!>(value)                         ::  simple scry response
?~  p.sign  [~ this]                     ::  success is no-op

::  GOOD: break out when there's conditional logic or multiple steps
=+  !<(=rumor q.cage.sign)
?:  (gth when.rumor (add now.bowl ~h1))  ::  guard
  [~ this]
?:  (gth (met 3 what.rumor) 1.024)       ::  guard
  [~ this]
:-  [[%give %fact [/rumors]~ %rumor !>(rumor)] (update-widget bowl what.rumor)]
=-  this(fresh -)                        ::  state update below
...                                      ::  complex insertion logic

::  BAD: too much inlined, hard to follow
:-  [[(invent:gossip %rumor !>([now.bowl (crip (cass (trip +.q.vase)))]))]~ this(avoid [(crip (cass (trip +.q.vase))) avoid])]
```

### Producing Both State Change and Cards Together

A common pattern returns `(quip card _state)` from helper arms, letting the caller thread state:

```hoon
::  helper returns cards and updated state
++  start-stream
  |=  =source
  ^-  (quip card _state)
  ?:  (~(has in streams) source)
    [~ state]                            ::  no-op: already streaming
  :-  [(watch-chat our.bowl source)]~    ::  cards
  state(streams (~(put in streams) source))  ::  new state

::  caller threads the result
=^  cards  state  (start-stream:do +.action)
[cards this]
```

### When Not to Use the Big Patterns

For a small agent, you can usually skip:
- ACUR-style message families if there is no client usage, or no agent-to-agent usage, or those two will necessarily always have the exact same API design.
- Versioned mark stacks if the only client updates with the desk.
- Wrapper libraries if plain `on-poke` / `on-agent` logic is still legible.
- Card accumulators if simple `=^` threading is enough.

Default to the simpler shape first. Add the larger patterns when a real compatibility, composition, or maintenance problem appears, and remain aware of the complexity/portability trade-offs.

### Set Operations as Pipelines

Chain set operations for readable permission/filtering logic:

```hoon
::  "reasonable targets" = all pals minus ourselves, source, and bad peers
=-  (~(dif in (~(del in (~(del in -) our.bowl)) src.bowl)) misses)
?-  tell.manner
  %anybody  (~(uni in (targets:pals ~.)) leeches:pals)
  %targets  (targets:pals ~.)
  %mutuals  (mutuals:pals ~.)
==
```

### Comment Discipline

```hoon
::  top-of-file: name and one-line description
::  face: see your friends

::  section dividers with double-colon whitespace
::
::  host logic
::

::  explain WHY, not what (the code says what)
::  reasonable targets do not include ourselves, whoever
::  caused us to want to (re)send this rumor, or ships that failed
::  to proxy for us before.

::  inline notes for non-obvious details
::NOTE  we could account for this above, but +del:ju is just easier there
::TODO  retry if nack?

::  document scry endpoints and subscription paths in the file header
::      scry endpoints (all %noun marks)
::    x  /                       records     full pals state
::    x  /leeches                (set ship)  foreign one-sided friendships
::    x  /targets(/[list])       (set ship)  local one-sided friendships
```

Comments are lowercase. Don't capitalize sentence starts the way prose
would — `::  mirrors +se-member-join`, not `::  Mirrors +se-member-join`.

When commenting a code segment inside an arm, prefer a *flag comment*: the
comment block sits above the segment and is closed with a bare `::` line
before the code resumes.

```hoon
    ::  single-shot note reference preview: answer from state if we can,
    ::  else proxy one watch to the host and relay its answer in +agent.
    ::
    =/  =flag:n  [(slav %p ship.pole) `@tas`name.pole]
```

No tutorial-style or self-help comments. A comment that restates what a
well-chosen name already says, reassures the reader, or explains the
obvious is noise — delete it. The same goes for shell scripts and other
supporting code: a self-explanatory improvement needs no comment.

```hoon
::  bad: "a snippet is not the full document" is implied by the word
::  $note-preview: trimmed note view — snippet is the leading slice of
::  body-md, not the full document.

::  good
::  $note-preview: trimmed note view
::
::  .snippet is the leading slice of body-md.note
::
```

---

## Error Handling

Hoon has no exceptions. Errors either crash the computation or are handled structurally.

### One Meaning per Error Response

A semantic error answer (a `%denied` fact, a `%not-found` response) must
mean exactly one condition. Never coerce unrelated failures — crashes,
nacks, malformed data — into the nearest semantic answer: it masks real
bugs as clean errors and poisons error semantics system-wide. Give
genuine failures their own generic channel (an `%error` mark or response
that may carry a server-generated message), and keep the semantic answer
reserved for its one condition.

Where the interface already answers in responses — a venter-style poke
result, a single-shot preview fact — a semantically clear error should
get the right error response rather than a crash. That is not a mandate
to replace crashes with error facts everywhere: rejecting a subscription
for lack of permission by crashing is still the architecturally correct
choice in most places, and crashing remains right for programming errors
and untrusted input.

### Crash with Context (~|)

`~|` (sigbar) adds context to a crash. If any expression below it crashes, the context is included in the stack trace:

```hoon
~|  [%illegal-empty-list-name in=-.cmd]
!!                                       ::  crash with that context

~|  [%negotiate %poke-to-mismatching-gill gill]
!!

~|  commit                               ::  attach the commit hash to any crash below
?+  mark  ~|(bad-mark+mark !!)          ::  crash with mark info if unknown
```

### Assertions

```hoon
?>  condition                    ::  crash if false (positive assertion)
?<  condition                    ::  crash if true (negative assertion)
!!                               ::  unconditional crash
```

### Safe Coercion with `soft`

`soft` attempts to coerce a noun to a type, returning a `unit` instead of crashing:

```hoon
?~  meta=((soft ,hops=@ud) meta.rumor)   ::  try to parse meta as hops
  [~ state]                              ::  failed: bail
=*  hops  hops.u.meta                    ::  succeeded: use it
```

### Catching Crashes with `mule`

`mule` runs a computation and catches crashes, returning an `each`:

```hoon
=/  res=(each (quip card _this) tang)
  %-  mule  |.
  (on-poke %million-action !>(action))
?:  ?=(%| -.res)  'invalid action'      ::  crashed: use error message
=^  caz  this  p.res                     ::  succeeded: use result
```

### Logging Errors Without Crashing

```hoon
::  slog: print to terminal and continue
%-  (slog 'pals: failed to notify' u.p.sign)
[~ this]

::  shorthand: gate-call returns the second arg, prints the first
((slog leaf+"failed poke on {(spud wire)}" u.p.sign) [~ this])

::  ~& for debug printing (removed in production)
~&  [dap.bowl %unexpected-wire wire]
[~ this]

::  ~? for conditional debug printing
~?  !accepted.sign-arvo
  [dap.bowl 'eyre bind rejected!' binding.sign-arvo]
[~ this]
```

### The `tang` Type

A `tang` is `(list tank)` — a structured error trace. You'll see it in crash handlers:

```hoon
++  on-fail
  |=  [=term =tang]
  ^-  (quip card _this)
  %-  (slog term tang)                   ::  print the error
  [~ this]
```

---

## Test Conventions (/lib/test-agent)

Monadic agent tests spell `bind:m` at each step — don't alias it away
(`=*  b  bind:m` saves keystrokes but departs from convention and reads
worse):

```hoon
++  test-said-host-answers-member
  %-  eval-mare
  =/  m  (mare ,~)
  ^-  form:m
  ;<  ~  bind:m  init-zod
  ;<  caz=(list card)  bind:m  (do-watch /v0/some/path)
  (ex-cards caz ~)
```

Use `+do-as` for a temporarily different `src.bowl` instead of paired
`set-src` calls — it scopes the change to one step and restores src
afterward:

```hoon
;<  caz=(list card)  bind:m
  ((do-as ~bus) (do-watch (said-watch-path f 3)))
```

Test headers use the same flag-comment style as everything else — never
asciidoc-style banners (`::  ====  test-x  ====`), which are not a hoon
convention no matter how many of them appear in a file you're imitating:

```hoon
::  +test-join-public-accepts: non-member join of a public notebook
::
++  test-join-public-accepts
```

---

## Common Pitfalls

Things to watch out for when writing Hoon:

### 1. Forgetting `==` Terminators

Runes like `?+`, `?-`, `:~`, `:*`, `|%`, `|_` need explicit `==` or `--` terminators. Missing one shifts all subsequent code into the wrong context. `|%`/`|_` cores end with `--`, everything else uses `==`.

### 2. Wrong Indentation in Switch Arms

In `?+` and `?-`, the case tag is indented 4 spaces, and its body is indented 2 from the rune (which happens to be 2 from the tag):

```hoon
::  CORRECT
?+  mark  default
    %foo                         ::  4 spaces
  handle-foo                     ::  2 spaces from rune
::
    %bar
  handle-bar
==

::  WRONG — this is a common mistake
?+  mark  default
  %foo                           ::  only 2 spaces — will confuse the parser
    handle-foo
==
```

### 3. Confusing =/ and =+

`=+` pins a value to the subject. If the value happens to include a face, that comes along with it. `=/` syntactically enforces that you provide either a face, or a type, or both. `=+` is often used for one-liners, including shorthand of some kind for adding a face (such as when extracting a vase to a type with a face). `=/` is often used for storing the result of more complex computations, or simply to explicate a type for improved legibility.

```hoon
::  PREFERRED: =+ with =type face — uses the type's auto-name
=+  !<(=action:v1:gs vase)
?-  -.action  ...

::  PREFERRED: =/ with type annotation
=/  foo=my-type  (~(got by things) some-key)

::  AVOID: =/ without type annotation
=/  act  something
?-  -.act  ...
```

### 4. Forgetting that Cells are Right-Associative

`[a b c]` is `[a [b c]]`, not `[[a b] c]`. This matters for pattern matching:

```hoon
::  this matches a 3-element path
?=  [@ @ @ ~]  path              ::  [@ [@ [@ ~]]]

::  NOT this (which would be a different shape)
?=  [[@ @] @ ~]  path
```

### 5. Wet Stdlib Gates on `?~`-Narrowed Lists

`+rear`, `+snip`, and friends are wet gates specced on `(lest)`. Calling
them with a `?~`-narrowed `lest` sample can fail to mull: their internal
`$(a t.a)` can't nest the list-typed tail back into the lest-typed input
(`have %~, need [i=@c ...]`, with `mull-nice` in the trace).

```hoon
::  bad: mull-fails at the +snip call
?~  keep  ''
=/  last  (rear keep)
$(keep (snip keep))

::  good: plain head/tail walk (flop first to work from the end)
=/  peek  (flop keep)
|-
?~  peek  ''
?.  (glue i.peek)  (crip (tufa (flop peek)))
$(peek t.peek)
```

### 6. `=.` Only Changes the Subject for the Body Below

`=.` doesn't "return" the changed value — it changes the subject for the rest of the expression:

```hoon
::  CORRECT: =. then continue with body that uses changed state
=.  field  new-value
body-that-sees-new-field

::  WRONG mental model: =. as a standalone "set" statement
::  There is no such thing. =. MUST have a body below it.
```

### 6. Paths Need `/` Between Interpolated Segments

```hoon
::  CORRECT
/[dap.bowl]                      ::  interpolate into path
/(scot %p our.bowl)/[app]/(scot %da now.bowl)

::  WRONG — missing / separators
/[dap.bowl][app]
```

### 7. `~` Is Both Null and a Sigil Prefix

`~` alone is null (empty list, empty unit). But `~[a b c]` is a list literal, `~&` is a debug print rune, `~paldev` is a ship name. Context determines meaning.

### 8. Cord vs Tape

A cord (`@t`, written with single quotes: `'hello'`) is an atom. A tape (`(list @t)`, written with double quotes: `"hello"`) is a list of characters. String interpolation (`"{expr}"`) only works in tapes. Most internal Hoon operations prefer cords; tapes are for display and manipulation.

```hoon
'cord'                           ::  @t atom
"tape"                           ::  (list @tD), supports interpolation
(trip 'cord')                    ::  cord -> tape
(crip "tape")                    ::  tape -> cord
```

### 9. `_type` Means "the Type of This Example"

`_this` means "whatever type `this` happens to be." `_state` means "the type of `state`." This is used extensively in return types:

```hoon
^-  (quip card _this)            ::  list of cards + same type as this agent
^-  (quip card _state)           ::  list of cards + same type as state
=_outgoing                       ::  "a value with the same type as outgoing"
```

### 10. `$` Is the Default Arm Name

When you write `|-` (bartis), it creates a core with one arm called `$`. Calling `$` re-enters that arm (recursion). This also applies to unnamed gates — `$` is the gate itself. So `$(x new-x)` means "re-call this gate/trap with `x` changed to `new-x`."
