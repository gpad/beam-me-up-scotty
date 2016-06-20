## OTP
Remember the Greater ?

```elixir
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
```

What happen if we send a message that are not _handled_ ???

```elixir

send pid, {:msg, "ciao"}

send pid, {:other, "ciao"}

(1..1000) |> Enum.each(fn _ -> send pid, {:other, "ciao" } end)

:erlang.process_info pid, [:memory, :message_queue_len]

(1..1000) |> Enum.each(fn _ -> send pid, {:other, "ciao" } end)

:erlang.process_info pid, [:memory, :message_queue_len]
```

If you need more ...

```elixir
:observer.start
```

when should create a receiver like this

```elixir
defmodule Greater do
  def run do
    receive do
      {:msg, s} ->
        IO.puts "Grater say: #{s}"
      _ ->
        IO.puts "Unknown message"
    end
    Greater.run()
  end
end

pid = spawn Greater, :run, []

send pid, {:msg, "ciao"}

send pid, {:other, "ciao"}
```

But also with `spawn`, `spawn_monitor` and `spawn_link`, OTP have a lot of
defined pattern.

```elixir
defmodule GreaterServer do
  use GenServer

  def start() do
    GenServer.start(__MODULE__, [])
  end

  def handle_info({:msg, s}, state) do
    IO.puts "Received from server: #{s}"
    {:noreply, state}
  end
end

{:ok, pid } = GreaterServer.start

send pid, {:msg, "Other ciaoo" }

send pid, {:other, "Other ciaoo" } # --> Boom

Process.alive? pid

```

We can solve in this way

```elixir
defmodule GreaterServer do
  use GenServer

  def start() do
    GenServer.start(__MODULE__, [])
  end

  def handle_info({:msg, s}, state) do
    IO.puts "Received from server: #{s}"
    {:noreply, state}
  end

  def handle_info(_, state) do
    IO.puts "Unknown message ..."
    {:noreply, state}
  end
end

{:ok, pid } = GreaterServer.start

send pid, {:msg, "Other ciaoo" }

send pid, {:other, "Other ciaoo" } # --> Boom

Process.alive? pid

```

Or we can manage the crash with OTP library using Supervisor

```elixir
defmodule GreaterServerSupervisor do
  use Supervisor

  def start_link do
    Supervisor.start_link(__MODULE__, [])
  end

  def init([]) do
    children = [ worker(GreaterServer, [[], [name: :greater]]) ]
    supervise(children, strategy: :one_for_one)
  end
end

defmodule GreaterServer do
  use GenServer

  def start_link(_, opts \\ []) do
    GenServer.start_link(__MODULE__, [], opts)
  end

  def handle_info({:msg, s}, state) do
    IO.puts "Received from server: #{s}"
    {:noreply, state}
  end
end

```

So remember the wrong message

```elixir
{:ok, pid} = GreaterServer.start_link([])
send pid, {:msg, "Ciao"}
send pid, {:other, "Ciao"}
```

Use the Supervisor

```elixir
GreaterServerSupervisor.start_link

pid = Process.whereis :greater

send pid, {:msg, "Ciao"}
send pid, {:other, "Ciao"}

pid = Process.whereis :greater
Process.alive? pid

send :greater, {:msg, "Ciao"}
send :greater, {:other, "Ciao"}
send :greater, {:msg, "Ciao"}
pid = Process.whereis :greater
send :greater, {:other, "Ciao"}
pid = Process.whereis :greater
Process.alive? pid

```

But not only we can create async process cna comunicate in very _simple_ way.

```elixir
defmodule GreaterServer do
  use GenServer

  def start_link(name, opts \\ []) do
    GenServer.start_link(__MODULE__, name, opts)
  end

  def say(pid, message) do
    GenServer.call(pid, {:say, message})
  end

  def handle_call({:say, message}, from, name) do
    IO.puts "#{name} say: #{message} from: #{inspect from}"
    {:reply, nil, name}
  end

end

{:ok, pid} = GreaterServer.start_link("Gino")

GreaterServer.say(pid, "Ciao")

```

All calls have a timeout

```elixir
defmodule GreaterServer do
  use GenServer

  def start_link(name, opts \\ []) do
    GenServer.start_link(__MODULE__, name, opts)
  end

  def say(pid, message, time) do
    GenServer.call(pid, {:long_say, message, time})
  end

  def handle_call({:long_say, message, time}, from, name) do
    IO.puts "#{name} say: #{message} from: #{inspect from}"
    IO.puts "Wait #{time} msec"
    :timer.sleep(time)
    IO.puts "DONE!!!"
    {:reply, nil, name}
  end
end

spawn_link fn ->
  {:ok, pid} = GreaterServer.start_link("Gino")
  GreaterServer.say(pid, "Ciao", 1000)
end

spawn_link fn ->
  {:ok, pid} = GreaterServer.start_link("Gino")
  GreaterServer.say(pid, "Ciao", 7000)
end

```
