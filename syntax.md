# Hoon Syntax Reference

Complete rune, type, and standard library reference.

---

## Desk Structure

A desk (application package) is organized into these directories:

```
desk/
  app/    :: agents (main applications, gall apps)
  sur/    :: structure definitions (shared types)
  lib/    :: libraries (reusable logic)
  mar/    :: marks (serialization/deserialization)
  gen/    :: generators (one-shot CLI computations)
  ted/    :: threads (async multi-step computations)
  tests/  :: test files
```

Configuration files at the desk root:
- `desk.bill` — list of agents to start on install
- `desk.docket-0` — app metadata (title, version, color, etc.)
- `sys.kelvin` — kernel version compatibility

---

## Comments

```hoon
::  this is a comment
::  file-level doc comment at the top of every file describes purpose
::
::    indented doc comments describe usage, scry endpoints, etc.
::
```

Every file starts with `::  name: short description` followed by longer doc comments. Type definitions get inline `::` comments explaining each field.

---

## Imports

```hoon
/-  sur-name                     ::  import /sur/sur-name.hoon
/-  g=groups                     ::  import with alias (access as g)
/-  *pals                        ::  import and expose namespace
/-  g=groups, c=chat, meta       ::  multiple imports
/+  default-agent, dbug, verb    ::  import /lib/ libraries
/+  gc=groups-conv               ::  library with alias
/~  pages  (page:rudder records command)  /app/pals/webui  ::  directory import
/=  create-thread  /ted/group/create      ::  build a file as a mark
/%  m-noun  %noun                ::  warm a mark (pre-compile)
/*  commit  %txt  /commit/txt    ::  read file contents at build time
/$  grab-rumor  %noun  %rumor    ::  build mark conversion gate
```

---

## Atoms (Primitive Types)

```
@t     text (cord)          'hello'
@ta    URL-safe text (knot) ~.hello
@tas   term                 %hello
@p     ship                 ~zod, ~paldev
@da    absolute date        ~2025.3.20
@dr    relative duration    ~h1, ~m30, ~s15
@ud    unsigned decimal     42
@uv    hash                 0v1a2b3c
@ux    hex                  0xdead.beef
@      generic atom
```

---

## Molds (Types)

```hoon
+$  name  type                   ::  named type definition

::  product types (named tuples / structs)
+$  seat
  $:  roles=(set role-id)
      joined=time
  ==

::  tagged unions (discriminated unions / enums)
+$  gesture
  $%  [%hey ~]
      [%bye ~]
  ==

::  value unions (simple enums)
+$  privacy  ?(%public %private %secret)

::  default value for a type
+$  tile
  $~  [%forsale ~]
  $%  [%forsale ~]
      [%pending who=ship]
      [%managed who=ship body]
  ==

::  type aliases
+$  flag     (pair ship term)
+$  card     card:agent:gall
+$  groups   (map flag group)

::  atom-or-cell dispatch
+$  brief  ?(~ @t)              ::  either ~ or a cord

::  parameterized types
(map key val)                    ::  map
(set val)                        ::  set
(jug key val)                    ::  map of key to set of val
(list item)                      ::  list
(unit val)                       ::  optional (~ or [~ u=val])
(pair a b)                       ::  [p=a q=b]
(quip card _this)                ::  [(list card) _this]
```

---

## Cores and Arms

```hoon
|%                               ::  core (namespace of arms)
++  arm-name                     ::  arm definition
  expression
+$  type-name  type              ::  type arm
+*  alias  expression            ::  virtual arm (computed alias)
+|  %section-label               ::  labeled section divider
--                               ::  end core

|_  sample=type                  ::  door (core with state/sample)
++  arm  ...
--

|=  arg=type                     ::  gate (function)
  body

|^  body                         ::  gate with local arms
++  helper  ...
--

|-                               ::  trap (recursion point, re-enter with $)
  ?~  list  result
  $(list t.list)

|*  arg=type                     ::  wet gate (generic/polymorphic)
  body

|~  arg=type                     ::  iron gate (abstract interface)
  body
```

---

## Assignment and Mutation

```hoon
=/  name=type  value             ::  let-binding
  body
=|  name=type                    ::  bind default value of type
  body
=.  field  new-value             ::  mutate a field in subject
  body
=*  alias  expression            ::  create alias (not evaluated until used)
  body
=^  var  state  expression       ::  extract value and updated state
  body                           ::    expression must produce [value new-state]
=?  field  test                  ::  conditional mutation
  new-value                      ::    only mutates if test is true
  body
=+  value                        ::  push value onto subject (unnamed)
  body
=;  name=type                    ::  reverse let-binding (body first, value second)
  body                           ::    useful when value is long or a gate
  value
=,  namespace                    ::  pull names into scope
  body
```

---

## Conditionals and Pattern Matching

```hoon
?:  test  if-true  if-false      ::  if-else
?.  test  if-false  if-true      ::  inverted if (false branch first, preferred
                                 ::    when true branch is short)
?~  val   if-null  if-exists     ::  null check (unit or list)
?^  val   if-cell  if-atom       ::  cell test

?-  expression                   ::  exhaustive switch (must cover all cases)
    %foo  handle-foo
    %bar  handle-bar
==

?+  expression  default          ::  switch with default
    %foo  handle-foo
    %bar  handle-bar
==

?>  test                         ::  assert true (crash if false)
?<  test                         ::  assert false (crash if true)

?=  [type value]                 ::  type test (returns ?)
?&(a b c)                        ::  logical AND
?|(a b c)                        ::  logical OR
```

---

## Function Application

```hoon
%-  gate  arg                    ::  call gate with one arg
%+  gate  arg1  arg2             ::  call gate with two args
%^  gate  arg1  arg2  arg3       ::  call gate with three args
%~  arm  door  arg               ::  call arm on door with arg
%_  door  field  value  ==       ::  mutate door fields
```

---

## Tuple and List Construction

```hoon
:-  head  tail                   ::  cons (2-tuple)
:_  tail  head                   ::  reversed cons (write tail first)
:+  a  b  c                     ::  3-tuple
:^  a  b  c  d                  ::  4-tuple
:*  a  b  c  d  ==              ::  n-tuple
:~  a  b  c  ==                 ::  list literal
```

---

## Casting

```hoon
^-  type  value                  ::  enforce type (cast)
^+  example  value               ::  cast to type of example
^~  expression                   ::  compile-time evaluation
!>  value                        ::  wrap in vase [type value]
!<  type  vase                   ::  extract from vase (crash if wrong type)
```

---

## Vases and Cages

A `vase` is `[type-noun value-noun]` — a dynamically typed value. A `cage` is `[mark vase]` — a typed, serializable value. These are the fundamental units of inter-agent communication.

```hoon
!>(value)                        ::  create vase from value
!<(type vase)                    ::  extract typed value from vase
%noun                            ::  generic mark
mark+!>(value)                   ::  shorthand cage construction
[%pals-command !>(cmd)]          ::  explicit cage construction
```

---

## Wings (Field Access)

```hoon
field.structure                  ::  named field access
-.tagged-union                   ::  head (tag) of a cell
+.cell                           ::  tail of a cell
p.pair  q.pair                   ::  pair fields
i.list  t.list                   ::  list head and tail
u.unit                           ::  unit value (must check for ~ first)
our.bowl  src.bowl  now.bowl     ::  bowl fields
dap.bowl                         ::  current agent name
eny.bowl                         ::  entropy
```

---

## Standard Library Operations

```hoon
::  door initialization and arm calls
~(. core sample)                 ::  initialize door with sample
~(arm core sample)               ::  call arm on initialized door

::  map operations (by)
~(put by map) key val            ::  insert
~(get by map) key                ::  lookup -> (unit val)
~(got by map) key                ::  lookup -> val (crash if missing)
~(gut by map) key default        ::  lookup with default
~(has by map) key                ::  membership test -> ?
~(del by map) key                ::  delete
~(tap by map)                    ::  map -> list of [key val]
~(key by map)                    ::  keys as set
~(val by map)                    ::  values as list
~(gas by *(map k v)) list        ::  build map from list of [k v]
~(wyt by map)                    ::  count entries
~(run by map) gate               ::  transform values
~(rep by map) gate               ::  fold over map
~(dif by map) other-map          ::  difference
~(uni by map) other-map          ::  union
~(int by map) other-map          ::  intersection

::  set operations (in)
~(put in set) val                ::  add
~(has in set) val                ::  membership test
~(del in set) val                ::  delete
~(tap in set)                    ::  set -> list
~(gas in *(set t)) list          ::  build set from list
~(uni in set) other-set          ::  union
~(int in set) other-set          ::  intersection
~(dif in set) other-set          ::  difference
~(wyt in set)                    ::  count
~(run in set) gate               ::  transform elements
~(rep in set) gate               ::  fold over set

::  jug operations (ju) — map of key to (set val)
~(put ju jug) key val            ::  add val to key's set
~(del ju jug) key val            ::  remove val from key's set
~(get ju jug) key                ::  get set for key

::  list operations
(turn list gate)                 ::  map
(skip list gate)                 ::  filter out (remove where gate returns &)
(skim list gate)                 ::  filter in (keep where gate returns &)
(weld list-a list-b)             ::  concatenate
(flop list)                      ::  reverse
(snag idx list)                  ::  index (0-based)
(lent list)                      ::  length
(scag n list)                    ::  take first n
(slag n list)                    ::  drop first n
(snoc list val)                  ::  append to end
(roll list gate)                 ::  fold left
(reel list gate)                 ::  fold right
(murn list gate)                 ::  filter-map (gate returns unit)
(levy list gate)                 ::  all? (every element satisfies gate)
(lien list gate)                 ::  any? (some element satisfies gate)
(sort list comparator)           ::  sort
(join separator list)            ::  join with separator
(zing list-of-lists)             ::  flatten
(spin list state gate)           ::  stateful map

::  unit operations
(need unit)                      ::  unwrap (crash if ~)
(fall unit default)              ::  unwrap with default
(bind unit gate)                 ::  map over unit (~ stays ~)
(some value)                     ::  wrap in unit [~ value]

::  text operations
(trip cord)                      ::  cord -> tape (list @t -> list @tD)
(crip tape)                      ::  tape -> cord
(cass tape)                      ::  lowercase tape
(cuss tape)                      ::  uppercase tape
(cat 3 cord-a cord-b)            ::  concatenate cords
(rap 3 list-of-cords)            ::  concatenate list of cords
(scot %p ship)                   ::  atom -> knot
(slav %p knot)                   ::  knot -> atom (crash if invalid)
(slaw %p knot)                   ::  knot -> (unit atom) (safe parse)
(scow %p ship)                   ::  atom -> tape
```

---

## JSON Encoding/Decoding

```hoon
=,  enjs:format                  ::  pull JSON encoders into scope
%-  pairs                        ::  object from key-value list
  :~  'key1'^s+'string-value'
      'key2'^(numb 42)
      'key3'^b+&                 ::  boolean
      'key4'^a+(turn list (lead %s))  ::  array of strings
  ==
(frond key json-value)           ::  single-key object
(ship ship-value)                ::  ship -> json string

=,  dejs:format                  ::  pull JSON decoders into scope
(of key1+decoder1 key2+decoder2 ~)   ::  tagged union
(ot 'key1'^decoder1 'key2'^decoder2 ~)  ::  object fields
(su parser)                      ::  string parsed by rule
(as decoder)                     ::  array -> set
so                               ::  string -> cord
ni                               ::  number -> @ud
bo                               ::  boolean -> ?
```

---

## Sail (XML/HTML in Hoon)

```hoon
;div
  =style  "color: red;"
  =class  "my-class"
  ;span:"text content"
  ;p: paragraph text
==
```

---

## Scries (Reads from System/Agents)

```hoon
::  basic scry pattern
.^(type %gx /(scot %p our)/agent-name/(scot %da now)/path/noun)

::  care values
%gx    ::  read from agent (on-peek)
%gy    ::  read arch (directory listing) from agent
%gu    ::  check if agent is running (returns ?)

::  through a helper library (preferred)
(scry:io type /gx/agent/path/noun)
```
