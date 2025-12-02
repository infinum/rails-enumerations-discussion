# PATTERN 9: Storing Model Classes in Enums

## Original Example



```ruby
class ServiceType < Enumerations::Base
  value(
    :ocean,
    id: :ocean,
    name: 'Ocean',
    order: Ocean::Order,
    offer: Ocean::Offer,
    order_index_includes: [:from_location, :to_location, :offers]
  )

  value(
    :lcl,
    id: :lcl,
    name: 'LCL',
    order: Lcl::Order,
    offer: Lcl::Offer,
    order_index_includes: [:from_location, :to_location, :offers]
  )

  value(
    :air,
    id: :air,
    name: 'Air',
    order: Air::Order,
    offer: Air::Offer,
    order_index_includes: [:from_location, :to_location, :offers]
  )

  value(
    :land,
    id: :land,
    name: 'Land',
    order: Land::Order,
    offer: Land::Offer,
    order_index_includes: [:from_location, :to_location, :offers]
  )

  value(
    :ltl,
    id: :ltl,
    name: 'LTL',
    order: Ltl::Order,
    offer: Ltl::Offer,
    order_index_includes: [:from_location, :to_location, :offers]
  )

  value(
    :warehouse,
    id: :warehouse,
    name: 'Warehouse',
    order: Warehouse::Order,
    offer: Warehouse::Offer,
    order_index_includes: [:location, :offers]
  )

  value(
    :customs_clearance,
    id: :customs_clearance,
    name: 'Customs Clearance',
    order: CustomsClearance::Order,
    offer: CustomsClearance::Offer,
    order_index_includes: [:location, :origin_country, :destination_country, :offers]
  )

  def to_s
    symbol.to_s
  end

  private

  def self.with_order_expiration
    [ocean, air, lcl, ltl, land]
  end
end
```

```ruby
class Service < ActiveRecord::Base
  enumeration :transport_type, class_name: ServiceType,
                                foreign_key: :transport_type
end
```

```ruby
li
  = active_link_to polymorphic_path([controller_context.path, "#{service.transport_type}_orders"]), active: /^\/#{Regexp.quote(controller_context.path)}\/#{Regexp.quote(service.transport_type.to_s)}*/ do
  = service.name
  - if controller_context.admin?
  span.pull-right
  = pending_review_count(service.transport_type.order)
```

```ruby
module AdminHelper
  def pending_review_count(clazz)
    pending_rewiews_count_for_clazz = clazz.pending_review.count
    pending_rewiews_count_for_clazz.zero? ? nil : pending_rewiews_count_for_clazz
  end
end
```


```ruby
class Ocean::Order < ActiveRecord::Base
  enum status: [:pending_review, :active, :booked, :in_progress, :delivered, :canceled].index_with(&:to_s)
end
```


## Analysis

Polymorphism taken to an extreme - using enums to select which
ActiveRecord model class to query.

| Pros | Cons |
|---------|---------|
| Extreme DRY - everything defined in one place | impossible to understand without deep context |
| Polymorphism without explicit conditionals in helpers | breaks standard Rails conventions |
| "Clever" solution that reduces code duplication | makes refactoring dangerous |
| | violates all SOLID principles |
| | creates hidden coupling between enum and ALL model classes |
| | Can't grep for Ocean::Order usage |
| | Can't use IDE "find usages" reliably |
| | Autoloading nightmares in development vs production |
| | Impossible to test in isolation |
| | Helper depends on enum which depends on models |
| | Each model class must have identical interface (pending_review scope) |
| | Breaking changes in one model affect all services |
| | New developers will be completely lost |
| | Mixes Rails enum with enumerations gem (confusing!) |
| | Law of Demeter violated massively |
| | Makes codebase appear "too clever" and unmaintainable |

## Rails Enum Alternative


### Option 1: Single Table Inheritance (Best if models share behavior)
```ruby
class Order < ApplicationRecord
  self.abstract_class = false # Base class
  enum status: [:pending_review, :active, :booked, :in_progress, :delivered, :canceled].index_with(&:to_s)

  scope :pending_review, -> { where(status: :pending_review) }

  # Shared behavior for all order types
  def calculate_total
    # ...
  end
end
```
Each specific order inherits from Order
```ruby
class Ocean::Order < Order
  # Ocean-specific logic
end
class Lcl::Order < Order
  # LCL-specific logic
end
class Air::Order < Order
  # Air-specific logic
end
class Service < ApplicationRecord
  enum transport_type: [:ocean, :lcl, :air, :land, :ltl, :warehouse, :customs_clearance].index_with(&:to_s)
  has_many :orders
end
module AdminHelper
  def pending_review_count(service)
  count = service.orders.pending_review.count
  count.zero? ? nil : count
  end
end
```
Usage in view:
pending_review_count(service)

Benefits:
- Standard Rails pattern (STI)
- Polymorphic through database (type column)
- Easy to query across all order types
- No enum trickery needed


### Option 2: Registry Pattern (Best if models are truly different)
```ruby
module OrderRegistry
  ORDERS = {
    ocean: Ocean::Order,
    lcl: Lcl::Order,
    air: Air::Order,
    land: Land::Order,
    ltl: Ltl::Order,
    warehouse: Warehouse::Order,
    customs_clearance: CustomsClearance::Order
  }.freeze

  def self.class_for(service_type)
    ORDERS.fetch(service_type.to_sym)
  end

  def self.pending_review_for(service_type)
    class_for(service_type).pending_review
  end

  def self.pending_review_count_for(service_type)
    count = pending_review_for(service_type).count
    count.zero? ? nil : count
  end
end

class Service < ApplicationRecord
  enum transport_type: [:ocean, :lcl, :air, :land, :ltl, :warehouse, :customs_clearance].index_with(&:to_s)

  def order_class
    OrderRegistry.class_for(transport_type)
  end

  def pending_review_count
    OrderRegistry.pending_review_count_for(transport_type)
  end
end

module AdminHelper
  def pending_review_count(service)
    service.pending_review_count
  end
end
```
Usage in view:
pending_review_count(service)
OrderRegistry.class_for(:ocean) # => Ocean::Order

Benefits:
- Explicit mapping in one place (OrderRegistry)
- Easy to find all model class usages
- No enum trickery
- Testable in isolation
- Clear dependencies


### Option 3: Polymorphic Association (Best for true polymorphism)
```ruby
class Service < ApplicationRecord
  enum transport_type: [:ocean, :lcl, :air, :land, :ltl, :warehouse, :customs_clearance].index_with(&:to_s)
  has_many :orders, as: :orderable
end
```
Each order type has inverse polymorphic association
```ruby
class Ocean::Order < ApplicationRecord
  belongs_to :orderable, polymorphic: true
  scope :pending_review, -> { where(status: :pending_review) }
end

class Service < ApplicationRecord
  def pending_review_count
    count = orders.where(status: :pending_review).count
    count.zero? ? nil : count
  end
end

module AdminHelper
  def pending_review_count(service)
    service.pending_review_count
  end
end
```
Benefits:
- Standard Rails pattern (polymorphic associations)
- Database-backed relationships
- Easy to query
- No class reference trickery


### Option 4: Service Object per Transport Type (Most explicit)
```ruby
class OrderQueryService
  def initialize(service)
    @service = service
  end

  def pending_review_count
    count = order_class.pending_review.count
    count.zero? ? nil : count
  end

  def pending_review_orders
    order_class.pending_review
  end

  private

  def order_class
    case @service.transport_type
    when 'ocean' then Ocean::Order
    when 'lcl' then Lcl::Order
    when 'air' then Air::Order
    when 'land' then Land::Order
    when 'ltl' then Ltl::Order
    when 'warehouse' then Warehouse::Order
    when 'customs_clearance' then CustomsClearance::Order
    else raise ArgumentError, "Unknown transport type: #{@service.transport_type}"
    end
  end
end
class Service < ApplicationRecord
  enum transport_type: [:ocean, :lcl, :air, :land, :ltl, :warehouse, :customs_clearance].index_with(&:to_s)

  def order_query
    @order_query ||= OrderQueryService.new(self)
  end
end

module AdminHelper
  def pending_review_count(service)
    service.order_query.pending_review_count
  end
end
```
Benefits:
- Most explicit - case statement shows all mappings
- Easy to understand
- Easy to test
- Centralized in service object


### Option 5: Delegate to namespaced services
```ruby
module TransportTypes
  class Ocean
    ORDER_CLASS = Ocean::Order
    OFFER_CLASS = Ocean::Offer
    INCLUDES = [:from_location, :to_location, :offers].freeze

    def self.pending_review_count
      count = ORDER_CLASS.pending_review.count
      count.zero? ? nil : count
    end
  end

  class Air
    ORDER_CLASS = Air::Order
    OFFER_CLASS = Air::Offer
    INCLUDES = [:from_location, :to_location, :offers].freeze

    def self.pending_review_count
      count = ORDER_CLASS.pending_review.count
      count.zero? ? nil : count
    end
  end

  # ... other transport types

  REGISTRY = {
    ocean: Ocean,
    air: Air,
    # ...
  }.freeze

  def self.for(transport_type)
    REGISTRY.fetch(transport_type.to_sym)
  end
end

class Service < ApplicationRecord
  enum transport_type: [:ocean, :air, :lcl, :land, :ltl, :warehouse, :customs_clearance].index_with(&:to_s)

  def transport_service
    TransportTypes.for(transport_type)
  end
end

module AdminHelper
  def pending_review_count(service)
    service.transport_service.pending_review_count
  end
end
```

## Verdict

This pattern represents one of the more problematic uses of enumerations. Storing model class references in enums creates significant architectural issues that compound over time.

### Why this approach is problematic:

1. **Comprehensibility Impact:**
 When a developer encounters `service.transport_type.order.pending_review.count`, the `.order` appears to be a method but is actually an attribute returning a class constant. This implicit behavior increases cognitive load and makes the codebase harder to reason about, especially for new team members or when revisiting code after time has passed.

2. **Discoverability Challenges:**
 Finding where `Ocean::Order` is referenced becomes difficult since it's stored as a value rather than explicitly referenced in code. Standard grep searches and IDE "find usages" tools won't locate these references, making impact analysis during refactoring significantly more challenging.

3. **Autoloading Complications:**
 Rails uses different class loading strategies in development (lazy-loading) versus production (eager-loading). Storing class references as enum attributes can lead to "uninitialized constant" errors that only manifest in production environments, creating debugging scenarios that are difficult to reproduce locally.

4. **Refactoring Brittleness:**
 Renaming classes like `Ocean::Order` to `Shipping::OceanOrder` becomes error-prone because the enum attribute references won't appear in automated refactoring tools. This hidden coupling means changes that should be safe can silently break the application.

5. **Testing Complexity:**
 Testing the helper requires loading the ServiceType enum, which in turn requires loading Ocean::Order, Lcl::Order, Air::Order, and all other order classes. This tight coupling makes it difficult to write focused unit tests and increases test setup complexity.

6. **Increased Coupling:**
 The enum becomes coupled to 7+ different model classes simultaneously. Any change to a model's interface (like renaming or removing the pending_review scope) can break the enum. This coupling also assumes all models maintain the same interface, creating implicit contracts that are easy to violate accidentally.

7. **Convention Violation:**
 Rails provides established patterns for polymorphism: Single Table Inheritance (STI), polymorphic associations, and explicit service objects. Storing class references in enums bypasses these well-understood patterns, making the codebase less predictable for developers familiar with Rails conventions.

8. **Pattern Mixing:**
 The code simultaneously uses Rails enum (in Ocean::Order for status) and the enumerations gem (in ServiceType), creating confusion about when and why each pattern is chosen. This inconsistency suggests the architecture evolved without clear guidelines.

9. **Law of Demeter Violation:**
 The chain `service.transport_type.order.pending_review.count` requires the helper to understand the internal structure of multiple objects. Each additional method call increases the fragility of the code and the number of reasons it might need to change.

The correct approach depends on your needs:

- If order types share behavior: Use STI (Option 1)
- If order types are different: Use Registry Pattern (Option 2)
- If true polymorphism needed: Use Polymorphic Associations (Option 3)
- If you want maximum clarity: Use Service Object (Option 4)
- If you want namespaced services: Use Option 5

My recommendation: Option 2 (Registry Pattern) for this case.

Why? Because the order types seem genuinely different (Ocean::Order,
Air::Order, etc. probably have different attributes and behavior), but
you need a way to query them based on transport type.

### Then in helper:

def pending_review_count(service)
 OrderRegistry.pending_review_count_for(service.transport_type)
end
