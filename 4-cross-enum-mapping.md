# PATTERN 4: Cross-Enum Mapping (Status Transformation)

## Original Example


```ruby
class OrderStatus < BaseEnumeration
  values all_orders: { name: 'All' },
         created: { name: 'Created' },
         authorized: { name: 'Authorized' },
         queued: { name: 'Queued' },
         approved: { name: 'Approved' },
         cmt_created: { name: 'Cmt created' }

  def status_for_device
    case symbol
    when :created, :cmt_created
      KioskStatus.opened # => another enumeration
    when :queued, :queued_merchandise, :charged
      KioskStatus.queued
    when :approved, :producing, :verified_at_lab, :authorized
      KioskStatus.ordered
    when :declined
      KioskStatus.declined
    when :ready_for_collection
      KioskStatus.ready_for_collection
    when :collected
      KioskStatus.collected
    when :canceled, :reordered
      KioskStatus.canceled
    when :refunded, :partly_refunded
      KioskStatus.refunded
    end
  end
end
```

```ruby
render_api json: {
  response: {
    order_status_id: @order.order_status.status_for_device.id,
  }
}
```

```ruby
- if order.order_status.status_for_device != KioskStatus.opened
  .row
    .col-xs-12.order-detail-row
      %span.order-detail-box
        = order.created_at.strftime("%d %b")
```

```ruby
module Api
  module V1
    module Serializers
      class Order < Api::V1::Serializers::Serializer
        def order_status_id
          object.order_status.status_for_device.id
        end
      end
    end
  end
end
```

## Analysis


### What This Pattern Does
Maps OrderStatus enumeration values to KioskStatus enumeration values.
This is essentially a translator between two different domain representations
of order state - one for internal use, one for kiosk devices.

| Pros | Cons |
|---------|---------|
| Encapsulates the mapping logic in one place | 🚨 major CODE SMELL: This is a mapper/translator, NOT enum behavior |
| Polymorphic transformation based on status | Creates tight coupling between two enumerations |
| Clear transformation from one domain to another (internal → device) | Violates Single Responsibility Principle |
| | Hard to test in isolation |
| | OrderStatus shouldn't "know about" KioskStatus |
| | Difficult to maintain when either enum changes |
| | Makes dependencies unclear and circular |
| | Can't use OrderStatus without KioskStatus loaded |

## Rails Enum Alternative


### Option 1: Dedicated Mapper Class (Best - Clean and testable)
```ruby
class OrderToKioskStatusMapper
  # Clear, explicit mapping in one place
  MAPPING = {
    'created' => 'opened',
    'cmt_created' => 'opened',
    'queued' => 'queued',
    'queued_merchandise' => 'queued',
    'charged' => 'queued',
    'approved' => 'ordered',
    'producing' => 'ordered',
    'verified_at_lab' => 'ordered',
    'authorized' => 'ordered',
    'declined' => 'declined',
    'ready_for_collection' => 'ready_for_collection',
    'collected' => 'collected',
    'canceled' => 'canceled',
    'reordered' => 'canceled',
    'refunded' => 'refunded',
    'partly_refunded' => 'refunded'
  }.freeze

  def self.call(order_status)
    MAPPING.fetch(order_status) do
      raise ArgumentError, "Unknown order status: #{order_status}"
    end
  end

  # Alternative: return the ID directly if needed
  def self.kiosk_status_id(order_status)
    kiosk_status = call(order_status)
    Order.kiosk_statuses[kiosk_status]
  end
end

class Order < ApplicationRecord
  enum order_status: [:created, :cmt_created, :authorized, :queued, :approved].index_with(&:to_s)
  # ... all statuses

  enum kiosk_status: [:opened, :queued, :ordered, :declined, :ready_for_collection, :collected, :canceled, :refunded].index_with(&:to_s)

  def status_for_device
    OrderToKioskStatusMapper.call(order_status)
  end
end
```

**In controller:**

```ruby
render_api json: {
  response: {
    order_status_id: @order.status_for_device
  }
}
```

**In view:**

```ruby
- if @order.status_for_device != 'opened'
  .row
  .col-xs-12.order-detail-row
```

**In serializer:**

```ruby
def order_status_id
  OrderToKioskStatusMapper.call(object.order_status)
end
```
### Option 2: Service Object (Good for complex transformation logic)
```ruby
class DeviceStatusTransformer
  def initialize(order)
    @order = order
  end

  def kiosk_status
    case @order.order_status
    when 'created', 'cmt_created'
      'opened'
    when 'queued', 'queued_merchandise', 'charged'
      'queued'
    when 'approved', 'producing', 'verified_at_lab', 'authorized'
      'ordered'
    when 'declined'
      'declined'
    when 'ready_for_collection'
      'ready_for_collection'
    when 'collected'
      'collected'
    when 'canceled', 'reordered'
      'canceled'
    when 'refunded', 'partly_refunded'
      'refunded'
    else
      'opened' # default fallback
    end
  end

  def kiosk_status_id
    Order.kiosk_statuses[kiosk_status]
  end
end

class Order < ApplicationRecord
  def device_status
    @device_status ||= DeviceStatusTransformer.new(self)
  end
end

# usage
# @order.device_status.kiosk_status
# @order.device_status.kiosk_status_id
```


### Option 3: Concern for Reusability (If multiple models need this)
```ruby
module DeviceStatusMapping
  extend ActiveSupport::Concern

  included do
    def status_for_device
      self.class.device_status_mapper.call(order_status)
    end
  end

  class_methods do
    def device_status_mapper
      OrderToKioskStatusMapper
    end
  end
end

class Order < ApplicationRecord
  include DeviceStatusMapping
  enum order_status: [:created, :authorized, :queued, :approved].index_with(&:to_s)
end
```


## Verdict


This pattern misuses the enumerations gem by placing mapping logic where it doesn't belong. Dedicated mapper classes provide a cleaner solution.

Why this is a problem:

1. COUPLING: OrderStatus shouldn't know about KioskStatus. These are
 separate domains that happen to need translation.

2. TESTING: To test OrderStatus, you now need KioskStatus loaded.
 To test the mapping, you need to instantiate enumeration objects.

3. DISCOVERABILITY: Where does status transformation happen? In OrderStatus?
 In a service? How do I find all places that use this mapping?

4. MAINTENANCE: If KioskStatus changes, you have to modify OrderStatus.
 If you add a new OrderStatus, you have to remember to add mapping.

5. REUSABILITY: What if a different device needs a different mapping?
 Do you add another method to OrderStatus? It becomes a dumping ground.

The Mapper Pattern solves all of these:
- Clear separation of concerns
- Easy to test independently
- No coupling between enums
- Easy to find (named *Mapper or *Transformer)
- Easy to extend (create new mappers for new devices)
- Follows Single Responsibility Principle
- Works with any enum approach (Rails enum or enumerations gem)

### Recommendation:
always use Option 1 (Mapper class) for cross-enum transformations.
This pattern is so common in Rails that many teams have a mappers/
directory just for these classes.
