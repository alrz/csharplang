
# List patterns

## Summary

Lets you to match an array or a list with a sequence of patterns e.g. `array is [1, 2, 3]` will match an integer array of the length three with 1, 2, 3 as its elements, respectively.

## Detailed Design

### Fixed-length list patterns

The syntax will be modified to include a *list_pattern* defined as below:

```antlr
primary_pattern
	: list_pattern
list_pattern
	: '[' (pattern (',' pattern)* ','?)? ']'
```

**Pattern compatibility:** A *list_pattern* is compatible with any type that conforms to the ***range indexer pattern***:

1. Has an accessible property getter that returns an `int` and has the name `Length` or `Count`
2. Has an accessible indexer with a single `int` parameter
3. Has an accessible `Slice` method that takes two `int` parameters (for slice patterns)

This rule includes `T[]`,  `string`,  `Span<T>`, `ImmutableArray<T>` and more.

**Subsumption checking:**  This construct will be built entirely on top of the existing DAG nodes. The subsumption is checked just like recursive patterns with `ITuple` - corresponding subpatterns are matched by position plus an additional node for testing length.

**Lowering:** A pattern of the form `expr is [1, 2, 3]` is equivalent to the following code:
```cs
expr.Length is 3
&& expr[0] is 1
&& expr[1] is 2
&& expr[2] is 3
```

### Variable-length list patterns

The syntax will be modified to include a *slice_pattern* defined as below:
```antlr
primary_pattern
	: slice_pattern
slice_pattern
	: '..'
```

A *slice_pattern* is only permitted once and only directly in a *list_pattern* and discards ***zero or more*** elements. Note that it's possible to use a *slice_pattern* in a nested *list_pattern* e.g. `[.., [.., 1]]` will match `new int[][]{new[]{1}}`.

A *slice_pattern* acts like a proper discard i.e. no tests will be emitted for such pattern, rather it only affects other nodes, namely the length and indexer. For instance, a pattern of the form `expr is [1, .., 3]`  is equivalent to the following code: 
```cs
expr.Length is >= 2
&& expr[0]  is 1
&& expr[^1] is 3
```
Note: the lowering is presented in the pattern form here to show how subsumption checking works, for example, the following code produces an error because both patterns yield the same DAG:

```cs
case [_, .., 1]: // expr.Length is >= 2 && expr[^1] is 1
case [.., _, 1]: // expr.Length is >= 2 && expr[^1] is 1
```
Unlike:
```cs
case [_, 1, ..]: // expr.Length is >= 2 && expr[1] is 1
case [.., 1, _]: // expr.Length is >= 2 && expr[^2] is 1
```

Note: the pattern `[..]` lowers to `expr.Length >= 0` so it would not be considered as a catch-all.

### Slice patterns

We can further extend the *slice_pattern* to be able to capture the skipped sequence:


```antlr
slice_pattern
	: '..' pattern?
```

So, a pattern of the form `expr is [1, ..var s, 3]` would be equivalent to the following code:

```cs
expr.Length    is >= 2
&& expr[0]     is 1
&& expr[1..^1] is var s
&& expr[^1]    is 3
```


## Questions

- Should we support multi-dimensional (including non-zero-based) arrays?
- Should we support a trailing designator to capture the input? e.g. `[] v`?
