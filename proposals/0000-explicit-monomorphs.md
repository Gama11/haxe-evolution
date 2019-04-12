# Explicit monomorphs

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Jens Fischer](https://github.com/Gama11)

## Introduction

Introduce a `_` syntax for explicit monomorphs in type hints.

## Motivation

Haxe's type system has monomorphs, but no way to _explicitly_ express this in syntax. Usually this is done _implicitly_ by omitting the type hint:

```haxe
var m; // Unknown<0>
```

However, this approach does not work for parameterized types:

```haxe
var m:Map; // Invalid number of type parameters for Map
```

In accordance with the "Don't Repeat Yourself" principle, it is considered best practice to leverage type inference to _decrease_ redundant information and to _increase_ readability. However, this becomes impossible as soon as parameterized types are involved. We cannot let type parameters be inferred while also specifying the outer type.

A good example where this is problematic is the use case of typing a map literal as a read-only abstract. To the reader it's obvious what the type parameters should be, yet they have to be written out explicitly:

```haxe
var m:ReadOnlyMap<String, (Int, Int) -> Int> = [
	"add" => (a, b) -> a + b,
	"subtract" => (a, b) -> a - b
];
```

Even worse, if not careful, information can be lost this way. Normally Haxe would infer the function type as `(a:Int, b:Int) -> Int`, including argument names. But if these are spelled out explicitly as well, the example becomes even more verbose.

Contrast this with the much more readable `_` syntax:

```haxe
var m:ReadOnlyMap<_, _> = [
	"add" => (a, b) -> a + b,
	"subtract" => (a, b) -> a - b
];
```

Syntax-wise, underscores are the natural choice, as they are already used as wildcards in pattern matching. Furthermore, it's also a well-established convention to use `_` for local identifiers that are "ignored", though it does not yet have special semantics here. So the proposed syntax is both familiar and concise.

Another possibility might be `?`, but this seems problematic since this is already used for optional types. `_` currently has no meaning in type hints and does not parse.

A natural extension of the syntax is to allow the last `_` type parameter to act as a _rest argument_, effectively a wildcard for however many type parameters follow. This too follows the precedent set by pattern matching:

```haxe
var m:ReadOnlyMap<_>; // _ acting as a rest argument in type parameters

enum Foo {
	Pattern(a:Int, b:Int);
}
switch Pattern(1, 2) {
	case Pattern(_): // _ acting as a rest argument in pattern matching
}
```

## Detailed design

The implementation seems straightforward:

- add a new argument-less `CTMono` to `Ast.complex_type`
- parse `_` in type hints as `CTMono`
- type `CTMono` as `TMono`

## Impact on existing code

The new `ComplexType.TMono` constructor in the macro API would break any `switch` on `ComplexType` that does not have a default case.

## Drawbacks

None apart from the usual concerns with adding new syntax.

## Alternatives

While Haxe does not have a syntax or built-in type for monomorphs, a `Mono<T>` type can already be implemented with a `@:genericBuild` macro that returns `null`:

```haxe
class Main {
	static function main() {
		var m:Array<Mono>;
		$type(m); // Array<Unknown<0>>
	}
}

@:genericBuild(Macro.build()) class Mono {}
```

```haxe
class Macro {
	public static function build() {
		return null;
	}
}
```

Unfortunately, realistically speaking the most concise syntax possible with this approach that it still readable is `Mono` (`_` is not a valid type name, and a single-character name like `M` seems questionable). Since `Mono` looks like a regular type name, it does not reduce the visual noise when reading code by nearly as much as `_`.

However, it does raise the question if allowing `_` as a type name could be a viable alternative to introducing `ComplexType.TMono`. The standard library would possibly contain a `@:coreType abstract _ {}` in this case. Still, this might end up being just as or even more breaking than introducing a new `ComplexType` constructor, as various tools will have to support types named `_`. It also means the toplevel `_` type could be shadowed by user types, which seems undesirable.

## Opening possibilities

### Printing monomorphs

Not having any way to express monomorphs in syntax is also reflected in error messages and tooling, where usually `Unknown<0>` is used. Depending on the user, this can be intimidating and confusing at worst, and a lot more verbose than it needs to be at best. Both haxe-language-server and haxe-TmLanguage instead use a special `?` syntax, which is more readable, but also potentially confusing because it is not valid Haxe syntax.

If `_` is established as the de facto syntax for monomorphs, it should also be used anywhere where monomorphs have to be printed (error messages, hover hints in IDEs...).

### Default type parameters

Furthermore, though neither proposal depends on the other, explicit monomorphs would play well together with [default type parameters](https://github.com/HaxeFoundation/haxe-evolution/pull/50), as it would allow explicit usage of a default value:

```haxe
class T<A = String, B> {}
var t:T<_, Int>;
```

Not only does this make the code more readable because the intent is clearer, it also allows switching out the concrete default value of `A` at the declaration without needing to update the usage.

## Unresolved questions

It is unclear whether it makes sense to allow explicit monomorphs outside of type parameters. `var foo:_;` should possibly result in a typer error, but might be useful for macros.

Similarly, it is unclear whether `_` should itself be allowed to be parameterized (`var foo:_<Int> = [];`).
