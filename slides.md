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
$ ecto.load --dump-path legacy-dumps/rails-db-seed.sql
```


---
# Importer la base Rails vers Ecto, automatiquement, avant les tests

```ruby
defp aliases do
  [
    "test": [
    	"ecto.create --quiet", 
    	"ecto.load --dump-path legacy-dumps/rails-db-seed.sql", 
    	"test"
    ]
  ]
end
```

---

<br>

## Continuous integration :heart:

### [fit] \(quelques heures pour la mise en place initiale sur CircleCI\)

<br>

--- 

# Se mettre à l'aise pour expérimenter

* Feedback-loop rapide avec `mix_test_watch`
* Tests "exploratoires"
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

# Adaptations Ecto <-> ActiveRecord

<br>

```ruby
  schema "entries" do
    timestamps inserted_at: :created_at
  end
```

---

# Evènements récurrents

Plusieurs tentatives:

1. Postgres `generate_series` (ne colle pas au besoin) :disappointed:
2. Fonction PL/pgSQL (des contournements à faire) :persevere:
3. Fonction Elixir (terminée en quelques heures) :sunglasses:

Merci `Timex.shift`

---

# [fit] Queries SQL sur des fonctions (avec Ecto 2)

```ruby

Logger.configure(level: :debug)

Ecto.Adapters.SQL.query!(WisecashEx.Repo, query, [from, to])

def postgres_setup!(repo) do
  source = File.read!(query_file("generate_recurring_events.sql"))
  Ecto.Adapters.SQL.query!(repo, source, [])
end
```

* Ecto = plus limité que Sequel en Ruby

---

# Authentification

Choix large (comme avec Rails en 2005):

* Guardian (pas de gestion de BDD, JWT)
* Openmaize (BDD, code généré, JWT) :thumbsup:
* Addict
* ...

Mais quelle solution sera réellement pérenne ???

---

# [fit] Adaptation du schéma de Rails & Devise vers Openmaize

```ruby
schema "users" do
  field :email, :string
  field :encrypted_password, :string
  field :password, :string, virtual: true
  field :role, :string, virtual: true, default: "user"
  timestamps inserted_at: :created_at
end

config :openmaize, hash_name: :encrypted_password
```

---

# Openmaize dans le contrôleur

```ruby
plug Openmaize.Login, [
  db_module: WisecashEx.OpenmaizeEcto,
  unique_id: :email
] when action in [:login_user]
```

Reprise en main du code généré par Openmaize.

--- 

# [fit] Tests d'acceptance (hound + phantomjs :heart:)

```ruby
navigate_to("/")
"/" = current_path

find_element(:link_text, "Login")
|> click
"/login" = current_path

find_element(:id, "user_email")
|> fill_field("john@example.com")
``
find_element(:id, "user_password")
|> fill_field("testtest")
```

--- 

# [fit] Tests d'acceptance (hound + phantomjs :heart:)

```ruby
find_element(:tag, "form")
|> submit_element

assert visible_page_text =~ ~r/Logged as john@example.com/i

find_element(:link_text, "Logout")
|> click

assert visible_page_text =~ ~r/You have been logged out/i
```

---

# Déploiement

* Heroku: build-pack fonctionnel rapidement
* EngineYard: custom cookbook mais pas trop compliqué
* Pas de hot-reloading pour le moment

---

# Mes conclusions pour le moment

* Données immutables = fondation solide
* TDD = aide beaucoup pour prendre en main
* Librairies jeunes (perte de temps - maintenance)
* Confort très similaire à Ruby en régime de croisière
* Moins de "compromis duct-tape" qu'avec Ruby
* Usage confirmé pour web, scalabilité et ETL

---
