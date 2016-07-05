footer: Thibaut Barrère (thibaut.barrere@gmail.com) - juillet 2016

## [fit] Rails -> Phoenix

## [fit] Retour d'expérience sur une
## [fit] rewrite (en cours) d'application SaaS.

---

# Pourquoi une rewrite?

* Elixir flaggé "stratégique" sur mon radar tech :sunglasses:
  * Produits SaaS et applications internes (dev solo)
  * Consulting (scalabilité, ETL, ...)
  * Robotique, IoT, ETL, Streaming. 
* Rewrite = apprentissage & comparaison
* Comparaison des écosystèmes

---

# Comment débuter

* "Programming Elixir" de A à Z
* Koans (http://github.com/thbar/elixir-playground)

```elixir
  test "function declaration and invocation" do
    sum = fn (a, b) -> a + b end
    assert sum.(2, 10) == 12

    tuple_sum = fn { a, b, c, d } -> a + b + c + d end
    assert tuple_sum.({ 10, 100, 1000, 10000 }) == 11110
  end
```

--- 

# Frictions

* Vendor lock-in Ruby:
  * Productivité très forte.
  * Ecosystème très mûr et large.
  * Zone de confort énorme.

* Accepter d'être lent au début.

--- 

# Motivations

* Faire plus avec moins de personnes.
* Scaler plus fort avec moins de machines.
* Aller vers l'internet des choses, le streaming, le flux continu de données, le très scalable, le redondant.

* Même "bon feeling" qu'avec Rails en 2005.

--- 

# [fit] T.B.D.D.D.D

### [fit] Tests & base de données driven-development

---

<br>

## Exporter une base de dév depuis Rails

```
$ rake db:seed
$ pg_dump -O -x wisecash_dev -f legacy-dumps/rails-db-seed.sql
```

---

<br>
# Importer la base Rails vers Ecto

```
$ ecto.load --dump-path legacy-dumps/rails-db-seed.sql", 
```


---

# Importer la base Rails vers Ecto, automatiquement, avant les tests

```ruby
defmodule WisecashEx.Mixfile do
  use Mix.Project

  defp aliases do
    [
      ...
      "test": [
      	"ecto.create --quiet", 
      	"ecto.load --dump-path legacy-dumps/rails-db-seed.sql", 
      	"test"
      ]
    ]
  end
end
```

---

<br>

## Continuous integration :heart:

<br>

--- 

# Se mettre à l'aise pour expérimenter

* Feedback loop rapide avec `mix_test_watch`
* Tests _exploratoires_
* Colorisation de l'output avec `apex`

```ruby
defp deps do
  [
    {:mix_test_watch, "~> 0.2", only: :dev},
    {:apex, "~> 0.4.0", only: [:dev, :test]}
  ]
end
```

---

# Zoomer sur un problème à la fois

```ruby
ExUnit.configure(
	exclude: :test,
	include: :focus,
  ...
)
```

```ruby
@tag :focus
def test_this do
  # snip
end
```

---

# Débugger une dépendance facilement

```
$ atom deps/openmaize_jwt/lib
```

```ruby
config :mix_test_watch, 
  [
    "deps.compile openmaize_jwt",
    "test"
  ]
```

--- 

# Mise sous intégration continue

circle.yml

```
machine:
  environment:
    TEST_POSTGRES_USERNAME: postgres
dependencies:
  pre:
    - ./circle_pre_build.sh
  cache_directories:
    - ~/dependencies
    - ~/.mix
    - _build
    - deps
test:
  override:
    - ./run_ci.sh
    - mkdir -p $CIRCLE_TEST_REPORTS/exunit
    - cp _build/test/test-junit-report.xml $CIRCLE_TEST_REPORTS/exunit
```

mix.exs -> junit_formatter
formatters = [JUnitFormatter, ExUnit.CLIFormatter]

# Queries SQL sur des fonctions

```
Logger.configure(level: :debug)
```

```
Ecto.Adapters.SQL.query!(WisecashEx.Repo, query, [from, to])
```

```
  def postgres_setup!(repo) do
    source = File.read!(query_file("generate_recurring_events.sql"))
    Ecto.Adapters.SQL.query!(repo, source, [])
  end
```

# Ecto <=> ActiveRecord

```
  schema "entries" do
    # remap Rails created_at
    timestamps inserted_at: :created_at
  end
```

# Calculs - dans la base? Ou en Elixir?

* récurrence de dates infinies avec prise en compte d'exceptions locales
* c'est un peu compliqué :-)
* j'ai commencé en SQL (et je finirai par utiliser ça)
* pour l'instant je recalcule en Elixir
* petite réécriture minimaliste

0/ ruby recurrence gem
1/ generate_series => pas bon
2/ custom generate_series => compliqué
3/ elixir -> 2h de travail et terminé

# Authentification

JWT WTF. Guardian -> pas suffisamment clé en main au début (ne gère pas la base, les routes, le reset password etc). Ou est devise? Comme Rails en 2005 -> librairies d'authent changent tous les 6 mois.

```
  schema "users" do
    field :email, :string
    field :encrypted_password, :string
    field :password, :string, virtual: true
    field :role, :string, virtual: true, default: "user"
    timestamps inserted_at: :created_at
  end
```

```
config :openmaize,
  hash_name: :encrypted_password
```

Génère beaucoup de code, mais au final ça fonctionne.

# Test d'acceptance

hound + phantomjs. Sandbox <3.

```
  test "login and logout" do
    navigate_to("/")
    "/" = current_path
    
    find_element(:link_text, "Login")
    |> click
    "/login" = current_path

    find_element(:id, "user_email")
    |> fill_field("john@example.com")

    find_element(:id, "user_password")
    |> fill_field("testtest")
    
    find_element(:tag, "form")
    |> submit_element

    assert visible_page_text =~ ~r/Logged as john@example.com/i
    
    find_element(:link_text, "Logout")
    |> click

    assert visible_page_text =~ ~r/You have been logged out/i
  end
```

# Beaucoup de duct-tape en moins.

* Pusher (externe) -> ActionCable (interne) -> Channels (encore plus interne)


Je dump le seed de Rails pour obtenir non seulement le schéma mais aussi un jeu de données valable. Je commence à développer une application Elixir/Phoenix/Ecto. Je trouve comment dire à Ecto qu’il faut restaurer la base depuis le dump SQL. Je mets le tout en intégration continue avec CircleCI. Je commence à jouer avec un modèle qui requiert des calculs un peu complexes de récurrence. Je me place dans un contexte où je peux avoir une boucle rétro-active rapide avec mix test watch. J’ai appris Elixir avant tout avec une méthode de Koans, en implémentant un test ExUnit par exercice de “programming elixir”. Je vois qu’il faut remapper un champ pour le inserted_at. J’essaye de déplacer de la logique vers la base de données. Pas forcément simple d’écrire la query qu’on veut avec Ecto (au moins avant la v2), mais je finis par trouver comment on génère une stored procedure, puis comment on l’invoque. C’est moins puissant que Sequel comme query builder pour le moment. Par contre les données immutables donnent une sensation de solidité du code qui est agréable. Je tatonne puis je finis par ré-écrire la partie logique en Elixir, adaptation de la librairie de récurrence Ruby, en peu de lignes. Je compte utiliser du fuzzy testing qui compare ruby et elixir sur les données de dév puis de production pour comparer l’output sur la totalité des questions possibles. Je déploie relativement facilement sur Heroku. Je vois comment déployer à terme sur EngineYard, mais non supporté en standard. Je me mets en recherche d’une solution “prête à l’emploi” pour l’authentification, je trouve Guardian, je me dis “WTF JWT”. Il s’avère que c’est moins “clé en main” que Devise. Je cherche un peu plus loin et je trouve OpenMaize. Il m’a fallu pas mal d’effort pour comprendre les paradigmes et adapter, et j’ai grosso-modo ce qu’il faut. J’ai vu que ça “auto-modifiait” les clés avec un process. Je ne suis pas fan de l’approche. J’aimerais une librairie clé en main avec des patchs de sécurité et le moins de code custom possible. Pour bien développer, j’ai directement sauté sur une librairie qui permette de faire des tests d’acceptance end-to-end, et j’ai trouvé hound. C’est vraiment un équivalent de Capybara. Il faut lancer phantom --wd à côté pour le moment. Grosso-modo c’est encore un peu jeune mais je trouve que les bases sont solides et que la maintenance longue durée (TCO) sera plus basse qu’avec Rails. Je vais m’appuyer sur des features comme les channels pour faire du simple “refresh page outdated”.