# PATTERN 2: Query Factory Pattern

## Original Example


```ruby
class ApplicationType < BaseEnumeration
  values amk: { id: 1, name: 'AMK' },
         aek: { id: 2, name: 'AEK' },
         dpf: { id: 3, name: 'DPF' }

  def releases
    case symbol
    when :amk, :aek then Release.amk
    when :dpf then Release.dpf
    end
  end
end
```

## Analysis

| Pros | Cons |
|---------|---------|
| Encapsulates query logic with the enum definition | Creates coupling between enum and query/scope logic |
| Polymorphic behavior per enum value | Not immediately obvious where query logic lives |
| Single place to see which types map to which queries | Violates Single Responsibility Principle (enum shouldn't know about queries) |
| | Hard to test in isolation |
| | Mixes domain logic with data definition |
| | Makes enum class a "God object" over time |

## Rails Enum Alternative

### Option 1: Simple class with query methods (Best for simple cases)

```ruby
class ApplicationType
  AMK = 'amk'
  AEK = 'aek'
  DPF = 'dpf'

  def self.releases_for(type)
    case type
    when AMK, AEK then Release.amk
    when DPF then Release.dpf
    end
  end
end

class Application < ApplicationRecord
  enum application_type: [:amk, :aek, :dpf].index_with(&:to_s)

  def releases
    ApplicationType.releases_for(application_type)
  end
end

# Usage:
# application.releases
```
### Option 2: Service Object (Better for complex logic)

```ruby
class ApplicationReleaseQuery
  def self.call(application_type)
    case application_type
    when 'amk', 'aek'
      Release.amk
    when 'dpf'
      Release.dpf
    else
      Release.none
    end
  end
end

class Application < ApplicationRecord
  enum application_type: [:amk, :aek, :dpf].index_with(&:to_s)

  def releases
    ApplicationReleaseQuery.call(application_type)
  end
end

# Usage:
# application.releases
# ApplicationReleaseQuery.call('amk')
```
### Option 3: Query Object Pattern (Most explicit and testable)

```ruby
module ApplicationQueries
  class ReleasesQuery
    def initialize(application_type)
      @application_type = application_type
    end

    def call
      case @application_type
      when 'amk', 'aek'
        Release.amk
      when 'dpf'
        Release.dpf
      else
        Release.none
      end
    end
  end
end

class Application < ApplicationRecord
  enum application_type: [:amk, :aek, :dpf].index_with(&:to_s)

  def releases
    ApplicationQueries::ReleasesQuery.new(application_type).call
  end
end

# Usage:
# application.releases
# ApplicationQueries::ReleasesQuery.new('amk').call
```
### Option 4: Scope on Release model (Most Rails-idiomatic)

```ruby
class Release < ApplicationRecord
  scope :amk, -> { where(application_type: ['amk', 'aek']) }
  scope :dpf, -> { where(application_type: 'dpf') }

  def self.for_application_type(type)
    case type
    when 'amk', 'aek' then amk
    when 'dpf' then dpf
    else none
    end
  end
end

class Application < ApplicationRecord
  enum application_type: [:amk, :aek, :dpf].index_with(&:to_s)

  def releases
    Release.for_application_type(application_type)
  end
end
```
## Verdict

This pattern creates unclear boundaries. Enumerations should define values and their attributes, not business logic about how to query other models.

### Problems with the enumerations approach:
- Where do I look for query logic? The enum? The model? A service?
- How do I test the query logic separately from the enum?
- The enum becomes a dumping ground for any type-specific behavior

### Recommendation by use case:
- Simple cases: Use Option 1 (class method)
- Medium complexity: Use Option 2 (Service Object)
- Complex queries: Use Option 3 (Query Object)
- When queries are Release-focused: Use Option 4 (scope on model)

### All alternatives are:
- More testable (can test query logic independently)
- More maintainable (clear separation of concerns)
- More discoverable (know where to look for query logic)
- More Rails-idiomatic (follows standard patterns)
