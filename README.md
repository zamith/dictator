# Dictator

Dictator is a plug-based authorization mechanism.

## Installation

First, you need to add `:dictator` to your list of dependencies on your `mix.exs`:

```elixir
def deps do
  [{:dictator, "~> 0.1"}]
end
```

## Usage

**IMPORTANT: `Dictator` assumes you have your current user in your
`conn.assigns`. See our [demo app](https://github.com/subvisual/dictator_demo)
for an example on integrating with guardian.**

To authorize your users, just add in your controller:

```elixir
defmodule ClientWeb.ThingController do
  use ClientWeb, :controller

  plug Dictator.Plug.Authorize

  # ...
end
```

That plug will automatically look for a `ClientWeb.Policies.Thing` module, which
should `use Dictator.Policy`, provide the resource that is being authorized
access to, and you can define two functions: `can?/3` and `load_resource/1`.

In `lib/client_web/policies/thing.ex`:

```elixir
defmodule ClientWeb.Policies.Thing do
  alias Client.Context.Thing

  use Dictator.Policy, for: Thing

  # User can edit, update, delete and show their own things
  def can?(%User{id: user_id}, action, %Thing{user_id: user_id})
    when action in [:edit, :update, :delete, :show], do: true

  # Any user can index, new and create things
  def can?(_, action, _) when action in [:index, :new, :create], do: true

  # Users can't do anything else (users editing, updating, deleting and showing)
  # on things they don't own
  def can?(_, _, _), do: false
end
```

This exact scenario is, in fact, so common that already comes bundled as
`Dictator.Policies.Standard`. This is equivalent to the previous definition:

```elixir
defmodule ClientWeb.Policies.Thing do
  alias Client.Context.Thing

  use Dictator.Policies.Standard, for: Thing
end
```

### Custom Options

The following params can be passed to `Dictator.Policy` and
`Dictator.Policies.Standard`:

- **`for:` (required)** - the module of the resource being accessed.
- **`repo:` (optional, automatically inferred)** - repo to load the resource.
- **`key:` (optional, default: `:id`)** - primary key of the resource being
  accessed.

The following params can be passed to `Dictator.Policies.Standard`:

- **`foreign_key:` (optional, `default: :user_id`)** - foreign key of the
  resource being accessed. If you're accessing a `Post` with that has an
  `organization_key` instead of a `user_id`, use this option to override it.
- **`owner_key:` (optional, default: `:id`)** - primary key of the owner of the
  resource. If you're accessing a `Post` with that belongs to an `Organization`
  that uses a `:slug` key as primary instead of an `id`, use this option.

The following params can be passed to `Dictator.Plug.Authorize`:

- **`only:` (optional, defaults to all actions)** - actions subject to
  authorization.
- **`policy:` (optional, infers the policy)** - policy to be used
- **`resource_key:` (optional, default: `:current_user`)** - key to use in the
  `conn.assigns` to load the currently logged in resource.

#### Overriding the Repo

By `use`ing `Dictator.Policy`, it will automatically infer the correct repo.  If
it cannot do so, it will fail compiling. If that happens, you need to pass the
correct repo module:

```elixir
defmodule ClientWeb.Policies.Thing do
  alias Client.Context.Thing
  alias Client.FunkyRepoForThings

  use Dictator.Policies.Standard, for: Thing, repo: FunkyRepoForThings
end
```

#### Using a different primary key

When getting the resource being accessed from the database, `Dictator` calls
`YourRepo.get_by(YourModule, id: id)`. But if you use a primary key other than
`id`, you can set it by overriding the `key` option:

```elixir
defmodule ClientWeb.Policies.Thing do
  alias Client.Context.Thing

  use Dictator.Policies.Standard, for: Thing, key: :uuid
end
```

If you need further customizing how the resource is loaded from the database,
you can override the `load_resource/1`, which receives the route params as
argument.

```elixir
defmodule ClientWeb.Policies.Thing do
  alias Client.Context.Thing
  alias Client.FunkyRepoForThings

  use Dictator.Policies.Standard, for: Thing

  def load_resource(params) do
    if params["uuid"] && params["name"] do
      FunkyRepoForThings.get_by(Thing, uuid: params["uuid"], name: [params["name"]])
    else
      FunkyRepoForThings.get(Thing, params["id"])
    end
  end
end
```

#### Overriding the resource foreign key

Sometimes the resource you're trying to access (e.g a `Post`) has a different
foreign key than `user_id`. Let's assume it belongs to an organization and has
an `organization_id` foreign key. You can override this by doing

```elixir
defmodule ClientWeb.Policies.Post do
  alias Client.Context.Post

  use Dictator.Policies.Standard, for: Post, foreign_key: :organization_id
```

#### Overriding the primary key of the resource being authorized

This option allows you to override the primary key of the resource that is being
granted or denied access. Imagine your User has a `uuid` primary key instead of
an `id`. You can use the standard police by doing the following:

```elixir
defmodule ClientWeb.Policies.Post do
  alias Client.Context.Post

  use Dictator.Policies.Standard, for: Post, owner_key: :id
```

#### Limitting the actions to be authorized

If you want to only limit authorization to a few actions you can use the `:only`
option when calling the plug. In your controller.

```elixir
defmodule ClientWeb.ThingController do
  use ClientWeb, :controller

  plug Dictator.Plug.Authorize, only: [:edit, :update]

  # ...
end
```

This way, all other actions will not go through the authorization plug and the
policy will only be enforced for the `edit` and `update` actions.


#### Overriding the policy to be used

By default, the `Authorize` plug will automatically infer the policy to be used.
`MyWebApp.UserController` would mean a `MyWebApp.Policies.User` policy to use.

However, by using the `:policy` option, that can be overriden

```elixir
defmodule ClientWeb.ThingController do
  use ClientWeb, :controller

  plug Dictator.Plug.Authorize, policy: MyPolicy

  # ...
end
```

#### Overriding the current user key

By default, the `Authorize` plug will automatically search for a `current_user`
in the `conn.assigns`. You can change this behaviour by using the `resource_key`
option in the `plug` call.

```elixir
defmodule ClientWeb.ThingController do
  use ClientWeb, :controller

  plug Dictator.Plug.Authorize, resource_key: :current_organization

  # ...
end
```

# Contributing

Feel free to contribute.

If you found a bug, open an issue. You can also open a PR for bugs or new
features. Your PRs will be reviewed and subject to our style guide and linters.

All contributions **must** follow the [Code of
Conduct](https://github.com/subvisual/dictator/blob/master/CODE_OF_CONDUCT.md)
and [Subvisual's guides](https://github.com/subvisual/guides).

## Setup

To clone and setup the repo:

```bash
git clone git@github.com:subvisual/dictator.git
cd dictator
bin/setup
```

And everything should automatically be installed for you.

To run the development server:

```bash
bin/server
```



# About

`Dictator` is maintained by [Subvisual](http://subvisual.com).

[<img alt="Subvisual logo" src="https://raw.githubusercontent.com/subvisual/guides/master/github/templates/subvisual_logo_with_name.png" width="350px" />](https://subvisual.com)
