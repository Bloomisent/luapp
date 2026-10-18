# Lua++ Manual

Oct 18, 2026 · @Mayai

## Overview

Lua++ is a fork of the Lua 5.5 C source that adds extra syntax while keeping Lua's tables, metatables, closures and 1-based indexing. Anything not listed in this manual behaves as in stock Lua 5.5.

What Lua++ adds:

- `!=` as a replacement for `~=`, for clarity's sake.
- compound assignment: `+=`, `-=`, `*=`, `/=`, `//=`, `^=`, `%=`
- `++` and `--` as statements
- `??`, a nil-coalescing operator
- `continue` in every loop type
- optional runtime-checked type annotations on locals and parameters
- `class`, `extends` and `init`, built on metatables
- `##` as the line-comment marker

What changes for existing Lua code:

- `continue` and `class` are now reserved words, so they can no longer be used as variable names.
- `--` is now the decrement token, so Lua's `--` comments no longer work. Use `##`.
- `a --b` lexes as a decrement, so write subtraction of a negative with a space: `a - -b`.

## Running programs

Run a script by passing its path to the interpreter: `Lua++.exe script.lua++`. Source files use the `.lua++` extension by convention, and the standard Lua libraries are loaded as usual.

Comments start with `##` and run to the end of the line:

```lua
## this is a comment
local x = 1   ## so is this
```

Semicolons stay optional, as in Lua. A global function named `__class` is defined at startup because the class system uses it. Do not overwrite it.

## Operators

Lua++ adds shorthand for the most common assignment patterns and one new binary operator, `??`.

### Not equal

`!=` means the same as `~=`. Only `!``=` works.

### Compound assignment

A compound operator reads the target, applies the operator, and stores the result back. The right side can be any expression, so `a += b + c` adds the whole sum.

| Operator | Equivalent | Notes |
| --- | --- | --- |
| `x += y` | `x = x + y` |  |
| `x -= y` | `x = x - y` |  |
| `x *= y` | `x = x * y` |  |
| `x /= y` | `x = x / y` | always a float, as with `/` |
| `x //= y` | `x = x // y` | floor division; stays an integer for integer operands |
| `x ^= y` | `x = x ^ y` | always a float |
| `x %= y` | `x = x % y` |  |

Valid targets are local variables, upvalues, globals, and table fields or indexes (`t.x += 1`, `t[i] *= 2`). Compound assignment is a statement, not an expression, so `y = x += 1` is a syntax error. It also takes a single target: `a, b += 1` is not allowed.

### Increment and decrement

`x++` and `x--` add or subtract 1 from the target. They are statements only and cannot be used inside an expression.

### Nil coalescing

`a ?? b` evaluates to `a` unless `a` is `nil`, in which case it evaluates to `b`. The right side is only evaluated when the left side is `nil`.

This differs from `or` in one case: `false` is a real value for `??`.

```lua
local debug = false
print(debug or true)    ## true   (false is treated as missing)
print(debug ?? true)    ## false  (false is kept)
print(nil ?? 5)         ## 5
print(0 ?? 5)           ## 0
```

`??` is right-associative, so `a ?? b ?? c` returns the first value that is not `nil`. Put parentheses around it whenever you mix it with `and`, `or` or comparisons, because `x == nil ?? y` reads as `(x == nil) ?? y`.

## continue

`continue` skips the rest of the current loop iteration and moves to the next one. It works in `while`, numeric `for`, generic `for` and `repeat ... until` loops.

```lua
for i = 1, 5 do
  if i % 2 == 0 then continue end
  print(i)              ## 1 3 5
end

local n = 0
while n < 5 do
  n += 1
  if n == 3 then continue end
  print(n)              ## 1 2 4 5
end
```

Rules:

- In nested loops, `continue` applies to the innermost loop only.
- Closures created in the loop body still capture the right variables when `continue` skips the rest of the iteration.
- In a `repeat ... until` loop, `continue` jumps to the `until` condition. Like `goto`, it cannot jump forward over a `local` declaration that is still in scope at that point, and the compiler reports `jumps into the scope of local` if you try.
- `continue` outside a loop is a compile-time error: `continue outside a loop at line N`.
- Internally `continue` is a jump to a hidden label at the end of the loop body, so it needs no new bytecode.

## Type annotations

Type annotations are runtime contracts, not a static type system: the compiler parses them, and the VM checks them when the code runs. Code without annotations behaves exactly as in Lua.

```lua
local x: number = 1
local s: string? = nil              ## '?' means "or nil"
local u: number | string = "a"      ## union
local n: integer = 10 // 2

function add(a: number, b: number)
  return a + b
end
```

### Supported types

| Type | Accepts |
| --- | --- |
| `number` | integers and floats |
| `integer` | integers only; a float such as `3.0` is rejected |
| `string`, `boolean`, `table`, `function`, `thread` | the matching Lua type |
| `userdata` | full and light userdata |
| `nil` | only `nil` |
| `any` | every value |

Combine types with `|`, and write `T?` as shorthand for `T | nil`. An unknown type name is a compile-time error.

### Where checks run

- when a typed local is declared
- on function entry, for each typed parameter
- after each assignment to a typed local: plain (`x = v`), multiple (`a, b = v1, v2`) and compound (`x += 1`)

A failed check raises a runtime error that starts with `type mismatch` and names the variable, for example `type mismatch (local 'y')`.

### Not checked

- return types
- table fields and array elements
- assignments to a typed local from inside a closure (the upvalue carries no type)
- values flowing into untyped variables

### Things that surprise people

`/` and `^` always produce floats, so `local a: integer = 10 / 2` fails. Use `10 // 2`. Each typed local or parameter adds one VM instruction to check it, and there is currently no switch to turn the checks off.

## Classes

A class is a table whose metatable provides construction and inheritance, so everything you know about Lua tables and metatables still applies.

```lua
class Animal
  function init(name)
    self.name = name
  end
  function speak()
    print(self.name .. " makes a sound")
  end
end

class Dog extends Animal
  function init(name)
    Animal.init(self, name)
  end
  function speak()
    print(self.name .. " barks")
  end
end

local d = Dog("Rex")
d:speak()                         ## Rex barks
print(getmetatable(d) == Dog)     ## true
```

Rules:

- `class Name ... end` creates the class and assigns it to the name `Name`.
- `extends Base` is optional and takes any expression that evaluates to a class. `extends` is a contextual word, so it is not reserved.
- The class body can contain only `function` definitions. Every one of them gets an implicit `self` parameter, as if declared with a colon.
- Calling the class, `Dog("Rex")`, creates an object and passes the arguments to `init` if the class defines one.
- Methods are found through the `__index` chain, so a subclass inherits everything its base defines.
- Single inheritance only. To call a base method, name the base explicitly, as in `Animal.init(self, name)`.
- Each class stores its name in `__name` and its base class in `__base`.

## Known limitations

These are the gaps to know about before relying on a feature.

### Behaviours to be careful with

- Class names are ordinary assignments, so two classes with the same name silently replace each other.
- `__class` is a normal global. Overwriting it, or running a script that replaces `_ENV` with an empty table, breaks class definitions.
- Compound assignment accepts only single targets, and it is a statement, not an expression.

## Quick reference

| Feature | Syntax |
| --- | --- |
| Comment | `## text` |
| Not equal | `a != b` |
| Compound assignment | `x += 1`, `-=`, `*=`, `/=`, `//=`, `^=`, `%=` |
| Increment, decrement | `x++`, `x--` (statements only) |
| Nil coalescing | `a ?? b` |
| Skip to next iteration | `continue` |
| Typed local | `local x: number = 1` |
| Typed parameter | `function f(a: string?)` |
| Union type | `number \| string` |
| Class | `class Name ... end` |
| Subclass | `class Name extends Base ... end` |
| Construct | `Name(args)` calls `init` |

### Starter regression test

Save this as a `.lua++` file and run it after every build. Each line should pass silently; a failing `assert` names the line.

```lua
assert(1 != 2)

local x = 1
x += 1;   assert(x == 2)
x *= 5;   assert(x == 10)
x //= 3;  assert(x == 3)
x ^= 2;   assert(x == 9)
x %= 4;   assert(x == 1)

assert((nil ?? 5) == 5)
assert((false ?? 5) == false)
assert((nil ?? nil ?? 3) == 3)

local t = {1, 2, 3}
t[1] += 1
assert(t[1] == 2)

local seen = {}
for i = 1, 5 do
  if i % 2 == 0 then continue end
  seen[#seen + 1] = i
end
assert(#seen == 3)

assert(not pcall(function() local y: number = "hi" end))
assert(not pcall(function() local y: integer = 3.0 end))
assert(pcall(function(a: number) return a end, 5))

class A
  function init(v) self.v = v end
  function get() return self.v end
end
class B extends A
end
assert(B(4):get() == 4)

print("all passed")
```
