---
title: Elixir boolean operators	
tags: [elixir]
categories: [TIL, Elixir]
---

There's subtle difference between Elixir boolean operators. Namely between `or` and `||`, `and` and `&&`, `not` and `!`.

These operators expect `true` or `false` as their first argument:
```elixir
a or b	# true if a is true; otherwise b
a and b # false if a is false; otherwise b
not a	# false if a is true; otherwise b
```

These operators take arguments of any type. Any value apart from `nil` or `false` is interpreted as `true`:
```elixir
a || b	# a if a is truthy; otherwise b
a && b	# b if a is truthy; otherwise a
!a	# false if a is truthy; otherwise true
```

Examples:
```elixir
iex(1)> 1 or 2
** (BadBooleanError) expected a boolean on left-side of "or", got: 1

iex(1)> 1 || 2
1

iex(1)> true or false
true

iex(2)> "foo" and "bar"
** (BadBooleanError) expected a boolean on left-side of "and", got: "foo"

iex(2)> "foo" && "bar"
"bar"

iex(3)> not "foo"
** (ArgumentError) argument error
    :erlang.not("foo")

iex(3)> !"foo"
false
```

