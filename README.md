![Circle CI](https://circleci.com/gh/klarna/phobos.svg?style=shield&circle-token=2289e0fe5bd934074597b32e7f8f0bc98ea0e3c7)]

# Phobos

Simplifying Kafka for ruby apps.

Phobos is a microframework and library for kafka based applications, it wraps common behaviors needed by consumers and producers in an easy an convenient API. It uses [ruby-kafka](https://github.com/zendesk/ruby-kafka) as it's kafka client and core component.

## Table of Contents

1. [Installation](#installation)
2. [Usage](#usage)
  1. [Standalone apps](#usage-standalone-apps)
  2. [Programmatically](#usage-programmatically)
  3. [Configuration file](#usage-configuration-file)
  4. [Consuming messages from kafka (handlers)](#usage-consuming-messages-from-kafka)
  4. [Producing messages to kafka](#usage-producing-messages-to-kafka)
  4. [Instrumentation](#usage-instrumentation)
3. [Development](#development)

## <a name="installation"></a> Installation

Add this line to your application's Gemfile:

```ruby
gem 'phobos'
```

And then execute:

```sh
$ bundle
```

Or install it yourself as:

```sh
$ gem install phobos
```

## <a name="usage"></a> Usage

Phobos can be used to power standalone ruby applications, it comes with a CLI to help loading your code and running it as a daemon or service. It can also be used as a library to bring kafka features to your projects, including Rails apps.

### <a name="usage-standalone-apps"></a> Standalone apps

Standalone apps have the benefits of individual deploys, simple and small code, and many others. If consuming from kafka is your version of microservices, Phobos can be of great help.

### Setup

To create this sort of apps you need two things:
  * A configuration file (more details in the [Configuration file](#usage-configuration-file) section)
  * A `phobos_boot.rb` (or the name of your choice) to properly load your code into Phobos executor

To help with the preparations Phobos ships with a CLI powering two commands: __init__ and __start__. To setup your project do the following:

```sh
# call this command inside your app folder
$ phobos init
    create  config/phobos.yml
    create  phobos_boot.rb
```

`phobos.yml` is an example of the configuration file and `phobos_boot.rb` is the place to load your code.

### Consumers (listeners and handlers)

In Phobos apps __listeners__ are configured against kafka, those are our consumers. A listener requires a __handler__ (a ruby class where you should process incoming messages), a __topic__, and a __group_id__. Consumer groups are used to coordinate the listeners across machines. We write the __handlers__ and Phobos makes sure to run them for us.

When using Phobos executor you don't care about how listeners are created, just provide the configuration under the `listeners` section in the configuration file and you are good to go. An example of a handler is:

```ruby
class MyHandler
  include Phobos::Handler

  def consume(payload, metadata)
    # payload  - This is the content of your kafka message, Phobos does not attempt to
    #            parse this content, it is delivered raw to you
    # metadata - A hash with useful information about this event, it contains: The event key,
    #            partition number, offset, retry_count, topic, group_id, and listener_id
  end
end
```

You can read more about the handlers in a [section bellow](#usage-consuming-messages-from-kafka).

Writing your handler is all you need to allow Phobos to work, it will take care of execution, retries and concurrency for you.

To start your application you can use the __start__ command, example:

```sh
$ phobos start
[2016-08-13T17:29:59:218+0200Z] INFO  -- Phobos : <Hash> {:message=>"Phobos configured", :env=>"development"}
______ _           _
| ___ \ |         | |
| |_/ / |__   ___ | |__   ___  ___
|  __/| '_ \ / _ \| '_ \ / _ \/ __|
| |   | | | | (_) | |_) | (_) \__ \
\_|   |_| |_|\___/|_.__/ \___/|___/

phobos_boot.rb - find this file at ~/Projects/example/phobos_boot.rb

[2016-08-13T17:29:59:272+0200Z] INFO  -- Phobos : <Hash> {:message=>"Listener started", :listener_id=>"6d5d2c", :group_id=>"test-1", :topic=>"test"}
```

By default, the __start__ command will look for the configuration file at `config/phobos.yml` and it will load the file `phobos_boot.rb` if it exists, in the example above I'm using all example files generated by the __init__ command. It is possible to change both files, use `-c` for the configuration file and `-b` for the boot file. For example:

```sh
$ phobos start -c /var/configs/my.yml -b /opt/apps/boot.rb
```

### <a name="usage-programmatically"></a> Programmatically

Besides the handler and the producer (look at their specific sections bellow), you can use `Listener` and `Executor`

But first, call the method `configure` with the path of your configuration file

```ruby
Phobos.configure('config/phobos.yml')
```

__Listener__ connects to kafka and acts as your consumer. To create a listener you need a handler class, a topic, and a group id.

```ruby
listener = Phobos::Listener.new(
  handler: Phobos::EchoHandler,
  group_id: 'group1',
  topic: 'test'
)

# start method blocks
Thread.new { listener.start }

listener.id # 6d5d2c (all listeners have an id)
listener.stop # stop doesn't block
```

This is all you need to consume from kafka with backoff retries.

__Executor__ is the supervisor of all listeners. It loads all listeners configured in `phobos.yml`. The executor keeps the listeners running and restart them when needed.

```ruby
executor = Phobos::Executor.new

# start doesn't block
executor.start

# stop will block until all listers are properly stopped
executor.stop
```

### <a name="usage-configuration-file"></a> Configuration file

The configuration file is organized in 6 sections. Take a look at the example file, [config/phobos.yml.example](https://github.com/klarna/phobos/blob/master/config/phobos.yml.example).

__logger__ configures the logger for all Phobos components, it automatically outputs to `STDOUT` and it saves the log in the configured file

__kafka__ provides configurations for all `Kafka::Client` created over the application. All options presented are from `ruby-kafka`

__producer__ provides configurations for all producers created over the application, the options are the same for normal and async producers. All options presented are from `ruby-kafka`

__consumer__ provides configurations for all consumer groups created over the application. All options presented are from `ruby-kafka`

__backoff__ Phobos provides automatic retries for your handlers, if an exception is raised the listener will retry following the backoff configured here

__listeners__ is the list of listeners configured, each listener represents a consumers group

### <a name="#usage-consuming-messages-from-kafka"></a> Consuming messages from kafka (handlers)

You can use Phobos executor or do it programmatically but handlers classes will always be used. To create a handler class just include the module `Phobos::Handler`. This module allows Phobos to manage the life cycle of your handler.

A handler must implement the method `#consume(payload, metadata)`.

A new instance of your handler will be created for every message, keep a constructor without arguments. If `consume` raises an exception, Phobos will retry the message indefinitely, applying the backoff configurations presented in the configuration file. Metadata will contain a key called `retry_count` with the current number of retries for this message. To skip a message just return from `consume`.

When the listener starts, the class method `start` will be called with the `kafla_client` used by the listener. Use this callback as a chance to setup necessary code for your handler. The class method `stop` will be called during listener shutdown.

```ruby
class MyHandler
  include Phobos::Handler

  def self.start(kafka_client)
  end

  def self.stop
  end

  def consume(payload, metadata)
  end
end
```

It is also possible to control the execution of `consume` with the class method `around_consume(payload, metadata)`. This method receives payload, metadata and the "consume method" in a form of a block, example:

```ruby
class MyHandler
  include Phobos::Handler

  def self.around_consume(payload, metadata)
    Phobos.logger.info "consuming..."
    output = yield
    Phobos.logger.info "done, output: #{output}"
  end

  def consume(payload, metadata)
  end
end
```

Take a look at the examples folder for some ideas.

We can resume the hander life cycle as:

  `.start` -> `#consume` -> `.stop`

  `.start` -> `.around_consume` [ `#consume` ] -> `.stop`

### <a name="#usage-producing-messages-to-kafka"></a> Producing messages to kafka

`ruby-kafka` provides several options for publishing messages, Phobos offers them through the module `Phobos::Producer`. It is possible to turn any ruby class into a producer (including your handlers), just include the producer module, example:

```ruby
class MyProducer
  include Phobos::Producer
end
```

Phobos is designed for multithreading thus the producer is always bound to the current thread. It is possible to publish messages from objects and classes, pick the option that suits your code better.
The producer module doesn't pollute your classes with a thousand methods, it includes a single method the class and in the instance level: `producer`.

```ruby
my = MyProducer.new
my.producer.publish('topic', 'message-payload', 'partition and message key')

# The code above has the same effect of this code:
MyProducer.producer.publish('topic', 'message-payload', 'partition and message key')
```

It is also possible to publish several messages at once:

```ruby
MyProducer
  .producer
  .publish_list([
    { topic: 'A', payload: 'message-1', key: '1' },
    { topic: 'B', payload: 'message-2', key: '2' },
    { topic: 'B', payload: 'message-3', key: '3' }
  ])
```

There are two flavors of producers: __normal__ producers and __async__ producers.

Normal producers will deliver the messages synchronously and disconnect, it doesn't matter if you use `publish` or `publish_list` after the messages get delivered the producer will disconnect.

Async producers will accept your messages without blocking, use the methods `async_publish` and `async_publish_list` to use async producers. One detail regarding async producers is that you need to shutdown them manually before you close the application. Use the class method `async_producer_shutdown` to safely shutdown the producer.

An example using your handlers to publish messages would be:

```ruby
class MyHandler
  include Phobos::Handler
  include Phobos::Producer

  PUBLISH_TO = 'topic2'

  def self.stop
    producer.async_producer_shutdown
    producer.kafka_client.close
  end

  def consume(payload, metadata)
    producer.async_publish(PUBLISH_TO, {key: 'value'}.to_json)
  end
end
```

Without configuring the kafka client, the producers will create a new one when needed (once per thread). To configure the kafka client use the class method `configure_kafka_client`, example:

```ruby
class MyHandler
  include Phobos::Handler
  include Phobos::Producer

  def self.start(kafka_client)
    producer.configure_kafka_client(kafka_client)
  end

  def self.stop
    producer.async_producer_shutdown
  end

  def consume(payload, metadata)
    producer.async_publish(PUBLISH_TO, {key: 'value'}.to_json)
  end
end
```

Using the same client as the listener is a good idea because it will be managed by Phobos and properly closed when needed.

### <a name="#usage-instrumentation"></a> Instrumentation

Some operations are instrumented using [Active Support Notifications](http://api.rubyonrails.org/classes/ActiveSupport/Notifications.html).

In order to receive notifications you can use the module `Phobos::Instrumentation`, example:

```ruby
Phobos::Instrumentation.subscribe('listener.start') do |event|
  puts(event.payload)
end
```

`Phobos::Instrumentation` is just a convenience module around `ActiveSupport::Notifications`, feel free to use it or not. All Phobos events are in the `phobos` namespace. `Phobos::Instrumentation` will always look at `phobos.` events.

#### Executor notifications
  * `executor.retry_listener_error` is sent when the listener crashes and the executor wait for a restart. It includes the following payload:
    * listener_id
    * retry_count
    * waiting_time
    * exception_class
    * exception_message
    * backtrace
  * `executor.stop` is sent when executor stops

#### Listener notifications
  * `listener.start_handler` is sent when invoking `handler.start(kafka_client)`. It includes the following payload:
    * listener_id
    * group_id
    * topic
  * `listener.start` is sent when listener starts. It includes the following payload:
    * listener_id
    * group_id
    * topic
  * `listener.process_batch` is sent after process a batch. It includes the following payload:
    * listener_id
    * group_id
    * topic
    * batch_size
    * partition
    * offset_lag
    * highwater_mark_offset
  * `listener.process_message` is sent after process a message. It includes the following payload:
    * listener_id
    * group_id
    * topic
    * key
    * partition
    * offset
    * retry_count
  * `listener.retry_handler_error` is sent after waited for `handler#consume` retry. It includes the following payload:
    * listener_id
    * group_id
    * topic
    * key
    * partition
    * offset
    * retry_count
    * waiting_time
    * exception_class
    * exception_message
    * backtrace
  * `listener.retry_aborted` is sent after waiting for a retry but the listener was stopped before the retry happened. It includes the following payload:
    * listener_id
    * group_id
    * topic
  * `listener.stop_handler` is sent after stopping the handler
    * listener_id
    * group_id
    * topic
  * `listener.stop` is send after stopping the listener
    * listener_id
    * group_id
    * topic

## <a name="#development"></a> Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

The `utils` folder contain some shell scripts to help with the local kafka cluster. It uses docker to start kafka and zookeeper.

```sh
sh utils/start-all.sh
sh utils/stop-all.sh
```

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/klarna/phobos.

## License

Copyright 2016 Klarna

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License.

You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
