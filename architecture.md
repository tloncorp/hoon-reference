# Urbit Application Architecture

How to build, structure, and version Gall agents. This file mixes Gall facts with production defaults for large, compatibility-conscious applications.

---

## Agent Architecture

Most patterns in this file are written from the perspective of long-lived agents that may serve clients at different update levels. For a small single-desk app, some of this machinery is optional. For Tlon-style apps, it is the default.

### The 10-Arm Agent Interface

Every Gall agent implements this interface:

```hoon
|_  =bowl:gall
++  on-init   :: -> (quip card _this)     — called once on first install
++  on-save   :: -> vase                  — serialize state for upgrade
++  on-load   :: vase -> (quip card _this) — deserialize/migrate after upgrade
++  on-poke   :: [mark vase] -> (quip card _this) — handle incoming commands
++  on-peek   :: path -> (unit (unit cage))       — handle scry reads
++  on-watch  :: path -> (quip card _this)        — handle subscription requests
++  on-agent  :: [wire sign] -> (quip card _this) — handle sub updates & poke acks
++  on-arvo   :: [wire sign-arvo] -> (quip card _this) — handle kernel responses
++  on-leave  :: path -> (quip card _this)        — handle subscription departures
++  on-fail   :: [term tang] -> (quip card _this) — handle crash reports
--
```

### Standard Agent Boilerplate

```hoon
::  app-name: short description
::
/-  sur-imports
/+  default-agent, dbug, verb
::
|%
+$  state-0  [%0 field1=type1 field2=type2]
+$  card     card:agent:gall
--
::
=|  state-0
=*  state  -
::
%-  agent:dbug                   ::  debugging wrapper (enables :app-name +dbug)
%+  verb  |                      ::  verbosity toggle (| = quiet, & = loud)
^-  agent:gall
::
=<                               ::  helper core goes below, agent above
|_  =bowl:gall
+*  this  .
    def   ~(. (default-agent this %|) bowl)
::
++  on-init   ...
++  on-save   !>(state)
++  on-load   ...
++  on-poke   ...
...
++  on-leave  on-leave:def       ::  delegate unhandled arms to default
++  on-fail   on-fail:def
--
::                               ::  helper core
|%
++  helper-arm  ...
--
```

### State Versioning and Migration

For persistent production agents, default to tagging state with a version number in the head and migrating through each version sequentially in `on-load`:

```hoon
++  on-load
  |=  ole=vase
  |^  ^-  (quip card _this)
      =/  old=state-any  !<(state-any ole)
      =?  old  ?=(%0 -.old)  (state-0-to-1 old)
      =?  old  ?=(%1 -.old)  (state-1-to-2 old)
      ?>  ?=(%2 -.old)
      [~ this(state old)]
  ::
  +$  state-any  $%(state-0 state-1 state-2)
  +$  state-0    [%0 old-field=@t]
  ++  state-0-to-1
    |=  state-0
    ^-  state-1
    [%1 old-field new-field=*default*]
  --
```

### Cards (Effects / Messages)

Cards are the agent's output — instructions to the runtime:

```hoon
::  poke another agent
[%pass /wire %agent [ship agent-name] %poke mark+!>(value)]

::  subscribe to another agent
[%pass /wire %agent [ship agent-name] %watch /subscription/path]

::  unsubscribe
[%pass /wire %agent [ship agent-name] %leave ~]

::  send update to our subscribers
[%give %fact [/path1 /path2]~ mark+!>(value)]

::  send update on the subscription that triggered this handler
[%give %fact ~ mark+!>(value)]

::  kick subscribers off a path
[%give %kick [/path]~ ~]          ::  kick all
[%give %kick [/path]~ `ship]      ::  kick specific ship

::  set a timer
[%pass /wire %arvo %b %wait (add now.bowl ~m30)]

::  cancel a timer
[%pass /wire %arvo %b %rest time-value]

::  bind an eyre (HTTP) route
[%pass /eyre/connect %arvo %e %connect [~ /app-name] %app-name]
```

### Wire Convention

Wires identify the source of a response in `on-agent` and `on-arvo`. Use descriptive path segments:

```hoon
/hey                             ::  simple identifier
/eyre/connect                    ::  system / subsystem
/profile/widget/mutuals          ::  hierarchical context
/~/gossip/gossip/(scot %p ship)  ::  library-namespaced (/~/ prefix)
```

### on-poke Pattern

Dispatch on mark, enforce permissions, extract typed value:

```hoon
++  on-poke
  |=  [=mark =vase]
  ^-  (quip card _this)
  ?+  mark  (on-poke:def mark vase)     ::  default for unknown marks
      %pals-command
    ?>  =(our src):bowl                  ::  assert local poke only
    =+  !<(=command vase)                ::  extract typed value
    ... handle command ...
  ::
      %pals-gesture
    ?<  =(our src):bowl                  ::  assert foreign poke only
    =+  !<(=gesture vase)
    ... handle gesture ...
  ==
```

### on-peek Pattern

Return `(unit (unit cage))` — `~` for "no binding", `[~ ~]` for "path exists but no value", `[~ ~ cage]` for value:

```hoon
++  on-peek
  |=  =path
  ^-  (unit (unit cage))
  ?>  =(our src):bowl                    ::  local only (common)
  ?+  path  [~ ~]
    [%x ~]            ``noun+!>(full-state)
    [%x %leeches ~]   ``noun+!>(leeches)
    [%x %targets @ ~] ...
  ==
```

### on-watch / on-agent Pattern

`on-watch` guards subscription access and optionally gives initial data. `on-agent` handles responses on wires you previously `%pass`ed:

```hoon
++  on-watch
  |=  =path
  ^-  (quip card _this)
  ?>  =(our.bowl src.bowl)               ::  local subscribers only
  ?+  path  (on-watch:def path)
      [%targets ~]
    :_  this
    %+  turn  ~(tap in targets)
    |=(=@p [%give %fact ~ %pals-effect !>(`effect`[%meet p])])
  ==

++  on-agent
  |=  [=wire =sign:agent:gall]
  ^-  (quip card _this)
  ?+  wire  (on-agent:def wire sign)
      [%hey ~]
    ?+  -.sign  (on-agent:def wire sign)
        %poke-ack
      ?~  p.sign  [~ this]              ::  success: no-op
      ((slog u.p.sign) [~ this])        ::  failure: log error
    ==
  ==
```

### =^ State Threading

The `=^` rune is the core pattern for threading state changes through multiple operations:

```hoon
=^  cards  state   (some-operation args)   ::  get cards + new state
=^  more   state   (another-operation)     ::  chain another
[(weld cards more) this]                   ::  combine all cards
```

This is especially important in wrapper libraries that intercept and transform cards.

---

## Structure Design (sur/)

### The ACUR Model (Actions, Commands, Updates, Responses)

This is a production pattern for larger agents. It is useful when local UI actions, remote peer-to-peer commands, canonical updates from a host/source of truth, and frontend responses have meaningfully different requirements. Small agents and agents that exclusively get either client or backend interactions do not need this split.

Large agents separate message types into distinct roles. This is not just naming convention — each category has a different trust model, direction of flow, and serialization strategy:

```hoon
::  a-*  actions: requests from the local client UI
::    originate ONLY locally (from our own frontend).
::    most actions become commands sent to the group host.
::    the client core (go-core) handles these.
::
+$  a-groups
  $%  [%group =flag =a-group]
      [%invite =flag ships=(set ship) =a-invite]
      [%leave =flag]
  ==

::  c-*  commands: requests to the agent AS a group host
::    arrive from remote ships (or locally for hosted groups).
::    checked for permissions on the server side.
::    the server core (se-core) handles these.
::
+$  c-groups
  $%  [%create =create-group]
      [%group =flag =c-group]
      [%ask =flag story=(unit story:s)]
      [%join =flag token=(unit token)]
      [%leave =flag]
  ==

::  u-*  updates: canonical state changes
::    generated by the server in response to valid commands.
::    stored in the group log for replay.
::    propagated to group members (server-to-server).
::
+$  u-group
  $%  [%create =group]
      [%meta =data:meta]
      [%seat ships=(set ship) =u-seat]
      [%channel =nest =u-channel]
      [%delete ~]
  ==

::  r-*  responses: facts sent to local client subscribers
::    generated alongside updates, but specifically for the frontend.
::    sent on versioned subscription paths (/v1/groups, /groups/ui).
::    may be converted to multiple versions for backwards compat.
::
+$  r-groups  [=flag =r-group]
+$  r-group
  $%  [%create =group]
      [%meta meta=data:meta]
      [%entry =r-entry]
      [%seat ships=(set ship) =r-seat]
      [%channel =nest =r-channel]
      [%delete ~]
  ==
::  most r-* subtypes alias to u-* subtypes:
+$  r-ban      u-ban
+$  r-token    u-token
+$  r-seat     u-seat
```

**The data flow:**

```
Client UI --(a-action)--> go-core --(c-command)--> se-core (host)
                                                      |
                                                      v
                                              validate + apply
                                                      |
                                          +-----------+-----------+
                                          |                       |
                                    u-update                 r-response
                                    (to log,                 (to client
                                     to members)              subscribers)
```

The response path in the agent converts to multiple versions simultaneously:

```hoon
++  go-response
  |=  =r-group:g
  ^+  go-core
  ?.  go-is-init  go-core           ::  only send after group is initialized
  ::  v1 response for modern clients
  =/  r-groups-9=r-groups:v9:gv  [flag r-group]
  =.  cor  (give %fact ~[/v1/groups] group-response-1+!>(r-groups-9))
  ::  v0 backcompat for old clients
  =/  diffs-2=(list diff:v2:gv)
    (diff:v2:r-group:v9:gc r-group [seats admissions]:group)
  =.  cor
    %+  roll  diffs-2
    |=  [=diff:v2:gv =_cor]
    =/  action-2=action:v2:gv  [flag now.bowl diff]
    (give:cor %fact ~[/groups/ui] group-action-3+!>(action-2))
  go-core
```

### Type Naming Conventions

- `flag` — group/app identifier `(pair ship term)`
- `nest` — channel identifier `(pair term flag)`
- `dock` — agent address `[ship dude]`
- `cage` — typed data `[mark vase]`
- `wire` — response routing path
- `bowl` — agent execution context

### Documentation Style

Document every type with its purpose. Document fields with trailing `::` comments:

```hoon
::  $seat: group membership
::
::  .roles: the set of roles assigned to a seat
::  .joined: the time a ship has joined
::
+$  seat
  $:  roles=(set role-id)
      joined=time
  ==
```

---

## Mark System (mar/)

Marks define how data is serialized and converted between formats. Every mark is a door:

```hoon
::  pals-command: friend management
::
/-  *pals
::
|_  cmd=command
++  grad  %noun                  ::  revision control granularity
++  grow                         ::  how to convert FROM this mark
  |%
  ++  noun  cmd                  ::  to noun: just the value itself
  --
++  grab                         ::  how to convert TO this mark
  |%
  ++  noun  command              ::  from noun: the type acts as validator
  ++  json                       ::  from JSON (optional)
    ^-  $-(^json command)
    =,  dejs:format
    |^  (of meet+arg part+arg ~)
    ++  arg
      (ot 'ship'^(su fed:ag) 'in'^(as so) ~)
    --
  --
--
```

- `grow` — outbound conversions (this mark -> other marks)
- `grab` — inbound conversions (other marks -> this mark)
- `grad` — revision control strategy (usually `%noun`)
- JSON arms in grow/grab enable HTTP API integration

---

## Versioning Strategy

The codebase maintains backwards compatibility across client versions through a layered versioning system. This is one of the most important architectural patterns for apps with independently updating clients — it lets old clients keep working while the backend evolves.

### Type Versioning (sur/*-ver.hoon)

Use this machinery when clients or peer agents may lag behind the desk that serves the latest state. If the only client is bundled with the desk and always updated in lockstep, client-facing type backcompat may be unnecessary. State migrations and inter-desk compatibility can still justify versioning.

Versioned types live in a `-ver` file (e.g., `groups-ver.hoon`) as nested version cores. Each version imports the previous one with `=,` and selectively overrides types:

```hoon
::  /sur/groups-ver.hoon

++  v10
  =,  v9                        ::  inherit all v9 types
  |%
  +$  progress  ?(%ask %join %watch %done %leave %error)  ::  added %leave
  +$  foreign                    ::  depends on new progress
    $:  invites=(list invite)
        lookup=(unit lookup)
        preview=(unit preview)
        progress=(unit progress)
        token=(unit token)
    ==
  --

++  v9
  =,  v8                        ::  inherit all v8 types
  |%
  +$  group  ...                 ::  modified group type
  +$  admissions  ...            ::  new admissions fields
  --

++  v8
  =,  v7
  |%
  ...
  --

::  ... all the way down to v0
```

**Key principle**: types are referenced as `type-name:vN:ver-file` — e.g., `group:v9:gv` where `gv` is the import alias for `groups-ver`. The agent's internal state uses the latest version:

```hoon
+$  current-state
  $:  %11
      groups=net-groups:v9:gv    ::  canonical internal format
      =channels-index:v7:gv
      =foreigns:v10:gv
  ==
```

### Conversion Libraries (lib/*-conv.hoon)

A separate conversion library provides functions to convert **down** from the current version to older versions:

```hoon
::  /lib/groups-conv.hoon

++  v9                           ::  conversions from v9
  |%
  ++  group
    |%
    ++  v7                       ::  v9 -> v7
      |=  =group:v9:gv
      ^-  group:v7:gv
      %=  group  requests.admissions
        %-  ~(run by requests.admissions.group)
        |=  [at=@da note=(unit story:s)]
        note                     ::  drop the new 'at' field
      ==
    ::
    ++  v5                       ::  v9 -> v5 (chains through v7)
      |=  =group:v9:gv
      ^-  group:v5:gv
      %-  v5:group:^v7           ::  convert v7 -> v5
      (v7:^group group)          ::  first convert v9 -> v7
    ::
    ++  v2                       ::  v9 -> v2 (chains through v7)
      |=  =group:v9:gv
      ^-  group:v2:gv
      %-  v2:group:^v7
      (v7:^group group)
    --
  --
```

**Conversions chain downward**: v9 → v7 → v5 → v2. Each step drops or transforms fields that didn't exist in the older version. These conversions are lossy by design.

### Mark Versioning

Multiple mark files serve different client versions. The naming convention:

```
groups.hoon     ::  "base" mark, historically serves v2 format
groups-1.hoon   ::  serves v5 format
groups-2.hoon   ::  serves v9 format (current)
```

Each mark file wraps the appropriate version:

```hoon
::  /mar/groups-2.hoon (newest)
/-  gv=groups-ver
|_  =groups:v9:gv               ::  wraps v9 types
++  grad  %noun
++  grow
  |%
  ++  noun  groups
  --
++  grab
  |%
  ++  noun  groups:v9:gv
  --
--
```

### Scry Path Versioning

The agent serves multiple versions on different scry paths. Version routing happens in `on-peek`:

```hoon
[%x ver=?(%v0 %v1 %v2) %groups ~]
  =/  groups-9=groups:v9:gv  (~(run by groups) tail)
  ?-  ver.pole
      %v0  ``groups+!>((~(run by groups-9) v2:group:v9:gc))
      %v1  ``groups-1+!>((~(run by groups-9) v5:group:v9:gc))
      %v2  ``groups-2+!>(groups-9)
  ==
```

- `/x/v0/groups` → converts v9 to v2, returns as `%groups` mark
- `/x/v1/groups` → converts v9 to v5, returns as `%groups-1` mark
- `/x/v2/groups` → returns v9 as-is, as `%groups-2` mark

The discipline library declares all valid scry/fact path-to-mark mappings up front:

```hoon
::  scries
:~  [/x/v0/groups %groups]
    [/x/v1/groups %groups-1]
    [/x/v2/groups %groups-2]
==
::  facts (subscription paths)
:~  [/v1/groups %group-response-1 ~]
    [/groups/ui %group-action-3 ~]
==
```

### Subscription Path Versioning

Watch paths follow the same version prefix pattern. Clients subscribe to the version they understand:

```hoon
::  modern client subscribes to /v1/groups
::  legacy client subscribes to /groups/ui

++  go-response
  |=  =r-group:g
  ::  send v1 response for modern clients
  =.  cor  (give %fact ~[/v1/groups] group-response-1+!>(r-groups-9))
  ::  convert and send v0 backcompat for old clients
  =/  diffs-2  (diff:v2:r-group:v9:gc r-group ...)
  =.  cor
    %+  roll  diffs-2
    |=  [=diff:v2:gv =_cor]
    (give:cor %fact ~[/groups/ui] group-action-3+!>(action-2))
  go-core
```

### Version Bump Checklist

When evolving a type in a compatibility-sensitive app:

1. **Add a new version core** in `sur/*-ver.hoon` that inherits the previous version
2. **Add downconversion** in `lib/*-conv.hoon` from the new version to the old
3. **Create a new mark file** (e.g., `groups-3.hoon`) wrapping the new types
4. **Add new scry/watch paths** with the new version prefix (e.g., `/v3/groups`)
5. **Keep old paths working** — they convert from the latest internal format down
6. **Update the discipline declaration** to register the new marks and paths
7. **Bump agent state version** and add a migration in `on-load`

In long-lived production apps, old paths usually do not go away. Old clients keep working because every scry and subscription path continues to serve its version through on-the-fly downconversion. In a lockstep deployment, you may intentionally skip or retire older client-facing paths.

---

## Wrapper Library Pattern

Libraries like `gossip`, `negotiate`, and `discipline` wrap an inner agent to add cross-cutting behavior. They intercept cards and signs, managing their own state alongside the inner agent's. This is a powerful but invasive production pattern for specialized use-cases, not a requirement for ordinary agents:

```hoon
++  agent
  |=  inner=agent:gall
  =|  state-N
  =*  state  -
  ^-  agent:gall
  |_  =bowl:gall
  +*  this  .
      og    ~(. inner bowl)              ::  inner agent access
      up    ~(. helper bowl state)       ::  wrapper helper core
  ::
  ++  on-poke
    |=  [=mark =vase]
    ^-  (quip card _this)
    ::  intercept library-specific marks, pass rest to inner
    ?.  =(%my-mark mark)
      =^  cards  inner  (on-poke:og +<)
      =^  cards  state  (play-cards:up cards)  ::  transform inner's cards
      [cards this]
    ... handle library mark ...
  --
```

Key pattern: `play-cards` processes cards from the inner agent, intercepting watches/pokes/facts to add library behavior (version checking, gossip relay, mark validation).

### Library Wire Namespacing

Wrapper libraries use `/~/library-name/` wire/path prefixes to avoid collisions with the inner agent:

```hoon
/~/gossip/gossip         ::  gossip subscription path
/~/gossip/source         ::  locally-originating data path
/~/gossip/config         ::  configuration path
/~/negotiate/version     ::  version advertisement
/~/negotiate/notify      ::  match/mismatch notifications
```

### Key Wrapper Libraries

- **negotiate** — automatic protocol version negotiation between agents. Captures watches and only establishes subscriptions once protocol versions match.
- **discipline** — mark type enforcement. Validates that agents only emit facts with declared marks on declared paths.
- **gossip** — P2P data sharing through pals network with configurable hop count, proxy relaying, and deduplication.
- **subscriber** — subscription lifecycle management with automatic retry on failure.

---

## Generator Pattern (gen/)

One-shot computations invoked from the dojo CLI:

```hoon
/-  g=groups
:-  %say
|=  $:  [now=@da eny=@uvJ =beak]
        [[ship=^ship name=term ~] ~]    ::  required args
    ==
mark+value
```

---

## Thread Pattern (ted/)

Multi-step async computations using the strand monad:

```hoon
/-  spider
/+  io=strandio
::
=,  strand=strand:spider
^-  thread:spider
|=  arg=vase
=/  m  (strand ,vase)
^-  form:m
=+  !<(arg=(unit input-type) arg)
?>  ?=(^ arg)
::
;<  =bowl:spider  bind:m  get-bowl:io
;<  result=type   bind:m  (async-op args)
;<  ~             bind:m  (poke:io [ship agent] mark+!>(value))
(pure:m !>(result))
```

`;<` is the monadic bind — chain async steps where each step can use results of previous steps.

---

## Web Framework (rudder)

For agents serving web UIs, the rudder library handles HTTP routing:

```hoon
::  in on-poke, handle %handle-http-request:
%.  [bowl !<(order:rudder vase) app-state]
%-  (steer:rudder _app-state cmd-type)
:^  pages                                ::  map of page handlers
    (point:rudder /[dap.bowl] & ~(key by pages))  ::  router
  (fours:rudder app-state)               ::  404 handler
|=  cmd=cmd-type                         ::  command solver
[brief cards new-state]
```

Page handlers implement three arms:
- `build` — render GET requests
- `argue` — parse POST body into a command
- `final` — render response after POST processing

---

## Scry Helper Library Pattern

Wrap scry calls in a library door for clean external access:

```hoon
::  /lib/pals.hoon
|_  bowl:gall
++  mutuals  |=  list=@ta  (s (set ship) %mutuals ?~(list / /[list]))
++  leeche   |=  =ship     (s _| /leeches/(scot %p ship))
::
++  base     ~+  `path`/(scot %p our)/pals/(scot %da now)
++  running  ~+  .^(? %gu (snoc base %$))
++  s
  |*  [=mold =path]  ~+
  ?.  running  *mold
  .^(mold %gx (weld base (snoc `^path`path %noun)))
--
```

`~+` memoizes the result for the current event, avoiding redundant scries.
