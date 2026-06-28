# Mutator catalog (cxx-mutant)

`cxx-mutant` uses token-mode mutators by default and supports additional AST-informed
clang mode where supported.

## Built-in mutators

| Mutator | Description | Default? |
|---|---|---|
| `ConditionalBoundary` | `<` `<=` `>` `>=` branch boundary swaps | yes |
| `EqualityOperator` | `==` ↔ `!=` | yes |
| `LogicalOperator` | `&&` ↔ `||` | yes |
| `BooleanLiteral` | `true` ↔ `false` | yes |
| `ArithmeticOperator` | `+ - * /` swaps | opt-in |
| `AssignmentOperator` | `+= -= *= /=` swaps | opt-in |
| `BitwiseOperator` | `& | ^` swaps | opt-in |
| `UnaryOperator` | `!` removal/duplication | opt-in |
| `ReturnValue` | `return true` ↔ `return false` | opt-in |

## Recommended production selection

For PR-sized gates and low-noise behavior, start with the defaults:

```text
ConditionalBoundary,EqualityOperator,LogicalOperator,BooleanLiteral
```

Then add arithmetic and mutation-heavy operators only if signal quality is still poor.

