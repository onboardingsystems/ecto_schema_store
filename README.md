# Ecto Schema Store

This library is used to create customizable data stores for individual ecto schemas.

## Getting Started ##
With the following schema:

```elixir
defmodule Person do
  use Ecto.Schema

  schema "people" do
    field :name, :string
    field :email, :string

    timestamps
  end

  def changeset(model, params) do
    model
    |> cast(params, [:name, :email])
  end
end
```

You can create a store with the following:

```elixir
defmodule PersonStore do
  use EctoSchemaStore, schema: Person, repo: MyApp.Repo
end
```

## Querying ##

The following functions are provided in a store for retrieving data.

* `all`                 - Fetch all records
* `one`                 - Return a single record
* `refresh`             - Reloads an existing record from the database.
* `preload_assocs`      - Preload record associations. Same as `Repo.preload`. Providing `:all` will cause all associations to be preloaded.
* `count_records`       - Count the number of records returned by the provided filters.
* `exists?`             - Returns true if any records exists for the provided filters.
* `to_map`              - Return the model as generic Elixir maps without the Ecto content

Sample Queries:

```elixir
# Get all records in a table.
PersonStore.all

# Get all records fields that match the provided value.
PersonStore.all %{name: "Bob"}
PersonStore.all %{name: "Bob", email: "bob@nowhere.test"}
PersonStore.all name: "Bob", email: "bob@nowhere.test"

# Return a single record.
PersonStore.one %{name: "Bob"}
PersonStore.one name: "Bob"

# Return a specific record by id.
PersonStore.one 12

# Refresh
record = PersonStore.one 12
record = PersonStore.refresh record

# Preload after query
PersonStore.preload_assocs record, :field_name
PersonStore.preload_assocs record, :all
PersonStore.preload_assocs record, [:field_name_1, :field_name_2]

# To Map
record = PersonStore.to_map PersonStore.one 12
```

Options:

* preload            - An atom or list of atoms with the model association keys to preload. Providing `:all` will cause all associations to be preloaded.
* to_map             - Return the models as generic Elixir maps without the Ecto content. Defaults: `false`
* order_by           - Provides a means to pass order by information to Ecto.Repo. Uses same format as `https://hexdocs.pm/ecto/Ecto.Query.html#order_by/3`.

```elixir
# Get all records in a table.
PersonStore.all %{}, preload: :field_name

# Get all records fields that match the provided value.
PersonStore.all %{name: "Bob"}, preload: [:field_name_1, :field_name_2]

# Return a single record.
PersonStore.one %{name: "Bob"}, preload: :all

# Return a specific record by id.
PersonStore.one 12, preload: :all, to_map: true

# Order by
PersonStore.all %{}, order_by: :email
PersonStore.all %{}, order_by: [:name, :email]
PersonStore.all %{}, order_by: [asc: :name, desc: :email]
```

## Filter Operators ##

Stores support a special syntax for changing the comparison operator in the passed filter map or keyword list.

Operators:

* `{:==, value}`            - The field equals this value. This is the default operator when the value is not `nil`.
* `{:!=, value}`            - The field does not equal the value.
* `{:==, nil}`              - The field is `nil` or `null` in the database. This is the default operator when the value is `nil`.
* `{:!=, nil}`              - The field is not `nil` or `null` in the database.
* `{:<, value}`             - The field is less than the value.
* `{:<=, value}`            - The field is less than or equal the value.
* `{:>, value}`             - The field is greater than the value.
* `{:>=, value}`            - The field is greater than or equal the value.
* `{:in, []}`               - The field is in the list of provided values.
* `{:like, value}`          - The field performs a case sensitive like against the provided value.
* `{:ilike, value}`         - The field performs a case insensitive like against the provided value.

```elixir
PersonStore.all %{name: nil}
PersonStore.all %{name: {:==, nil}}
PersonStore.all name: {:!=, nil}
PersonStore.all name: {:!=, "Bob"}
PersonStore.all name: {:in, ["Bob"]}, email: "bob@nowhere.test"
```

## Editing ##

The following functions are provided in a store for editing data.

* `insert`                       - Insert a record based upon supplied parameters map.
* `insert_fields`                - Insert the record without using a changeset.
* `insert!`                      - Same as `insert` but throws an error instead of returning a tuple.
* `insert_fields!`               - Same as `insert_fields` but throws an error instead of returning a tuple.
* `update`                       - Update a record based upon supplied parameters map.
* `update_fields`                - Update the record without using a changeset.
* `update!`                      - Same as `update` but throws an error instead of returning a tuple.
* `update_fields!`               - Same as `update_fields` but throws an error instead of returning a tuple.
* `update_or_create`             - Updates a record if the provided query values are found. Otherwise creates the record.
* `update_or_create!`            - Same as `update_or_create` but throws an error instead of returning a tuple.
* `update_or_create_fields`      - Updates a record if the provided query values are found. Otherwise creates the record. Does not use a changeset.
* `update_or_create_fields!`     - Same as `update_or_create_fields` but throws an error instead of returning a tuple.
* `find_or_create`               - Returns the record if it already exists. Otherwise creates the record.
* `find_or_create!`              - Same as `find_or_create` but throws an error instead of returning a tuple.
* `find_or_create_fields`        - Returns the record if it already exists. Otherwise creates the record. Does not use a changeset.
* `find_or_create_fields!`       - Same as `find_or_create_fields` but throws an error instead of returning a tuple.
* `delete`                       - Delete a record.
* `delete!`                      - Same as `delete` but throws an error instead of returning a tuple.

Options:

* `changeset`                    - Provide and atom, or function reference for the changeset to use. Default `:changeset`
* `errors_to_map`                - If an error occurs, the changeset error is converted to a JSON encoding firendly map. When given an atom, sets the root id to the atom. Default: `false`
* `timeout`                      - Number of milliseconds to wait before returning when a :before_* event is being sent and processed. Default: 5000
* `sync`                         - Should the operation wait for a :before_* event to be complete before returning. If not, then an :ok will be return and the action will be asynchronous. Default: true

Sample Usage:

```elixir
# Using Map
bob = PersonStore.insert! %{name: "Bob", email: "bob@nowhere.test"}
bob = PersonStore.update! bob, %{email: "bob2@nowhere.test"}
PersonStore.delete bob

# Using Keyword List
bob = PersonStore.insert! name: "Bob", email: "bob@nowhere.test"
bob = PersonStore.update! bob, email: "bob2@nowhere.test"

# Updates/deletes can also occur by id.
PersonStore.update! 12, %{email: "bob2@nowhere.test"}
PersonStore.delete 12

# Update a single record based upon a query.
# This will only update the first record retrieved by the database, it is not meant
# to be an update_all style function. Instead it is useful if you use another id to reference
# the record that is not the primary id.
PersonStore.update %{name: "bob"}, email: "otheremail@nowhere.test"

# Update or create
attributes_to_update =
query = %{name: "Bob"}
PersonStore.update_or_create attributes_to_update, query
PersonStore.update_or_create! %{email: "new@nowhere.test"}, name: "Bob"
```

## Changesets ##

The `insert` and `update` functions by default use a changeset on the provided schema name `:changeset` for inserting and updating.
This can be overridden and a specific changeset name provided.¬

```elixir
bob = PersonStore.insert! %{name: "Bob", email: "bob@nowhere.test"}, changeset: :insert_changeset
bob = PersonStore.insert! [name: "Bob", email: "bob@nowhere.test"], changeset: :insert_changeset
bob = PersonStore.update! bob, %{email: "bob2@nowhere.test"}, changeset: :update_changeset
bob = PersonStore.update! bob, [email: "bob2@nowhere.test"], changeset: :update_changeset
bob = PersonStore.update! bob, %{email: "bob2@nowhere.test"}, changeset: :my_other_custom_changeset
```

It is also possible to pass a function reference in as the changeset.

```elixir
def my_changeset(model, params) do
  model
  |> cast(params, [:name, :email])
  |> validate_required([:name])
end

insert [name: "Bob"], changeset: &my_changeset/2
```

## Validate Insert/Update Params Only ##

Sometimes it is convienent to check if a changeset will pass before actually attempting to insert or update
a record. There are two validation functions that can be used to check the changesets the same way they would
be checked on an insert or update action.

* `validate_insert`           - Validate the params against a new instance of the schema.
* `validate_update`           - Validate the params against an existing instance.

Options:

* `changeset`                    - Provide and atom, or function reference for the changeset to use. Default `:changeset`
* `errors_to_map`                - If an error occurs, the changeset error is converted to a JSON encoding firendly map. When given an atom, sets the root id to the atom. Default: `false`

```elixir
# Checking an insert

with :ok <- PersonStore.validate_insert(%{name: "Bob"}, changeset: :my_changeset, errors_to_map: :person) do
  # Perform some action on validation, the :error tuple is return directly from the with statement.
end

# Chacking an update

existing_record = Person.Store.insert_fields! name: "Will"

with :ok <- PersonStore.validate_update(existing_record, %{name: "Bob"}, changeset: :my_changeset, errors_to_map: :person) do
  # Perform some action on validation, the :error tuple is return directly from the with statement.
end
```

## Preconfigured Options ##

Each of the insert and update functions takes a set of options. It may be inconvienent to set these options every
time one of theses functions is used. A duplicate set of these funcitons can be generated with a predefined set
of options.

The `preconfigure` function will generate a customized variation of the following store functions:

* `insert`              - `insert_{name}`
* `insert!`             - `insert_{name}!`
* `insert_fields`       - `insert_fields_{name}`
* `insert_fields!`      - `insert_fields_{name}!`
* `validate_insert`     - `validate_insert_{name}`
* `update`              - `update_{name}`
* `update!`             - `update_{name}!`
* `update_fields`       - `update_fields_{name}`
* `update_fields!`      - `update_fields_{name}!`
* `validate_update`     - `validate_update_{name}`

```elixir
defmodule PersonStore do
  use EctoSchemaStore, schema: Person, repo: MyApp.Repo

  # (custom_name, options)
  preconfigure :for_api, changeset: :my_changeset, errors_to_map: true
end

PersonStore.insert_for_api! name: "Bob", email: "bob@nowhere.test"
PersonStore.insert_for_api! name: "Will", email: "will@nowhere.test"
PersonStore.insert_for_api! name: "Carl", email: "carl@nowhere.test"
PersonStore.insert_for_api! name: "Mike", email: "mike@nowhere.test"

# Similar statements without preconfigure would have looked like this.
PersonStore.insert! [name: "Bob", email: "bob@nowhere.test"], changeset: :my_changeset, errors_to_map: true
PersonStore.insert! [name: "Will", email: "will@nowhere.test"], changeset: :my_changeset, errors_to_map: true
PersonStore.insert! [name: "Carl", email: "carl@nowhere.test"], changeset: :my_changeset, errors_to_map: true
PersonStore.insert! [name: "Mike", email: "mike@nowhere.test"], changeset: :my_changeset, errors_to_map: true
```

## Update/Delete All ##

Ecto allows batch updates and deletes by passing changes directly to the database to effect multiple records.

* `update_all`         - Set values based upon a provided query map/keyword list. If `updated_at` field present on schema, will update the date/time.
* `delete_all`         - Delete all records that match the provided query map/keyword list.

```elixir
# Update records by query
values_to_set = [name: "Generic Name"]
query_params = [email: {:like, "%@test.dev"}]

PersonStore.update_all values_to_set, query_params

# Update all records
PersonStore.update_all name: "Generic Name"

# Delete records by query
query_params = [email: {:like, "%@test.dev"}]

PersonStore.delete_all query_params

# Delete All Records on a Table
PersonStore.delete_all
```

## Logging Edit Actions ##

A store can enable logging for all edit actions performed through the store module. Successful actions use `Logger.info` while failures use `Logger.warn`.

```elixir
defmodule PersonStore do
  use EctoSchemaStore, schema: Person, repo: MyApp.Repo, logging: true
end
```

Sample Insert Output:

```
iex(1)> PersonStore.insert [], changeset: :insert_changeset, errors_to_map: :person
[warn] Elixir.Person action `insert` using opts `[timeout: 5000, sync: true, changeset: :insert_changeset, errors_to_map: :person]` failed due to %{"person.email" => ["can't be blank"], "person.name" => ["can't be blank"]}
```

## References ##

The internal references to the schema and the provided Ecto Repo are provided as convience functions.

* `schema`         - returns the schema reference used internally by the store.
* `repo`           - returns the Ecto Repo reference used internally by the store.

## Custom Actions ##

Since a store is just an ordinary module, you can add your actions and build off private APIs to the store. For convience
`Ecto.Query` is already fully imported into the module.

A store is provided the following custom internal API:

* `build_query`       - Builds a `Ecto.Query` struct based upon the map params input.
* `build_query!`      - Like `build_query` but throws an error.


```elixir
defmodule PersonStore do
  use EctoSchemaStore, schema: Person, repo: MyApp.Repo

  def get_all_ordered_by_name do
    build_query!
    |> order_by([:name])
    |> all
  end

  def find_by_email(email) do
    %{email: email}
    |> build_query!
    |> order_by([:name])
    |> all
  end

  def get_all_ordered_by_name_using_ecto_directly do
    query = from p in schema,
            order_by: [p.name]

    repo.all query
  end
end
```

## Schema Field Aliases ##

Sometimes field names get changed or the developer wishes to have an alias that represents another field.
These work for both querying and editing schema models.

```elixir
defmodule PersonStore do
  use EctoSchemaStore, schema: Person, repo: MyApp.Repo

  alias_fields email_address: :email
end

PersonStore.all %{email_address: "bob@nowhere.test"}
PersonStore.update! 12, %{email_address: "bob@nowhere.test"}
```

## Filter or Params Map/Keyword List ##

Many of the API calls used by a store take a map of fields as input. Normal Ecto
requires param maps to be either all atom or string keyed but not mixed. A schema
store will convert every map provided into atom keys before aliasing and passing
on to Ecto. This means you can provide a mixture of both. This will allow a
developer to combine multiple maps together and not worry about what kind of
keys were used.

However, if you provide the same value twice as both an atom and string key then
only one will be used.

```elixir
PersonStore.insert! %{"name" => "Bob", email: "bob2@nowhere.test"}
```

## Transaction ##

Under normal circumstances, the regular Ecto Repo `transaction` function can be used normally; if you would like to use
the `!` functions in a store you will need to use the `EctoSchemaStore.transaction` or the store specific
`transaction` function to return a friendlier error tuple instead of passing the throw up the chain.

```elixir
{:error, changeset_or_map} =
  PersonStore.transaction fn ->
    PersonStore.insert! age: "bad value"
  end

# Manual rollback
{:error, message} =
  PersonStore.transaction fn ->
    PersonStore.repo.rollback(message)
  end

# Using directly
{:error, changeset_or_map} =
  EctoSchemaStore.transaction MyApp.Repo, fn ->
    PersonStore.insert! name: "Bob"
    OtherStore.update! field: "bad value"
  end
```

Although the `transaction` function can be called on a store module, it only proxies to the `EctoSchemaStore` module
passing in the repo associated with the store. The call can be used with any combination of stores or the Ecto Repo
itself.

## Generator Factories ##

Sometimes, such as in unit testing, a developer would like to create a common predefined data structure
to work against. Generator Factories provide a composable method to create such a data structure. There
two types of factory methods that can be used. One to create a common factory base for the store and
another to append upon that base if it is used. The base factory is optional and not required.

If a default factory does not exist then the defaults will be used as defined in the Struct ecto will
generate for the schema.

Macros:

* `factory`               - Generates a factory segment of values. When no name is provided the fatcory is the automatic base for all factories.

Functions:

* `generate`              - Takes a list of atoms and generates a new record in the database based upon the factories matching the atom.
* `generate!`             - Like `generate` but throws the error instead of returning a tuple.
* `generate_default`      - Like `generate` but only applies the base factory. When no base is defined, uses the defaults from the Struct Ecto generates.
* `generate_default!`     - Like `generate_default` but throws the error instead of returning a tuple.

```elixir
defmodule PersonStore do
  use EctoSchemaStore, schema: Person, repo: MyApp.Repo

  # Create the common base for starting factory generated record.
  factory do
    %{
      name: "Test Person",
      email: "test@nowhere.test"
    }
  end

  # Create a factory segment to override the base factory values.
  factory bob do
    %{
      name: "Bob"
    }
  end

  factory karen do
    %{
      name: "Karen"
    }
  end
end

# Sample Usage

# Using just the base
%Person{name: "Test Person", email: "test@nowhere.test"} = PersonStore.generate!
{:ok, %Person{name: "Test Person", email: "test@nowhere.test"}} = PersonStore.generate

# Using bob Factory
%Person{name: "Bob", email: "test@nowhere.test"} = PersonStore.generate! :bob

# Using karen factory
%Person{name: "Karen", email: "test@nowhere.test"} = PersonStore.generate! :karen

# Using multiple factories, each is overlayed in order.
%Person{name: "Karen", email: "test@nowhere.test"} = PersonStore.generate! [:bob, :karen]
%Person{name: "Bob", email: "test@nowhere.test"} = PersonStore.generate! [:karen, :bob]

# Manually setting values
%Person{name: "Test Person", email: "ignore@nowhere.test"} = PersonStore.generate_default! email: "ignore@nowhere.test"
%Person{name: "Bob", email: "bob@nowhere.test"} = PersonStore.generate! :bob, email: "bob@nowhere.test"
%Person{name: "Karen", email: "karen@nowhere.test"} = PersonStore.generate! [:bob, :karen], email: "karen@nowhere.test"
```

## Event Announcements ##

A store supports the concept of an event through the [Event Queues](https://hex.pm/packages/event_queues) library on Hex.
Event Queues must be included in your application and each queue and handler added to your application supervisor. Visit
the instructions at (https://hexdocs.pm/event_queues) for more details.

Events:

* `:before_insert`
* `:before_update`
* `:before_delete`
* `:after_insert`
* `:after_update`
* `:after_delete`

Macros:

* `create_queue`               - Creates a Queue for instances where one is not already set up. Accessible at {store module name}.Queue
* `announces`                   - Register a what events to announce and what modules to send the event. By default will use {store}.Queue

```elixir
defmodule PersonStore do
  use EctoSchemaStore, schema: Person, repo: MyApp.Repo

  create_queue

  announces events: [:after_delete, :after_update]
end

defmodule PersonEventHandler do
  use EventQueues, type: :handler, subscribe: PersonStore.Queue

   def handle(%{category: Person, name: :after_delete, data: data}) do
    IO.inspect "Delete #{data.schema} id: #{data.id}"
   end
   def handle(%{category: Person, name: :after_update, data: data}) do
    IO.inspect "Changed #{data.schema} id: #{data.id}"
   end
   def handle(_), do: nil
end
```

A store can also use existing queues (1 or more):

```elixir
defmodule Queue1 do
  use EventQueues, type: :queue
end

defmodule Queue2 do
  use EventQueues, type: :queue
end

defmodule PersonStore do
  use EctoSchemaStore, schema: Person, repo: MyApp.Repo

  announces events: [:after_delete, :after_update],
           queues: [Queue1, Queue2]
end

defmodule PersonDeleteEventHandler do
  use EventQueues, type: :handler, subscribe: Queue1

   def handle(%{category: Person, name: :after_delete, data: data}) do
    IO.inspect "Delete #{data.schema} id: #{data.id}"
   end
   def handle(_), do: nil
end

defmodule PersonUpdateEventHandler do
  use EventQueues, type: :handler, subscribe: Queue2

   def handle(%{category: Person, name: :after_update, data: data}) do
    IO.inspect "Changed #{data.schema} id: #{data.id}"
   end
   def handle(_), do: nil
end
```

For certain events, actions can be taken before the action is to take place. In order to continue, an event must be resubmitted
after handling the initial event to tell the Store to continue or cancel the action originally submitted.

```elixir
defmodule PersonStore do
  use EctoSchemaStore, schema: Person, repo: MyApp.Repo

  create_queue

  announces events: [:before_update]
end

defmodule PersonEventHandler do
  use EventQueues, type: :handler, subscribe: PersonStore.Queue

   def handle(%{category: Person, name: :before_update} = event) do
    # Perform some action

    if success do
      event.data.originator.continue event
    else
      event.data.originator.cancel event, "Something failed"
    end

    # The event must be continued or canceled, otherwise an error will be returned back to
    # the original function calling the store. Even if the operation was successful.
   end
   def handle(_), do: nil
end
```

## Proxy Store Functions Through Schema ##

If you happen to come from other programming environments, you may have used an ORM that places store style functions
directly on the entity or what in the case of Ecto is called a schema. EctoSchemaStore provides a modules that will
allow you to include some of the store functions directly into the schema module.

```elixir
defmodule Person do
  use Ecto.Schema
  use EctoSchemaStore.Proxy, store: PersonStore

  schema "people" do
    field :name, :string
    field :email, :string

    timestamps
  end

  def changeset(model, params) do
    model
    |> cast(params, [:name, :email])
  end
end

defmodule PersonStore do
  use EctoSchemaStore, schema: Person, repo: MyApp.Repo
end

# Get all records in a table.
Person.all

# Get all records fields that match the provided value.
Person.all %{name: "Bob"}
Person.all %{name: "Bob", email: "bob@nowhere.test"}
Person.all name: "Bob", email: "bob@nowhere.test"

# Return a single record.
Person.one %{name: "Bob"}
Person.one name: "Bob"

# Return a specific record by id.
Person.one 12

# Refresh
record = Person.one 12
record = Person.refresh record

# Preload after query
Person.preload_assocs record, :field_name
Person.preload_assocs record, :all
Person.preload_assocs record, [:field_name_1, :field_name_2]

# To Map
record = Person.to_map Person.one 12
```

If a store is not provided, the proxy take the current schema module name and append `.Store` to the end of it.
So the default for `Person` would be `Person.Store`.

To figure out what store is proxied into a module you can call the `store` function on the schema module.
