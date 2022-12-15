# Speakeasy

[![build](https://github.com/avenueplace/speakeasy/actions/workflows/build.yml/badge.svg?branch=main)](https://github.com/avenueplace/speakeasy/actions/workflows/build.yml)

> This project was forked from [coryodaniel/speakeasy] and its maintenance is focused
> on our internal usage at Avenue. Feel free to use it as is, and reach out to
> us through an issue or pull-request. We'll gladly consider your suggestions
> and contributions.

[Speakeasy](https://hexdocs.pm/speakeasy/readme.html) is authentication agnostic middleware based authorization for [Absinthe](https://hexdocs.pm/absinthe) GraphQL powered by [Bodyguard](https://hexdocs.pm/bodyguard).

[Docs](https://hexdocs.pm/speakeasy/readme.html)

## Installation

[Speakeasy](https://hex.pm/packages/speakeasy) can be installed
by adding `speakeasy` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:speakeasy, github: "avenueplace/speakeasy", tag: "0.4"}
  ]
end
```

## Configuration

Configuration can be done in each Absinthe middleware call, but you can set global defaults as well.

```elixir
config :speakeasy,
  user_key: :current_user,                # the key the current user will be under in the GraphQL context
  authn_error_message: :unauthenticated   # default authentication failure message
```

_Note:_ no `authz_error_message` is provided because it is set from Bodyguard.

## Usage

**tl;dr:** A full example authentication, authorizing, loading, and resolving an Absinthe schema:

_This example assumes:_

- You are authorizing a standard phoenix context
- You already have a [bodyguard policy](https://github.com/schrockwell/bodyguard#policies)
- Your `:current_user` is already in the Absinthe context _or_ you are using [`Speakeasy.Plug`](#speakeasy-plug)

```elixir
defmodule MyApp.Schema.PostTypes do
  use Absinthe.Schema.Notation
  alias Spectra.Posts

  object :post do
    field(:id, non_null(:id))
    field(:name, non_null(:string))
  end

  object :post_mutations do
    @desc "Create post"
    field :create_post, type: :post do
      arg(:name, non_null(:string))
      middleware(Speakeasy.Authn)
      middleware(Speakeasy.Authz, {Posts, :create_post})
      middleware(Speakeasy.Resolve, &Posts.create_post/2)
      middleware(MyApp.Middleware.ChangesetErrors) # :D
    end

    @desc "Update post"
    field :update_post, type: :post do
      arg(:name, non_null(:string))
      middleware(Speakeasy.Authn)
      middleware(Speakeasy.Authz, {Posts, :update_post})
      middleware(Speakeasy.Resolve, &Posts.update_post/3)
      middleware(MyApp.Middleware.ChangesetErrors) # :D
    end
  end

  object :post_queries do
    @desc "Get posts"
    field :posts, list_of(:post) do
      middleware(Speakeasy.Authn)
      middleware(Speakeasy.Resolve, fn(attrs, user) -> MyApp.Posts.search(attrs, user) end)
    end

    @desc "Get post"
    field :post, type: :post do
      arg(:id, non_null(:string))
      middleware(Speakeasy.Authn)
      middleware(Speakeasy.LoadResourceByID, &Posts.get_post/1)
      middleware(Speakeasy.Authz, {Posts, :get_post})
      middleware(Speakeasy.Resolve)
    end
  end
end
```

And of course you can use Absinthe's resolve function as well:

```elixir
@desc "Get post"
field :post, type: :post do
  arg(:id, non_null(:string))
  middleware(Speakeasy.Authn)
  middleware(Speakeasy.LoadResourceByID, &Posts.get_post/1)
  middleware(Speakeasy.Authz, {Posts, :get_post})
  resolve(fn(_parent, _args, ctx) ->
    {:ok, ctx[:speakeasy].resource}
  end)
end
```

### Differences from [coryodaniel/speakeasy]

The main difference in this fork is the added support of "[#13 - Allow Authn to bypass nil users, given an option](https://github.com/coryodaniel/speakeasy/pull/13)".

> NOTE: At the time of this fork, `#13` was open. As we need to support this
> internally, we opted to fork and release a tagged version on GitHub. If and
> when `#13` (or an equivalent fix) is merged, we **highly** recommend using the
> [main version](https://github.com/coryodaniel/speakeasy) of Speakeasy.

This PR adds a new `require: boolean` option to `Speakeasy.Authn`, allowing for
the `current_user` to be `nil` when `require` is set to `false`. The main
purposes is to facilitate the usage of API calls that can be made both
authenticated and unauthenticated.

Example:

```elixir
object :post_mutations do
  @desc "Create post"
  field :create_post, type: :post do
    arg(:name, non_null(:string))
    middleware(Speakeasy.Authn, require: false)
  end
end
```

Note that the `current_user` value then passed to `Speakeasy.Authz`,
`Speakeasy.Resolve` and subsequent middlewares can be `nil` and you should take
steps to verify this.

### Middleware

Speakeasy is a collection of Absinthe middleware:

- [Speakeasy.Authn](https://hexdocs.pm/speakeasy/Speakeasy.Authn.html#content) - Resolution middleware for Absinthe.

- [Speakeasy.LoadResource](https://hexdocs.pm/speakeasy/Speakeasy.LoadResource.html#content) - Loads a resource into the speakeasy context.

- [Speakeasy.LoadResourceById](https://hexdocs.pm/speakeasy/Speakeasy.LoadResourceByID.html#content) - A convenience middleware to `LoadResource` using the `:id` in the Absinthe arguments.

- [Speakeasy.LoadResourceBy](https://hexdocs.pm/speakeasy/Speakeasy.LoadResourceBy.html#content) - A convenience middleware to `LoadResource` using a value from the attrs with the given key in the Absinthe arguments.

- [Speakeasy.AuthZ](https://hexdocs.pm/speakeasy/Speakeasy.Authz.html#content) - Authorization middleware for Absinthe.

- [Speakeasy.Resolve](https://hexdocs.pm/speakeasy/Speakeasy.Resolve.html#content) - Resolution middleware for Absinthe.

### Speakeasy.Plug

Speakeasy includes a Plug for loading the current user into the Absinthe context. It isn't required if you already have a method for loading the user into your Absinthe context.

```elixir
defmodule MyAppWeb.Router do
  use MyAppWeb, :router

  pipeline :graphql do
    plug(Speakeasy.Plug, load_user: &MyApp.Users.whoami/1, user_key: :current_user)
  end
end
```

[coryodaniel/speakeasy]: https://github.com/coryodaniel/speakeasy
