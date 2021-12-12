---
title: How Ecto infers field types from a changeset or a schema struct
tags: [elixir, ecto, cast]
categories: [TIL, Elixir]
---

I was wondering how [`Ecto.Changeset.cast/4`](https://hexdocs.pm/ecto/Ecto.Changeset.html#cast/4) function casts attributes from a map into their approriate types in the DB since we do not provide any information about the types explicitly.

Turns out that if you are passing a schema struct to `cast/4` Ecto uses the [`__changeset__`](https://github.com/elixir-ecto/ecto/blob/master/lib/ecto/schema.ex#L598-L600) function on it to figure out the types of the fields. Otherwise if you are passing an `Ecto.Changeset` to `cast/4` then Ecto uses the [`types`](https://hexdocs.pm/ecto/Ecto.Changeset.html#module-the-ecto-changeset-struct) field of the changeset struct.

---

To show you what I mean, we are going to create a Phoenix app called `God` with only a `Human` schema so we can **c**reate, **r**ead, **u**pdate, **d**elete a `Human` (wish it was that easy lol).

*If you already know what I mean then you can basically skip to [the breakdown](#the-breakdown).*

Create a Phoenix app:

```bash
$ mix phx.new god
``` 

After our project is created, and we ran `mix ecto.create` inside the project directory we need to generate our controller, view and context for `Human` to be able to CRUD `Human`s.

```bash
╰─$ mix phx.gen.html Creature Human humans name surname to_be_born_on:utc_datetime age_to_die_at:integer will_get_married:boolean
                                                                      
* creating lib/god_web/controllers/human_controller.ex
* creating lib/god_web/templates/human/edit.html.eex
* creating lib/god_web/templates/human/form.html.eex
* creating lib/god_web/templates/human/index.html.eex
* creating lib/god_web/templates/human/new.html.eex
* creating lib/god_web/templates/human/show.html.eex
* creating lib/god_web/views/human_view.ex
* creating test/god_web/controllers/human_controller_test.exs
* creating lib/god/creature/human.ex
* creating priv/repo/migrations/20210716083409_create_humans.exs
* creating lib/god/creature.ex
* injecting lib/god/creature.ex
* creating test/god/creature_test.exs
* injecting test/god/creature_test.exs

Add the resource to your browser scope in lib/god_web/router.ex:

    resources "/humans", HumanController


Remember to update your repository by running migrations:

    $ mix ecto.migrate
```

Basically our `Human` will have a `name`, `surname` and we will also have a `utc_datetime` for them `to_be_born_on`, an `age_to_die_at` and finally whether they `will_get_married`.

After we follow the instructions given after running `mix phx.gen.html` above we will have a working `God` app. :D


![A look at our human control center](https://imgur.com/4oKBi4I.png)

We've already created (scheduled maybe?) a human to be born on 05/05/2022 at 07:07 who will sadly die at the age of 88. 🎉

![I am God](https://media.giphy.com/media/G3fPad8N68GfS/giphy.gif)

---

# Let's get serious

So when we navigate to `http://localhost:4000/humans/new` we get the form below to create a human.

![New human form](https://i.imgur.com/B46JIH5.png)


After we fill in the details in the form and click the save button, our request follows a path of

`endpoint.ex -> router.ex -> HumanController.create/2`

Below you can see the code for `HumanController.create/2` which gets run eventually,

```elixir
def create(conn, %{"human" => human_params}) do
  case Creature.create_human(human_params) do
    {:ok, human} ->
      conn
      |> put_flash(:info, "Human created successfully.")
      |> redirect(to: Routes.human_path(conn, :show, human))

    {:error, %Ecto.Changeset{} = changeset} ->
      render(conn, "new.html", changeset: changeset)
  end
end
```

which in fact calls a function `Creature.create_human/1` which you can see below.

```elixir
def create_human(attrs \\ %{}) do
  %Human{}
  |> Human.changeset(attrs)
  |> Repo.insert()
end
```

This function also calls `Human.changeset/2` which you can see below.

```elixir
def changeset(human, attrs) do
  human
  |> cast(attrs, [:name, :surname, :to_be_born_on, :age_to_die_at, :will_get_married])
  |> validate_required([:name, :surname, :to_be_born_on, :age_to_die_at, :will_get_married])
end
```

If we insert a `require IEx; IEx.pry()` after the function head and before the function end like below,

```elixir
def changeset(human, attrs) do
  require IEx; IEx.pry()
  human = human
  |> cast(attrs, [:name, :surname, :to_be_born_on, :age_to_die_at, :will_get_married])
  |> validate_required([:name, :surname, :to_be_born_on, :age_to_die_at, :will_get_married])
  require IEx; IEx.pry()
  human
end
```

and inspect the `attrs` map. We will get:

```bash
pry(1)> attrs
%{
  "age_to_die_at" => "99",
  "name" => "John",
  "surname" => "Doe",
  "to_be_born_on" => %{
    "day" => "1",
    "hour" => "0",
    "minute" => "0",
    "month" => "1",
    "year" => "2016"
  },
  "will_get_married" => "true"
}
```

As you can see all the values are of string which is what we can only get after a form submission anyway.

If we also inspect the `human` after the piping we get the changeset below.

```elixir
pry(1)> human
#Ecto.Changeset<
  action: nil,
  changes: %{
    age_to_die_at: 99,
    name: "John",
    surname: "Doe",
    to_be_born_on: ~U[2016-01-01 00:00:00Z],
    will_get_married: true
  },
  errors: [],
  data: #God.Creature.Human<>,
  valid?: true
>
```

### The question is...

So here, how does `Ecto.Changeset.cast/4` know how to cast attributes into their DB appropriate types? We have only passed `Ecto.Changeset.cast/4` a `Human` schema struct and a map of attributes consisting of only string values.

### The Breakdown

The breakdown will be two-fold. One for when you are passing a schema struct to `cast/4` and the other one for when you are passing a `Ecto.Changeset` to `cast/4`.

---

In the case, when you pass a schema struct to `Ecto.Changeset.cast/4`, it turns out that there's a `__changeset__` function you can call on a schema struct which will give you

```elixir
iex(12)> God.Creature.Human.__changeset__
%{
  age_to_die_at: :integer,
  id: :id,
  inserted_at: :naive_datetime,
  name: :string,
  surname: :string,
  to_be_born_on: :utc_datetime,
  updated_at: :naive_datetime,
  will_get_married: :boolean
}
```

and this `__changeset__` function gets injected by a private `schema/4` function which is called by `schema` and `embedded_schema` macros. You can [take a look at them on GitHub](https://github.com/elixir-ecto/ecto/blob/master/lib/ecto/schema.ex#L516).

If you further inspect the [`Ecto.Changeset.cast/4`](https://github.com/elixir-ecto/ecto/blob/master/lib/ecto/changeset.ex#L489-L491) function which accepts a struct, you will see that the `__changeset__` function is being used like so:

```elixir
def cast(%{__struct__: module} = data, params, permitted, opts) do
  cast(data, module.__changeset__(), %{}, params, permitted, opts)
end
```

If you create your own struct and try to pass it to `Ecto.Changeset.cast/4` you will obviously get an error.

```bash
iex(18)> defmodule Foo do
...(18)> defstruct [:name, :age]
...(18)> end
{:module, Foo,
 <<70, 79, 82, 49, 0, 0, 6, 152, 66, 69, 65, 77, 65, 116, 85, 56, 0, 0, 0, 180,
   0, 0, 0, 18, 10, 69, 108, 105, 120, 105, 114, 46, 70, 111, 111, 8, 95, 95,
   105, 110, 102, 111, 95, 95, 10, 97, 116, ...>>, %Foo{age: nil, name: nil}}
   
iex(19)> Ecto.Changeset.cast(%Foo{}, %{name: "Burak"}, [:name])
** (UndefinedFunctionError) function Foo.__changeset__/0 is undefined or private
    Foo.__changeset__()
    (ecto 3.6.2) lib/ecto/changeset.ex:489: Ecto.Changeset.cast/4
```

---

In the case, when you pass an `Ecto.Changeset` to [`Ecto.Changeset.cast/4`](https://github.com/elixir-ecto/ecto/blob/master/lib/ecto/changeset.ex#L482-L487) below function gets invoked.

```elixir
def cast(%Changeset{changes: changes, data: data, types: types, empty_values: empty_values} = changeset,
		    params, permitted, opts) do
  opts = Keyword.put_new(opts, :empty_values, empty_values)
  new_changeset = cast(data, types, changes, params, permitted, opts)
  cast_merge(changeset, new_changeset)
end
```

As you can also see below, there's a field called `types` in an `Ecto.Changeset` hence when you pass an `Ecto.Changeset` to `cast/4`, Ecto will use the `types` field to figure out how to cast attributes properly.

```bash
iex(63)> ch = God.Creature.Human.changeset(%God.Creature.Human{}, %{name: "John", surname: "Doe", to_be_born_on: "2022-05-05 00:00:00", age_to_die_at: "99"})
#Ecto.Changeset<
  action: nil,
  changes: %{
    age_to_die_at: 99,
    name: "John",
    surname: "Doe",
    to_be_born_on: ~U[2022-05-05 00:00:00Z]
  },
  errors: [],
  data: #God.Creature.Human<>,
  valid?: true
>
iex(64)> ch.types
%{
  age_to_die_at: :integer,
  id: :id,
  inserted_at: :naive_datetime,
  name: :string,
  surname: :string,
  to_be_born_on: :utc_datetime,
  updated_at: :naive_datetime,
  will_get_married: :boolean
}
```

---

### Lastly

You don't always need a schema struct or an `Ecto.Changeset` to be able to use `cast/4`. As you can see in [`lib/ecto/changeset.ex`](https://github.com/elixir-ecto/ecto/blob/master/lib/ecto/changeset.ex#L474-L476)

```elixir
def cast({data, types}, params, permitted, opts) when is_map(data) do
  cast(data, types, %{}, params, permitted, opts)
end
```

You can just pass a tuple consisting of a `data` map and a `types` map which has the types of the values like so:

```bash
iex(15)> data = %{}
%{}
iex(16)> types = %{name: :string, age: :integer, likes_elixir: :boolean}
%{age: :integer, likes_elixir: :boolean, name: :string}
iex(17)> Ecto.Changeset.cast({ %{}, types}, %{name: "Burak", age: "99", likes_elixir: "true"}, [:name, :age, :likes_elixir])
#Ecto.Changeset<
  action: nil,
  changes: %{age: 99, likes_elixir: true, name: "Burak"},
  errors: [],
  data: %{},
  valid?: true
>
```

![Cheers!](https://media.giphy.com/media/8Iv5lqKwKsZ2g/source.gif)

