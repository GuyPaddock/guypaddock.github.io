---
layout: article
title: "Demystifying Rails 3.x Template Rendering"
categories: posts
modified: 2017-09-27T14:00:00-04:00
tags: [ruby, templates, erb, rails3, ruby on rails]
comments: true
ads: false
---
Now that Rails 5.x is out, I know that it's very rare for anyone to still be talking about Rails 3.x. But, for those poor souls (like us) who still are maintaining system written with that version of the framework, I wanted to share some knowledge I've pieced together that did not appear elsewhere on the web.

_(If you're running Rails 4.x, this topic is covered in nauseating detail here: http://climber2002.github.io/blog/2015/02/21/how-rails-finds-your-templates-part-1/)_

## Questions
First, let me start with the questions we were recently asking here that prompted this research:

1. When a controller inherits from `ActionController::Base` instead of `ApplicationController`, Rails looks for its template under `views` (e.g. `views/awesome_sauce/my_orders/new.html.erb`), but if the controller inherits from `ApplicationController` this does not happen (i.e. it always uses `views/layouts/application.html.erb`). Why?

1. If a controller inherits from `Devise::SessionsController`, Rails tries to find a layout called `views/layout/devise/sessions_controller.html.erb`, and then ends up using `views/layout/application.html.erb`, even when there is a template for the current controller action `views/my_controller/action_name.html.erb` (where `action_name` is something like `new`, `show`, etc). Why?

1. How does rails determine what layout to use when several parent controllers have layouts that match their names? Are they nested?

## Answers
All right, now the answers:

1. _When a controller inherits from `ActionController::Base` instead of `ApplicationController`, Rails looks for its template under `views` (e.g. `views/awesome_sauce/my_orders/new.html.erb`), but if the controller inherits from `ApplicationController` this does not happen (i.e. it always uses `views/layouts/application.html.erb`). Why?_

The answer is related to the fact that there are actually two distinct concepts here in Rails that are often conflated &ndash; _views_ and _layouts_. Even [the Rails guides](http://guides.rubyonrails.org/v3.2/layouts_and_rendering.html#finding-layouts) seem to group these together as once concept &ndash; "views" &ndash; but they're actually separate things that Rails handles differently.

Here's how the two things are different:
- **views** provide the content to display for a specific controller action. This content may be:
-- Rendered inside a layout. In this case, the content of the view should not be a full HTML page, but just enough HTML to fill out a content area of the larger page; or
-- Rendered stand-alone, without a layout. In this case, the content of the view must be a full HTML page, complete with the `<html>` tag, headers, body, etc.
- **layouts** provide the skeleton / boilerplate content of the HTML page into which a view is rendered. If a layout is available for a given controller, or its parents, Rails will render the view for the controller action into the layout whereever the layout calls `yield`.

Rails determines the name of the _layout_ to search for by taking the controller name as follows _(the steps might actually be ordered differently inside Rails ActionPack, but these are the steps Rails effectively does)_:

1. It removes the word "Controller" at the end (e.g. "AwesomeSauce::MyOrdersController" becomes "AwesomeSauce::MyOrders").
2. It converts double-colons (for namespaces) in the remaining name into forward slashes (e.g. "AwesomeSauce::MyOrders" becomes "AwesomeSauce/MyOrders").
3. It converts the result from CamelCase to snake_case (e.g. "AwesomeSauce/MyOrders" becomes "awesome_sauce/my_orders").
4. It looks for an HTML ERB file under "views/layouts/" that matches the result (e.g. it looks for "views/layouts/awesome_sauce/my_orders.html.erb").
5. If it does not find a template for the current controller, it repeats steps 1-5 for each of the ancestors of the current controller.

Now, to actually answer the question:
- *If a controller inherits from `ApplicationController`:* per the algorithm above, Rails will look for a layout under `views/layouts/application.html.erb` since that's the name of the current controller's parent. If it finds a file with that name, it renders the view for the controller action in the content area of that layout.
- *If a controller inherits from `ActionController::Base`:* the algorithm above will fail to find a layout, so Rails will render the view "bare" / "raw" &nbdash; without a layout. So, if the content of the view is a full HTML page, that's what gets rendered. That's why it seems strange when you change base classes that the HTML page rendered changes.

2. _If a controller inherits from `Devise::SessionsController`, Rails tries to find a layout called `views/layout/devise/sessions_controller.html.erb`, and then ends up using `views/layout/application.html.erb`, even when there is a template for the current controller action `views/my_controller/action_name.html.erb` (where `action_name` is something like `new`, `show`, etc). Why?_

All of Devise's controllers inherit from `DeviseController`, which has a _dynamic parent class_ specified by `Devise.parent_controller`. By default, the parent controller Devise uses is [`ApplicationController`](https://github.com/plataformatec/devise/blob/v2.2/lib/devise.rb#L205) unless you change that in your application configuration. So, if you combine that knowledge with the answer from step #1, that's why devise controllers will tend to use the `application.html.erb` file if there is no `devise/sessions_controller.html.erb` layout.

3. _How does rails determine what layout to use when several parent controllers have layouts that match their names? Are they nested?_

They are not nested -- they are replaced. Rails injects an internal method called `_layout` into each controller that looks something like this:

```Ruby
def _layout
  lookup_context.find_all("awesomesauce/my_orders", ["layouts"]).first || super
end
```
(For conditional layouts, the definition looks a bit different; I haven't yet figured out how to interpret [the code that handles them](https://github.com/rails/rails/blob/3-2-stable/actionpack/lib/abstract_controller/layouts.rb#L274)).

This means that it starts at the current controller and navigates up. As soon as it encounters a controller in the hierarchy for which there is a layout, it stops looking for another one. If it doesn't find a layout in the hierarchy, per my answer to question #1, it renders the view for the controller action without wrapping it in a layout.

## Conclusion
I hope this is helpful to someone! I know we certainly spent a few days trying to understand why our controllers weren't behaving the way we expected when we changed their parent classes a bit.

![The More You Know](https://media.giphy.com/media/3og0IMJcSI8p6hYQXS/giphy.gif)
