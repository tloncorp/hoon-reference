# Hoon Fundamentals

Core concepts needed before reading or writing any Hoon code.

---

## The Subject Model

Hoon has no variables, scope, or closures in the traditional sense. Every expression is evaluated against a **subject** — a single noun that is the entire environment. When you write `=/  x  5`, you're not declaring a variable; you're creating a *new subject* with `x` pinned to it, then evaluating the body against that new subject.

This means:
- `=.  field  new-value` replaces a part of the subject, producing a new subject for the body below.
- Arms (`++`) in a core resolve by searching the subject. An arm's body evaluates against the core itself as its subject.
- When you write `this(state new-state)`, you're producing a copy of `this` with the `state` face replaced — not mutating anything.
- There is no mutation. Every "state change" produces a new noun.

### Faces (Names)

Faces are labels attached to parts of a noun. They're how you name things:

```hoon
=/  x=@ud  5                     ::  bind face 'x' to 5
=*  ship  src.bowl               ::  alias face 'ship' to src.bowl
```

The `=` prefix convention auto-names a value with its type's face:

```hoon
|=  =ship                        ::  sample is named 'ship', type is 'ship' (i.e. @p)
|=  =bowl:gall                   ::  sample is named 'bowl', type is 'bowl:gall'
=/  =cage  [%noun !>(42)]        ::  binds face 'cage' to the value
```

This is syntactic sugar: `=ship` means `ship=ship` — a value of type `ship` with face `ship`.

### Type Narrowing Through Branches

After a type test with `?=`, the subject's type is narrowed in the appropriate branch. This is why you can access union-specific fields after a check:

```hoon
=/  val=(unit @ud)  [~ 42]
?~  val  ~                       ::  in this branch, val is known to be ~
u.val                            ::  in this branch, val is known to be [~ u=@ud]
                                 ::  so u.val is valid

?+  -.sign  default
    %fact                        ::  in this branch, sign is known to have cage
  p.cage.sign                    ::  safe to access cage fields
==
```

This is why the cascading guard pattern works — each `?:` / `?~` / `?=` check narrows the type for everything below it.

---

## Formatting Conventions

### Tall Form vs Wide Form

Every rune has two forms. **Tall form** is multi-line with two-space gaps between children. **Wide form** is single-line with parentheses and commas/spaces:

```hoon
::  TALL FORM (standard for most code)
?:  test
  branch-a
branch-b

::  WIDE FORM (for short expressions)
?:(test branch-a branch-b)

::  TALL
=/  x  5
(add x 1)

::  WIDE
=/(x 5 (add x 1))
```

**Use tall form by default.** Use wide form only for very short expressions inside other expressions, such as inside list literals or gate arguments.

### Indentation

Hoon uses **two-space indentation** with a specific "backstep" convention:

```hoon
::  runes that take a fixed number of children: indent children by 2
?:  condition
  true-branch                    ::  2 spaces in
false-branch                     ::  "backstep" — last child at same level as rune

::  this applies recursively
?:  condition-a
  result-a
?:  condition-b
  result-b
result-c                         ::  cascade of checks, each backsteps

::  for switch runes, indent cases by 2 from the rune
?+  mark  (on-poke:def mark vase)
    %pals-command                ::  4 spaces in (2 for case + 2 for body)
  body-of-case                   ::  2 spaces in from case tag
::
    %pals-gesture
  body-of-case
==

::  for cores, arms at same level as core rune
|%
++  arm-one
  body
++  arm-two
  body
--
```

### Vertical Rhythm

Use `::` on its own line to create visual separation between logical sections:

```hoon
++  on-poke
  |=  [=mark =vase]
  ^-  (quip card _this)
  ?+  mark  (on-poke:def mark vase)
    ::  %noun: misc bespoke self-management
    ::
      %noun
    ...
  ::
    ::  %pals-command: local app control
    ::
      %pals-command
    ...
  ==
```

---

## Irregular (Sugar) Syntax

Hoon has extensive syntactic sugar. This is **not optional knowledge** — real code uses these forms constantly. If you don't recognize them, the code is unreadable.

### Cell Construction

```hoon
[a b]                            ::  regular cell
a^b                              ::  same as [a b] (ket-dot irregular)
[a b c]                          ::  right-associative: [a [b c]]
'key'^value                      ::  pair in maps: ['key' value]
```

### Cage / Tagged Noun Shorthand

```hoon
noun+!>(value)                   ::  same as [%noun !>(value)]
%pals-effect+!>(effect)          ::  same as [%pals-effect !>(effect)]
mark+vase                        ::  general pattern: mark+vase = [mark vase]
```

This `a+b` syntax is the irregular form of `[%a b]` when `a` is a term. It's extremely common for cage construction.

### Unit Construction

```hoon
`value                           ::  same as [~ value] — wrap in unit
~                                ::  null unit
`[~ value]`                      ::  `` wraps the whole thing: [~ [~ value]]
```

Backtick is the irregular form of `(some value)`.

### List Construction

```hoon
~[a b c]                         ::  same as [a b c ~] — proper list
[a b c ~]                        ::  same thing, explicit null terminator
:~  a  b  c  ==                  ::  tall form list literal
```

### Arm / Limb Lookups

```hoon
our.bowl                         ::  wing: lookup 'our' inside 'bowl'
p.cage.sign                      ::  chained: p inside cage inside sign
-.sign                           ::  head of sign
+.state                          ::  tail of state
src.bowl                         ::  field access on bowl
=(our src):bowl                  ::  =(our.bowl src.bowl) — :bowl applies to both
[our dap]:bowl                   ::  [our.bowl dap.bowl]
```

The `a:b` syntax means "resolve `a` with `b` as the subject." `=(our src):bowl` is sugar for `=(our.bowl src.bowl)`.

### Gate Calls

```hoon
(gate arg)                       ::  same as %-  gate  arg
(gate arg1 arg2)                 ::  same as %+  gate  arg1  arg2
(add 2 3)                        ::  => 5
```

### Door Arm Calls

```hoon
~(arm door sample)               ::  call arm on door, regular
~(tap by my-map)                 ::  => list of [key val] pairs
~(. door sample)                 ::  initialize door (return the whole door)
```

### Scry

```hoon
.^(type %care /path)             ::  dotket — read from the system namespace
.^(? %gu /(scot %p our)/app/(scot %da now)/$)  ::  check if agent running
```

### Prettyprint in Error Messages

```hoon
~|  [%error-tag >some-expression<]   ::  >< wraps in a "prettyprint" vase
    !!                                ::  the >< makes it print nicely in traces
```

### String Interpolation (Tapes Only)

```hoon
"{(scow %p ship)} says hello"    ::  interpolation in tape (list @tD)
                                 ::  expression must produce a tape
```

Note: interpolation only works in tape literals (`""`), NOT in cord literals (`''`).

### Type Normalization

```hoon
;;(type value)                   ::  clam: coerce value to type (crash if invalid)
;;(@p +.q.vase)                  ::  "I know this is a @p, enforce it"
```

### Misc Sugar

```hoon
state(field new-value)           ::  copy with change — irregular form of
                                 ::  %_(state field new-value)
`type`value                      ::  cast: same as ^-  type  value
&                                ::  true (%.y)
|                                ::  false (%.n)
~&  >>  expression               ::  debug print with priority (> >> >>>)
```
