---
layout: article
title: "Re-ordering Existing Rack Middleware in Rails 3.x"
categories: posts
modified: 2017-09-28T21:30:00-04:00
tags: [ruby, rails3, ruby on rails, middleware, logging, mix-ins]
comments: true
ads: false
---
Time for yet another Rails 3.x-themed post! My typical note applies -- now that Rails 5.x is out, I know that it's very rare for anyone to still be talking about Rails 3.x. But, for those poor souls (like us) who still are maintaining systems written with that version of the framework, I wanted to share some knowledge I've pieced together that did not appear elsewhere on the web.

## The Problem
In our application, we wanted each Rails log line to print out the ID of the user who was logged-in at the time he or she made his or her request. Here's what we added in our `application.rb` to do this:

```Ruby
config.log_tags = [
  proc do |req|
    if req.session["warden.user.user.key"].nil?
      "Anonym"
    else
      "user_id:#{req.session["warden.user.user.key"][1][0]}"
    end
  end
]
```

Unfortunately, this just lead to every log line indicating the user as `Anonym`. After some research, we discovered it was a combination of the fact that we use cookie-based sessions and the fact that the default Rails middleware stack configures Rack with `Rails::Rack::Logger` before `ActionDispatch::Session::CookieStore`. In order for the logger to have access to the user information, it needs to come _after_ `CookieStore`.

## What the Docs Say
Ideally, what we wanted to do was simply move the existing `Logger` middleware instance further down in the stack, below the instance for `CookieStore. The docs gave us hope that this would be pretty straightforward.

The [Ruby on Rails Guide for 3.2](http://guides.rubyonrails.org/v3.2/rails_on_rack.html) says:
> The middleware stack behaves just like a normal `Array`. You can use any `Array` methods to insert, reorder, or remove items from the stack.

Unfortunately, **that's incorrect**. The guide instructs you to work with `config.middleware` in `application.rb`, but that object is a `Rails::Configuration::MiddlewareStackProxy`, which only implements a handful of methods and most definitely does not behave like an `Array` or `Enumerable`. `ActionDispatch::MiddlewareStack` _does_ behave like an `Enumerable`, but you don't get direct access to it.

In fact, as an aside, the documentation for this is even incorrect in [the Rails 4.0 Rails guide](http://guides.rubyonrails.org/v4.0/rails_on_rack.html), as far as I can tell. It says:
> The middleware stack behaves just like a normal `Enumerable`. You can use any `Enumerable` methods to manipulate or interrogate the stack. The middleware stack also implements some `Array` methods including `[]`, `unshift` and `delete`.

Luckily this was corrected in the Rails guide for Rails 4.1 and later, but where does that leave us?

## What We Originally Tried
Originally, we tried leveraging the few methods `Rails::Configuration::MiddlewareStackProxy` does provide to remove and re-insert the appropriate middleware.

The first time around, here's what our code looked like:

```Ruby
# Middleware config for user logging (take 1)
config.middleware.delete(Rails::Rack::Logger)
config.middleware.insert_after(ActionDispatch::Session::CookieStore, Rails::Rack::Logger)
```

But, this had one big problem &ndash; *the logger lost its configuration*! Then we tried this:
```Ruby
# Middleware config for user logging (take 2)
config.middleware.delete(ActionDispatch::Cookies)
config.middleware.delete(ActionDispatch::Session::CookieStore)
config.middleware.insert_before(Rails::Rack::Logger, ActionDispatch::Session::CookieStore)
config.middleware.insert_before(ActionDispatch::Session::CookieStore, ActionDispatch::Cookies)
```

That worked; *sort of*. This time around, the problem was that our session store settings in `config/initializers/session_store.rb` were being lost.

As it turns out, the methods that Rails provides for you to use end up blowing away perfectly-good, already-configured middleware. Then you end up having to reconfigure the new instance of the same middleware after you insert it back in the Rack.

There had to be a better way!

## An Actual Solution
Luckily, we found a way that _is_ better (we think). It just so happens that the default stack of middleware that Rails initializes is provided by a protected method on `Rails::Application` called `default_middleware_stack`. Since that's the case, we can just override that method in our own Rails `Application` class to get direct access to the middleware.

That approach ended up looking like this:
```Ruby
# This is inside application.rb
module MyApplication
  class Application < Rails::Application
    # ... normal Rails `config` initialization stuff here ...

    config.log_tags = [
      proc do |req|
        if req.session["warden.user.user.key"].nil?
          "Anonym"
        else
          "user_id:#{req.session["warden.user.user.key"][1][0]}"
        end
      end
    ]

    protected

    ##
    # Modifies the default Rack middleware stack to put the Rails logger
    # after cookie-based session loading, to allow us to capture user IDs in
    # log files.
    #
    def default_middleware_stack
      stack = super

      ::CoreExtensions::ActionDispatch::MiddlewareStack.import

      stack.move_middleware_after(
        Rails::Rack::Logger,
        ActionDispatch::Session::CookieStore)

      stack
    end
  end
end
```

The `move_middleware_after` method is provided by a mix-in, as follows:
```Ruby
##
# Custom extensions to the +ActionDispatch::MiddlewareStack+ class.
#
module CoreExtensions
  module ActionDispatch::MiddlewareStack
    ##
    # Imports the methods of this module into the
    # +ActionDispatch::MiddlewareStack+ class.
    #
    def self.import
      ::ActionDispatch::MiddlewareStack.send(:include, self)
    end

    ##
    # Moves an existing middleware instance after another.
    #
    # If either the middleware being moved (i.e. the "target") or the
    # middleware after which its being moved are not found, this method has
    # no effect.
    #
    # If the same type of middleware appears multiple times in this stack,
    # an error is raised.
    #
    # @param [String,Class] target_name
    #   The name or type of the middleware to move.
    # @param [String,Class] after_name
    #   The name or type of the middleware after which the target should be
    #   moved.
    #
    def move_middleware_after(target_name, after_name)
      # We fetch both to ensure each only appears once in the middleware stack
      target = find_middleware(target_name)
      after = find_middleware(after_name)

      return unless target && after

      middlewares.delete(target)

      after_index = middlewares.find_index(after)
      new_index   = after_index + 1

      middlewares.insert(new_index, target)
    end

    ##
    # Finds middleware in this stack.
    #
    # @param [String,Class] name
    #   The name or type of the middleware to find.
    #
    # @return [ActionDispatch::MiddlewareStack::Middleware,NilClass]
    #   Either the desired middleware; or, `nil` if it could not be found.
    #
    def find_middleware(name)
      matches = middlewares.select do |middleware|
        middleware.name == name.to_s
      end

      middleware_count = matches.length

      if middleware_count > 1
        raise ArgumentError,
              "Found the `#{target_name}` middleware `#{middleware_count}` "\
              "times in the stack, but expected to find it only once."
      else
        matches.first
      end
    end
  end
end
```

If your team is of the adventurous sort that doesn't really need to keep close tabs on monkey-patches, you could move this core extension to an initializer that's automatically included, instead of having to call `::CoreExtensions::ActionDispatch::MiddlewareStack.import` before the first time you need to use the methods it exposes. In our case, we like to explicitly declare which monkey-patch mix-ins are needed in each file so we can track where unexpected methods come from.

So, there you have it. I hope it's useful!
