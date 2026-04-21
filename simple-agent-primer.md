# Simple Gall Agent Primer

Smallest useful mental model for building a Gall agent before you add production machinery.

This file is for agents with a small state model, a single desk-served client, or a narrow protocol surface. It intentionally leaves out many patterns from `architecture.md` that matter more for long-lived, compatibility-sensitive apps.

---

## What a Simple Agent Needs

A Gall agent is a door over `bowl:gall` with ten fixed arms, most of which return cards (effects) plus updated agent (with updated state). `dbug` is optional but broadly useful:

```hoon
/-  *my-types
/+  default-agent, dbug
::
=|  state-0
=*  state  -
::
%-  agent:dbug
^-  agent:gall
|_  =bowl:gall
+*  this  .
    def   ~(. (default-agent this %|) bowl)
++  on-init
  ^-  (quip card _this)
  [~ this]
++  on-save
  ^-  vase
  !>(state)
++  on-load
  |=  old=vase
  ^-  (quip card _this)
  =/  old-state=state-0  !<(state-0 old)
  [~ this(state old-state)]
++  on-poke
  |=  [=mark =vase]
  ^-  (quip card _this)
  ?+  mark  (on-poke:def mark vase)
      %my-command
    =+  !<(=command vase)
    ...handle command...
  ==
++  on-peek
  |=  =path
  ^-  (unit (unit cage))
  (on-peek:def path)
++  on-watch
  |=  =path
  ^-  (quip card _this)
  (on-watch:def path)
++  on-agent
  |=  [=wire =sign:agent:gall]
  ^-  (quip card _this)
  (on-agent:def wire sign)
++  on-arvo
  |=  [=wire =sign-arvo]
  ^-  (quip card _this)
  (on-arvo:def wire sign-arvo)
++  on-leave
  |=  =path
  ^-  (quip card _this)
  (on-leave:def path)
++  on-fail
  |=  [=term =tang]
  ^-  (quip card _this)
  (on-fail:def term tang)
--
```

Start here. Only add more structure when the simple shape stops being clear.

---

## The 10 Arms, Split by Practical Importance

### Usually implement yourself

- `on-init`: initialize state or emit setup cards.
- `on-save`: serialize state for upgrades.
- `on-load`: load or migrate saved state.
- `on-poke`: handle commands and requests.

### Implement when your agent needs them

- `on-peek`: expose scry reads.
- `on-watch`: accept subscriptions and optionally send initial facts.
- `on-agent`: handle subscription updates and poke acknowledgements.
- `on-arvo`: handle kernel responses like timers firing or Eyre binding confirmation.

### Usually safe to delegate early

- `on-leave`
- `on-fail`

For many small agents, the default implementations are enough until you have a concrete need.

---

## Minimal State Versioning

Put a version tag in state from the start. It is the most basic form of future-proofing and costs almost nothing up front:

```hoon
+$  state-0  [%0 count=@ud]
+$  state-1  [%1 count=@ud label=@t]
::
=|  state-1
=*  state  -
::
++  on-load
  |=  old=vase
  ^-  (quip card _this)
  =/  any-state  !<($%(state-0 state-1) old)
  =?  any-state  ?=(%0 -.any-state)  [%1 count.any-state 'default']
  ?>  ?=(%1 -.any-state)
  [~ this(state any-state)]
```

This is separate from versioned marks and multiple client-facing scry/subscription paths. Those are about client/peer compatibility and can be skipped when the desk ships the only client in lockstep — see `architecture.md`. State versioning stays useful either way.

---

## A Good First `on-poke`

Handle a single mark, unpack and dispatch immediately, and keep the control flow obvious. Gate on `src.bowl` so only the local ship can drive the agent — this is the most common access check for a simple agent:

```hoon
++  on-poke
  |=  [=mark =vase]
  ^-  (quip card _this)
  ?>  =(our.bowl src.bowl)
  ?+  mark  (on-poke:def mark vase)
      %counter-command
    =+  !<(=counter-command vase)
    ?-  -.counter-command
      %inc  [~ this(count +(count.state))]
      %dec  [~ this(count (dec count.state))]
      %set  [~ this(count value.counter-command)]
    ==
  ==
```

Note the backstep on the one-line `?-` case branches. This is boring, which is good. Boring agents are easier to debug.

---

## A Good First `on-peek`

Expose the smallest useful read surface:

```hoon
++  on-peek
  |=  =path
  ^-  (unit (unit cage))
  ?+  path  [~ ~]
    [%x %count ~]  ``noun+!>(count.state)
  ==
```

Scry result semantics:
- `[~ ~]` — invalid read, reject this path.
- `~` — cannot answer right now but the path could become valid; treated as blocking.
- `[~ ~ cage]` — successful read.

Use `[~ ~]` as the default for unknown paths.

The leading `%x` in the path is the scry *care*. The current API prepends it to the `path` argument directly. `%x` is the general-purpose read and may return any mark. `%u` is an existence check and must return a `%loob`. Other cares are rare in agents.

Do **not** add a `/x/state` endpoint to expose raw agent state. The `dbug` wrapper already provides `/x/dbug/state` for development inspection. A custom state endpoint duplicates that, clutters the scry namespace, and tends to become a liability when state evolves.

---

## A Good First `on-watch`

Accept a path, optionally authorize it, and send an initial fact:

```hoon
++  on-watch
  |=  =path
  ^-  (quip card _this)
  ?>  =(our.bowl src.bowl)
  ?+  path  (on-watch:def path)
      [%count ~]
    :_  this
    :~  [%give %fact ~ %counter-state !>(count.state)]
    ==
  ==
```

To reject an incoming subscription you **must crash inside `on-watch`**. There is no "deny" return value. Typically this is a `?>` assertion on `src.bowl` (as above) or some other access check — falling through to `on-watch:def` on an unknown path also crashes, which is what you want.

If you do not need subscriptions yet, delegate `on-watch` and add it later.

---

## When To Stay Simple

Stay with the simple agent shape when:
- The desk ships the only client and updates in lockstep.
- There is one clear command type and one clear state model.
- You are not translating between multiple protocol versions.
- You can understand the whole event flow without helper wrappers.

Move toward the patterns in `architecture.md` when:
- Local UI actions and remote commands have different trust models.
- Multiple client versions may exist at the same time.
- You need mark/path versioning.
- Cross-cutting concerns like discipline, gossip, or negotiation start obscuring the core app logic.

---

## Suggested Reading Path

1. `fundamentals.md`
2. `syntax.md`
3. this file
4. selected sections of `architecture.md`
5. `patterns.md` once the agent works and needs cleanup
