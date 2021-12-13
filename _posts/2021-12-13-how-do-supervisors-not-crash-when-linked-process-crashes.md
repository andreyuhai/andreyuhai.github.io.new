---
title: How do supervisors not crash when one of the children processes crashes
tags: [elixir, supervisor, agent, genserver]
categories: [TIL, Elixir]
---

When you start an `Agent` or `GenServer` they will be **linked** to the current process and the current process will crash if the **linked** `Agent` or `GenServer` crashes.
However `Supervisor`s do not crash even if the child processes that are **linked** to the supervisor crash. Well, the answer is obvious.

Because `Supervisor`s trap the exit signals.<sup>[source code](https://github.com/erlang/otp/blob/maint/lib/stdlib/src/supervisor.erl#L329)</sup>

Assuming that you've at least one child spec in your `children` list below, you can also confirm that doing:

```elixir
{:ok, supervisor} = Supervisor.start_link(children, strategy: :one_for_one)

Process.info(supervisor, :trap_exit)
# => {:trap_exit, true}
```
