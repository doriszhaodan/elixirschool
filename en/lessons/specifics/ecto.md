---
version: 2.0.0
title: Ecto
---

Ecto is an official Elixir project providing a database wrapper and integrated query language.  With Ecto we're able to create migrations, define schemas, insert and update records, and query them.

{% include toc.html %}

## Setup

Create a new app with a supervision tree:

```shell
$ mix new example_app --sup
$ cd example_app
```

To get started we need to include Ecto and a database adapter in our project's `mix.exs`. Ecto has 2 hex packages, `ecto` and `ecto_sql`. `ecto` only focus on the building blocks of its features
we'll talk about in this lesson, `ecto_sql` provides SQL adapters. So to use Ecto with a database, we need to include `ecto_sql`. You can find a list of supported database adapters in
the [Usage](https://github.com/elixir-lang/ecto/blob/master/README.md#usage) section of the Ecto README.  For our example we'll use PostgreSQL:

```elixir
defp deps do
  [
    {:ecto_sql, "~> 3.0"},
    {:postgrex, ">= 0.0.0"}
  ]
end
```

Then we'll fetch our dependencies using

```shell
$ mix deps.get
```

### Repository

Finally we need to create our project's repository, the database wrapper.  This can be done via the `mix ecto.gen.repo -r ExampleApp.Repo` task.  We'll cover Ecto mix tasks next.  The Repo can be found in `lib/example_app/repo.ex`:

```elixir
defmodule ExampleApp.Repo do
  use Ecto.Repo,
    otp_app: :example_app,
    adapter: Ecto.Adapters.Postgres
end
```

### Supervisor

Once we've created our Repo we need to set up our supervisor tree, which is found in `lib/example_app/application.ex`. Add the Repo to the `children` list:

```elixir
defmodule ExampleApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      ExampleApp.Repo
    ]

    opts = [strategy: :one_for_one, name: ExampleApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

For more info on supervisors check out the [OTP Supervisors](../../advanced/otp-supervisors) lesson.

### Configuration

To configure Ecto we need to add a section to our `config/config.exs`.  Here we'll specify the repository, database, and account information:

```elixir
config :example_app, :ecto_repos, [ExampleApp.Repo]

config :example_app, ExampleApp.Repo,
  database: "example_app",
  username: "postgres",
  password: "postgres",
  hostname: "localhost"
```

## Mix Tasks

Ecto includes a number of helpful mix tasks for working with our database:

```shell
mix ecto.create         # Create the storage for the repo
mix ecto.drop           # Drop the storage for the repo
mix ecto.gen.migration  # Generate a new migration for the repo
mix ecto.gen.repo       # Generate a new repository
mix ecto.migrate        # Run migrations up on a repo
mix ecto.rollback       # Rollback migrations from a repo
```

## Migrations

The best way to create migrations is the `mix ecto.gen.migration <name>` task.  If you're acquainted with ActiveRecord these will look familiar.

Let's start by taking a look at a migration for a users table:

```elixir
defmodule ExampleApp.Repo.Migrations.CreateUser do
  use Ecto.Migration

  def change do
    create table(:users) do
      add(:username, :string, unique: true)
      add(:encrypted_password, :string, null: false)
      add(:email, :string)
      add(:confirmed, :boolean, default: false)

      timestamps()
    end

    create(unique_index(:users, [:username], name: :unique_usernames))
  end
end
```

By default Ecto creates an auto-incrementing primary key called `id`.  Here we're using the default `change/0` callback but Ecto also supports `up/0` and `down/0` in the event you need more granular control.

As you might have guessed, adding `timestamps` to your migration will create and manage `inserted_at` and `updated_at` for you.

To apply our new migration run `mix ecto.migrate`.

For more on migrations take a look at the [Ecto.Migration](https://hexdocs.pm/ecto_sql/3.0.0/Ecto.Migration.html) section of the docs.

## Schemas

Now that we have our migration we can move on to the schema. Schema is a module, that defines mappings to the underlying database table and it's fields, helper functions, and our changesets.  We'll cover changesets more in the next sections.

For now let's look at what the schema for our migration might look like:

```elixir
defmodule ExampleApp.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field(:username, :string)
    field(:encrypted_password, :string)
    field(:email, :string)
    field(:confirmed, :boolean, default: false)
    field(:password, :string, virtual: true)
    field(:password_confirmation, :string, virtual: true)

    timestamps()
  end

  @required_fields [:username, :encrypted_password, :email]
  @optional_fields []

  def changeset(user, params \\ :empty) do
    user
    |> cast(params, @required_fields ++ @optional_fields)
    |> validate_required(@required_fields)
    |> unique_constraint(:username)
  end
end
```

The schema we define closely represents what we specified in our migration.  In addition to our database fields we're also including two virtual fields.  Virtual fields are not saved to the database but can be useful for things like validation.  We'll see the virtual fields in action in the [Changesets](#changesets) section.

## Querying

Before we can query our repository we need to import the Query API.  For now we only need to import `from/2`:

```elixir
import Ecto.Query, only: [from: 2]
```

The official documentation can be found at [Ecto.Query](https://hexdocs.pm/ecto/Ecto.Query.html).

### Basics

Ecto provides an excellent Query DSL that allows us to express queries clearly.  To find the usernames of all confirmed accounts we could use something like this:

```elixir
alias ExampleApp.{Repo, User}

query =
  from(
    u in User,
    where: u.confirmed == true,
    select: u.username
  )

Repo.all(query)
```

In addition to `all/2`, Repo provides a number of callbacks including `one/2`, `get/3`, `insert/2`, and `delete/2`.  A complete list of callbacks can be found at [Ecto.Repo#callbacks](https://hexdocs.pm/ecto/Ecto.Repo.html#callbacks).

### Count

If we want to count the number of users that have confirmed account we could use `count/1`:

```elixir
query =
  from(
    u in User,
    where: u.confirmed == true,
    select: count(u.id)
  )
```

There is `count/2` function that counts the distinct values in given entry:

```elixir
query =
  from(
    u in User,
    where: u.confirmed == true,
    select: count(u.id, :distinct)
  )
```

### Group By

To group users by their confirmation status we can include the `group_by` option:

```elixir
query =
  from(
    u in User,
    group_by: u.confirmed,
    select: [u.confirmed, count(u.id)]
  )

Repo.all(query)
```

### Order By

Ordering users by their creation date:

```elixir
query =
  from(
    u in User,
    order_by: u.inserted_at,
    select: [u.username, u.inserted_at]
  )

Repo.all(query)
```

To order by `DESC`:

```elixir
query =
  from(
    u in User,
    order_by: [desc: u.inserted_at],
    select: [u.username, u.inserted_at]
  )
```

### Joins

Assuming we had a profile associated with our user, let's find all confirmed account profiles:

```elixir
query =
  from(
    p in Profile,
    join: u in assoc(p, :user),
    where: u.confirmed == true
  )
```

### Fragments

Sometimes, like when we need specific database functions, the Query API isn't enough.  The `fragment/1` function exists for this purpose:

```elixir
query =
  from(
    u in User,
    where: fragment("downcase(?)", u.username) == ^username,
    select: u
  )
```

Additional query examples can be found in the [Ecto.Query.API](https://hexdocs.pm/ecto/Ecto.Query.API.html) module description.

## Changesets

In the previous section we learned how to retrieve data, but how about inserting and updating it?  For that we need Changesets.

Changesets take care of filtering, validating, and maintaining constraints when changing a schema.

For this example we'll focus on the changeset for user account creation.  To start we need to update our schema:

```elixir
defmodule ExampleApp.User do
  use Ecto.Schema
  import Ecto.Changeset
  import Comeonin.Bcrypt, only: [hashpwsalt: 1]

  schema "users" do
    field(:username, :string)
    field(:encrypted_password, :string)
    field(:email, :string)
    field(:confirmed, :boolean, default: false)
    field(:password, :string, virtual: true)
    field(:password_confirmation, :string, virtual: true)

    timestamps()
  end

  @required_fields [:username, :email, :password, :password_confirmation]
  @optional_fields []

  def changeset(user, params \\ :empty) do
    user
    |> cast(params, @required_fields ++ @optional_fields)
    |> validate_required(@required_fields)
    |> validate_length(:password, min: 8)
    |> validate_password_confirmation()
    |> unique_constraint(:username, name: :email)
    |> put_change(:encrypted_password, hashpwsalt(params[:password]))
  end

  defp validate_password_confirmation(changeset) do
    case get_change(changeset, :password_confirmation) do
      nil ->
        password_incorrect_error(changeset)

      confirmation ->
        password = get_field(changeset, :password)
        if confirmation == password, do: changeset, else: password_mismatch_error(changeset)
    end
  end

  defp password_mismatch_error(changeset) do
    add_error(changeset, :password_confirmation, "Passwords does not match")
  end

  defp password_incorrect_error(changeset) do
    add_error(changeset, :password, "is not valid")
  end
end
```

We've improved our `changeset/2` function and added three new helper functions: `validate_password_confirmation/1`, `password_mismatch_error/1`, and `password_incorrect_error/1`.

As its name suggests, `changeset/2` creates a new changeset for us.  In it we use `cast/3` to convert our parameters to a changeset from a set of required and optional fields.  Then we validate the presence of required fields. Next we validate the changeset's password length, we use our own function to validate the password confirmation matches, and we validate username uniqueness.  Finally we update our actual password database field.  For this we use `put_change/3` to update a value in the changeset.

Using `User.changeset/2` is relatively straightforward:

```elixir
alias ExampleApp.{User,Repo}

pw = "passwords should be hard"
changeset = User.changeset(%User{}, %{username: "doomspork",
                    email: "sean@seancallan.com",
                    password: pw,
                    password_confirmation: pw})

case Repo.insert(changeset) do
  {:ok, record}       -> # Inserted with success
  {:error, changeset} -> # Something went wrong
end
```

That's it!  Now you're ready to save some data.
