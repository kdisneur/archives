---
layout: post
title: Sharing settings between front & back end application
tags:
  - javascript
  - ruby
  - rails
  - gem
---
How many times do you need to share some configuration between your server side app and your client side app? Many times, for sure!

What's the possibilities?

## DRY as Do Repeat Yourself

The first solution seen in lot of applications: duplicate the Rails settings in the JavaScript.

###### Yaml

```yaml
# config/settings.yml
mailer:
  from: 'John Doe <noreply@company.com>'
maps:
  defaults:
    latitude: 42.00
    longitude: 31.00
    zoom: 4
```

###### Rails

```ruby
# app/models/settings
class Settings < Settingslogic
  source Rails.root.join('config', 'settings', '**', '*.yml')
end

# app/controllers/contact_controller.rb
class ContactController < ApplicationController
  def show
    @map_configuration = Settings.maps.defaults
  end
end
```

###### Javascript

```javascript
var Settings = {
  maps: {
    defaults: {
      latitude: 42.00,
      longitude: 31.00,
      zoom: 4
    }
  }
};

new google.maps.Map(document.getElementById('map'), {
  zoom:      Settings.maps.defaults.zoom,
  center:    new google.maps.LatLng(Settings.maps.defaults.latitude, Settings.maps.defaults.longitude),
  mapTypeId: google.maps.MapTypeId.ROADMAP
});
```

| Pros              | Cons                                                  |
|-------------------|-------------------------------------------------------|
| Easy to implement | Configuration duplicated between JavaScript and Rails |
|                   | Cannot be updated without rebooting the app           |

## Ajax loading

Another solution could be a controller that responds with a JSON containing all the shared settings.

###### Yaml

```yaml
# config/settings.yml
mailer:
  from: 'John Doe <noreply@company.com>'
maps:
  defaults:
    latitude: 42.00
    longitude: 31.00
    zoom: 4
```

###### Rails

```ruby
# config/initializers/settings_js.rb
SettingsJs.configuration do |config|
  config.klass   = Settings
  config.keys    = %w(maps)
end

# app/models/settings.rb
class Settings < Settingslogic
  source Rails.root.join('config', 'settings', '**', '*.yml')
end

# app/controllers/contact_controller.rb
class ContactController < ApplicationController
  def show
    @map_configuration = Settings.maps.defaults
  end
end

# app/controllers/configuration_controller.rb
class ConfigurationController < ApplicationController
  respond_to :json

  def show
    @settings = { maps: Settings.maps }
  end
end
```

###### Javascript

```javascript
jQuery.getJSON('/configuration', success: function(data, textStatus, jqXHR) {
  new google.maps.Map(document.getElementById('map'), {
    zoom:      data.maps.defaults.zoom,
    center:    new google.maps.LatLng(data.maps.defaults.latitude, data.maps.defaults.longitude),
    mapTypeId: google.maps.MapTypeId.ROADMAP
  });
});
```

| Pros                                                                                                       | Cons                                                                               |
|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| If the configuration comes from the database it could update JavaScript settings without rebooting the app | One additional HTTP request on every page                                          |
| Share the same settings between JavaScript and Rails                                                       | More shared settings are needed, more complex becomes your ConfigurationController |

## Settings-js gem

The last solution is to use the [settings-js](https://github.com/fanaticio/settings-js) gem that allows
to share the same configuration between Rails and JavaScript without Ajax request.

The gem can be bound with any backend but, for now, the only implemented backend is SettingsLogic.

###### Configuration

```yaml
# config/settings.yml
mailer:
  from: 'John Doe <noreply@company.com>'
maps:
  defaults:
    latitude: 42.00
    longitude: 31.00
    zoom: 4
```

###### Rails

```ruby
# config/initializers/settings_js.rb
SettingsJs.configuration do |config|
  config.klass   = Settings
  config.keys    = %w(maps)
end

# app/models/settings.rb
class Settings < Settingslogic
  source Rails.root.join('config', 'settings', '**', '*.yml')
end

# app/controllers/contact_controller.rb
class ContactController < ApplicationController
  def show
    @map_configuration = Settings.maps.defaults
  end
end
```

###### Javascript

```javascript
//= require settings-js/settings

new google.maps.Map(document.getElementById('map'), {
  zoom:      Settings.maps.defaults.zoom,
  center:    new google.maps.LatLng(Settings.maps.defaults.latitude, Settings.maps.defaults.longitude),
  mapTypeId: google.maps.MapTypeId.ROADMAP
});
```

| Pros                                                        | Cons                                        |
|-------------------------------------------------------------|---------------------------------------------|
| No additional HTTP requests                                 | Cannot be updated without rebooting the app |
| Share the same configuration between Rails and JavaScript   |                                             |
