# PATTERN 3: Enum Class Methods for Subset Filtering

## Original Example


```ruby
class OrderStatus < BaseEnumeration
  values all_orders: { name: 'All' },
         created: { name: 'Created' },
         authorized: { name: 'Authorized' },
         queued: { name: 'Queued' },
         approved: { name: 'Approved' },
         cmt_created: { name: 'Cmt created' }

  def self.for_report
    (all - [created, cmt_created]).map(&:to_s)
  end
end
```

```ruby
class Order < ActiveRecord::Base
  scope :for_report, -> { where(order_status_id: OrderStatus.for_report) }
  scope :by_status, -> { where(order_status_id: OrderStatus.for_report).order(touched_at: :desc) }
end
```

```ruby
module Guests
  module GuestAccounts
    class SearchPresenter
      def orders
        @orders ||= if passengers.present?
                      Order.includes(:passenger, :order_items, :coupons)
                           .for_passengers(passengers)
                           .for_report
                    else
                      []
                    end
      end
    end
  end
end
```

## Analysis


### What This Pattern Does

| Pros | Cons |
|---------|---------|
| Centralizes business logic about status groupings | Mixes data definition with business logic |
| Easy to find and maintain - single source of truth | Can grow into a "God object" for enum-related logic |
| Self-documenting - method name describes the subset | Not immediately obvious these methods exist (discoverability) |
| Reduces duplication across models/controllers | Testing requires loading the entire enumeration class |
| Changes to "reportable statuses" happen in one place | |

## Rails Enum Alternative


### Option 1: Constants in model (Simplest and most discoverable)
```ruby
class Order < ApplicationRecord
  enum status: [:all_orders, :created, :authorized, :queued, :approved, :cmt_created].index_with(&:to_s)

  # All reportable statuses in one place
  REPORTABLE_STATUSES = %w[authorized queued approved].freeze
  scope :for_report, -> { where(status: REPORTABLE_STATUSES) }
  scope :by_status, -> { for_report.order(touched_at: :desc) }
end

# Usage:
# Order.for_report
# Order.by_status
```


### Option 2: Separate module for organization (Better for many groupings)
```ruby
module OrderStatus
  ALL_ORDERS = 'all_orders'
  CREATED = 'created'
  AUTHORIZED = 'authorized'
  QUEUED = 'queued'
  APPROVED = 'approved'
  CMT_CREATED = 'cmt_created'

  ALL = [ALL_ORDERS, CREATED, AUTHORIZED, QUEUED, APPROVED, CMT_CREATED].freeze
  REPORTABLE = [AUTHORIZED, QUEUED, APPROVED].freeze
  NON_REPORTABLE = [CREATED, CMT_CREATED].freeze
end

class Order < ApplicationRecord
  enum status: [
    OrderStatus::ALL_ORDERS,
    OrderStatus::CREATED,
    OrderStatus::AUTHORIZED,
    OrderStatus::QUEUED,
    OrderStatus::APPROVED,
    OrderStatus::CMT_CREATED
  ].index_with(&:to_s)

  scope :for_report, -> { where(status: OrderStatus::REPORTABLE) }
  scope :by_status, -> { for_report.order(touched_at: :desc) }
end

# Usage:
# Order.for_report
# OrderStatus::REPORTABLE
```

## Verdict


The enumerations gem doesn't add much value here. You're essentially creating
a method that returns a subset of IDs. This can be done equally well (or better)
with standard Rails patterns.

### Recommendation by complexity:
- 1-2 subsets: Use Option 1 (constants in model) - most straightforward
- 3-5 subsets: Use Option 2 (separate module) - better organization

### Benefits of Rails approach:
- Standard Rails pattern (any dev understands immediately)
- No additional dependencies
- Easy to see all subsets at a glance
