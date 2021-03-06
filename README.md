# solidus_tracking

[![CircleCI](https://circleci.com/gh/solidusio-contrib/solidus_tracking.svg?style=svg)](https://circleci.com/gh/solidusio-contrib/solidus_tracking)

This extension provides a data tracking platform (think [Segment](https://segment.com), but
on-premise) for your Solidus store.

It has out-of-the-box support for the most common eCommerce events, as well as for delivering
transactional emails through an external service.

| Service | Extension |
| ------- | --------- |
| [Klaviyo](https://klaviyo.com) | [solidus_klaviyo](https://github.com/solidusio-contrib/solidus_klaviyo) |

## Installation

Add solidus_tracking to your Gemfile:

```ruby
gem 'solidus_tracking'
```

Bundle your dependencies and run the installation generator:

```console
$ bundle
$ bundle exec rails g solidus_tracking:install
```

The generator will create an initializer at `config/initializers/solidus_tracking.rb` with the
default configuration. Take a look at the file and customize it to fit your environment.

## Usage

### Tracking events

The extension will track the following events:

- `Started Checkout`: when an order transitions from the `cart` state to `address`.
- `Placed Order`: when an order is finalized.
- `Ordered Product`: for each item in a finalized order.
- `Fulfilled Order`: when all of an order's shipments are shipped.
- `Cancelled Order`: when an order is cancelled.
- `Created Account`: when a user is created.
- `Reset Password`: when a user requests a password reset.

For the full payload of these events, look at the source code of the serializers and events.

#### Implementing custom events

If you have custom events you want to track with this gem, you can easily do so by creating a new
event class and implementing the required methods:

```ruby
module MyApp
  module Events
    class SubscribedToNewsletter < SolidusTracking::Event::Base
      self.payload_serializer = 'MyApp::Serializers::User'

      def name
        'SubscribedToNewsletter'
      end

      def email
        user.email
      end

      def customer_properties
        self.class.customer_properties_serializer.serialize(user)
      end

      def properties
        self.class.payload_serializer.serialize(user)
      end

      def time
        Time.zone.now
      end

      private

      def user
        payload.fetch(:user)
      end
    end 
  end 
end
```

As you can see, you will also have to create a serializer for your users:

```ruby
module MyApp
  module Serializers
    class UserSerializer < SolidusTracking::Serializer::Base
      def user
        object
      end

      def as_json(_options = {})
        {
          'FirstName' => user.first_name,
          'LastName' => user.last_name,
          # ...
        }
      end
    end
  end
end
```

Once you have created the event and serializer, the next step is to register your custom event when
initializing the extension:

```ruby
# config/initializers/solidus_tracking.rb
SolidusTracking.configure do |config|
  config.events['subscribed_to_newsletter'] = MyApp::Events::SubscribedToNewsletter
end
```

Your custom event is now properly configured! You can track it by calling `.track_later`:

```ruby
SolidusTracking.track_later('subscribed_to_newsletter', user: user)
```

*NOTE:* You can follow the same exact pattern to override the built-in events.

### Changing the serializers for built-in events

If you need to change the customer properties serializer or the payload serializer for one or more
events, there's no need to monkey-patch or override them. You can simply set 
`customer_properties_serializer` and/or `payload_serializer` in an initializer:

```ruby
# config/initializers/solidus_tracking.rb

SolidusTracking::Event::CancelledOrder.customer_properties_serializer = 'MyApp::Serializers::CustomerProperties'
SolidusTracking::Event::CancelledOrder.payload_serializer = 'MyApp::Serializers::Order'
```

If need to change the customer properties serializer for _all_ your events (which will usually be
the case), you can loop through the `.events` setting:

```ruby
# config/initializers/solidus_tracking.rb

SolidusTracking.configuration.events.each_value do |event_klass|
  event_klass.constantize.customer_properties_serializer = 'MyApp::Serializers::CustomerProperties'
end
```

Make sure to do this _after_ defining any custom events!

### Delivering emails through a tracker

If you plan to deliver your transactional emails through a tracker (e.g., Klaviyo), you may want to
disable the built-in emails that are delivered by Solidus and solidus_auth_devise.

In order to do that, you can set the `disable_builtin_emails` option in the extension's initializer:

```ruby
# config/initializers/solidus_tracking.rb
SolidusTracking.configure do |config|
  config.disable_builtin_emails = true
end
```

This will disable the following emails:

- Order confirmation
- Order cancellation
- Password reset
- Carton shipped

You'll have to re-implement the emails with a tracker.

### Disabling automatic event tracking

If you want to disable the out-of-the-box event tracking, you can do so on a per-event basis by
acting on the `automatic_events` configuration option:

```ruby
# config/initializers/solidus_tracking.rb
SolidusTracking.configure do |config|
  config.automatic_events.delete('placed_order')
end
```

You may also disable the built-in event tracking completely, if you only want to track events
manually:

```ruby
# config/initializers/solidus_tracking.rb
SolidusTracking.configure do |config|
  config.automatic_events.clear
end
```

### Test mode

You can enable test mode to mock all API calls instead of performing them:

```ruby
# config/initializers/solidus_tracking.rb
SolidusTracking.configure do |config|
  config.test_mode = true
end
```

This spares you the need to use VCR and similar.
 
When in test mode, you can also use our custom RSpec matchers to check if an event has been tracked:

```ruby
require 'solidus_tracking/testing_support/matchers'

RSpec.describe 'My tracking integration' do
  it 'tracks events' do
    SolidusTracking.track_now 'custom_event', foo: 'bar'

    expect(SolidusTracking).to have_tracked_event(CustomEvent)
      .with(foo: 'bar')
  end
end
```

## Development

### Testing the extension

First bundle your dependencies, then run `bin/rake`. `bin/rake` will default to building the dummy
app if it does not exist, then it will run specs. The dummy app can be regenerated by using
`bin/rake extension:test_app`.

```console
$ bundle
$ bin/rake
```

To run [Rubocop](https://github.com/bbatsov/rubocop) static code analysis run

```console
$ bundle exec rubocop
```

When testing your application's integration with this extension you may use its factories.
Simply add this require statement to your spec_helper:

```ruby
require 'solidus_tracking/factories'
```

### Running the sandbox

To run this extension in a sandboxed Solidus application, you can run `bin/sandbox`. The path for
the sandbox app is `./sandbox` and `bin/rails` will forward any Rails commands to
`sandbox/bin/rails`.

Here's an example:

```console
$ bin/rails server
=> Booting Puma
=> Rails 6.0.2.1 application starting in development
* Listening on tcp://127.0.0.1:3000
Use Ctrl-C to stop
```

### Releasing new versions

Your new extension version can be released using `gem-release` like this:

```console
$ bundle exec gem bump -v VERSION --tag --push --remote upstream && gem release
```

## License

Copyright (c) 2020 Nebulab Srls, released under the New BSD License.
