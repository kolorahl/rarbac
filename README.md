rarbac
======

## What is rarbac?

`rarbac` stands for "Rails Role-Based Access Control". I couldn't find a good
gem that did RBAC and integrated with a Rails app, so I made one. Since this is
basically a stripped down Rails project, I created it as a Rails engine
(a.k.a. plugin).

However, there was a UC Berkley project which did well but fell a little short
and does not appear to be maintained any more, called
[rails_access](https://github.com/ucberkeley/rails_access). You will probably
see similarities in the data structure between that project and this
one.

## Installation

You simply add the gem and then mount the engine.

In your Gemfile:

```ruby
gem 'rarbac'
```

In `config/routes.rb`:

```ruby
mount Rarbac::Engine, at: "rarbac"
```

The project includes migrations for non-user data. Your Rails app must have, and
supply, its own user model. To generate the migrations for `rarbac` simply run:

    rake rarbac:install:migrations

Then run migrations. One more step to ensure everything works properly. Open
your user model and add the following to it:

```ruby
has_many :user_roles,                         class_name: "Rarbac::UserRole"
has_many :roles,       through: :user_roles,  class_name: "Rarbac::Role"
has_many :permissions, through: :roles,       class_name: "Rarbac::Permission"
has_many :actions,     through: :permissions, class_name: "Rarbac::Action"

include Rarbac::UserHelper
```

That may seem like a lot, but really we're just off-loading a lot of the
querying work to the underlying `ActiveRecord` base (or whatever else you are
using). If we didn't do this we would have to do things like:

```ruby
user.user_roles.roles.permissions.where(...)
```

By including all those `has_many X through: Y` statements, we can go straight to
what we want:

```ruby
user.permissions.where(...)
```

With migrations installed, run, and the user model setup, start your application
and enjoy! Read on to make the most out of `rarbac`.

## How it works

You will have a few new models, under the namespace `Rarbac`, called `Role`,
`Action`, `Permission`, and `UserRole`.

The `Role` model/table holds a collections of roles, with an optional
description. e.g. "guest", "user", "admin".

The `Action` model/table holds a collection of actions that a user may perform
in the system. e.g. "new_post", "delete_post", "new_comment".

The `Permission` model/table links roles to actions. e.g. "the `admin` role has
permission to the `delete_post` action".

The `UserRole` model/table links users to roles. e.g. "Jill has the `user`
role".

It is important to note that user ids are not uniquely kept in the `UserRole`
model, so you may have a single user with more than one role.

### Controller Filters

You may use the `Rarbac` controller filters/functions by including
`Rarbac::ApplicationHelper`. The only assumption made is the presence of a
resource (function or attribute) named `current_user`. This is modeled after the
popular authentication gem [Devise](https://github.com/plataformatec/devise).

You may add `before_filter :ensure_permission!` to create role-based authorization
on the controller and action being executed. For example:

```ruby
class PostsController < ApplicationController::Base
  include Rarbac::ApplicationHelper

  before_filter :ensure_permission!

  def show
    ...
  end

  def destroy
    ...
  end
end
 ```

If you had an `Action` specified for "posts#destroy", this would ensure that
`current_user` had permission to that action. If no such action existed,
however, it is assumed that you don't need permission and anyone, even
un-authenticated users, would be allowed access to that route.

You may also customize your filters using the normal application controller
options:

```ruby
before_filter :ensure_permission!, only: :destroy
```

Now the filter only runs when the `destroy` action is being executed. You may
also use the block method of invoking a filter to further customize your
options:

```ruby
before_filter only: :destroy do |controller|
  controller.ensure_permission!(:delete_post)
end
```

But wait, there's more! What if you want to check a role rather than an
individual permission to some action? Maybe an entire controller should be
off-limits to everyone but those with the `admin` role. That's easy:

```ruby
before_filter do |controller|
  controller.ensure_role!(:admin)
end
```

Bonus: it accepts arrays of roles.

```ruby
before_filter do |controller|
  controller.ensure_role!(:author, :admin)
  # The above uses an "OR" operation; use the plural to perform "AND"
  # e.g. controller.ensure_roles!(:author, :admin)
end
```

Don't forget that you may create your own filter functions and call `ensure_*`
from there. However, if you create a custom filter and have access to
`current_user`, it might be better, and more configurable, to use the model
functions defined in the next section.

### Model Functions

With the appropriate permissions added to your user model you may very easily
check on a user's roles:

```ruby
user.has_role?(:admin)
user.has_role?(:admin, :author)  # Singular noun implies "OR" operation
                                 # e.g. has_role?(:admin) || has_role?(:author)
user.has_roles?(:admin, :author) # Plural noun implies "AND" operation
                                 # e.g. has_role?(:admin) && has_role?(:author)
```

And also permissions:

```ruby
user.has_permission?(:delete_post)
```

It's important to note that permissions don't accept arrays. Why? Because very
rarely should you be checking for a collection of permissions in a single
"question". If you want to know if a user has permission to multiple actions,
more likely you actually want to know if the user is assigned to one or more
roles instead. That's why the `has_role?` and `has_roles?` functions have the
argument-array option.

### View Helpers

There is no explicit view helper system. The idea is that the controller filters
will help control when a view is rendered, and the model functions can further
control what is rendered within a template.
