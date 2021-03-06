# Simple Metrics Engine

This plugin provides event log based simple metrics.

## Installation

```
  script/plugin install git://github.com/ss/simple_metrics_engine.git
```

## Testing

```
  rake test 
  rake sme:coverage
  script/server
  point your browser to http://localhost:3000/coverage/
```

## Verification

```
  rake sme:generate_dummy_data
  script/server
  point your browser to http://localhost:3000/sme
```
  
## Usage

Configure the plugin.

in *config/initializers/simple_metrics_engine.rb*

```ruby
  Sme.configure do |config|
    # only allow admins to access metrics
    config.permission_check { redirect_to login_path and return false unless current_user.admin? }
   
    # (Optional) Metrics will be rolled up into 15-minute periods starting every hour on the hour.
    config.rollup_window = 15.minutes
  end

  # add user details to event hash
  def add_user_info(event_hash)
    event_hash[:user_id] ||= current_user.id
    event_hash
  end

  Sme.wrap!(method(:add_user_info))
```

Modify your User controller to log a visits and conversions.

in *app/controller/users_controller.rb*:
```ruby
  class UsersController < ApplicationController

    def index
      log_visitation
      ...
    end

    def create
      ...
      log_conversion if @user.valid?
    end

  private

    def log_visitation
      Sme.log('user|visit')
      Sme.log("user|visit|#{country}")
    end

    def log_conversion
      Sme.log('user|conversion', metrics_notes)
      Sme.log("user|conversion|#{country}")
    end

    def metrics_notes
      {
        :ab_test_button_text => ab_test_button_text,
        :foo => 'bar',
      }
    end

  end # class UsersController
```

Alternatively, you can use an after_create filter to your User model:

in *app/models/user.rb*:
```ruby
  class User < ActiveRecord::Base

    after_create :log_conversion

    def log_conversion_event
      Sme.log('user|conversion', :user_id => id)
    end

  end # class User
```

Create a cron job that runs the *sme:rollup_logs* rake task:

```cron
HOME=<rails root>
0/15 * * * * rake sme:rollup_logs >> $HOME/log/sme_rollup_logs.log 2>&1
```


