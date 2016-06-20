## BEAM

```elixir
length Process.list

Process.list

Process.registered

```

Create some process and check new count

```elixir
spawn(fn -> :timer.sleep(10000); IO.puts("Ciao") end)

length Process.list

```

Can we create a lot of process ...

Execute htop and check memeory and CPU utilizzation.

```elixir

(1..10_000) |>
  Stream.map(fn n ->
    spawn(fn -> :timer.sleep(10000); IO.puts("Ciao #{n}") end)
  end) |>
  Stream.run

(1..100_000) |>
  Stream.map(fn n ->
    spawn(fn -> :timer.sleep(10000); IO.puts("Ciao #{n}") end)
  end) |>
  Stream.run

(1..1_000_000) |>
  Stream.map(fn n ->
    spawn(fn -> :timer.sleep(10000); IO.puts("Ciao #{n}") end)
  end) |>
  Stream.run

```
Can we try in another way ...

```elixir
(1..1_000_000) |>
  Stream.map(fn n ->
    spawn(fn -> :timer.sleep(10000); end)
  end) |>
  Stream.run

```

Ops ...

```
09:39:46.467 [error] Too many processes


** (SystemLimitError) a system limit has been reached
             :erlang.spawn(:erlang, :apply, [#Function<20.50752066/0 in :erl_eval.expr/5>, []])
             :erlang.spawn/1
    (elixir) lib/stream.ex:454: anonymous fn/4 in Stream.map/2
    (elixir) lib/range.ex:80: Enumerable.Range.reduce/5
    (elixir) lib/stream.ex:1247: Enumerable.Stream.do_each/4
    (elixir) lib/stream.ex:494: Stream.run/1
```


```elixir
:erlang.system_info(:process_limit)
```

```bash
iex --erl "+P 1000000"
```

```elixir
:erlang.system_info(:process_limit)

length Process.list

(1..1_000_000) |>
  Stream.map(fn n ->
    spawn(fn -> :timer.sleep(30000); end)
  end) |>
  Stream.run

length Process.list

(1..1_000_000) |>
  Stream.map(fn n ->
    spawn(fn -> :timer.sleep(60000); end)
  end) |>
  Stream.run

length Process.list
```
## Comunication between process

```elixir
send self, { :msg, "Ciao"}
flush
send self, { :msg, "Ciao"}
receive do
  {:msg, s} ->
    IO.puts "Received: #{s}"
end

pid = spawn fn ->
  receive do
    {:msg, s} ->
      IO.puts "Received: #{s}"
  end
end

send pid, {:msg, "Hello" }
send pid, {:msg, "Hello" }

```
 Mmmh??? I need a loop

```elixir
run = fn ->
  receive do
    {:msg, s} ->
      IO.puts "Received: #{s}"
  end
end

pid = spawn run

send pid, {:msg, "Hello" }
send pid, {:msg, "Hello" }

defmodule Greater do
  def run do
    receive do
      {:msg, s} ->
        IO.puts "Grater say: #{s}"
        Greater.run()
    end
  end
end

pid = spawn Greater, :run, []

send pid, {:msg, "Hello" }
send pid, {:msg, "Hello" }

```

## Link & monitor process

```elixir
defmodule TicToc do

  def start_monitor(args \\ [:tic, 10]) do
    spawn_monitor(__MODULE__, :run, args)
  end

  def run(_, 0) do
    IO.puts "BOOOM !!!!"
    1 / 0
  end

  def run(:tic, timeout) do
    IO.puts "tic"
    :timer.sleep(1000)
    run(:toc, timeout - 1)
  end

  def run(:toc, timeout) do
    IO.puts "toc"
    :timer.sleep(1000)
    run(:tic, timeout - 1)
  end
end

{pid, _} = TicToc.start_monitor

Process.alive? pid

flush
```

Can create a process can see this event

```elixir
defmodule TicTocWatcher do

  def start() do
    spawn(__MODULE__, :watch, [])
  end

  def watch() do
    {pid, _} = TicToc.start_monitor
    IO.puts "Process #{inspect self} monitor #{inspect pid}"
    receive do
      {:DOWN, _, _, _, _} ->
        IO.puts "Tic Toc down :-("
    end
  end
end

TicTocWatcher.start

```

There is no bidirectionl relationship ...

```elixir
pid = spawn fn -> TicToc.start_monitor(); :timer.sleep(2000); 1/0 end
Process.alive? pid
```

If you want bidirectionl link you should use link!!!

```elixir
defmodule TicToc do

  def start_link(args \\ [:tic, 10]) do
    spawn_link(__MODULE__, :run, args)
  end

  def run(_, 0) do
    IO.puts "BOOOM !!!!"
    1 / 0
  end

  def run(:tic, timeout) do
    IO.puts "tic"
    :timer.sleep(1000)
    run(:toc, timeout - 1)
  end

  def run(:toc, timeout) do
    IO.puts "toc"
    :timer.sleep(1000)
    run(:tic, timeout - 1)
  end
end

self
pid = TicToc.start_link
```
wait the crash and see the new console

```elixir
self
```

BUT also viceversa

```elixir
pid = spawn fn -> TicToc.start_link(); :timer.sleep(5000); 1/0 end
```

It's possible trap the signal:

```elixir
self
:erlang.process_flag(:trap_exit, true)
spawn_link fn -> 1/0 end
self
flush
```

In the same way as for the monitor we can trap the signal and do whatever we want.
