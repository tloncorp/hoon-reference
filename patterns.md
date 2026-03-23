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

### Pattern: check-then-crash for permissions

```hoon
?>  =(our src):bowl              ::  must be local
?<  =(our src):bowl              ::  must be foreign
```

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

For `on-agent`, cascade on wire first, then sign type:

```hoon
++  on-agent
  |=  [=wire =sign:agent:gall]
  ^-  (quip card _this)
  ?+  wire  (on-agent:def wire sign)    ::  unknown wire -> default
      [%hey ~]
    ?+  -.sign  (on-agent:def wire sign) :: unknown sign -> default
        %poke-ack
      ?~  p.sign  [~ this]              ::  ack: success, no-op
      ((slog u.p.sign) [~ this])        ::  nack: log and continue
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
- ACUR-style message families if there is no real trust-boundary split.
- Versioned mark stacks if the only client updates with the desk.
- Wrapper libraries if plain `on-poke` / `on-agent` logic is still legible.
- Card accumulators if simple `=^` threading is enough.

Default to the simpler shape first. Add the larger patterns when a real compatibility, composition, or maintenance problem appears.

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

---

## Error Handling

Hoon has no exceptions. Errors either crash the computation or are handled structurally.

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

`=/` gives a face (name) to the value: `=/  x=@ud  5`. `=+` pushes an unnamed value onto the subject: `=+  !<(cmd=command vase)`. Use `=/` when you'll refer to the value by name. Use `=+` when the expression itself already has a face (like `!<` which names its output).

### 4. Forgetting that Cells are Right-Associative

`[a b c]` is `[a [b c]]`, not `[[a b] c]`. This matters for pattern matching:

```hoon
::  this matches a 3-element path
?=  [@ @ @ ~]  path              ::  [@ [@ [@ ~]]]

::  NOT this (which would be a different shape)
?=  [[@ @] @ ~]  path
```

### 5. `=.` Only Changes the Subject for the Body Below

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
