# RFC: `where` Bindings for Arrow Functions

## Introduction

This RFC proposes adding optional `where (...)` bindings to PHP arrow functions.

It introduces a concise, expression-level way to define local variables directly on an arrow function, eliminating the need for multi-line anonymous functions to introduce a single local, or awkward inline assignments.

The design mirrors the placement and feel of `use (...)` but defines local variables as opposed to captured variables.

PHP is a multi-paradigm language; this proposal embraces practical ideas seen in other ecosystems (notably Haskell’s `where`) while adapting them to PHP’s eager, expression‑oriented arrows and familiar semantics.

Recent additions and proposals also show PHP taking cues from the functional world. Notably, the pipe operator (`|>`) and the Function Composition RFC (https://wiki.php.net/rfc/function-composition). Expression‑level building blocks like `where` fit naturally alongside these.

## Motivation

Arrow functions are popular for short, expression-based callbacks. However, any non-trivial body today typically expands into a multi-line anonymous function just to introduce a single local:

```php
// Price filter with a single derived local.
function (stdClass $product): bool {
    $priceWithTax = $product->price * 1.2;
    return $priceWithTax > 4 && $priceWithTax < 10;
}
```

This is verbose compared to arrows. Alternatively, assignment inside the arrow expression is clunky or hacky:

```php
fn(stdClass $product): bool => ($price = $product->price * 1.2) > 4 && $price < 10; // easy to miss
```

Adding `where (...)` allows a clean, composable idiom:

```php
fn(stdClass $product): bool where ($price = $product->price * 1.2) => $price > 4 && $price < 10;
```

### Existing alternatives

Another workaround is to define a helper arrow function:

```php
$getPrice = fn(stdClass $product): float => $product->price * 1.2;
$products = array_filter($allProducts, fn($product): bool => $getPrice($product) > 4 && $getPrice($product) < 10);
```

With `where`, you can keep the computation local to the arrow function without introducing a helper:

```php
$products = array_filter(
    $allProducts,
    fn($product): bool where ($price = $product->price * 1.2) => $price > 4 && $price < 10
);
```

As mentioned, you can also rely on inline assignment inside expressions or array literals, but this tends to hide intent and is error prone:

```php
// Inline assignment in a condition works, but is easy to miss.
fn($product): bool => ($price = $product->price * 1.2) > 4 && $price < 10;

// Inline assignment inside an array literal; relies on evaluation order.
fn(User $u): array => [
    'full' => ($full = "{$u->first} {$u->last}"),
    'tag'  => strtoupper($u->first[0] . $u->last[0]),
    'desc' => "$full ({$u->last})",
];
```

`where` makes these intermediate values explicit, evaluated left‑to‑right before the body, without resorting to clever (and fragile) expression tricks.

## Proposal

### Syntax

Extend the arrow function syntax to optionally include a **`where` clause** between the parameter list and the `=>` token.

```php
fn(parameter_list) where (binding_list) => expression;
```

Each `binding` is an **assignment expression**:

```php
variable = expression
```

Notes:
- Bindings are separated by commas
- Trailing commas are allowed
- Destructuring patterns are permitted

### Examples

```php
fn(float $x, float $mean, float $sd): array where ($z = ($x - $mean) / $sd) => [
    'z'        => $z,
    'z²'       => $z ** 2,
    'distance' => abs($z),
];

fn(array $pair): int where ([$a, $b] = $pair, $sum = $a + $b) => $sum;
```

## Semantics

- The `where (...)` clause defines **local variables** visible only inside the arrow function
- Bindings are evaluated **left-to-right** at **invocation time**
- Each binding can refer to parameters and any previously defined binding
- Variable names cannot shadow parameters (error)
- Shadowing outer variables is allowed
- `&` references are **disallowed** inside `where (...)`
- Bindings are expressions; no statements or control structures
- Evaluation is eager, not lazy

## Desugaring (Conceptual)

```php
fn(stdClass $product): bool where ($price = $product->price * 1.2) => $price > 4 && $price < 10;
```

Desugars logically to:

```php
fn(stdClass $product): bool => (function() use ($product) {
    $price = $product->price * 1.2;
    return $price > 4 && $price < 10;
})();
```

The engine would internally treat this as a blockified arrow function with initializer statements followed by an implicit `return` of the arrow expression. Note: The IIFE illustration above is only a conceptual device, not implementation guidance.

## Grammar

```
arrow_function:
    'static'? 'fn' '(' parameter_list? ')' return_type?
    where_clause? '=>' expr

where_clause:
    'where' '(' binding_list? ')'

binding_list:
    binding (',' binding)* (',' )?

binding:
    variable '=' expr
  | list_pattern '=' expr
```

The new keyword `where` is reserved in this context.

## Examples in Practice

### Cleaner mapping

Avoid duplicate work when building structured results.

```php
array_map(
    fn(User $u): array
        where (
            $full = trim("{$u->first} {$u->last}"),
            $initials = strtoupper($u->first[0] . $u->last[0]),
            $slug = strtolower(preg_replace('/[^a-z0-9]+/i', '-', $full))
        )
        => ['full' => $full, 'initials' => $initials, 'slug' => $slug],
    $users
);
```

### Composed helpers

```php
fn($xs) where (
  $head = fn($a) => $a[0] ?? null,
  $tail = fn($a) => array_slice($a, 1),
  $h = $head($xs),
  $t = $tail($xs)
) => [$h, $t];
```

### Conditional shorthand

```php
static fn(int $n) where ($even = $n % 2 === 0) => $even ? 'even' : 'odd';
```

### Dependent bindings

Sequential bindings allow later values to depend on earlier ones.

```php
$squareandcube = fn(int $n): array where ($square = $n ** 2, $cube = $square * $n) => [$square, $cube];

array_map($squareandcube, [2, 4, 6]); //[[4, 8], [16, 64], [36, 216]]
```

This behaves like Haskell’s `where`:

```haskell
squareAndCube n = [square, cube]
  where
    square = n ^ 2
    cube = square * n
```

Except with PHP’s eager semantics and without nested scopes.

These semantics mirror Scheme’s `let*` (sequential, dependent bindings), not parallel `let`.

## Design Intent and Non-Goals

This feature is **inspired by** Haskell’s `where` but **deliberately simpler**:

| Feature                        | Haskell | PHP Proposal      |
|--------------------------------|---------|-------------------|
| Defines functions & values     | ✅      | ❌ values only    |
| Recursive / mutually recursive | ✅      | ❌                |
| Lazy                           | ✅      | ❌ eager          |
| Nested `where`                 | ✅      | ❌ not applicable |

**Non-Goals**

- No recursion or hoisting between bindings.
- No statements, loops, or conditionals in `where (...)`.
- No multi-level `where of where` nesting.
- Not a replacement for blocks or full closures.

This keeps the feature small, predictable, and consistent with PHP’s expression only arrow functions.

## Implementation Notes

* **AST**: add an optional `where_bindings` property to the `ast_arrow_function` node.
* **Parser**: allow an optional `T_WHERE '(' ... ')'` between parameter list and `T_DOUBLE_ARROW`.
* **Compiler**: emit opcodes that assign each binding before evaluating the arrow body.
* **Engine behavior**: identical to current arrow functions, with local variables initialised beforehand.
* **Reflection**: expose bindings as part of the AST node (for tooling).

## Alternative: Overloading `use`

One considered alternative was to overload `use (...)` on arrow functions to carry bindings, e.g. `fn(...) use ($x = expr) => ...`.

Pros

- Reuses an existing keyword and familiar syntax; no new reserved word.

Cons

- Semantic inversion: `use` historically means "capture from outer scope" whereas here we want "define locals" - different concepts sharing syntax.
- Grammar and readability: `use` currently accepts only variable names (and references) on closures; allowing full assignment expressions would create a special case just for arrows.
- Mental model: arrows already implicitly capture; adding a second, different meaning for `use` on arrows conflicts with that design and increases confusion (`use($a)` vs `use($a = ...)`).
- Future evolution: keeping `use`’s meaning consistent across closures/arrows preserves space for potential extensions without ambiguity.

A dedicated `where (...)` clause makes intent explicit, keeps local initialisation clearly separate, and avoids overloading `use` with two meanings.


## Relationship to `use`

`use` and `where` serve complementary roles:

- `use (...)` declares external dependencies; i.e., variables captured from the surrounding scope.
- `where (...)` declares internal derivations; i.e., local variables computed inside the function from parameters (and prior bindings).

Keeping these concerns distinct improves readability: the signature shows parameters; `use` communicates "what this function needs from outside"; `where` communicates "what this function derives before evaluating the body”. They are orthogonal features and can conceptually coexist (e.g., a future extension for block closures: `function (...) use ($dep) where ($tmp = expr) { ... }`).

## Backward Compatibility

- `where` will become a **reserved keyword**.
- No changes to existing arrow or closure semantics.
- Minimal parser impact: the new production is unambiguous.

## Performance Impact

Negligible. Bindings are compiled to sequential assignment opcodes preceding the return expression.

## Interaction with the Pipe Operator (`|>`)

The proposed `where (...)` clause is orthogonal to the `|>` operator. The pipe structures the sequence of computations between expressions; `where` provides clear, named locals inside a single arrow step. Used together, they enable a clean, functional pipeline style without sacrificing PHP’s readability.

Example (combining both):

```php
$data |> fn($v) where (
    $sum   = array_sum($v),
    $mean  = $sum / count($v),
    $sq    = array_map(fn($x) => ($x - $mean) ** 2, $v),
    $sd    = sqrt(array_sum($sq) / count($v))
) => ['mean' => $mean, 'sd' => $sd];
```

Conceptually:

- `|>` improves flow between steps; it passes the left value as the first argument to the right expression.
- `where` improves clarity within a step; it names intermediate values used by the arrow's body.

This complements PHP’s multi-paradigm nature. The pipe operator offers a functional sequencing idiom, while `where` offers local naming ideas inspired by languages like Haskell, adapted to PHP’s eager semantics and single‑expression arrows.

### Function Composition

The Function Composition RFC proposes a first‑class composition operator to build functions from functions (https://wiki.php.net/rfc/function-composition). Composition remains orthogonal to `where`:

- Composition glues reusable steps together (reusability and abstraction).
- `where` clarifies the internals of a single step (local names, dependent bindings).

Used together with `|>`, pipelines can be written as composed stages, each with a clear internal story via `where`.

## Prior Art & Direction

Many languages offer local binding constructs that improve readability of small transformations:
- Haskell: `where` and `let/in` establish local names; often lazy and may permit recursive groups.
- ML family (OCaml/F#): `let` bindings in expressions; eager evaluation, typically non‑recursive by default.
- Scheme: `let` (parallel) vs `let*` (sequential, dependent); `where` mirrors `let*` semantics.
- C# LINQ: `let` introduces computed names within query comprehensions.
- SQL: `WITH` (CTE) names intermediate results at statement scope.

In parallel, PHP’s current and proposed features draw on functional idioms:
- Pipelines: `|>` enables left‑to‑right function application (cf. Elixir, F#, Hack).
- Composition: the Function Composition RFC proposes building functions from functions.
- Local bindings: `where` names intermediates within a single step.

These are orthogonal and complementary: `|>` expresses flow between steps; composition builds reusable steps; `where` clarifies internals within a step.

## Future Scope

- Potential extension to normal closures (`function (...) use (...) where (...) { ... }`) if desired
- Static analysis tools and IDEs can infer types for bound locals

## Voting

Simple Yes/No vote: Add `where (...)` bindings to arrow functions.

## References

1. **Haskell 2010 Language Report**
   Simon Marlow (ed.), Cambridge University Press, 2010.
   Chapter 4: *Declarations and Bindings* — includes function and pattern bindings, `let`, and `where` constructs.
   [https://www.haskell.org/onlinereport/haskell2010/](https://www.haskell.org/onlinereport/haskell2010/)
   [https://www.haskell.org/onlinereport/haskell2010/haskellch4.html](https://www.haskell.org/onlinereport/haskell2010/haskellch4.html)

2. **PHP RFC: Arrow Functions 2.0 (Short Closures)**
   Nikita Popov, 2019.
   Defines the `fn` arrow syntax and captures semantics, serving as the baseline for this proposal.
   [https://wiki.php.net/rfc/arrow_functions_v2](https://wiki.php.net/rfc/arrow_functions_v2)

3. **PHP Manual — Arrow Functions**
   Official documentation for short closures, including `fn` semantics and `use` capture behavior.
   [https://www.php.net/manual/en/functions.arrow.php](https://www.php.net/manual/en/functions.arrow.php)

4. **F# Language Reference — `let` and `where` bindings**
   Shows similar semantics for local bindings in functional expressions, where `let` defines locals in expression scope.
   [https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/functions/let-bindings](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/functions/let-bindings)

5. **OCaml Manual — Local Bindings with `let`**
   Describes eager local binding semantics in a strict functional language, close to PHP’s evaluation model.
   [https://ocaml.org/manual/expr.html#let-expressions](https://ocaml.org/manual/expr.html#let-expressions)

6. **Scala Language Specification**
   Section 6.18: *Block Expressions and Local Definitions* — demonstrates local definitions inside expressions.
   [https://scala-lang.org/files/archive/spec/2.13/06-expressions.html](https://scala-lang.org/files/archive/spec/2.13/06-expressions.html)

7. **Kotlin Language Reference — `let` Scope Function**
   Illustrates local expression‑scoped bindings within lambda expressions, similar in purpose to `where`.
   [https://kotlinlang.org/docs/scope-functions.html#let](https://kotlinlang.org/docs/scope-functions.html#let)

8. **Swift Language Guide — Local Constants and Variables**
    Demonstrates expression‑local binding within closures and block scopes; a parallel to `where` semantics.
    [https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html#ID326](https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html#ID326)

9. **PHP RFC: Function Composition**
   Proposal to introduce a function composition operator, complementing `|>` and `where`.
   [https://wiki.php.net/rfc/function-composition](https://wiki.php.net/rfc/function-composition)

These references collectively show that expression‑local bindings are a recurring idea across functional and multi‑paradigm languages. Haskell’s `where` is the conceptual origin; PHP’s arrow functions provide the structural base; and modern languages like F#, Scala, Kotlin, and Swift illustrate pragmatic adaptations of the same idea in eager, expression‑centric contexts.

## Appendix: Rationale Summary

`where (...)` gives PHP arrows a **structured "let" expression** without making arrows into full statement blocks. It matches PHP’s ergonomics and keeps the language consistent with modern expression syntax while staying implementable in php-src with minimal changes.
