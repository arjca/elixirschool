%{
  version: "1.0.4",
  title: "Concurrence dans OTP",
  excerpt: """
  Nous avons précédemment passé en revue les abstractions d'Elixir pour la programmation concurrente. 
  Cependant, nous avons parfois besoin d'un plus grand contrôle, ce pour quoi nous pouvons utiliser OTP.
  
  Dans cette leçon, nous nous concentrons sur la partie la plus importante de OTP : GenServer.
  """
}
---

## GenServer

Un serveur *OTP* est un module qui implémente le comportement (en anglais : *Behaviour*) `GenServer`. Pour faire simple, un `GenServer` est un processus qui répond en boucle à des messages, ainsi qu'un état.

Afin de présenter l'*API* `GenServer`, nous allons implémenter une file basique pour stocker et retrouver des valeurs.

Pour commencer, il nous faut le nécessaire pour démarrer le serveur *OTP* et initialiser l'état de la file.
Pour démarrer le processus, nous emploierons `GenServer.start_link/3` dans la plupart des cas ; ici, nous passons en argument le module courant (nous créons un processus `GenServer` du module `SimpleQueue`), l'état initial de la file, ainsi que des options (ici, un nom que nous pourrons à la place du *PID* du serveur *OTP*).
L'état initial de la file est utilisé par `GenServer.init/1`, dont la valeur de retour initialise l'état du serveur *OTP*. Dans notre exemple, l'état passé en argument dans `SimpleServer.start_link/1` sera l'état initial de la file :

```elixir
defmodule SimpleQueue do
  use GenServer

  @doc """
  Start our queue and link it.
  This is a helper function
  """
  def start_link(state \\ []) do
    GenServer.start_link(__MODULE__, state, name: __MODULE__)
  end

  @doc """
  GenServer.init/1 callback
  """
  def init(state), do: {:ok, state}
end
```

### Fonctions synchrones

Il est souvent utile d'interagir avec un `GenServer` de façon synchrone, c'est-à-dire en attendant la réponse. Pour traiter des requêtes synchrones, nous devons implémenter `GenServer.handle_call/3`, qui prend en argument la requête, le *PID* du processus auteur de la requête, et l'état actuel. La réponse attendue d'une telle fonction est normalement un tuple de la forme : `{:reply, response, state}` ; la liste des valeurs acceptées en retour de cette fonction peut être trouvé dans la documentation de [`GenServer.handle_call/3`](https://hexdocs.pm/elixir/GenServer.html#c:handle_call/3)

Grâce au filtrage par motif, nous pouvons définir une fonction `GenServer.handle_call/3` par type de requête et état courant. Dans notre exemple, implémentons deux fonctions pour montrer l'état courant de la file et pour retirer la prochaine valeur :

```elixir
defmodule SimpleQueue do
  use GenServer

  ### GenServer API

  @doc """
  GenServer.init/1 callback
  """
  def init(state), do: {:ok, state}

  @doc """
  GenServer.handle_call/3 callback
  """
  def handle_call(:dequeue, _from, [value | state]) do
    {:reply, value, state}
  end

  def handle_call(:dequeue, _from, []), do: {:reply, nil, []}

  def handle_call(:queue, _from, state), do: {:reply, state, state}

  ### Client API / Helper functions

  def start_link(state \\ []) do
    GenServer.start_link(__MODULE__, state, name: __MODULE__)
  end

  def queue, do: GenServer.call(__MODULE__, :queue)
  def dequeue, do: GenServer.call(__MODULE__, :dequeue)
end
```

Testons dans le *REPL* :

```elixir
iex> SimpleQueue.start_link([1, 2, 3])
{:ok, #PID<0.90.0>}
iex> SimpleQueue.dequeue
1
iex> SimpleQueue.dequeue
2
iex> SimpleQueue.queue
[3]
```

### Fonctions asynchrones

De manière analogue aux fonctions synchrones, une requête asynchrone (c.-à-d. dont nous n'attendons pas le résultat) est traitée par `GenServer.handle_cast/2`. Une différence : le *PID* de l'appelant ne fait pas partie des arguments.

Nous allons implémenter l'ajout d'une valeur à une file comme une fonction asynchrone, c'est-à-dire que la mise à jour de la file ne bloquera pas l'exécution du programme :

```elixir
defmodule SimpleQueue do
  use GenServer

  ### GenServer API

  @doc """
  GenServer.init/1 callback
  """
  def init(state), do: {:ok, state}

  @doc """
  GenServer.handle_call/3 callback
  """
  def handle_call(:dequeue, _from, [value | state]) do
    {:reply, value, state}
  end

  def handle_call(:dequeue, _from, []), do: {:reply, nil, []}

  def handle_call(:queue, _from, state), do: {:reply, state, state}

  @doc """
  GenServer.handle_cast/2 callback
  """
  def handle_cast({:enqueue, value}, state) do
    {:noreply, state ++ [value]}
  end

  ### Client API / Helper functions

  def start_link(state \\ []) do
    GenServer.start_link(__MODULE__, state, name: __MODULE__)
  end

  def queue, do: GenServer.call(__MODULE__, :queue)
  def enqueue(value), do: GenServer.cast(__MODULE__, {:enqueue, value})
  def dequeue, do: GenServer.call(__MODULE__, :dequeue)
end
```

Testons :

```elixir
iex> SimpleQueue.start_link([1, 2, 3])
{:ok, #PID<0.100.0>}
iex> SimpleQueue.queue
[1, 2, 3]
iex> SimpleQueue.enqueue(20)
:ok
iex> SimpleQueue.queue
[1, 2, 3, 20]
```

Pour plus de renseignements, veuillez consulter la documentation officielle de [GenServer](https://hexdocs.pm/elixir/GenServer.html#content).
