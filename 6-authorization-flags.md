# PATTERN 6: Authorization Flags in Enums

## Original Example


```ruby
class Role < Enumerations::Base
  value :client, name: 'Client', non_dev?: true
  value :collaborator, name: 'Collaborator', non_dev?: true
  value :developer, name: 'Developer', non_dev?: false
end
```

```ruby
class ProjectPolicy < ApplicationPolicy
  def show?
    user_membership.role.developer? || non_dev_on_project?
  end

  private

  def non_dev_on_project?
    user_membership.role.non_dev? && user.projects.include?(record)
  end
end
```

## Analysis

| Pros | Cons |
|---------|---------|
| Clean predicate methods (`role.non_dev?`) | Mixing authorization concerns with enum data |
| Self-documenting - flags are visible in enum definition | Limited to boolean flags (can't express complex authorization rules) |
| Single source of truth for role characteristics | Authorization logic should be in policies, not enums |
| | Hard to add role-specific authorization logic |
| | Can't easily override or extend for specific contexts |
| | Makes enum coupled to authorization domain |

## Rails Enum Alternative


### Option 1: Constant array in model (Simplest)
```ruby
class Membership < ApplicationRecord
  enum role: [:client, :collaborator, :developer].index_with(&:to_s)

  NON_DEV_ROLES = %w[client collaborator].freeze

  def non_dev_role?
    NON_DEV_ROLES.include?(role)
  end

  # Or more explicit:
  def non_dev?
    client? || collaborator?
  end
end

class ProjectPolicy < ApplicationPolicy
  def show?
    user_membership.developer? || non_dev_on_project?
  end

  private

  def non_dev_on_project?
    user_membership.non_dev? && user.projects.include?(record)
  end
end
```


### Option 2: Module with role definitions (Better organization)
```ruby
module Roles
  CLIENT = 'client'
  COLLABORATOR = 'collaborator'
  DEVELOPER = 'developer'

  ALL = [CLIENT, COLLABORATOR, DEVELOPER].freeze
  NON_DEV = [CLIENT, COLLABORATOR].freeze
  DEV = [DEVELOPER].freeze

  def self.non_dev?(role)
    NON_DEV.include?(role)
  end

  def self.dev?(role)
    DEV.include?(role)
  end
end

class Membership < ApplicationRecord
  enum role: [Roles::CLIENT, Roles::COLLABORATOR, Roles::DEVELOPER].index_with(&:to_s)

  def non_dev?
    Roles.non_dev?(role)
  end
end
```


### Option 3: Dedicated authorization object (Most explicit)
```ruby
class RoleAuthorizer
  attr_reader :role

  def initialize(role)
    @role = role
  end

  def non_dev?
    %w[client collaborator].include?(role)
  end

  def developer?
    role == 'developer'
  end

  def can_view_all_projects?
    developer?
  end

  def can_manage_members?
    developer?
  end
end

class Membership < ApplicationRecord
  enum role: [:client, :collaborator, :developer].index_with(&:to_s)

  def authorizer
    @authorizer ||= RoleAuthorizer.new(role)
  end

  delegate :non_dev?, :can_view_all_projects?, :can_manage_members?,
           to: :authorizer
end

class ProjectPolicy < ApplicationPolicy
  def show?
    user_membership.authorizer.developer? || non_dev_on_project?
  end

  private

  def non_dev_on_project?
    user_membership.authorizer.non_dev? && user.projects.include?(record)
  end
end
```


### Option 4: In policy itself (Most Rails-idiomatic with Pundit)
```ruby
class Membership < ApplicationRecord
  enum role: [:client, :collaborator, :developer].index_with(&:to_s)
end

class ProjectPolicy < ApplicationPolicy
  def show?
    developer? || non_dev_on_project?
  end

  private

  def developer?
    user_membership.developer?
  end

  def non_dev_on_project?
    non_dev? && user.projects.include?(record)
  end

  def non_dev?
    user_membership.client? || user_membership.collaborator?
  end
end
```
Most explicit - authorization logic stays in policy


## Verdict


Authorization logic should live in policies or dedicated authorization
objects, not in enum definitions. Enums should define states/categories,
not business rules about what those states can do.

### Benefits of keeping auth logic in policies:
- Single Responsibility - enums define roles, policies define permissions
- Testable - can test authorization independently
- Flexible - easy to add complex rules like time-based or context-based auth
- Standard Rails/Pundit pattern
- Easy to audit - all auth rules in policy files

### Recommendation by complexity:

- Simple flags (like non_dev?): Use Option 1 (method in model)
- Multiple role groups: Use Option 2 (module with constants)
- Complex authorization: Use Option 3 (dedicated authorizer)
- Standard Pundit setup: Use Option 4 (keep logic in policy)
