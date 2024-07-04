%{
  version: "1.1.2",
  title: "Superviseurs OTP",
  excerpt: """
  Les superviseurs sont des processus spécialisés pour accomplir une mission : surveiller les autres processus.
  Les superviseurs permettent de construire des applications tolérantes aux pannes en redémarrant automatiquement les processus-enfants lors d'un échec.
  """
}
---

## Configuration

La magie des superviseurs *OTP* réside dans la fonction `Supervisor.start_link/2`.
En plus de démarrer le superviseur et ses enfants, elle permet de définir la stratégie du superviseur pour la gestion de ses enfants.

Reprenons l'exemple `SimpleQueue` de la leçon [Concurrence dans OTP](/fr/lessons/advanced/otp_concurrency). Commençons par créer un nouveau projet avec un arbre de supervision en exécutant la commande suivante : `mix new simple_queue --sup`. Puis, déposons le code de `SimpleQueue` dans le fichier `lib/simple_queue.ex`. Le code lié au superviseur sera écrit dans `lib/simple_queue/application.ex`.

Les processus-enfants sont définis dans une liste. Cela peut être une liste de noms de modules, comme suit :

```elixir
defmodule SimpleQueue.Application do
  use Application

  def start(_type, _args) do
    children = [
      SimpleQueue
    ]

    opts = [strategy: :one_for_one, name: SimpleQueue.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

Si des informations supplémentaires sont nécessaires (p. ex. un état initial), la liste peut être peuplée de tuples :

```elixir
defmodule SimpleQueue.Application do
  use Application

  def start(_type, _args) do
    children = [
      {SimpleQueue, [1, 2, 3]}
    ]

    opts = [strategy: :one_for_one, name: SimpleQueue.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

En lançant `iex -S mix`, nous constatons que `SimpleQueue` est automatiquement démarré :

```elixir
iex> SimpleQueue.queue
[1, 2, 3]
```

Si le processus `SimpleQueue` devait être interrompu, le superviseur le redémarrerait automatiquement, comme si rien ne s'était produit.

### Stratégies du superviseur

Il y a actuellement trois stratégies disponibles pour les superviseurs :

+ `:one_for_one` - Redémarre uniquement les processus-enfants en panne.

+ `:one_for_all` - Redémarre tous les processus-enfants en cas de panne.

+ `:rest_for_one` - Redémarre les processus-enfants et tous les processus démarrés après lui.

## Spécification des processus-enfants

Une fois le superviseur démarré, il a besoin de savoir comment démarrer, stopper et redémarrer ses enfants.
Chaque module enfant devrait avoir une fonction `child_spec/1` pour définir ces comportements.
Les macros `use GenServer`, `use Supervisor` et `use Agent` définissent automatiquement cette fonction pour nous (ce qui est le cas de notre exemple `SimpleQueue`, car il est un `GenServer`).

S'il vous faut définir vous-même `child_spec/1`, sachez qu'elle doit retourner une table d'options :

```elixir
def child_spec(opts) do
  %{
    id: SimpleQueue,
    start: {__MODULE__, :start_link, [opts]},
    shutdown: 5_000,
    restart: :permanent,
    type: :worker
  }
end
```

+ `id` - Requise.
  Valeur utilisée par le superviseur pour identifier le processus-enfant.

+ `start` - Requise.
  Le module/fonction/arguments utilisés par le superviseur pour démarrer le processus-enfant.

+ `shutdown` - Optionnelle.
  Comportement du processus-enfant lors de son extinction.
  Les options sont :
  
  + `:brutal_kill` - Le processus-enfant est arrêté immédiatement.

  + `0` ou un entier positif : le nombre de millisecondes qu'attend le superviseur avant de tuer le processus-enfant.
  
    Si le processus est du type `:worker`, la valeur par défaut de `shutdown` est `5000`.

  + `:infinity` - Le superviseur attendra indéfiniment avant de tuer le processus-enfant.
  
    C'est le comportement par défaut pour le type `:supervisor`. Nous le recommandons pas pour les processus du type `:worker`.

+ `restart` - Optionnelle.

  Il y a plusieurs approche pour traiter l'interruption d'un processus-enfant :

  + `:permanent` - Le processus-enfant est toujours redémarré.
    C'est la valeur par défaut pour tous les processus.

  + `:temporary` - Le processus-enfant n'est jamais redémarré.

  + `:transient` - Le processus-enfant est redémarré uniquement s'il se termine anormalement.

+ `type` - Optionnelle.
  Les processus peuvent être `:worker` ou `:supervisor`.
  La valeur par défaut est `:worker`.

## Superviseur dynamique

Les superviseurs démarrent avec une liste de processus-enfants au lancement de l'application. Cependant, l'ensemble des processus-enfants peut parfois ne pas être connue d'avance. Par exemple, une application Web pourrait démarrer un nouveau processus pour chaque nouvel utilisateur qui se connecte.
Pour ce genre de cas, nous voulons un superviseur capable de démarrer des processus-enfants sur demande : un superviseur dynamique.

Comme nous ne spécifions pas les enfants, nous avons seulement à définir les options d'exécution du superviseur. `DynamicSupervisor` ne tolère que la stratégie `:one_for_one` :

```elixir
options = [
  name: SimpleQueue.Supervisor,
  strategy: :one_for_one
]

DynamicSupervisor.start_link(options)
```

Ensuite, afin de démarrer dynamiquement une file `SimpleQueue`, nous allons utiliser `DynamicSupervisor.start_child/2` qui prend en argument un superviseur et une spécification de processus-enfant. À nouveau, nous n'avons rien à préciser dans notre exemple car `SimpleQueue` est un `GenServer`.

```elixir
{:ok, pid} = DynamicSupervisor.start_child(SimpleQueue.Supervisor, SimpleQueue)
```

## Superviseur de tâches

Les tâches ont leur propre superviseur spécialisé : `Task.Supervisor`.
Il peut démarrer dynamiquement des tâches.

### Configuration

Configurer un `Task.Supervisor` est analogue aux autres superviseurs : 

```elixir
children = [
  {Task.Supervisor, name: ExampleApp.TaskSupervisor, restart: :transient}
]

{:ok, pid} = Supervisor.start_link(children, strategy: :one_for_one)
```

La principale différence entre `Supervisor` et `Task.Supervisor` réside dans sa stratégie de redémarrage par défaut : `:temporary` (une tâche ne sera jamais redémarrée).

### Tâches supervisées

Une fois le superviseur démarré, nous pouvons créer une nouvelle tâche supervisée avec `Task.Supervisor.start_child/2` :

```elixir
{:ok, pid} = Task.Supervisor.start_child(ExampleApp.TaskSupervisor, fn -> background_work end)
```
