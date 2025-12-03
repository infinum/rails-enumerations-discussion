# PATTERN 5: Rich Metadata per Enum Value

## Original Example


```ruby
class Platform < Enumerations::Base
  value :android,
        name: 'Android',
        installable?: true,
        form: 'releases/forms/android',
        extensions: ['.apk', '.aab'],
        icon: 'android',
        stash_platform_id: 1,
        icon_background: '#9BC472'

  value :ios,
        name: 'iOS',
        installable?: true,
        form: 'releases/forms/ios',
        extensions: ['.ipa'],
        icon: 'apple',
        stash_platform_id: 2,
        icon_background: '#000000'

  def valid_extension?(file)
    extensions.include? File.extname(file)
  end

# Usage:
# release.platform.valid_extension?(file_path)
end
```

**in controller or service object:**

```ruby
def assign_info
  @release.uploaded_by = current_user.id
  return true unless @release.installable?
  extract_release_info = ExtractReleaseInfo.new(@release)
  if extract_release_info.valid?
  @release.name = extract_release_info.name
  @release.version_code = extract_release_info.version_code
  @release.extra_info = extract_release_info.extra_info
  return true if @release.valid?
  else
  @release = extract_release_info.release
  false
  end
end
```

```ruby
class ExtractReleaseInfo
  def initialize(release)
    @release = release
    @release_info = detect_release_info
  end

  def valid?
    return false unless extension_valid?
    true
  end

  private

  def extension_valid?
    return true if uploaded_file_matches_platform?

    if @release.app.nil?
      release.errors.add(:app, 'Please upload app file')
    else
      release.errors.add(
        :app,
        "Wrong release filename extension (isn't #{release.platform.extensions.join('/')} file)"
      )
    end
    false
  end

  def uploaded_file_matches_platform?
    @release.app &&
      @release.platform.valid_extension?(@release.app.original_filename)
  end
end
```
#### Usage in views #1
```ruby
= render @platform.form, f: f
```
#### Usage in views #2
```ruby
= link_to [project, platform: platform] do
  = icon platform.icon
```
#### Usage in views #3 (for css class)
```ruby
= icon release.platform.icon, class: 'release__platform-icon', style: "background-color: #{release.platform.icon_background};"
```

## Analysis


### What This Pattern Does
Stores multiple related attributes for each platform enum value in one place.
When you add a new platform (e.g., :web), you specify ALL its attributes
at once: name, whether it's installable, valid file extensions, form path,
icon, styling, etc.

| Pros | Cons |
|---------|---------|
| everything in one place - This provides great convenience | Not easily queryable (can't do DB queries on these attributes) |
| When adding a new platform, you see all required attributes | All attributes loaded into memory even if not needed |
| Impossible to forget to add an attribute (you see the pattern) | Mixing data definition with behavior (valid_extension? method) |
| Self-documenting - clear what metadata each platform needs | Can become a junk drawer for platform-related logic |
| Strongly typed attributes on enum objects | No compile-time checking that all values have same attributes |
| Can add instance methods that use these attributes | Requires understanding of enumerations gem patterns |
| Excellent discoverability - just look at the enum definition | |
| Centralized - one file to understand all platform differences | |

## Rails Enum Alternative


### Option 1: Configuration Hash with Value Object (Closest equivalent)
app/values/platform_attributes.rb
```ruby
class PlatformAttributes
  attr_reader :name, :extensions, :form, :icon, :stash_platform_id, :icon_background

  def initialize(**attributes)
    @name = attributes[:name]
    @installable = attributes[:installable]
    @extensions = attributes[:extensions]
    @form = attributes[:form]
    @icon = attributes[:icon]
    @stash_platform_id = attributes[:stash_platform_id]
    @icon_background = attributes[:icon_background]
  end

  def installable?
    @installable
  end

  def valid_extension?(file)
    extensions.include?(File.extname(file))
  end
end
```
app/models/concerns/platform_config.rb
```ruby
module PlatformConfig
  extend ActiveSupport::Concern

  # All platform data in one place - addresses the "everything in one place" concern
  PLATFORMS = {
    android: PlatformAttributes.new(
      name: 'Android',
      installable: true,
      form: 'releases/forms/android',
      extensions: ['.apk', '.aab'],
      icon: 'android',
      stash_platform_id: 1,
      icon_background: '#9BC472'
    ),
    ios: PlatformAttributes.new(
      name: 'iOS',
      installable: true,
      form: 'releases/forms/ios',
      extensions: ['.ipa'],
      icon: 'apple',
      stash_platform_id: 2,
      icon_background: '#000000'
    )
    # When adding new platform, all attributes are visible here!
    # Same benefits as enumerations gem
  }.freeze

  class_methods do
    def platform_config(platform_symbol)
      PLATFORMS.fetch(platform_symbol)
    end
  end

  def platform_attributes
    @platform_attributes ||= self.class.platform_config(platform.to_sym)
  end

  # Delegate all platform methods for clean API
  delegate :name, :installable?, :extensions, :form, :icon,
           :icon_background, :valid_extension?,
           to: :platform_attributes, prefix: :platform
end

class Release < ApplicationRecord
  include PlatformConfig
  enum platform: [:android, :ios].index_with(&:to_s)
end

# Usage (same as enumerations):
# @release.platform_installable?
# @release.platform_extensions
# @release.platform_valid_extension?(filename)
# @release.platform_form
# @release.platform_icon
# @release.platform_icon_background
```


### Option 2: YAML Configuration (Best for non-developer editable config)
config/platforms.yml
```yaml
android:
 name: Android
 installable: true
 form: releases/forms/android
 extensions:
   - .apk
   - .aab
 icon: android
 stash_platform_id: 1
 icon_background: '#9BC472'

ios:
 name: iOS
 installable: true
 form: releases/forms/ios
 extensions:
   - .ipa
 icon: apple
 stash_platform_id: 2
 icon_background: '#000000'
```

```ruby
class PlatformConfig
  CONFIG = YAML.load_file(Rails.root.join('config/platforms.yml')).freeze

  def self.for(platform)
    CONFIG.fetch(platform.to_s).with_indifferent_access
  end

  def self.installable?(platform)
    for(platform)[:installable]
  end

  def self.valid_extension?(platform, file)
    for(platform)[:extensions].include?(File.extname(file))
  end
end

class Release < ApplicationRecord
  enum platform: [:android, :ios].index_with(&:to_s)

  def platform_config
    @platform_config ||= PlatformConfig.for(platform)
  end

  def installable?
    platform_config[:installable]
  end

  def valid_extension?(file)
    PlatformConfig.valid_extension?(platform, file)
  end

  def platform_form
    platform_config[:form]
  end

  def platform_icon
    platform_config[:icon]
  end

  def platform_icon_background
    platform_config[:icon_background]
  end
end
```
@release.installable?
@release.valid_extension?(filename)
@release.platform_form
@release.platform_icon


### Option 3: Registry with Constants (Most explicit)
```ruby
module Platforms
  class Android
    NAME = 'Android'
    INSTALLABLE = true
    FORM = 'releases/forms/android'
    EXTENSIONS = ['.apk', '.aab'].freeze
    ICON = 'android'
    STASH_PLATFORM_ID = 1
    ICON_BACKGROUND = '#9BC472'

    def self.valid_extension?(file)
      EXTENSIONS.include?(File.extname(file))
    end
  end

  class IOS
    NAME = 'iOS'
    INSTALLABLE = true
    FORM = 'releases/forms/ios'
    EXTENSIONS = ['.ipa'].freeze
    ICON = 'apple'
    STASH_PLATFORM_ID = 2
    ICON_BACKGROUND = '#000000'

    def self.valid_extension?(file)
      EXTENSIONS.include?(File.extname(file))
    end
  end

  REGISTRY = {
    android: Android,
    ios: IOS
  }.freeze

  def self.for(platform_key)
    REGISTRY.fetch(platform_key.to_sym)
  end
end

class Release < ApplicationRecord
  enum platform: [:android, :ios].index_with(&:to_s)

  def platform_class
    @platform_class ||= Platforms.for(platform)
  end

  def installable?
    platform_class::INSTALLABLE
  end

  def valid_extension?(file)
    platform_class.valid_extension?(file)
  end
end

# Usage:
# @release.installable?
# @release.valid_extension?(filename)
# Platforms::Android::ICON
```


## Verdict


The "everything in one place" benefit is really valuable. When you add
a new platform value, you immediately see all the attributes you need to
specify. You won't forget to add the icon or the extensions or the form path.

With Rails enum, this information gets scattered across:
- Constants for names
- Helper methods for icons
- Validator arrays for extensions
- Form logic for which partial to render
- CSS classes for styling

And when you add platform :web, how do you remember all these places?

The Rails alternatives CAN achieve the same "everything in one place"
benefit, as shown in Option 1 above. The key is using a configuration hash
or value object pattern.

My recommendation:

If you're considering adding the enumerations gem just for this pattern,
use Option 1 (Value Object with Config Hash) instead. You get 95% of
the benefits with 0% of the gem dependency.
