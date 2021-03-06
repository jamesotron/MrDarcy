# MrDarcy

A mashup of Async Promises and DCI in ruby.

![Build Status](https://travis-ci.org/jamesotron/MrDarcy.svg?branch=master)

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'mr_darcy'
```

And then execute:

```sh
bundle
```

Or install it yourself as:

```sh
gem install mr_darcy
```

## Documentation

API Documentation is available on [rdoc.info](http://rdoc.info/gems/mr_darcy/)

## Usage

### WARNING

MrDarcy is definitely experimental, and was mostly built over the weekend of
[RailsCamp NZ 5](http://camp.ruby.org.nz/) with the generous help of the
amazing and sexy [@breccan](https://twitter.com/breccan) and
[@eoinkelly](https://twitter.com/eoinkelly).

### API Changes (0.4.0 and above)

As of version `0.4.0` the `instance_exec` of promise blocks has been removed
(meaning that you can't call `resolve` or `reject` from directly inside your
promise blocks).  This is becuase `instance_exec` changes the binding of
`self` inside the promise, and will cause calls to enclosed methods to fail.

For example:

```ruby
class InstanceExecExample

  def example_method
    puts "I did a thing"
  end

  def make_promise
    MrDarcy.promise { resolve example_method }.raise
  end
end
```

Will raise:

```
NameError in `block in make_promise': undefined local variable or method `example_method' for #<MrDarcy::Promise::DSL:0x007fcc542d75d0>
```

As of `0.4.0` the DSL object wll be passed to the promise block as an argument,
therefore:

```
  def make_promise
    MrDarcy.promise { |promise| promise.resolve example_method }.raise
  end
```

will not raise an exception.

#### Should I use MrDarcy in Production?

No.

#### How can I help make MrDarcy production ready?

Run it in production. Report bugs.

### Such promise. Many then.

Promises are a way of structuring batches of (async) functionality into a
pipeline, in such a way as to make them seem synchronous.

Here's an example:

```ruby
# We're going to wrap an asynchronous web request using EventMachine
# in a promise:
data = MrDarcy.promise do |promise|
  EM.run do
    http = EM.HttpRequest.new('http://camp.ruby.org.nz/').get
    http.errback do
      promise.reject http.error
      EM.stop
    end
    http.callback do
      promise.resolve http.response
      EM.stop
    end
  end
end.then do |response|
  response.body
end.result

puts data
```

What's cool about MrDarcy is that we can switch between different methods of
doing async ruby:
  - Naive threads, using MRI's thread implementation.
  - Reactor pattern, using [EventMachine](http://rubyeventmachine.com/) to
    schedule promises on the a reactor thread.
  - Actor pattern, using [Celluloid](http://celluloid.io/) to schedule
    promises using Celluloid futures.

#### Key points to know about Promises

  1. You create them with a block, which is scheduled asynchronously, and
     inside of which you can place your long-running executable. Inside this
     block you call either `resolve <value>` or `reject <exception>` to resolve
     or reject the promise.

     ```ruby
     MrDarcy.promise do |promise|
       accelerate_the_delorean
       if speed >= 88
         promise.resolve :time_flux_initiated
       else
         promise.reject :engage_service_brake
       end
     end
     ```

  2. All promises have `then` and `fail` methods, to which you pass a block to
     be called when the promise resolves (`then`) or rejects (`fail`). These
     methods return new promises, upon which you can chain more `then` and
     `fail` calls.

     ```ruby
     MrDarcy.promise do |promise|
       i = rand
       i > 0.5 ? promise.resolve i : promise.reject i
     end.then |value|
       # success
     end.fail |value|
       # failure
     end
     ```

  3. `fail` is used to catch errors asynchronously, and deal with them.
     Therefore the result of a `fail` block will be a resolved promise.
     If you wish to keep processing a failure then you can `raise` it
     within the `fail` block to pass it along to the next `fail` block.

     ```ruby
     MrDarcy.promise do |promise|
       promise.reject 2
     end.fail |value|
       value * value
     end.then |value|
       # I am called with 4
     end
     ```

  4. Failures cascade until they're caught:

     ```ruby
     MrDarcy.promise do |promise|
       promise.reject :fail
     end.then
       # I am skipped
     end.then
       # as am I
     end.fail
       # I am called
     end
     ```

  5. If your block returns a new promise, then `then` or `fail` will defer
     their resolution until the new promise is resolved:

     ```ruby
     MrDarcy.promise do |promise|
       promise.resolve 1
     end.then do |value|
       MrDarcy.promise do |p|
         p.resolve value * 2
       end
     end.then |value|
       # I will be called with 2
     end
     ```

#### Promise collections

If you have a bunch of promises, and you'd like to know when they're all
complete then you can use the `MrDarcy.all_promises` method:

```ruby
MrDarcy.all_promises do
  10.times.map { |i| MrDarcy.promise { |p| sleep 1; p.resolve i } }
end.then do |values|
  puts "All done."
end
```

In the above example, all 10 promises will execute simultaneously, the message
`All done.` will be printed to standard out, after approximately one second.

It will also reject as soon as the first promise rejects.

### Sprinkle on some DCI goodness.

[DCI](http://fulloo.info) is a method of specifying interactions between
objects in a single location, by decorating or extending your data objects
within a context, running the interaction and then (optionally) removing
the extensions again.

Other takes on DCI in Ruby:
  - [playhouse](https://github.com/enspiral/playhouse) is an app framework
    for building entire apps with DCI from the lovable hippies at
    [enspiral](http://www.enspiral.com/),

  - [surrounded](https://github.com/saturnflyer/surrounded) by
    [Jim Gay](https://github.com/saturnflyer) is a gem for doing DCI in
    a simple, repeatable fashion.

MrDarcy is a little differnt to these approaches, as is builds an interesting
DSL on top of promises to create contexts that are alive as long as they need
to be to achieve their goal, even when code is being run asynchronously.

Here's how we define a classic bank trasfer example:

```ruby
class BankTransfer < MrDarcy::Context
  role :money_source do
    def has_available_funds? amount
      available_balance >= amount
    end

    def subtract_funds amount
      self.available_balance = available_balance - amount
    end
  end

  role :money_destination do
    def receive_funds amount
      self.available_balance = available_balance + amount
    end
  end

  action :transfer do |amount|
    if money_source.has_available_funds? amount
      money_source.subtract_funds amount
      money_destination.receive_funds amount
    else
      raise "insufficient funds"
    end
    amount
  end
end
```

  - The `role` class method defines roles, which the context will expect to be
    passed on object creation.  They also define the extra behaviour (methods)
    that we wish to see added to our role players at initialisation.

  - The `action` class method defines our interactions. In other words, this
    is where we define how the interaction will take place. These also define
    instance methods on the class of the same name, which return a promise
    when called.

  - You can have as many roles and actions as needed by your context.

Let's transfer some funds between two accounts:

```ruby
Account = Struct.new(:available_balance)
marty     = Account.new(10)
doc_brown = Account.new(15)

context = BankTransfer.new money_source: marty, money_destination: doc_brown
context.transfer(5).then do |amount|
  puts "Successfully transferred #{amount} from #{money_source} to #{money_destination}"
end

context.transfer(50).fail do |exception|
  puts "Failed to transfer funds: #{exception.message}"
end
```

What's super cool, however is that because promises can return and chain other
promises, we can come up with some pretty involved scenarios:

```ruby
marty     = Account.new(10)
jenn      = Account.new(10)
doc_brown = Account.new(200)

context = BankTransfer.new money_source: marty, money_destination: jenn
context.transfer(20).fail do
  # Oh no, Marty doesn't have enough money, let's borrow some from Doc.
  BankTransfer.new(money_source: doc_brown, money_destination: marty) \
    .transfer(20).then
      # Try transferring again.
      context.transfer(20)
    end
end
```

I hope that's enough to get you started.  Yup, it's a bit crazy, but it might
just work.

## Contributing

1. Fork it ( http://github.com/jamesotron/mr_darcy/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
