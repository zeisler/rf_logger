RF Logger provides a consistent API to log events of different severity levels.  Severities include:
```
    :debug # dev-only, for exploring issues
    :info  # end users, to audit process
    :warn  # weird thing happened, but isn't really an issue
    :error # someone fix the code
    :fatal # system-wide errors
```
The API provides helper methods for each severity (i.e. `Logger.debug`).  In addition to logging severity levels, the following fields are optional:
- actor (The user or caller of the event that you are logging)
- action (The event you are logging)
- target_type (We use this in cases where we want to tie a log record to a particular entity or object in the database, for example User)
- target_id (This is used in conjunction with target_type, as the unique identifier for the relevant entity)
- metadata (Additional information such as error details and backtraces or other messages, stored as a hash)

This gem includes RfLogger::SimpleLogger, RfLogger::SequelLooger and RfLogger::RailsLogger loggers as options that adhere to this API.  The fields above should be passed into helper methods as a hash.

##Integration##
Integrating RfLogger into your project requires the following steps:
* Include the rf_logger code in your project
* Create a migration 
* Add a model

#### Including rf_logger ####
Currently, RfLogger is included from the Github repo. Place the following in your Gemfile:

```gem 'rf_logger', :github => 'renewablefunding/rf_logger'```

Also make sure you include rf_logger and the logger you're going to be using:

```
require "rf_logger"
require "rf_logger/rails_logger"    # If using RailsLogger
# require "rf_logger/sequel_logger" # If using SequelLogger
```

#### Migration ####
Assuming your logger will persist to a database, you'll need to create a table. While the api should make it pretty easy to determine which fields you'll need, here are the guts of what you'd need for both the SequelLogger and RailsLogger:
######Sequel######
```
create_table :logs do
    primary_key :id
    column :actor, :text, :null => false
    column :action, :text, :null => false
    column :target_type, :text
    column :target_id, :text
    column :metadata, :text
    column :created_at, 'timestamp with time zone', :null => false
    Integer :level, :null => false, :default => 0
end
```

######Rails######
```
create_table :logs do |table|
    table.integer :level, null: false, default: 0
    table.string :actor
    table.string :action
    table.string :target_type
    table.string :target_id
    table.string :metadata
    table.timestamps null: false
end
```

#### Model ####
Again, assuming you'll be using the SequelLogger or RailsLogger (or some other logger that persists to a datasource), you'll want to create a Model that wraps your logger. This is as simple as creating a class that inherits from your logger (though you can make it more complex as your project needs dictate):

######Sequel######
```
class Log < RfLogger::SequelLogger
end
```

######Rails######
```
class Log < RfLogger::RailsLogger
end
```


##Configuration##
Configuration mostly sets up additional notifications beyond the actual logging.

```ruby
RfLogger.configure do |c|
  # environment attempts to determine this for Rails, Sinatra, Padrino, and Rory, but is otherwise
  c.environment = 'production'  required
  # fields from above can be interpolated into the notification subject
  c.notification_subject = "Oh no!  An error of {{level}} severity just occurred."
  c.set_notifier_list do |n|
    c.add_notifier Notification::DefinedElsewhere, :levels => [:error], :except => ['test', 'development']
    c.add_notifier Notification::OhNo, :levels => [:fatal, :error], :only => ['production']
    c.add_notifer Notifcation:VeryVerbose
  end
end
```

As you see above, you can specify different notifications for different levels or environments when you log an event.

##Notification##
While you have to implement notifiers yourself, the API is fairly simple.  The class must respond to .send_notification.  The argument passed in is an object that includes a #subject (which can be defined in the configuration (see above), and #details, which is the metadata in YAML format.  Future versions of this may allow for other transformations of the log data.

Example:
```ruby
require 'tinder'
module Notification
  class Campfire
    class << self
      def send_error_notification log_info
        message = log_info.subject + "\n" + log_info.details
        post_message message
      end

      def post_message message
        campfire = Tinder::Campfire.new(config['user'], :token => config['api_token'])
        room = campfire.find_room_by_id(config['room_id'])
        room.paste message
      end

      def config
        @config ||= YAML.load_file(File.join(Rails.root, 'config/campfire.yml'))
      end
    end
  end
end
```


