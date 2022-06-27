---
title: Creating a function with inputs from a file at compile time using Elixir AST
tags: [elixir, elixir-ast, abstract-syntax-tree]
categories: [TIL, Elixir]
---

Let me fill you in on what the problem I was trying to solve was, then we can get to how I've solved the problem by creating a function with inputs from a "defaults" file at compile time using Elixir's Abstract Syntax Tree (AST). 

We have a CSV file that has some defaults for our business logic, the file barely changes and is not huge.
The content of the file is transformed into a list of tuples using a script and the script is included in one of the modules as comments.
Using this script (in an `iex` shell), you would get the output, which is a list of tuples, and paste it into appropriate function body, which would be used at runtime.
Whenever the CSV file changes you would do the same things described above.

So the script looks something like this
```elixir
# example CSV just in case you want to run this script.
# Just copy & paste the values below into some CSV file.
# 
# service_name,base_price,duration
# Haircut,50,30
# Manicure,50,60
# Pedicure,70,90

"./lib/data.csv"
|> File.stream!()
|> CSV.decode!(headers: true)
|> Enum.map(fn row ->
  {
    row["service_name"],
    row["duration"],
    row["base_price"]
  }
end)
```

When run, this script outputs a list of tuples shown below.
```elixir
[
  {"Haircut", "30", "50"},
  {"Manicure", "60", "50"},
  {"Pedicure", "90", "70"}
]
```

You would then take the list above and put it inside one of the functions, which would be used at runtime on production for whatever logic and the final module would look something like this.
```elixir
defmodule Playground do
  # Script to use whenever necessary to
  # generate the body of some_important_function/0
  #
  # "/path/to/csv"
  # |> File.stream!()
  # |> CSV.decode!(headers: true)
  # |> Enum.map(fn row ->
  #   {
  #     row["service_name"],
  #     row["duration"],
  #     row["base_price"]
  #   }
  # end)

  def some_important_function do
    [
      {"Haircut", "30", "50"},
      {"Manicure", "60", "50"},
      {"Pedicure", "90", "70"}
    ]
  end
end
```

Everything is fine so far, except that you have to run the script manually every time the CSV file changes. It works anyway.
The problem started when we wanted to translate some of the strings from the CSV file, which means we would need to wrap some of the strings with `gettext/1` macro.

So the outcome we want to achieve is to have `some_important_function` as
```elixir
def some_important_function do
  [
    {gettext("Haircut"), "30", "50"},
    {gettext("Manicure"), "60", "50"},
    {gettext("Pedicure"), "90", "70"}
  ]
end
```

We can't directly wrap our strings with `gettext/1` in our "script" above because,

1. The strings are not known at compile time, hence can't be used with macros.
2. Moreover, it's just a script and it's not actually on production.

One solution would be to manually wrap the necessary strings with `gettext/1` inside `some_important_function/0` above. But that's too much manual work and error prone.

## Solving the problem at compile time with Elixir AST

Doing some research and asking around in [Elixir Slack](https://elixir-slackin.herokuapp.com/), I figured we could solve the problem using Elixir AST.

> What is the AST?
>
> An AST, acronym for Abstact Syntax Tree, is a representation of code as a data structure. This data structure is used by Elixir to run Elixir code: either by interpreting it directly: executing the commands in the AST by recursively walking it; or by compiling it: translating the AST into another format, namely BEAM bytecode instructions, which then are saved to disk as .beam files.<sup>[source: botsquad](https://www.botsquad.com/2019/04/11/the-ast-explained/)</sup>

We will first read the CSV at compile time and create the tuple list, which will then be used to create an AST with `gettext/1` macro used wherever necessary.
Lastly we will `unquote/1` this AST within our actual function body which will be used at runtime on production.
Moreover, we will use [`@external_resource`](https://hexdocs.pm/elixir/main/Module.html#module-external_resource) module attribute to make sure that whenever our CSV file changes our module will be recompiled, meaning that our code will always be up to date!
No more running scripts manually. 🎉

> `@external_resource`
>
> Tools may use this information to ensure the module is recompiled in case any of the external resources change, see for example: mix compile.elixir. <sup>[source: Hexdocs](https://hexdocs.pm/elixir/main/Module.html#module-external_resource)</sup>

```elixir
defmodule Playground do
  # This way our module will be recompiled whenever
  # you make changes to the CSV file.
  # Make some changes to the CSV file and recompile to see it in action.
  @external_resource "./lib/foo.csv"

	# Nothing fancy here, we're just creating a list of tuples
  data =
    @external_resource
    |> File.stream!()
    |> CSV.decode!(headers: true)
    |> Enum.map(fn row ->
      {
        row["service_name"],
        row["duration"],
        row["base_price"]
      }
    end)

  IO.inspect(data)
  # =>
  # [
  #   {"Haircut", "30", "50"},
  #   {"Manicure", "60", "50"},
  #   {"Pedicure", "90", "70"}
  # ]

  # This is where we build the AST.
  data_ast =
    Enum.map(data, fn tuple ->
      list =
        tuple
        |> put_elem(0, {:gettext, [], [elem(tuple, 0)]})
        |> Tuple.to_list()

      {:{}, [], list}
    end)

  IO.inspect(data_ast)
  # =>
  # [
  #   {:{}, [], [{:gettext, [], ["Haircut"]}, "30", "50"]},
  #   {:{}, [], [{:gettext, [], ["Manicure"]}, "60", "50"]},
  #   {:{}, [], [{:gettext, [], ["Pedicure"]}, "90", "70"]}
  # ]

  data_ast
  |> Macro.to_string()
  |> IO.puts()
  # =>
  # [
  #   {gettext("Haircut"), "30", "50"},
  #   {gettext("Manicure"), "60", "50"},
  #   {gettext("Pedicure"), "90", "70"}
  # ]

  # Now, this function would return the same thing
  # that's returned by Macro.to_string/1 above
  def some_important_function do
    unquote(data_ast)
  end
end
```
