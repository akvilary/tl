# Add `struct` keyword for ergonomic OOP over records

## Reviewing this PR

Changes were made in **10 files** — everything else is regenerated
build output:

- language & docs: `teal/reader.tl`, `teal/ast.tl`, `teal/block.tl`,
  `teal/types.tl`, `teal/check/{context,relations,visitors}.tl`,
  `teal/gen/lua_generator.tl`, `docs/src/structs.md`,
  `docs/src/SUMMARY.md`
- tests: `spec/lang/declaration/struct_spec.lua`

`teal/*.lua`, `teal.lua` and `tl.lua` are self-build artifacts (Teal
compiles its own `.tl` sources; the compiled output is committed per
repo convention). The blank-line runs there are the generator's
newline preservation: comments and type declarations in `.tl` produce
no code, and the output is padded 1:1 so line numbers stay aligned
between `.tl` and `.lua` (this is what makes `tl run` errors point at
the right source lines). Reviewers can skip those files entirely.

## Summary

This PR introduces a `struct` declaration: a thin layer over `record` that
removes the metatable boilerplate typically needed for object-oriented
Lua, while keeping Teal's compile-time philosophy intact. It is additive
at the language level — no previously-valid `record` program changes
behavior — but it is *built into* record's machinery rather than bolted
on beside it (see Compatibility for exactly what that means).

```lua
local struct Animal
   name: string
   sound: string
end

function Animal:init()
   self.sound = self.sound or "<silence>"
end

function Animal:speak(): string
   return self.name .. " says " .. self.sound
end

local struct Dog:Animal          -- single inheritance
   breed: string
end

function Dog:init()
   self.sound = "Woof"
end

local d = Dog.new { name = "Rex", breed = "Labrador" }
print(d:speak())                 --> Rex says Woof
```

## Motivation

The most common Teal OOP pattern today is hand-written metatable
plumbing, repeated for every type:

```lua
-- what users write today, over and over
function Animal.new(name: string): Animal
   local self: Animal = setmetatable({}, { __index = Animal })
   self.name = name
   return self
end
```

`struct` removes this boilerplate, and builds the pieces the hand-written
pattern cannot express on top of it:

- an auto-generated `.new` whose parameter is a *typed* record of the
  struct's instance fields (unknown keys are type errors, statics and
  methods are excluded);
- an `init` lifecycle hook with a **cascade**: `.new` of a child runs
  every `init` in the hierarchy, root parent first, each exactly once —
  something each hand-rolled constructor would have to re-implement
  (and usually doesn't);
- single inheritance with compile-time method flattening, default-value
  merging (child overrides win) and inheritable static fields.

`record` remains the tool for plain data; `struct` is for when you would
otherwise reach for `setmetatable` by hand.

## Design principle: inheritance is resolved at compile time

The core design decision, and the reason the runtime model looks the
way it does: **everything inheritance-related is paid once per struct at
load time — never per instance, never per method call.** There is no
dispatch anywhere.

### The generated `.new`

```lua
local struct Point
   x: number
   y: number = 0
   distance: number
end

function Point:init()
   self.distance = math.sqrt(self.x ^ 2 + self.y ^ 2)
end
```

compiles to:

```lua
local Point = {}
Point.__index = Point
Point.new = function(opts)
   local self = setmetatable({}, Point)
   for k, v in pairs(opts) do self[k] = v end
   if opts.y == nil then self.y = 0 end    -- one `if` per defaulted field
   Point.init(self)                        -- unconditional direct call
   return self
end
```

- The `opts` parameter is typed as a record of exactly the struct's
  instance fields — static fields and methods are excluded, so passing
  an unknown key is a type error.
- Defaults are applied with explicit `== nil` checks (falsy-safe:
  `false`/`0` work correctly), one per defaulted field, in declaration
  order. Child defaults override parent defaults; the merge happens at
  check time, so a constructor never contains duplicated parent
  assignments.
- If the struct declares no `init`, the call is omitted entirely — no
  `if X.init then` guard. The checker knows the answer at compile time.

### The `init` chain

`init` is a lifecycle hook, not a method: it is never flattened, never
inherited as a field. `.new` of a child contains a fixed list of direct
calls, root parent first, containing **only ancestors that declare
their own `init`**:

```lua
-- C extends B extends A; B declares no init
C.new = function(opts)
   local self = setmetatable({}, C)
   for k, v in pairs(opts) do self[k] = v end
   A.init(self)          -- B is absent from the chain: no cost, no guard
   C.init(self)
   return self
end
```

Every `init` runs exactly once, parent to child. Calling a parent hook
from user code is by name: `A.init(self)`.

### Method flattening

At the point a child struct is declared, every method known on its
parent is copied into it:

```lua
local Circle = {}
Circle.__index = Circle
Circle.describe = Shape.describe        -- flattened at declaration
Circle.new = function(opts) ... end
```

A method call on an instance is a **single table lookup**: instance
table → (miss) → `__index` → `Circle.describe`. No chain between
struct tables, no resolution at call time. Overrides are simply later
assignments — a method defined on the child after its declaration
overwrites the flattened copy; parent implementations stay reachable by
name (`Shape.describe(self)`).

### Subtyping

A child struct is accepted anywhere any ancestor is expected
(`Child <: Parent`, transitive), via nominal ancestry tracked at
declaration time. The synthesized `.new` signatures are excluded from
the comparison (each struct's opts record reflects its own fields).

## Feature overview

| Feature | Syntax |
|---|---|
| declaration | `local struct X ... end` (also `global`) |
| construction | `X.new { field = value }` — auto-generated, `.new` is reserved |
| init hook | `function X:init()` — no args besides `self`, chained root→leaf |
| instance methods | `function X:m(...)` (colon) |
| static methods | `function X.m(...)` (dot) |
| field defaults | `x: number = 0` — typechecked against the field type |
| static fields | `static ... end` block; excluded from `.new` opts; initializers emitted once on the type; inherited |
| inheritance | `local struct Dog:Animal` — single parent; fields, defaults, statics and methods are inherited |
| parent via alias | `local type P = Point; struct T:P` resolves to `Point` |
| cross-module parent | `local A = require("animal")` (module returns the struct directly) — see below |

### Nominal construction

Table literals are rejected where a struct instance is expected
(`local p: Point = { x = 1 }` is a type error, with a hint to use
`Point.new`): a bare table lacks the metatable wiring that methods and
`init` rely on. `as` remains the escape hatch.

### Cross-module parents

Supported through a top-level `local X = require("mod")` whose module
returns the struct directly — that local provably holds the struct's
runtime table, so the emitted copies and init calls are sound. Guarded
rejections, each with a clear error:

- field paths / bare globals (no runtime presence guarantee at the
  child's declaration site)
- parents whose *own* ancestors declare `init` (those tables are not
  visible outside the parent's module)
- parents with computed (non-literal) default values (their
  expressions may reference the parent module's locals)
- structs described by `.d.tl` declaration files (runtime shape is by
  contract)

### Explicit rejections (clear errors, not surprises)

- user-declared `X.new` (reserved; the error suggests `init`)
- data field named `new` in the body
- `static` blocks and `:Parent` in `record`/`interface` declarations
- nested structs — in both forms (`struct Inner ... end` and
  `type Inner = struct ... end` inside a type body); structs declared
  inside function bodies and as top-level `local type P = struct`
  declarations are supported and fully functional
- generic structs (`struct X<T>`) — not yet, see roadmap
- incompatible field overrides in children
- static fields shadowing instance fields

## Compatibility

- **No previously-valid `record` program changes behavior**; all
  pre-existing specs pass unchanged (1925 baseline).
- `struct` is not a parallel implementation, though — it shares record's
  machinery, which is worth spelling out for review:
  - `RecordType` carries the struct markers (`is_struct`, parent/chain
    tables, default-value maps) as new optional fields, `nil` on plain
    records;
  - the shared subtyping rule (`subtype_record`) and the table-literal
    checker each gained a small opt-in branch (`is_struct` checks);
  - struct-only syntax applied to a record — `X:Parent`, `static`
    blocks, `type Inner = struct` nested in a type body — is rejected
    with dedicated errors; these inputs were syntax errors before too,
    so only the messages are new;
  - `struct` is not a reserved word: it is parsed exactly like
    `record`/`enum` — usable as an identifier, treated as a type
    constructor only in type positions (same grammar slot).
- Struct instances are ordinary tables; the emitted Lua runs on all
  supported targets (verified `--gen-target=5.1` and `5.4`).
- Declaration files (`.d.tl`) can describe struct types for consumers.

## Testing

63 new specs in `spec/lang/declaration/struct_spec.lua`, covering:
construction, defaults (well-typed, mistyped, falsy, inherited,
overridden), init chaining (including skipping init-less ancestors and
exact-once semantics), method flattening and overrides, statics
(inheritance, `.new` rejection, single-block rule), subtyping and
upcasts, alias and cross-module parents (positive + all four guarded
rejections), reserved-name errors, and a general acceptance battery
(metamethods in struct bodies, array interfaces + inheritance,
recursion, 50-deep chains — the latter also verifies the generated
constructors stay linear: one `if` per defaulted field, zero spurious
init calls).

Full suite: 1988 passing (1781 lang / 96 api / 111 cli).

## Documentation

New chapter `docs/src/structs.md` (listed in the book summary),
including a "How it works: the generated code" section that spells out
the exact Lua emitted for `.new`, the init chain, and method
flattening, with a per-feature runtime cost table.

## Limitations & future work

- Parent methods must precede child struct declarations (flattening is
  a one-time, declaration-site operation). The checker enforces the
  order.
- Generic structs are rejected with a clear error; supporting
  `struct X<T>` is the natural follow-up.
- No multiple inheritance, no `super` — intentional; parent members
  are reachable by name.
- `init` takes no arguments besides `self`; construction data flows
  through the `.new` opts table.

We're happy to adjust naming, error messages, or split this into
smaller PRs (e.g. parser+codegen first, statics/cross-module as
follow-ups) if that eases review.
