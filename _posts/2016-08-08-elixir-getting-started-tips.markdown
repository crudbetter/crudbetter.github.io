---
layout: post
title:  "5 tips for starting an Elixir journey"
date:   2016-08-29 12:00:00 +0000
categories: elixir
---
I have been fortunate enough to start my Elixir journey in a commercial setting, some 8 months ago. I had neither heard of or dabbled with the language until then. It has become my third significant language change. I started out with C#, then I joined the JavaScript craze (which I'm still a member of), and now I'm a relative early adopter of Elixir.

I believe in the concepts of "learning in public" and "paying it forward", so here are my 5 tips for others starting an Elixir journey.

# Start with a purely functional problem

This was by luck really. The first thing I did with Elixir happened to be a specialised HTTP client for [InfluxDB][influxdb], making use of [HTTPotion][httpotion]. Just a module with a bunch of functions for abstracting away [InfluxQL][influxql] and the [Line Protocol][line_protocol] used to write points. This allowed me to concentrate on Elixir the language, i.e. no processes, no OTP.

For example, take a list of the following struct:

{% highlight elixir %}
defmodule Point do
  @moduledoc """
  Module to help persist a single field, multiple tag point within InfluxDB.
  """

  @typedoc "A single field, multipe tag point."
  @type t :: %__MODULE__{
    field_name: atom,
    field_value: integer,
    tags: Keyword.t,
    timestamp: non_neg_integer
  }
  defstruct field_name: nil, field_value: nil, tags: Keyword.new, timestamp: nil
end
{% endhighlight %}

and reduce it to the line protocol required to write to an InfluxDB measurement called `points`.

# Favour pipelines and function heads over conditionals

My code doesn't pass review and get merged to Master when it includes conditional statements, such as `if`, `case` and `cond`. This can be frustrating at times, but the resulting refactor to pipelines of functions is always easier to understand.

Improve the readability of a module's public functions by using the `|>` operator to chain private functions together, thus creating pipelines. Group public functions at the top of a module and private at the bottom. Use [Railway macros][railway] to elegantly handle error scenarios without making a pipeline harder to read.

Your future self, working on a growing codebase, will thank you. They'll be able to scan a module written several months ago and quickly comprehend what it's public functions are doing. If they want to understand the nitty-gritty details, they can scroll down and study the private functions. This is priceless.

I'll attempt to illustrate with a contrived example, as follows:

{% highlight elixir %}
defmodule Supermarket do
  @moduledoc """
  Module allowing customers to self-scan items and checkout at a Supermarket.

      iex> sm = Supermarket.new |> Supermarket.scan_item(1, :potatoes, 1.25)
      iex> sm |> Supermarket.checkout_case(1)
      1.25
      iex> sm |> Supermarket.checkout_pipeline(1)
      1.25
  """

  alias __MODULE__, as: Mod

  @typedoc "Map of shopping baskets keyed by customer id."
  @type t :: %Mod{
    baskets: %{pos_integer => Keyword.t}
  }
  defstruct baskets: %{}

  def new, do: %Mod{}

  def scan_item(%Mod{baskets: baskets}, cus_id, name, price) do
    initial_basket = Keyword.new([{name, price}])

    updated_baskets = 
      Map.update(baskets, cus_id, initial_basket, &Keyword.put(&1, name, price))
    
    %Mod{baskets: updated_baskets}
  end

  def checkout_case(%Mod{baskets: baskets}, cus_id) do
    case Map.fetch(baskets, cus_id) do
      :error -> 0
      {:ok, items} -> sum_items(items)
    end
  end

  def checkout_pipeline(%Mod{baskets: baskets}, cus_id) do
    baskets
    |> Map.fetch(cus_id)
    |> sum_basket
  end

  defp sum_items(items) do
    Enum.reduce(items, 0, fn ({_name, price}, acc) -> acc + price end)
  end

  defp sum_basket(:error), do: 0
  defp sum_basket({:ok, items}), do: sum_items(items)
end
{% endhighlight %}


# Understand processes before reaching for abstractions

Learn from my mistakes. Understand how to use processes with `Kernel.spawn` and `receive do ... end` blocks first. Then move on to the `GenServer`, `Agent` and `Task` abstractions.

My first attempt at concurrent design started by studying the excellent "Designing a Concurrent Application" chapter of [Learn You Some Erlang][lyse]. I couldn't grasp how to construct a single `GenServer` that used a pool of short-lived helper processes to achieve concurrency. Perhaps attempting to port Erlang to Elixir was a step too far at that time. More likely I simply didn't understand the basics of processes.

So before reaching for a `GenServer`, `Agent` or `Task` abstraction I recommended trying to solve the problem without them first.

The `Supermarket` example I introduced previously forces each caller to maintain state. Converting `Supermarket` to an `Agent` would allow it to maintain it's own state. But as a learning exercise can we achieve the same improvement using `Kernel.spawn` and `receive do ... end` blocks?

The answer is yes, as follows:

{% highlight elixir %}
defmodule Supermarket do
  @moduledoc """
  Module allowing customers to self-scan items and checkout at a Supermarket.

      iex> Supermarket.start_link
      iex> Supermarket.scan_item(1, :potatoes, 1.25)
      iex> Supermarket.scan_item(1, :carrots, 0.90)
      iex> Supermarket.checkout(1)
      2.15
  """

  alias __MODULE__, as: Mod

  @typedoc "Map of shopping baskets keyed by customer id."
  @type t :: %Mod{
    baskets: %{pos_integer => Keyword.t}
  }
  defstruct baskets: %{}

  def start_link do 
    server_pid = Kernel.spawn_link(fn -> loop(%Mod{}) end)
    true = Process.register(server, Mod)
    server_pid
  end

  def scan_item(cus_id, name, price) do
    _msg = Kernel.send(Mod, {:put, cus_id, name, price, Kernel.self})

    receive do
      basket -> basket
    end
  end

  def checkout(cus_id) do
    _msg = Kernel.send(Mod, {:get, cus_id, Kernel.self})

    receive do
      sum -> sum
    end
  end

  defp loop(%Mod{baskets: baskets}) do
    receive do
      {:put, cus_id, name, price, client} ->
        updated_basket = 
          baskets
          |> Map.fetch(cus_id)
          |> add_item(name, price)
        
        updated_baskets = Map.put(baskets, cus_id, updated_basket)

        _msg = Kernel.send(client, updated_basket)
  
        loop(%Mod{baskets: updated_baskets})
      {:get, cus_id, client} ->
        sum =
          baskets
          |> Map.fetch(cus_id)
          |> sum_items

        _msg = Kernel.send(client, sum)

        loop(%Mod{baskets: baskets})
    end
  end

  defp add_item(:error, name, price), do: Keyword.new([{name, price}])
  defp add_item({:ok, basket}, name, price), do: Keyword.put(basket, name, price)

  defp sum_items(:error), do: 0
  defp sum_items({:ok, items}), 
    do: Enum.reduce(items, 0, fn ({_name, price}, acc) -> acc + price end)
end
{% endhighlight %}

# Aid mental separation by enforcing one process per module

It may be my OO background, but for a long time I associated a single module to a single "thing". In reality a module is just a method for organising functions within a codebase. A "thing" is of course a process. I struggled to visualise what was going on when there were multiple processes to a module.

Our learning exercise with `Kernel.spawn` and `receive do ...end` blocks has served it's purpose. So I've migrated `Supermarket` to use the far less verbose `Agent` abstraction. However there still have two processes involved.

To counter any confusion, place spawned functions in separate modules. I'll introduce a `SupermarketServer` module to demonstrate, as follows:

{% highlight elixir %}
defmodule Supermarket do
  @moduledoc """
  Client half of Agent allowing customers to self-scan items 
  and checkout at a Supermarket.

      iex> Supermarket.start_link
      iex> Supermarket.scan_item(1, :potatoes, 1.25)
      iex> Supermarket.scan_item(1, :carrots, 0.90)
      iex> Supermarket.checkout(1)
      2.15
  """
  alias __MODULE__, as: Mod
  alias SupermarketServer, as: Server

  def start_link do 
    Agent.start_link(Server, :init, [], name: Mod)
  end

  def scan_item(cus_id, name, price) do
    Agent.update(Mod, Server, :update_basket, [cus_id, name, price])
  end

  def checkout(cus_id) do
    Agent.get(Mod, Server, :sum_basket, [cus_id])
  end
end

defmodule SupermarketServer do
  @moduledoc """
  Server half of Agent allowing customers to self-scan items 
  and checkout at a Supermarket.
  """
  alias __MODULE__, as: Mod

  @typedoc "Map of shopping baskets keyed by customer id."
  @type t :: %Mod{
    baskets: %{pos_integer => Keyword.t}
  }
  defstruct baskets: %{}

  def init(), do: %Mod{}

  def update_basket(%Mod{baskets: baskets}, cus_id, name, price) do
    updated_basket =
      baskets
      |> Map.fetch(cus_id)
      |> add_item(name, price)

    %Mod{baskets: Map.put(baskets, cus_id, updated_basket)}
  end

  def sum_basket(%Mod{baskets: baskets}, cus_id) do
    baskets
    |> Map.fetch(cus_id)
    |> sum_items
  end

  defp add_item(:error, name, price), do: Keyword.new([{name, price}])
  defp add_item({:ok, basket}, name, price), do: Keyword.put(basket, name, price)

  defp sum_items(:error), do: 0
  defp sum_items({:ok, items}), 
    do: Enum.reduce(items, 0, fn ({_name, price}, acc) -> acc + price end)
end
{% endhighlight %}


# Clarify intent by always pattern match function return values

You may have noticed I've adopted a strange practise throughout this post. I've pattern matched every value a function returns, even it's not used elsewhere. For example `_msg = Kernel.send(Mod, {:put, cus_id, name, price, Kernel.self})`. In functional programming not matching a return value suggests a side-effect only function. I prefer to be explicit with my code, any practise that adds greater clarity to the intent of my code is a good thing. If you're a [Dialyzer][dialyzer] user you may want to consider turning on the `Wunmatched_returns` warning.

# That's all for now folks

I've thoroughly enjoyed my journey with Elixir so far. During short periods of JavaScript development I've found the lack of immutable functions a jarring experience. This is something I hadn't truly appreciated prior to learning Elixir.

I hope you've found this post useful. If so, consider following me on [Twitter][twitter].

See you later!

[railway]: http://www.zohaib.me/railway-programming-pattern-in-elixir/
[influxdb]: https://influxdata.com/time-series-platform/influxdb/
[influxql]: https://docs.influxdata.com/influxdb/v0.13/query_language/data_exploration/
[line_protocol]: https://docs.influxdata.com/influxdb/v0.13/write_protocols/line/
[httpotion]: https://github.com/myfreeweb/httpotion
[lyse]: http://learnyousomeerlang.com/designing-a-concurrent-application
[dialyzer]: http://erlang.org/doc/man/dialyzer.html
[twitter]: https://twitter.com/crudbetter