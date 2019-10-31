# graphql-tutorial

This is a copy of absinthe graphql tutorial from the [absinthe repository](https://github.com/absinthe-graphql/absinthe). My only contribution is in combining everything in one page for better reading and fixing some links.


# 1. Getting Started

We'll be building a very basic GraphQL API for a blog, written in Elixir using
Absinthe.

## Background

Before you start, it's a good idea to have some background into GraphQL in general. Here are a few resources that might be helpful:

- The official [GraphQL](http://graphql.org/) website
- [How to GraphQL](https://www.howtographql.com/) (this includes another [brief tutorial](https://www.howtographql.com/graphql-elixir/0-introduction/) using Absinthe)

## The Example

 The tutorial expects you to have a properly set-up [Phoenix application](https://hexdocs.pm/phoenix/installation.html) with <a href="https://hex.pm/packages/absinthe">absinthe</a> and <a href="https://hex.pm/packages/absinthe_plug">absinthe_plug</a> added to the dependencies.

>  If you'd like to cheat, you can find the finished code for the tutorial
>  in the <a href="https://github.com/absinthe-graphql/absinthe_tutorial">Absinthe Example</a>
>  project on GitHub.

## First Step

Let's get started with our first query.

# 2. Our First Query

The first thing our viewers want is a list of our blog posts, so
that's what we're going to give them. Here's the query we want to
support:

```graphql
{
  posts {
    title
    body
  }
}
```

To do this we're going to need a schema. Let's create some basic types
for our schema, starting with a `:post`. GraphQL has several fundamental
types on top of which all of our types will be
built. The [Object](Absinthe.Type.Object.html) type is the right one
to use when representing a set of key value pairs.

Since our `Post` Ecto schema lives in the `Blog.Content` Phoenix
context, we'll define its GraphQL counterpart type, `:post`, in a
matching `BlogWeb.Schema.ContentTypes` module:

In `blog_web/schema/content_types.ex`:

```elixir
defmodule BlogWeb.Schema.ContentTypes do
  use Absinthe.Schema.Notation

  object :post do
    field :id, :id
    field :title, :string
    field :body, :string
  end
end
```

> The GraphQL specification requires that type names be unique, TitleCased words.
> Absinthe does this automatically for us, extrapolating from our type identifier
> (in this case `:post` gives us `"Post"`. If really needed, we could provide a
> custom type name as a `:name` option to the `object` macro.

If you're curious what the type `:id` is used by the `:id` field, see
the [GraphQL spec](https://facebook.github.io/graphql/#sec-ID). It's
an opaque value, and in our case is just the regular Ecto id, but
serialized as a string.

With our type completed we can now write a basic schema that will let
us query a set of posts.

In `blog_web/schema.ex`:

```elixir
defmodule BlogWeb.Schema do
  use Absinthe.Schema
  import_types BlogWeb.Schema.ContentTypes

  alias BlogWeb.Resolvers

  query do

    @desc "Get all posts"
    field :posts, list_of(:post) do
      resolve &Resolvers.Content.list_posts/3
    end

  end

end
```

> For more information on the macros available to build a schema, see
> their definitions in [Absinthe.Schema](https://github.com/absinthe-graphql/absinthe/blob/master/guides/schemas.md) and Absinthe.Schema.Notation.

This uses a resolver module we've created (again, to match the Phoenix context naming)
at `blog_web/resolvers/content.ex`:

```elixir
defmodule BlogWeb.Resolvers.Content do

  def list_posts(_parent, _args, _resolution) do
    {:ok, Blog.Content.list_posts()}
  end

end
```

Queries are defined as fields inside the GraphQL object returned by
our `query` function. We created a posts query that has a type
`list_of(:post)` and is resolved by our
`BlogWeb.Resolvers.Content.list_posts/3` function. Later we'll talk
more about the resolver function parameters; for now just remember
that resolver functions can take two forms:

- A function with an arity of 3 (taking a parent, arguments, and resolution struct)
- An alternate, short form with an arity of 2 (omitting the first parameter, the parent)

The job of the resolver function is to return the data for the
requested field. Our resolver calls out to the `Blog.Content` module,
which is where all the domain logic for posts lives, invoking its
`list_posts/0` function, then returns the posts in an `:ok` tuple.

> Resolvers can return a wide variety of results, to include errors and configuration
> for [advanced plugins](https://github.com/absinthe-graphql/absinthe/blob/master/guides/middleware-and-plugins.md) that further process the data.
>
> If you're asking yourself what the implementation of the domain logic looks like, and exactly how
> the related Ecto schemas are built, read through the code in the [absinthe_tutorial](http://github.com/absinthe-graphql/absinthe_tutorial)
> repository. The tutorial content here is intentionally focused on the Absinthe-specific code.

Now that we have the functional pieces in place, let's configure our
Phoenix router to wire this into HTTP:

In `blog_web/router.ex`:

```elixir
defmodule BlogWeb.Router do
  use BlogWeb, :router

  pipeline :api do
    plug :accepts, ["json"]
  end

  scope "/api" do
    pipe_through :api

    forward "/graphiql", Absinthe.Plug.GraphiQL,
      schema: BlogWeb.Schema

    forward "/", Absinthe.Plug,
      schema: BlogWeb.Schema

  end

end
```

In addition to our API, we've wired in a handy GraphiQL user interface to play with it. Absinthe integrates both the classic [GraphiQL](https://github.com/graphql/graphiql) and  more advanced [GraphiQL Workspace](https://github.com/OlegIlyenko/graphiql-workspace) interfaces as part of the [absinthe_plug](https://hex.pm/packages/absinthe_plug) package.

Now let's check to make sure everything is working. Start the server:

``` shell
$ mix phx.server
```

Absinthe does a number of sanity checks during compilation, so if you misspell a type or make another schema-related gaffe, you'll be notified.

Once it's up-and-running, take a look at [http://localhost:4000/api/graphiql](http://localhost:4000/api/graphiql):

<img style="box-shadow: 0 0 6px #ccc;" src="/guides/assets/tutorial/graphiql_blank.png" alt=""/>

Make sure that the `URL` is pointing to the correct place and press the play button. If everything goes according to plan, you should see something like this:

<img style="box-shadow: 0 0 6px #ccc;" src="/guides/assets/tutorial/graphiql.png" alt=""/>

## Next Step

Now let's look at how we can add arguments to our queries.


# 3. Query Arguments

Our GraphQL API would be pretty boring (and useless) if clients
couldn't retrieve filtered data.

Let's assume that our API needs to add the ability to look-up users by
their ID and get the posts that they've authored. Here's what a basic query to do that
might look like:

```graphql
{
  user(id: "1") {
    name
    posts {
      id
      title
    }
  }
}
```

The query includes a field argument, `id`, contained within the
parentheses after the `user` field name. To make this all work, we need to modify
our schema a bit.

## Defining Arguments

First, let's create a `:user` type and define its relationship to
`:post` while we're at it. We'll create a new module for the
account-related types and put it there; in
`blog_web/schema/account_types.ex`:

```elixir
defmodule BlogWeb.Schema.AccountTypes do
  use Absinthe.Schema.Notation

  @desc "A user of the blog"
  object :user do
    field :id, :id
    field :name, :string
    field :email, :string
    field :posts, list_of(:post)
  end

end
```

The `:posts` field points to a list of `:post` results. (This matches
up with what we have on the Ecto side, where `Blog.Accounts.User`
defines a `has_many` association with `Blog.Content.Post`.)

We've already defined the `:post` type, but let's go ahead and add an
`:author` field that points back to our `:user` type. In
`blog_web/schema/content_types.ex`:

``` elixir
object :post do

  # post fields we defined earlier...

  field :author, :user

end
```

Now let's add the `:user` field to our query root object in our
schema, defining a mandatory `:id` argument and using the
`Resolvers.Accounts.find_user/3` resolver function. We also need to
make sure we import the types from `BlogWeb.Schema.AccountTypes` so
that `:user` is available.

In `blog_web/schema.ex`:

```elixir
defmodule BlogWeb.Schema do
  use Absinthe.Schema

  import_types Absinthe.Type.Custom

  # Add this `import_types`:
  import_types BlogWeb.Schema.AccountTypes

  import_types BlogWeb.Schema.ContentTypes

  alias BlogWeb.Resolvers

  query do

    @desc "Get all posts"
    field :posts, list_of(:post) do
      resolve &Resolvers.Content.list_posts/3
    end

    # Add this field:
    @desc "Get a user of the blog"
    field :user, :user do
      arg :id, non_null(:id)
      resolve &Resolvers.Accounts.find_user/3
    end

  end

end
```

Now lets use the argument in our resolver. In `blog_web/resolvers/accounts.ex`:

```elixir
defmodule BlogWeb.Resolvers.Accounts do

  def find_user(_parent, %{id: id}, _resolution) do
    case Blog.Accounts.find_user(id) do
      nil ->
        {:error, "User ID #{id} not found"}
      user ->
        {:ok, user}
    end
  end

end
```

Our schema marks the `:id` argument as `non_null`, so we can be
certain we will receive it. If `:id` is left out of the query,
Absinthe will return an informative error to the user, and the resolve
function will not be called.

> If you have experience writing Phoenix controller actions, you might
> wonder why we can match incoming arguments with atoms instead of
> having to use strings.
>
> The answer is simple: you've defined the arguments in the schema
> using atom identifiers, so Absinthe knows what arguments will be
> used ahead of time, and will coerce as appropriate---culling any
> extraneous arguments given to a query. This means that all arguments
> can be supplied to the resolve functions with atom keys.

Finally you'll see that we can handle the possibility that the query,
while valid from GraphQL's perspective, may still ask for a user that
does not exist. We've decided to return an error in that case.

> There's a valid argument for just returning `{:ok, nil}` when a
> record can't be found. Whether the absence of data constitutes an
> error is a decision you get to make.

## Arguments and Non-Root Fields

Let's assume we want to query all posts from a user published within a
given time range. First, let's add a new field to our `:post` object
type, `:published_at`.

The GraphQL specification doesn't define any official date or time
types, but it does support custom scalar types (you can read more
about them in the [related guide](custom-scalars.html), and
Absinthe ships with several built-in scalar types. We'll use
`:naive_datetime` (which doesn't include timezone information) here.

Edit `blog_web/schema/content_types.ex`:

```elixir
defmodule BlogWeb.Schema.ContentTypes do
  use Absinthe.Schema.Notation

  @desc "A blog post"
  object :post do
    field :id, :id
    field :title, :string
    field :body, :string
    field :author, :user
    # Add this:
    field :published_at, :naive_datetime
  end
end
```

To make the `:naive_datetime` type available, add an `import_types` line to
your `blog_web/schema.ex`:

``` elixir
import_types Absinthe.Type.Custom
```

> For more information about how types are imported,
> read [the guide on the topic](https://github.com/absinthe-graphql/absinthe/blob/master/guides/importing-types.md).
>
> For now, just remember that `import_types` should _only_ be
> used in top-level schema module. (Think of it like a manifest.)

Here's the query we'd like to be able to use, getting the posts for a user
on a given date:

```graphql
{
  user(id: "1") {
    name
    posts(date: "2017-01-01") {
      title
      body
      publishedAt
    }
  }
}
```

To use the passed date, we need to update our `:user` object type and
make some changes to its `:posts` field; it needs to support a `:date`
argument and use a custom resolver. In `blog_web/schema/account_types.ex`:

```elixir
defmodule BlogWeb.Schema.AccountTypes do
  use Absinthe.Schema.Notation

  alias BlogWeb.Resolvers

  object :user do
    field :id, :id
    field :name, :string
    field :email, :string
    # Add the block here:
    field :posts, list_of(:post) do
      arg :date, :date
      resolve &Resolvers.Content.list_posts/3
    end
  end

end
```

For the resolver, we've added another function head to
`Resolvers.Content.list_posts/3`. This illustrates how you can use the
first argument to a resolver to match the parent object of a field. In
this case, that parent object would be a `Blog.Accounts.User` Ecto
schema:

``` elixir
# Add this:
def list_posts(%Blog.Accounts.User{} = author, args, _resolution) do
  {:ok, Blog.Content.list_posts(author, args)}
end
# Before this:
def list_posts(_parent, _args, _resolution) do
  {:ok, Blog.Content.list_posts()}
end
```

Here we pass on the user and arguments to the domain logic function,
`Blog.Content.list_posts/3`, which will find the posts for the user
and date (if it's provided; the `:date` argument is optional). The
resolver, just as when it's used for the top level query `:posts`,
returns the posts in an `:ok` tuple.

> Check out the full implementation of logic for
> `Blog.Content.list_posts/3`--and some simple seed data--in
> the
> [absinthe_tutorial](https://github.com/absinthe-graphql/absinthe_tutorial) repository.

If you've done everything correctly (and have some data handy), if you
start up your server with `mix phx.server` and head over
to <http://localhost:4000/api/graphiql>, you should be able to play
with the query.

It should look something like this:

<img style="box-shadow: 0 0 6px #ccc;" src="/guides/assets/tutorial/graphiql_user_posts.png" alt=""/>

## Next Step

Next up, we look at how to modify our data using mutations.


# 4. Mutations

A blog is no good without new content. We want to support a mutation
to create a blog post:

```graphql
mutation CreatePost {
  createPost(title: "Second", body: "We're off to a great start!") {
    id
  }
}
```

Now we just need to define a `mutation` portion of our schema and
a `:create_post` field:

In `blog_web/schema.ex`:

```elixir
mutation do

  @desc "Create a post"
  field :create_post, type: :post do
    arg :title, non_null(:string)
    arg :body, non_null(:string)
    arg :published_at, :naive_datetime

    resolve &Resolvers.Content.create_post/3
  end

end
```

The resolver in this case is responsible for making any changes and
returning an `{:ok, post}` tuple matching the `:post` type we defined
earlier:

In our `blog_web/resolvers/content.ex` module, we'll add the
`create_post/3` resolver function:

```elixir
def create_post(_parent, args, %{context: %{current_user: user}}) do
  Blog.Content.create_post(user, args)
end
def create_post(_parent, _args, _resolution) do
  {:error, "Access denied"}
end
```

> Obviously things can go wrong in a mutation. To learn more about the
> types of error results that Absinthe supports, read [the guide](https://github.com/absinthe-graphql/absinthe/blob/master/guides/errors.md).

## Authorization

This resolver adds a new concept: authorization. The resolution struct
(that is, an [`Absinthe.Resolution`](Absinthe.Resolution.html))
passed to the resolver as the third argument carries along with it the
Absinthe context, a data structure that serves as the integration
point with external mechanisms---like a Plug that authenticates the
current user. You can learn more about how the context can be used in
the [Context and Authentication](https://github.com/absinthe-graphql/absinthe/blob/master/guides/context-and-authentication.md)
guide.

Going back to the resolver code:

- If the match for a current user is successful, the underlying
  `Blog.Content.create_post/2` function is invoked. It will return a
  tuple suitable for return. (To read the Ecto-related nitty gritty,
  check out the [absinthe_tutorial](https://github.com/absinthe-graphql/absinthe_tutorial)
  repository.)
- If the match for a current user isn't successful, the fall-through
  match will return an error indicating that a post can't be created.





# 5. The Context and Authentication

Absinthe context exists to provide shared values to a given document execution.
A common use would be to pass in the current user of a given request. The context
is set at the call to `Absinthe.run`, and cannot be modified over the course of
a given execution.

## Basic Usage

As a basic example let's think about a profile page, where we want the current user
to be able to access basic information about themselves, but not other users.

First we'll need a very basic schema:

```elixir
defmodule MyAppWeb.Schema do
  use Absinthe.Schema

  @fakedb %{
    "1" => %{name: "Bob", email: "bubba@foo.com"},
    "2" => %{name: "Fred", email: "fredmeister@foo.com"},
  }

  query do
    field :profile, :user do
      resolve fn _, _, _ ->
        # How could we get a current user here?
      end
    end
  end

  object :user do
    field :id, :id
    field :name, :string
    field :email, :string
  end
end
```

A query we might want could look like:

```graphql
{
  profile {
    email
  }
}
```

If we're signed in as user 1, we should get only user 1's email, for example:

```json
{
  "profile": {
    "email": "bubba@foo.com"
  }
}
```

In order to set the context, our call to `Absinthe.run/3` should look like:

```elixir
Absinthe.run(document, MyAppWeb.Schema, context: %{current_user: %{id: "1"}})
```

To access this, we need to update our query's resolve function:

```elixir
query do
  field :profile, :user do
    resolve fn _, _, %{context: %{current_user: current_user}} ->
      {:ok, Map.get(@fakedb, current_user.id)}
    end
  end
end
```

And now it works!

## Context and Plugs

When using Absinthe.Plug you don't have direct access to the Absinthe.run call.
Instead, we can use `Absinthe.Plug.put_options/2` to set context values which
Absinthe.Plug will use to pass it along to Absinthe.run.

Setting up your GraphQL context is as simple as writing a plug that inserts the
appropriate values into the connection.

Let's use this mechanism to set our current_user from the previous example via
an authentication header. We will use the same Schema as before.

First, our plug. We'll be checking the for the `authorization` header, and calling
out to some unspecified authentication mechanism.

```elixir
defmodule MyAppWeb.Context do
  @behaviour Plug

  import Plug.Conn
  import Ecto.Query, only: [where: 2]

  alias MyApp.{Repo, User}

  def init(opts), do: opts

  def call(conn, _) do
    context = build_context(conn)
    Absinthe.Plug.put_options(conn, context: context)
  end

  @doc """
  Return the current user context based on the authorization header
  """
  def build_context(conn) do
    with ["Bearer " <> token] <- get_req_header(conn, "authorization"),
    {:ok, current_user} <- authorize(token) do
      %{current_user: current_user}
    else
      _ -> %{}
    end
  end

  defp authorize(token) do
    User
    |> where(token: ^token)
    |> Repo.one
    |> case do
      nil -> {:error, "invalid authorization token"}
      user -> {:ok, user}
    end
  end

end
```

This plug will use the `authorization` header to lookup the current user. If one
is found, it correctly sets the absinthe context. If you're using Guardian or
some other library that provides utilities for authenticating users you can use
those here too, and just add their output to the context.

If there is no current user it's better to simply not have the `:current_user`
key inside the map, instead of doing `%{current_user: nil}`. This way you an
just pattern match for `%{current_user: user}` in your code and not need to
worry about the nil case.

Using this plug is very simple. If we're just in a normal plug context we can
just make sure it's plugged prior to Absinthe.Plug

```elixir
plug MyAppWeb.Context

plug Absinthe.Plug,
  schema: MyAppWeb.Schema
```

If you're using a Phoenix router, add the context plug to a pipeline.

```elixir
defmodule MyAppWeb.Router do
  use Phoenix.Router

  resource "/pages", MyAppWeb.PagesController

  pipeline :graphql do
    plug MyAppWeb.Context
  end

  scope "/api" do
    pipe_through :graphql

    forward "/", Absinthe.Plug,
      schema: MyAppWeb.Schema
  end
end
```

## Next Step

Now let's take a look at more complex arguments.



# 6. Complex Arguments

In preparation for supporting comments on our blog, let's create users. We're building
a modern mobile first blog of course, and thus want to support either a phone number
or an email as the contact method for a user.

We want to support the following mutations.

Support creation of a user with their email address:

```graphql
mutation CreateEmailUser {
  createUser(contact: {type: EMAIL, value: "foo@bar.com"}, name: "Jane", password: "hunter1") {
    id
    contacts {
      type
      value
    }
  }
}
```

And by using their phone number:

```graphql
mutation CreatePhoneUser {
  createUser(contact: {type: PHONE, value: "+1 123 5551212"}, name: "Joe", password: "hunter2") {
    id
    contacts {
      type
      value
    }
  }
}
```

To do this we need the ability to create nested arguments. GraphQL has input objects
for this purpose. Input objects, like regular object, contain key value pairs, but
they are intended for input only (you can't do circular references with them for example).

Another notion we'll look at here is an enumerable type. We only want to support contact
types `"email"` and `"phone"` at the moment, and GraphQL gives us the ability to
specify this in our schema.

Let's start with our `:contact_type` Enum. In `blog_web/schema/account_types.ex`:

```graphql
enum :contact_type do
  value :phone, as: "phone"
  value :email, as: "email"
end
```

We're using the `:as` option here to make sure the parsed enum is represented by a string
when it's passed to our controllers; this is to ease integration with our Ecto schema
(by default, the enum values are passed as atoms).

> The standard convention for representing incoming enum values in
> GraphQL documents are in all caps. For instance, given our settings
> here, the accepted values would be `PHONE` and `EMAIL` (without
> quotes). See the GraphQL document examples above for examples.
>
> While the `enum` macro supports configuring this incoming format, we
> highly recommend you just use the GraphQL convention.

Now if a user tries to send some other kind of contact type they'll
get a nice error without any extra effort on your part. Enum types are
not a substitute for modeling layer validations however, be sure to
still enforce things like this on that layer too.

Now for our contact input object.

In `blog_web/schema/account_types.ex`:

```graphql
input_object :contact_input do
  field :type, non_null(:contact_type)
  field :value, non_null(:string)
end
```

Note that we name this type `:contact_input`. Input object types have
their own names, and the `_input` suffix is common.

> Important: It's very important to remember that only input
> types---basically scalars and input objects---can be used to model
> input.

Finally our schema, in `blog_web/schema.ex`:

```elixir
mutation do

  #... other mutations

  @desc "Create a user"
  field :create_user, :user do
    arg :name, non_null(:string)
    arg :contact, non_null(:contact_input)
    arg :password, non_null(:string)

    resolve &Resolvers.Accounts.create_user/3
  end

end
```

Suppose in our database that we store contact information in a different database
table. Our mutation would be used to create both records in this case.

There does not need to be a one to one correspondence between how data is structured
in your underlying data store and how things are presented by your GraphQL API.

Our resolver, `blog_web/resolvers/accounts.ex` might look something like this:

```elixir
def create_user(_parent, args, %{context: %{current_user: %{admin: true}}}) do
  Blog.Accounts.create_user(args)
end
def create_user(_parent, args, _resolution) do
  {:error, "Access denied"}
end
```

You'll notice we're checking for `:current_user` again in our Absinthe
context, just as we did before for posts. In this case we're taking
the authorization check a step further and verifying that only
administrators (in this simple example, an administrator is a user
account with `:admin` set to `true`) can create a user.

Everyone else gets an `"Access denied"` error for this field.

> To see the Ecto-related implementation of the
> `Blog.Accounts.create_user/1` function and the (stubbed) authentication logic we're
> using for this example, see the [absinthe_tutorial](https://github.com/absinthe-graphql/absinthe_tutorial)
> repository.

Here's our mutation in action in GraphiQL.

<img style="box-shadow: 0 0 6px #ccc;" src="/guides/assets/tutorial/graphiql_create_user.png" alt=""/>

> Note we're sending a `Authorization` header to authenticate, which a
> plug is handling. Make sure to read the
> related [guide](context-and-authentication.html) for more
> information on how to set-up authentication in your own
> applications.
>
> Our simple tutorial application is just using a simple stub: any
> authorization token logs you in the first user. Obviously not what
> you want in production!







# 7. Dataloader

Maybe you like good performance, or you realized that you are filling your objects with fields that need resolvers like 

```elixir
@desc "A user of the blog"
  object :user do
    field :id, :id
    field :name, :string
    field :contacts, list_of(:contact)
    field :posts, list_of(:post) do
      arg :date, :date
      resolve &Resolvers.Content.list_posts/3
    end
  end
```

This is going to get tedious and error-prone very quickly what if we could support a query that supports associations like

```elixir 
@desc "A user of the blog"
  object :user do
    field :id, :id
    field :name, :string
    field :contacts, list_of(:contact)
    field :posts, list_of(:post) do
       arg :date, :date
       resolve: dataloader(Content))
   end 
  end
```

This way associations are all handled in the context [business logic aware](https://github.com/absinthe-graphql/absinthe/issues/443#issuecomment-405929499) conditions, to support this is actually surprisingly simple.

Since we had already setup users to load associated posts we can change that to use dataloader to illustrate how much simpler this gets.

Let's start by adding `dataloader` as a dependency in `mix.exs`:

```elixir
defp deps do
  [
    {:dataloader, "~> 1.0.4"}
    << other deps >>
  ]
```

Next, we need to set up dataloader in our context which allows us to load associations using rules:

In `lib/blog/content.ex`:

```elixir
  def data(), do: Dataloader.Ecto.new(Repo, query: &query/2)
  
  def query(queryable, params) do
    
    queryable
  end 
```

This sets up a loader that can use pattern matching to load different rules for different queryables, also note this function is passed in the context as the second parameter and that can be used for further filtering.

Then let's add a configuration to our schema (in `lib/blog_web/schema.ex`) so that we can allow Absinthe to use Dataloader:

```elixir
defmodule BlogWeb.Schema do
  use Absinthe.Schema
  
  def context(ctx) do
     loader =
       Dataloader.new()
       |> Dataloader.add_source(Content, Content.data())

    Map.put(ctx, :loader, loader)
  end

  def plugins do
    [Absinthe.Middleware.Dataloader | Absinthe.Plugin.defaults()]
  end

  # << rest of the file>>
```

The loader is all set up, now let's modify the resolver to use Dataloader. In `lib/blog_web/schema/account_types.ex` modify the user object to look as follows:

```elixir
@desc "A user of the blog"
  object :user do
    field :id, :id
    field :name, :string
    field :contacts, list_of(:contact)
    field :posts, list_of(:post) do
       arg :date, :date
       resolve: dataloader(Content))
   end 
  end
```

That's it! You are now loading associations using [Dataloader](https://github.com/absinthe-graphql/dataloader)

## More Examples 
While the above examples are simple and straightforward we can use other strategies with loading associations consider the following:

```elixir
object :user do
  field :posts, list_of(:post), resolve: fn user, args, %{context: %{loader: loader}} ->
    loader
    |> Dataloader.load(Blog, :posts, user)
    |> on_load(fn loader ->
      {:ok, Dataloader.get(loader, Blog, :posts, user)}
    end)
  end
```

In this example, we are passing some args to the query in the context where our source lives. For example, this function now receives `args` as `params` meaning we can do now do fun stuff like apply rules to our queries like the following:

```elixir
def query(query, %{has_admin_rights: true}), do: query

def query(query, _), do: from(a in query, select_merge: %{street_number: nil})
```

This example is from the awesome [EmCasa Application](https://github.com/emcasa/backend/blob/master/apps/re/lib/addresses/addresses.ex) :) you can see how the [author](https://github.com/rhnonose) is only loading street numbers if a user has admin rights and the same used in a [resolver](https://github.com/emcasa/backend/blob/9a0f86c11499be6e1a07d0b0acf1785521eedf7f/apps/re_web/lib/graphql/resolvers/addresses.ex#L11).

Check out the [docs](https://hexdocs.pm/dataloader/) for more non-trivial ways of using Dataloader.


# 8. Subscriptions

When the need arises for near realtime data GraphQL provides subscriptions. We want to support subscriptions that look like


```graphql
subscription{
  newPost {
    id
    name
  }
}
```

Since we had already setup mutations to handle creation of posts we can use that as the event we want to subscribe to. In order to achieve this we have to do a little bit of set up


Let's start by adding `absinthe_phoenix` as a dependency

In `mix.exs`

```elixir
defp deps do
  [
    {:absinthe_phoenix, "~> 1.4.0"}
    << other deps >>
  ]
```

Then we need to add a supervisor to run some processes for the to handle result broadcasts

In `lib/blog/application.ex`:

```elixir
  children = [
    # other children ...
    {BlogWeb.Endpoint, []}, # this line should already exist
    {Absinthe.Subscription, [BlogWeb.Endpoint]}, # add this line
    # other children ...
  ]
```



The lets add a configuration to the phoenix endpoint so it can provide some callbacks Absinthe expects, please note while this guide uses phoenix. Absinthe's support for Subscriptions is good enough to be used without websockets even without a browser.

In `lib/blog_web/endpoint.ex`:


```elixir
defmodule BlogWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :blog # this line should already exist 
  use Absinthe.Phoenix.Endpoint # add this line

  << rest of the file>>
```

The `PubSub` stuff is now set up, let's configure our sockets

In `lib/blog_web/channels/user_socket.ex`

``` elixir
defmodule BlogWeb.UserSocket do
  use Phoenix.Socket # this line should already exist
  use Absinthe.Phoenix.Socket, schema: BlogWeb.Schema # add

  << rest of file>>
```

Lets now configure GraphQL to use this Socket.

In `lib/blog_web/router.ex` :

```elixir
defmodule BlogWeb.Router do
  use BlogWeb, :router

  pipeline :api do
    plug :accepts, ["json"]
    plug BlogWeb.Context
  end

  scope "/api" do
    pipe_through :api

    forward "/graphiql", Absinthe.Plug.GraphiQL,
      schema: BlogWeb.Schema,
      socket: BlogWeb.UserSocket # add this line


    forward "/", Absinthe.Plug,
      schema: BlogWeb.Schema
  end

end
```


Now let/s set up a subscription root object in our Schema to listen for an event. For this subscription we can set it up to listen every time a new post is created.


In `blog_web/schema.ex` :

```elixir

subscription do

  field :new_post, :post do
    config fn _args, _info ->
      {:ok, topic: "*"}
    end
  end

end
```

The `new_post` field is a pretty regular field only new thing here is the `config` macro, this is
here to help us know which clients have subscribed to which fields. Much like WebSockets subscriptions work by allowing t a client to subscribe to a topic.

Topics are scoped to a field and for now we shall use `*` to indicate we care about all the posts, and that's it!

If you ran the request at this moment you would get a nice message telling you that your subscriptions will appear once after they are published but you create a post and alas! no data what cut?

Once a subscription is set up it waits for a target event to get published in order for us to collect this information we need to publish to this subscription

In `blog_web/resolvers/content.ex`:

```elixir
def create_post(_parent, args, %{context: %{current_user: user}}) do
    # Blog.Content.create_post(user, args)
    case Blog.Content.create_post(user, args) do
      {:ok, post} ->
        # add this line in
        Absinthe.Subscription.publish(BlogWeb.Endpoint, post,
        new_post: "*"
        )

        {:ok, post}
      {:error, changeset} ->
        {:ok, "error"}
      end
  end
```

With this, open a tab and run the query at the top of this section. Then open another tab and run a mutation to add a post you should see a result in the other tab have fun.

<img style="box-shadow: 0 0 6px #ccc;" src="/guides/assets/tutorial/graphiql_new_post_sub.png" alt=""/>



# 9. Conclusion

With this we have a basic GraphQL based API for a blog. Head on over
to [the github page](https://github.com/absinthe-graphql/absinthe_tutorial) if
you want the final code.

We hope to expand this tutorial to include a comment system that will
acquaint you with Union types and Fragments in the coming days.

Head on over to the topic guides for further reading, and see
the [community page](community.html) for information
on how to get help, ask questions, or contribute!

## Please Help!

This tutorial is a work in progress, and while it covers the basics of
using Absinthe, there is plenty more that can be added and improved
upon. It's important that it's kept up-to-date, too, so if you notice
something that's slipped by us, please help us fix it!

Please contribute your GitHub issues (and pull requests!):

- The tutorial text is under `guides/tutorial` in the [absinthe](https://github.com/absinthe-graphql/absinthe)
  repository. It's in Markdown and easy to edit!
- The tutorial code located in the [absinthe_tutorial](https://github.com/absinthe-graphql/absinthe_tutorial) repository.
