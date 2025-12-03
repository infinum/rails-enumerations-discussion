# PATTERN 8: Type-Specific Strong Parameters

## Original Example


```ruby
class TherapyScheduleType < Enumerations::Base
  value(
    :by_hour,
    form: Therapy::ByHourForm,
    intake_record_generator: IntakeRecordGenerator::ByHour,
    ends_at_calculator: EndsAtCalculator::Hourly,
    specific_params: Therapy::Params::BY_HOUR_SPECIFIC
  )

  value(
    :every_day,
    form: Therapy::EveryDayForm,
    intake_record_generator: IntakeRecordGenerator::EveryDay,
    ends_at_calculator: EndsAtCalculator::EveryDay,
    specific_params: Therapy::Params::EVERY_DAY_SPECIFIC
  )

  value(
    :every_nth_day,
    form: Therapy::EveryNthDayForm,
    intake_record_generator: IntakeRecordGenerator::EveryNthDay,
    ends_at_calculator: EndsAtCalculator::EveryNthDay,
    specific_params: Therapy::Params::EVERY_NTH_DAY_SPECIFIC
  )

  value(
    :day_of_week,
    form: Therapy::DayOfWeekForm,
    intake_record_generator: IntakeRecordGenerator::DayOfWeek,
    ends_at_calculator: EndsAtCalculator::DayOfWeek,
    specific_params: Therapy::Params::DAY_OF_WEEK_SPECIFIC
  )

  value(
    :with_pause,
    form: Therapy::WithPauseForm,
    intake_record_generator: IntakeRecordGenerator::WithPause,
    ends_at_calculator: EndsAtCalculator::Null,
    specific_params: Therapy::Params::WITH_PAUSE_SPECIFIC
  )

  def create_params
    Therapy::Params::GENERIC_CREATE + specific_params
  end

  def update_params
    Therapy::Params::GENERIC_UPDATE + specific_params
  end

  def assign_and_update_params
    update_params.push(:profile_id)
  end

  def barcode_create_params
    create_params + Therapy::Params::BARCODE_SCAN_CREATE
  end
end
```

```ruby
class Therapy
  class CreateService
    attr_reader :params

    def initialize(current_profile, params)
      @params = params.merge(profile_id: current_profile.id)
    end

    def call
      therapy = therapy_form_class.create(permitted_params)

      therapy
    end

    private

    def permitted_params
      params.permit(schedule_type_enum.create_params)
    end

    def therapy_form_class
      schedule_type_enum.form
    end

    def schedule_type_enum
      TherapyScheduleType.find(schedule_type) || raise(UnknownScheduleType)
    end

    def schedule_type
      params[:schedule_type]
    end
  end
end
```

## Analysis


### What This Pattern Does
Stores type-specific strong parameter lists as enum attributes. Each therapy
schedule type has different required parameters (hourly needs `hour_interval`,
daily needs `time_of_day`, etc.). The enum defines which params are valid
for each type, and this is used in service objects for params.permit().

| Pros | Cons |
|---------|---------|
| DRY - parameter definitions co-located with the type | Mixes controller concerns (strong parameters) with domain logic (enum) |
| Type-specific param validation centralized | Strong parameters logic should live in controllers or form objects |
| Easy to see what params each type requires | Not easily testable in isolation |
| Single source of truth for allowed parameters | Hard to override or extend for specific contexts |
| | Makes enum responsible for HTTP/API concerns |
| | Violates Single Responsibility Principle |
| | Can't reuse enum in non-Rails contexts (e.g., background jobs) |

## Rails Enum Alternative


### Option 1: Dedicated Params module (Most Rails-idiomatic)
```ruby
module TherapyParams
  GENERIC_CREATE = [:name, :profile_id, :starts_at, :medication_id].freeze
  GENERIC_UPDATE = [:name, :starts_at, :notes].freeze

  BY_HOUR_SPECIFIC = [:hour_interval, :doses_per_day].freeze
  EVERY_DAY_SPECIFIC = [:time_of_day, :duration_days].freeze
  EVERY_NTH_DAY_SPECIFIC = [:day_interval, :time_of_day].freeze
  DAY_OF_WEEK_SPECIFIC = [:days_of_week, :time_of_day].freeze
  WITH_PAUSE_SPECIFIC = [:intake_days, :pause_days, :time_of_day].freeze

  PARAMS_MAP = {
    'by_hour' => BY_HOUR_SPECIFIC,
    'every_day' => EVERY_DAY_SPECIFIC,
    'every_nth_day' => EVERY_NTH_DAY_SPECIFIC,
    'day_of_week' => DAY_OF_WEEK_SPECIFIC,
    'with_pause' => WITH_PAUSE_SPECIFIC
  }.freeze

  def self.create_params_for(schedule_type)
    GENERIC_CREATE + PARAMS_MAP.fetch(schedule_type)
  end

  def self.update_params_for(schedule_type)
    GENERIC_UPDATE + PARAMS_MAP.fetch(schedule_type)
  end

  def self.barcode_create_params_for(schedule_type)
    create_params_for(schedule_type) + [:barcode, :scanned_at]
  end
end

class Therapy::CreateService
  def initialize(current_profile, params)
    @current_profile = current_profile
    @params = params
  end

  def call
    Therapy.create(permitted_params.merge(profile_id: @current_profile.id))
  end

  private

  def permitted_params
    @params.permit(TherapyParams.create_params_for(schedule_type))
  end

  def schedule_type
    @params[:schedule_type]
  end
end
```


### Option 2: Form objects (Best for complex validation)
```ruby
class Therapy::BaseForm
  include ActiveModel::Model

  attr_accessor :name, :profile_id, :starts_at, :medication_id
  validates :name, :profile_id, :starts_at, presence: true

  def self.permitted_params
    raise NotImplementedError
  end
end

class Therapy::ByHourForm < Therapy::BaseForm
  attr_accessor :hour_interval, :doses_per_day

  validates :hour_interval, :doses_per_day, presence: true
  validates :hour_interval, numericality: { greater_than: 0 }

  def self.permitted_params
    [:name, :profile_id, :starts_at, :medication_id, :hour_interval, :doses_per_day]
  end
end

class Therapy::EveryDayForm < Therapy::BaseForm
  attr_accessor :time_of_day, :duration_days

  validates :time_of_day, :duration_days, presence: true

  def self.permitted_params
    [:name, :profile_id, :starts_at, :medication_id, :time_of_day, :duration_days]
  end
end
```
Form factory
```ruby
class TherapyFormFactory
  FORMS = {
    'by_hour' => Therapy::ByHourForm,
    'every_day' => Therapy::EveryDayForm,
    'every_nth_day' => Therapy::EveryNthDayForm,
    'day_of_week' => Therapy::DayOfWeekForm,
    'with_pause' => Therapy::WithPauseForm
  }.freeze

  def self.form_for(schedule_type)
    FORMS.fetch(schedule_type)
  end

  def self.permitted_params_for(schedule_type)
    form_for(schedule_type).permitted_params
  end
end

class Therapy::CreateService
  def call
    form = form_class.new(permitted_params)
    if form.valid?
      Therapy.create(form.attributes)
    else
      # handle errors
    end
  end

  private

  def form_class
    TherapyFormFactory.form_for(schedule_type)
  end

  def permitted_params
    @params.permit(form_class.permitted_params)
  end
end
```

## Verdict


Dedicated params module or form objects are MORE Rails-idiomatic.

Strong parameters are a controller/API concern, not a domain logic concern.
Mixing them into enums violates the separation between HTTP layer and
business logic layer.

### Problems with storing params in enums:

1. COUPLING: Enum is now coupled to Rails' strong parameters API
2. CONTEXT: Can't reuse enum in non-HTTP contexts (background jobs, console)
3. TESTING: Must test enum to test parameter logic
4. DISCOVERABILITY: Developers expect params logic in controllers/forms
5. VIOLATION: Violates Single Responsibility Principle

### Benefits of dedicated params module/forms:
- Clear separation between domain logic and HTTP concerns
- Easy to find (developers know to look in params module or forms)
- Easy to test independently
- Follows Rails conventions (params logic near controllers)
- Form objects provide validation AND param filtering

### Recommendation by complexity:

- Simple cases: Use Option 1 (Params module) - cleanest and most standard
- Complex validation: Use Option 2 (Form objects) - best for complex rules

For this specific case, I recommend Option 2 (Form Objects):

Benefits:
- Encapsulates both params AND validation per type
- Each form is self-contained and testable
- Clear what params each schedule type accepts
- Easy to extend with custom validations
- Follows "Form Object" pattern
