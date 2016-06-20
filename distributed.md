## Distributed

We can start 4 node. To work in a distributed way, we need to set name to the node, to do taht we need to execute iex in this way:

```sh
iex --sname master
iex --sname s1
iex --sname s2
iex --sname s3
```
Check the name of the node and the list of connected nodes:

```elixir
node
Node.list
```
To connect to other nodes you can execute (tardis is the name of my machine):

```elixir
# from master
Node.connect(:s1@tardis)
Node.list

#from s1
Node.list
Node.connect(:s2@tardis)
Node.list

#from master
Node.list
```

Every node is connected with the others.
Now from one node we can send node to other nodes.
Copy the code in every nodes.

```elixir
defmodule PingPong do
  def start_ping_with(node_name) do
    pid = spawn(fn ->
      send({:pingpong, node_name}, {:ping, self})
      run()
    end)
    Process.register(pid, :pingpong)
  end

  def start_listen() do
    Process.register(spawn(PingPong, :run, []), :pingpong)
  end

  def run do
    receive do
      {:pong, from} ->
        log(:pong, from)
        send(from, {:ping, self})
      {:ping, from} ->
        log(:ping, from)
        send(from, {:pong, self})
      _ -> IO.pus "Unknown message"
    end
    run
  end

  defp log(msg, from) do
    IO.puts "Receive message: #{inspect msg} from: #{inspect from}"
  end
end
```

Auto connect:

```elixir
PingPong.start_listen
PingPong.start_ping_with :master@tardis
PingPong.start_ping_with :s1@tardis
PingPong.start_ping_with :master@tardis
```

Do you remember the GenServer?

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

# locally in s1@tardis

{:ok, pid} = GreaterServer.start_link("Gino")

GreaterServer.say(pid, "Ciao")

Process.register(pid, :greater_server)

GreaterServer.say(:greater_server, "Ciao")

```

On other node master@tardis

```elixir

GreaterServer.say({:greater_server, :s1@tardis}, "Ciao")

```

`Process.register` is not the only way to send message between nodes.
**Close the nodes**

```elixir
defmodule GreaterServer do
  use GenServer

  def start_link(name, opts \\ []) do
    GenServer.start_link(__MODULE__, [name: name, lmf: nil], opts)
  end

  def say(pid, message) do
    GenServer.call(pid, {:say, message})
  end

  def last_message_from(pid) do
    {pid, _} = GenServer.call(pid, {:lmf})
    pid
  end

  def handle_call({:say, message}, from, [name: name, lmf: lmf]) do
    IO.puts "#{name} say: #{message} from: #{inspect from}"
    {:reply, nil, [name: name, lmf: from]}
  end

  def handle_call({:lmf}, from, [name: name, lmf: lmf]) do
    {:reply, lmf, [name: name, lmf: lmf]}
  end

end

# on s1
{:ok, pid} = GreaterServer.start_link("Gino", [name: :greater_server])
GreaterServer.say(pid, "Ciao")
GreaterServer.say(:greater_server, "Ciao")

# on master
GreaterServer.say({:greater_server, :s1@tardis}, "Ciao")

# on s1
from = GreaterServer.last_message_from(pid)
send(from, "RESPONSE!!!!")

#on master
flush
```
