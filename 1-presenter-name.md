# PATTERN 1: Simple Name/Label Display

## Original Example


```ruby
# enumeration
class InstrumentCategory < BaseEnumerations
  value :bass, name: 'Bass'
  value :brass, name: 'Brass'
  value :drums, name: 'Drums'
  value :fx, name: 'FX'
  value :guitars, name: 'Guitars'
end

# model
class Instrument < ApplicationRecord
  enumeration :category, class_name: 'InstrumentCategory'
end

# view usage
td= instrument.category.name
```

## Analysis

| Pros | Cons |
|---------|---------|
|Clean, readable syntax in views | Overkill for simple name mapping|
|Name changes happen in one central location | Extra gem dependency for basic functionality |
| Self-documenting - easy to see all possible values | Creates additional enumeration objects in memory |
| | Developer needs to know gem-specific patterns |

## Rails Enum Alternative

### Option 1: Rails enum with I18n (Most Rails-idiomatic)

```ruby
class Instrument < ApplicationRecord
  enum category: [:bass, :brass, :drums, :fx, :guitars].index_with(&:to_s)

  def category_name
    I18n.t("instrument.categories.#{category}")
  end
end
```

```yaml
# config/locales/en.yml
en:
  instrument:
    categories:
      bass: "Bass"
      brass: "Brass"
      drums: "Drums"
      fx: "FX"
      guitars: "Guitars"
```


```haml
# view usage
td= instrument.category_name
```
### Option 2: Rails enum with hash (Simpler, no I18n needed)

```ruby
class Instrument < ApplicationRecord
  enum category: [:bass, :brass, :drums, :fx, :guitars].index_with(&:to_s)

  def category_name
    self.categories[category]&.camelize
  end
end
```

```haml
# view usage:
td= instrument.category_name
```

### Option 3: Decorator method (If presentation logic is complex)

```ruby
module InstrumentsDecorator
  CATEGORY_DISPLAY = {
    'bass' => 'Bass',
    'brass' => 'Brass',
    'drums' => 'Drums',
    'fx' => 'FX',
    'guitars' => 'Guitars'
  }.freeze

  def category_name
    CATEGORY_DISPLAY[category]
  end
end
```

```haml
# view usage:
td= instrument.category_name
```

## Verdict

**Rails enum with I18n or a constant hash is SUFFICIENT.**

### The enumerations gem adds unnecessary complexity for this simple use case. You're pulling in an entire gem dependency just to map strings to names.

### Recommendation:
- Use Option 1 (I18n) if you need internationalization
- Use Option 2 (constant hash) if English-only or simple names
- Use Option 3 (decorator) if you have complex presentation logic

### All three alternatives are:
- More performant (no object allocation per access)
- More standard (any Rails dev understands them immediately)
- Simpler to test and maintain
- No additional gem dependencies
