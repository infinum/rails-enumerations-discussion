# PATTERN 7: Storing Class References in Enums

## Original Example


```ruby
class FairType < EnumerationsBase
  value :demo,
        name: 'Demo',
        event_serializer: fair_defaults.serializer.event,
        nomenclature_definition: fair_defaults.nomenclature_definition,
        authentication: ::Users::Auth::Local

  value :ispo,
        name: 'ISPO',
        event_serializer: fair_defaults.serializer.event,
        nomenclature_definition: fair_defaults.nomenclature_definition,
        authentication: ::Users::Auth::Remote
end
```

```ruby
def serializers
  jsonapi_class.merge(
    Event: ActsAsTenant.current_tenant.fair_type.event_serializer
  )
end
```

## Analysis


### What This Pattern Does
Stores actual Ruby class references as enum attributes. The `authentication`
attribute stores `::Users::Auth::Local` or `::Users::Auth::Remote` classes,
and `event_serializer` stores serializer class references. These classes
are then used polymorphically based on the enum value.

| Pros | Cons |
|---------|---------|
| Polymorphic class selection based on enum value | 🚨 major code smell: Enums should not store class references |
| Centralized class mapping in one definition | Makes autoloading/eager loading extremely complex |
| Dynamic behavior per enum type | Hard to reason about dependencies (what classes are loaded when?) |
| | Breaks encapsulation and separation of concerns |
| | Makes code hard to grep (can't search for `Users::Auth::Local`) |
| | Difficult to test (must load all classes for enum) |
| | Creates hidden coupling between enum and implementation classes |
| | Rails autoloading can cause subtle bugs in development vs production |
| | Makes refactoring dangerous (change class name = break enum) |
| | Violates Dependency Inversion Principle |

## Rails Enum Alternative


### Option 1: Strategy pattern with registry (Best - Clean and testable)
```ruby
module EventSerializers
  class Demo
    def serialize(event)
      # Demo-specific serialization
    end
  end

  class Ispo
    def serialize(event)
      # ISPO-specific serialization
    end
  end

  class Registry
    SERIALIZERS = {
      'demo' => Demo,
      'ispo' => Ispo
    }.freeze

    def self.for(fair_type)
      SERIALIZERS.fetch(fair_type) do
        raise ArgumentError, "Unknown fair type: #{fair_type}"
      end
    end

    def self.serialize(fair_type, event)
      for(fair_type).new.serialize(event)
    end
  end
end

module AuthenticationStrategies
  STRATEGIES = {
    'demo' => Users::Auth::Local,
    'ispo' => Users::Auth::Remote
  }.freeze

  def self.for(fair_type)
    STRATEGIES.fetch(fair_type)
  end
end

class Fair < ApplicationRecord
  enum fair_type: [:demo, :ispo].index_with(&:to_s)

  def event_serializer
    EventSerializers::Registry.for(fair_type)
  end

  def authentication_strategy
    AuthenticationStrategies.for(fair_type)
  end
end
```

**In controller:**

```ruby
def serializers
  jsonapi_class.merge(
    Event: ActsAsTenant.current_tenant.fair.event_serializer
  )
end
```
fair.event_serializer # => EventSerializers::Demo
fair.authentication_strategy # => Users::Auth::Local


### Option 2: Service object with factory method (Good for complex logic)
```ruby
class FairServiceFactory
  def self.serializer_for(fair_type)
    case fair_type
    when 'demo'
      EventSerializers::Demo
    when 'ispo'
      EventSerializers::Ispo
    else
      raise ArgumentError, "Unknown fair type: #{fair_type}"
    end
  end

  def self.authentication_for(fair_type)
    case fair_type
    when 'demo'
      Users::Auth::Local
    when 'ispo'
      Users::Auth::Remote
    else
      raise ArgumentError, "Unknown fair type: #{fair_type}"
    end
  end
end

class Fair < ApplicationRecord
  enum fair_type: [:demo, :ispo].index_with(&:to_s)

  def event_serializer
    FairServiceFactory.serializer_for(fair_type)
  end

  def authentication_strategy
    FairServiceFactory.authentication_for(fair_type)
  end
end
```
### Option 3: Configuration object (Best for many attributes)
```ruby
class FairConfig
  CONFIGS = {
    demo: {
      serializer: EventSerializers::Demo,
      authentication: Users::Auth::Local,
      nomenclature: NomenclatureDefinitions::Demo
    },
    ispo: {
      serializer: EventSerializers::Ispo,
      authentication: Users::Auth::Remote,
      nomenclature: NomenclatureDefinitions::Ispo
    }
  }.freeze

  def self.for(fair_type)
    CONFIGS.fetch(fair_type.to_sym)
  end

  def self.serializer_for(fair_type)
    for(fair_type)[:serializer]
  end

  def self.authentication_for(fair_type)
    for(fair_type)[:authentication]
  end
end

class Fair < ApplicationRecord
  enum fair_type: [:demo, :ispo].index_with(&:to_s)

  def config
    @config ||= FairConfig.for(fair_type)
  end

  def event_serializer
    config[:serializer]
  end

  def authentication_strategy
    config[:authentication]
  end
end
```

## Verdict

Storing class references in enums introduces several architectural challenges that become more problematic as the codebase grows.

### Why this pattern creates issues:

**1. Autoloading Complications:**
Rails uses different class loading strategies in development (lazy-loading) versus production (eager-loading). When class references are stored as enum attributes, you may encounter "uninitialized constant" errors that only manifest in production environments, making them difficult to debug and reproduce locally.

**2. Tight Coupling:**
The enum becomes tightly coupled to specific implementation classes. Changing a class name or moving it to a different namespace requires updating the enum definition, creating a dependency that goes against the principle of loose coupling.

**3. Testing Complexity:**
Testing the enum requires loading all implementation classes. Similarly, testing one implementation requires loading the enum and all other implementations it references. This makes it difficult to write focused unit tests and increases test setup overhead.

**4. Discoverability Issues:**
Finding where `Users::Auth::Local` is used becomes challenging because it's stored as an attribute value rather than explicitly referenced in code. Standard grep searches and IDE "find usages" features won't locate these references reliably.

**5. Dependency Inversion Violation:**
The enum (high-level policy) depends directly on concrete implementation classes (low-level details). This violates the Dependency Inversion Principle, which suggests depending on abstractions rather than concretions.

This approach offers several advantages:

- The mapping between fair types and serializer classes is explicit and searchable
- Finding class usages is straightforward with standard search tools
- Autoloading works predictably since classes are referenced directly in the registry
- Each component can be tested independently without loading the entire dependency graph
- Adding new strategies only requires updating the registry in one place
- The pattern follows established dependency inversion principles

### Recommendation

For class selection based on enum values, the Strategy Pattern with an explicit registry (Option 1) provides the best balance of clarity, maintainability, and testability.
