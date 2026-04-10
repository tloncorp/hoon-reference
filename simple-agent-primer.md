# Simple Gall Agent Primer

Smallest useful mental model for building a Gall agent before you add production machinery.

This file is for agents with a small state model, a single desk-served client, or a narrow protocol surface. It intentionally leaves out many patterns from `architecture.md` that matter more for long-lived, compatibility-sensitive apps.

---

## What a Simple Agent Needs

At minimum, a Gall agent is a door over `bowl:gall` that returns cards plus updated state:

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
++  on-peek   on-peek:def
++  on-watch  on-watch:def
++  on-agent  on-agent:def
++  on-arvo   on-arvo:def
++  on-leave  on-leave:def
++  on-fail   on-fail:def
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
- `on-arvo`: handle kernel responses like timer or Eyre results.

### Usually safe to delegate early

- `on-leave`
- `on-fail`

For many small agents, the default implementations are enough until you have a concrete need.

---

## Minimal State Versioning

Even for a simple persistent agent, put a version tag in state if you expect upgrades:

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

For lockstep client deployments, you may not need versioned marks and multiple client-facing paths. State migration is still worth doing if you expect to upgrade the desk.

---

## A Good First `on-poke`

Use one typed mark, parse it immediately, and keep the control flow obvious:

```hoon
++  on-poke
  |=  [=mark =vase]
  ^-  (quip card _this)
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

This is boring, which is good. Boring agents are easier to debug.

---

## A Good First `on-peek`

Expose the smallest useful read surface:

```hoon
++  on-peek
  |=  =path
  ^-  (unit (unit cage))
  ?+  path  ~
    [%x %count ~]  ``noun+!>(count.state)
  ==
```

Use `~` for "no such binding". Return a cage only for paths you want to support deliberately.

Do **not** add a `/x/state` endpoint to expose raw agent state. The `dbug` wrapper already provides `/x/dbug/state` for development inspection. A custom state endpoint duplicates that, clutters the scry namespace, and tends to become a liability when state evolves.

---

## A Good First `on-watch`

Accept a path, optionally authorize it, and send an initial fact:

```hoon
++  on-watch
  |=  =path
  ^-  (quip card _this)
  ?+  path  (on-watch:def path)
      [%count ~]
    :_  this
    :~  [%give %fact ~ %counter-state !>(count.state)]
    ==
  ==
```

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
