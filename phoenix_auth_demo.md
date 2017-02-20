# auth

Note:
mix phoenix.new login_app
mix ecto.create

remember, `iex -S mix` is equivalent to `rails c`
---

+ models
  + user
+ controllers
  + user
  + session
+ views
+ templates
  + sign up
  + log in
  + user show

---

## create user model

Note:
mix phoenix.gen.model User users email:string password_digest:string

---

## migration

Note:
add `null:false` to email; `create unique_index(:users, [:email])`

---

## edit user model

validations & virtual password field

Note:
`field :password, :string, virtual: true`

```
@required_fields ~w(email)a
@optional_fields ~w()a

def changeset(struct, params \\ %{}) do
  struct
  |> cast(params, @required_fields ++ @optional_fields)
  |> validate_required(@required_fields)
end
```

---

## user controller

Note:
```
defmodule LoginApp.UserController do
  use LoginApp.Web, :controller

  alias LoginApp.User

  def show(conn, %{"id" => id}) do
    user = Repo.get!(User, id)
    render conn, "show.html", user: user
  end

  def new(conn, _params) do
    changeset = User.changeset(%User{})
    render conn, "new.html", changeset: changeset
  end

  def create(conn, %{"user" => user_params}) do
  end
end
```
---

## router

`resources "/users", UserController, only: [:show, :new, :create]`

---

## view & templates

Note:
```
defmodule LoginApp.UserView do
  use LoginApp.Web, :view
end
```

```
<h1>User Registration</h1>
<%= form_for @changeset, user_path(@conn, :create), fn f -> %>
  <%= if @changeset.action do %>
    <div class="alert alert-danger">
      <p>There are some errors</p>
    </div>
  <% end %>
  <div class="form-group">
    <%= text_input f, :email, placeholder: "Email",
                              class: "form-control" %>
    <%= error_tag f, :email %>
  </div>
  <div class="form-group">
    <%= password_input f, :password, placeholder: "Password",
                                     class: "form-control" %>
    <%= error_tag f, :password %>
  </div>
  <%= submit "Create User", class: "btn btn-primary" %>
<% end %>
```

```
# ../layout/app.html.eex
<ul class="nav nav-pills pull-right">
  <li>
    <%= link "Register", to: user_path(@conn, :new) %>
  </li>
</ul>
```
---

## creating users

add `comeonin` to list of dependencies; `mix deps.get` (like `bundle`)

create registration changeset in user model

implement `:create` action in user controller

Note:
```
def registration_changeset(struct, params) do
  struct
  |> changeset(params)
  |> cast(params, ~w(password)a, [])
  |> validate_length(:password, min: 6, max: 100)
  |> hash_password
end
defp hash_password(changeset) do
  case changeset do
    %Ecto.Changeset{valid?: true,
                    changes: %{password: password}} ->
      put_change(changeset,
                  :password_digest,
                  Comeonin.Bcrypt.hashpwsalt(password))
    _ ->
      changeset
  end
end
```

---

## creating sessions

session controller & routes

add `:guardian` as a dependency to handle sessions (via JWTs?)

find user by email, see if the pw matches, `Guardian.Plug.sign_in(user)`

add a current_user helper module and a `:with_session` pipeline

Note:
```
# config.exs
config :guardian, Guardian,
 issuer: "LoginApp.#{Mix.env}",
 ttl: {30, :days},
 verify_issuer: true,
 serializer: LoginApp.GuardianSerializer,
 secret_key: to_string(Mix.env) <> "SuPerseCret_aBraCadabrA"
```

```
# web/auth/guardian_serializer.ex
defmodule LoginApp.GuardianSerializer do
  @behaviour Guardian.Serializer
  alias LoginApp.Repo
  alias LoginApp.User
  def for_token(user = %User{}), do: { :ok, "User:#{user.id}" }
  def for_token(_), do: { :error, "Unknown resource type" }
  def from_token("User:" <> id), do: { :ok, Repo.get(User, id) }
  def from_token(_), do: { :error, "Unknown resource type" }
end
```

---

## deleting sessions

`Guardian.Plug.sign_out(conn)`; redirect

---

## refactor

refactor out common auth functionality into a helper module