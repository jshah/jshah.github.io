---
layout: post
title:  "Encapsulate For Easy Refactors"
date:   2020-12-14 10:00:00 -0700
categories: ruby rails refactor
---

An application is a living, breathing code base that will continually change over time. As the application evolves, early decisions won't scale, and shortcuts taken will reveal technical debt. When the time comes to address these problems, one thing you can start doing today to make refactoring tomorrow easier is using encapsulation.

# Example

Imagine your application has a `User` model, and the `User` can have a role. The role is stored as a column on the model.

```ruby
# == Schema Information
#
# Table name: users
#
#  id                     :bigint(8)        not null, primary key
#  email                  :string           default(""), not null
#  first_name             :string
#  last_name              :string
#  role                   :string
#  created_at             :datetime         not null
#  updated_at             :datetime         not null
#
class User < ApplicationRecord
end
```

You might have actions in your controller where you check if a user has a specific role. You might decide to directly access the role attribute and compare it with the role you care about:

```ruby
class ItemController < ApplicationController
  def update
    if user.role == 'admin' || user.role == 'operations'
       update_item(params[:item_id])
    end
  end
end
```

While this seems harmless at first, it becomes painful to refactor when you have to extend the relationship so a user can have many roles.

## Extending Role To Its Own Model

Imagine your product manager asks you to support users having multiple roles. You will have to move the role attribute from the `User` model to its own model.

### Models

The resulting model design might look like the following. The `User` model now has a `has_many` relationship through a joining class (`UserRole`) to a `Role` model.

```ruby
# == Schema Information
#
# Table name: users
#
#  id                     :bigint(8)        not null, primary key
#  email                  :string           default(""), not null
#  first_name             :string
#  last_name              :string
#  role                   :string
#  created_at             :datetime         not null
#  updated_at             :datetime         not null
#
class User < ApplicationRecord
  has_many :user_roles, dependent: :destroy
  has_many :roles, through: :user_roles
end
```

```ruby
# == Schema Information
#
# Table name: user_roles
#
#  id         :bigint(8)        not null, primary key
#  created_at :datetime         not null
#  updated_at :datetime         not null
#  role_id    :bigint(8)
#  user_id    :bigint(8)
#
class UserRole < ApplicationRecord
  belongs_to :user
  belongs_to :role
end
```

```ruby
# == Schema Information
#
# Table name: users
#
#  id                     :bigint(8)        not null, primary key
#  name                   :string
#  created_at             :datetime         not null
#  updated_at             :datetime         not null
#
class Role < ApplicationRecord
  has_many :user_roles, dependent: :destroy
  has_many :users, through: :user_roles
end
```

### Usage

Instead of checking if a `user.role` is equal to a role, we'll now check if a specific role is in the list of roles attached to a user.

```ruby
admin_role = Role.create(name: 'admin')
operations_role = Role.create(name : 'operations')
user = User.find(1)

# add roles to user
user.roles << admin_role
user.roles << operations_role
user.roles # [<Role name: 'admin'>, <Role name: 'operations'>]

# check if a user is an admin
user.roles.exists?(name: 'admin') # true
```

### Refactoring Usages

When we refactor the `ItemController` to use these new models and methods, we first want to bring all methods into the `User` class and then update usages.

```ruby
# Encapsulate all User role related methods.
# New feature is also gated behind a feature flag.

class User < ApplicationRecord
  has_many :user_roles, dependent: :destroy
  has_many :roles, through: :user_roles

  def has_role?(role_name)
    if feature_flag_on?
      roles.exists?(name: role_name)
    else
      role == role_name
    end
  end

  def admin?
    has_role?('admin')
  end

  def operations?
    has_role?('operations')
  end
end
```

With these methods now encapsulated in the `User` class, checking if a user has a certain role becomes easy!

```ruby
# No explicit feature flag check needed.
# All logic is encapsulated inside the User methods.

class ItemController < ApplicationController
  def update
    if user.admin? || user.operations?
       update_item(params[:item_id])
    end
  end
end
```
