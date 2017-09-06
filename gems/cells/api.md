---
layout: cells
title: "Cells API"
gems:
  - ["cells", "trailblazer/cells", "4.1"]
---

A cell is an object that can render views. It represents a fragment of the page, or the entire page.

Sometimes they're also called _object-oriented partials_.

The object has to define at least one method which in turn has to return the fragment. Per convention, this is `#show`. In this public method, you may compile arbitrary strings or `render` a cell view.

The return value of this public method (also called _state_) is what will be the rendered in the view using the cell.

## Anatomy

Cells usually inherit from `Cell::ViewModel`.

```ruby
class CommentCell < Cell::ViewModel
  def show
    render # renders app/cells/comment/show.haml
  end
end
```

When the `CommentCell` cell is invoked, its `show` method is called, the view is rendered, and returned as a HTML string.

This snippet illustrates a _suffix cell_, because it follows the outdated Rails-style naming and file structure. We encourage you to use [Trailblazer cells](trailblazer.html). However, this document mostly describes the generic API.

## Show

As per convention, `#show` is the only public method of a cell class.

The return value of this method is what gets rendered as the cell.

```ruby
def show
  "I don't like templates!"
end
```

You're free to return whatever string you desire, use your own rendering engine, or use cells' `render` for templates.

## Manual Invocation

In its purest form, a cell can be rendered as follows.

```ruby
Comment::Cell.new(comment).() #=> "I don't like templates!"
```

This can be split up into two steps: initialization and invocation.

## Initialize

You may instantiate cells manually, wherever you want.

```ruby
cell = Comment::Cell.new(comment)
```

This is helpful in environments where the helpers are not available, e.g. a Rails mailer or a `Lotus::Action`.

Note that usually you pass in an arbitrary object into the cell, the _"model"_. Here, this is the `comment` instance.

## Model

The model you pass into the cell's constructor is completely up to you! It could be an ActiveRecord instance, a `Struct`, or an array of items.

The model is available via the `model` reader.

```ruby
def show
  model.rude? ? "Offensive content." : render
end
```

The term *model* is really not to be confused with the way Rails uses it - it can be just anything.

## Property

Cells allow a short form to access model's attributes using the `property` class method.

```ruby
class CommentCell < Cell::ViewModel
  property :email #=> model.email

  def show
    model.email #=> "s@trb.to"
    email #=> "s@trb.to"
  end
end
```

Using `::property` will create a convenient reader method for you to the model.

## Options

Along with the model, you may also pass arbitrary options into the cell, for example the current user.

```ruby
cell = Comment::Cell.new(comment, current_user: current_user)
```

In the cell, you can access any options using  the `options` reader.

```ruby
def show
  options[:current_user] ? render : "Not logged in!"
end
```

## Invocation

Once you've got the cell instance, you may call the rendering state. This happens via `ViewModel#call`.

```ruby
cell.call(:show)
```

It's a common idiom in Ruby to skip the explicit `call` method name. The next snippet does the same as the above.

```ruby
cell.(:show)
```

Since `show` is the default state, you may simple _call_ the cell without arguments.

    cell.() #=> cell.(:show)

Note that in Rails controller views, this will be called automatically via cell's `ViewModel#to_s` method.

## Call

Always invoke cell methods via `call`. This will ensure that caching - if configured - is performed properly.

    CommentCell.new(comment).(:show)

As discussed, this will call the cell's `show` method and return the rendered fragment.

Note that you can invoke more than one state on a cell, if desired.

```ruby
- cell = CommentCell.new(Comment.last)      # instantiate.
= cell.call(:show)                          # render main fragment.
= content_for :footer, cell.(:footer)       # render footer.
```

See how you can combine cells with global helpers like `content_for`?

You can also provide additional arguments to `call`.

```ruby
cell.(:show, Time.now)
```

All arguments after the method name are passed to the invoked method.

```ruby
def show(time)
  time #=> Now!
end
```

Even blocks are allowed.


```ruby
cell.(:show) { "Yay!" }
```

Again, the block is passed through to the invoked method.


```ruby
def show(&block)
  yield #=> "Yay!"
end
```

This is particularly interesting when passing the block to `render` and using `yield` in the view. See [render](#render)'s docs for that.

## Default Show

Per default, every cell derived from `Cell::ViewModel` has a built-in `show` method.

```ruby
class CommentCell < Cell::ViewModel
  # #show is inherited.
end
```

The implementation looks as follows.

```ruby
def show(&block)
  render &block
end
```

An optional block is always passed to the `render` method.

Of course, you're free to override `show` to do whatever it needs to do.

## Instantiation Helper

In most environments you will instantiate cells with the `concept` or `cell` helper which internally does exactly the same as the manual invocation.

```ruby
cell = cell(:comment, comment)
```

This is identical to

```ruby
cell = CommentCell.new(comment)
```

Depending on your environment, the `cell` helper might inject dependencies into the created cell. For example, in Rails, the controller is passed on into the cell behind the scenes. When manually instantiating cells, you must not forget to do so, too.

The `cell` helper also allows passing in the cell constant. This means, it won't try to infer the class constant name.

```ruby
cell = cell(CommentCell, comment)
```

## File Structure

Having a cell to represent a fragment of your page is one thing. The real power, whatsoever, comes when rendering templates in cells. The `render` method does just that.

In a suffix environment, Cells expects the following file layout.

```
├── app
│   ├── cells
│   │   └── comment_cell.rb
│   │   └── comment
│   │       └── show.haml
```

Every cell - unless configured otherwise - has its own view  directory named after the cell's name (`comment`). Views do only have one extension to identify the template's format (`show.haml`). Again, you're free to provide arbitrary additional extensions.

Note that the _suffix_ style shown above is deprecated, and will be superseded in Cells 5 by the [Trailblazer](trailblazer.html)-style naming and file structure.

## Render


```ruby
class CommentCell < Cell::ViewModel
  def show
    render # renders show.haml.
  end
end
```

A simple `render` will implicitly figure out the method (or state) name and attempt to render that view. Here, the file will be resolved to `app/cells/comment/show.haml`.

Note that `render` literally renders the template and returns the HTML string. This allows you to call render multiple times, concatenate, and so on.

```ruby
def show
  render + render(:footer) + "<hr/>"
end
```

You can provide an explicit view name as the first argument.

```ruby
def show
  render :item # renders item.haml
end
```

When providing more than one argument to `render`, you have to use the `:view` option.


```ruby
def show
  render view: :item # renders item.haml
end
```

If you like the clunky Rails-style file naming, you're free to add a format to the view.

```ruby
render "shot.html" # renders show.html.haml
```

You can pass locals to the view using `:locals`.

```ruby
render locals: { logged_in: options[:current_user] }
```

## Instance Methods

While it is fine to use locals or instance variables in the view to access data, the preferred way is invoking instance methods in the view.

    %h1 Show comment

    = body
    = author_link

Every method call in the view is dispatched to the cell instance. You have to define your "helpers" there.

```ruby
class CommentCell < Cell::ViewModel
  property :body # creates #body reader.

  def author_link
    url_for model.author.name, model.author
  end
end
```

This allows slim, logic-less views.

Note that you can use Rails in the instance level, too, if you're in a Rails environment.

## Yield

A block passed to the cell constructor is passed on to the state method.

```ruby
CommentCell.new(comment) { "Yay!" }
cell(:comment, comment)  { "Yay!" }
```

It's up to you if you want to use this block, or provide your own.

```ruby
def show(&block)
  render(&block)
end
```

Passing the block `render` allows yielding it in the view.

    %h1 Comment

    = yield

## Collection

Instead of manually iterating an array of models and concatenating the output of the item cell, you can use the `:collection` feature.

```ruby
cell(:comment, collection: Comment.all).()
```

This will instantiate a cell per iterated model, invoke `call` and join the output into one fragment.

Pass the method name to `call` when you want to invoke a state different to `show`.

```ruby
cell(:comment, collection: Comment.all).(:item)
```

You're free to pass additional options to the call.

```ruby
cell(:comment, collection: comments, size: comments.size).()
```

This instantiates each collection cell as follows.

```ruby
CommentCell.new(comment, size: 9)
```

You can use the `join` method to customize how each item in the collection is invoked. The return value of the block is automatically inserted in between each rendered item in the collection0

```ruby
class CommentCell < Cell::ViewModel
  def odd
    "odd comment\n"
  end

  def even
    "even comment\n"
  end
end

cell(:comment, collection: Comment.all).join do |cell, i|
  i.odd? ? cell.(:odd) : cell(:even)
end
# => "odd comment\neven comment\nodd comment\neven comment"
```

An optional separator string can be passed to `join` when it concatenates the item fragments.

```ruby
cell(:comment, collection: Comment.all).join("<hr/>") do |cell, i|
  i.odd? ? cell.(:odd) : cell(:even)
end
# => "odd comment\n<hr/>even comment\n<hr/>odd comment\n<hr/>even comment"
```

Alternatively, if you just want to add some extra content in between each rendered item and don't need to customize how each item is invokved, you can call `join` with a separator and no block:

```ruby
class PostCell
  def show
    'My post'
  end
end

cell(:post, collection: Post.all).join("<hr/>")
# => "My post<hr/>My post<hr/>My post"
```

## External Layout

Since Cells 4.1, you can instruct your cell to use a second cell as a wrapper. This will first render your actual content cell, then pass the content via a block to the layout cell.

Cells desiring to be wrapped in a layout have to include `Layout::External`.

```ruby
class CommentCell < Cell::ViewModel
  include Layout::External
end
```

The layout cell usually can be an empty subclass.

```ruby
class LayoutCell < Cell::ViewModel
end
```

Its `show` view must contain a `yield` to insert the content.

    !!!
    %html
      %head
        %title= "Gemgem"
        = stylesheet_link_tag 'application', media: 'all'
        = javascript_include_tag 'application'
      %body
        = yield

The layout cell class is then injected into the actual invocation using `:layout`.

```ruby
cell(:comment, comment, layout: LayoutCell)
```

The context object will automatically be passed to the layout cell.

Note that `:layout` also works in combination with `:collection`.

## View Paths

Per default, the cell's view path is set to `app/cells`. You can set any number of view paths for the template file lookup.

```ruby
class CommentCell < Cell::ViewModel
  self.view_paths = ["app/views"]
```

Note that the default view paths are different if you're using the [Trailblazer-style file structure](trailblazer.html).

## Template Formats

Cells provides support for a handful of popular template formats like ERB, Haml, etc.

You need to add the specific template engine to your Gemfile:

* [cells-erb](https://github.com/trailblazer/cells-erb)
* [cells-hamlit](https://github.com/trailblazer/cells-hamlit) We strongly recommend using [Hamlit](https://github.com/k0kubun/hamlit) as a Haml replacement.
* [cells-haml](https://github.com/trailblazer/cells-haml) Make sure to bundle Haml 4.1: `gem "haml", github: "haml/haml", ref: "7c7c169"`. Use `cells-hamlit` instead.
* [cells-slim](https://github.com/trailblazer/cells-slim)

```ruby
gem "cells-erb"
```

In Rails, this is all you need to do. In other environments, you need to include the respective module into your cells.

```ruby
class CommentCell < Cell::ViewModel
  include ::Cell::Erb # or Cell::Hamlit, or Cell::Haml, or Cell::Slim
end
```

## HTML Escaping

Cells per default does no HTML escaping, anywhere. This is one of the reasons why Cells is much faster than Rails' ActionView.

Include `Escaped` to make property readers return escaped strings.


    class CommentCell < Cell::ViewModel
      include Escaped
      property :title
    end

    song.title                 #=> "<script>Dangerous</script>"
    Comment::Cell.(song).title #=> &lt;script&gt;Dangerous&lt;/script&gt;


Only strings will be escaped via the property reader.

You can suppress escaping manually.


    def raw_title
      "#{title(escape: false)} on the edge!"
    end

Of course this works in views too:

    <%= title(escape: false) %>

If you feel this is not enough, then have a look at `Decoration`.

## Decoration

If you don't like the HTML Escaping method above, because it is too easy to forget something, than maybe the Decoration feature is for you.

Include `Decoration` to wrap the model of the cell in a decorator class. The default decorator class is the EscapeDecorator.
It delegates every instance method to your model and HTML escapes the result if it is a string. If the result is an object it gets decorated again,
to escape the result of chaining method calls, too.


```ruby
class CommentCell < Cell::ViewModel
  include Decoration

  def show
    "Title: #{model.title} by #{model.artist.name}"
  end
end

song.title              #=> "<script>Dangerous</script>"
song.artist.name        #=> "<script>Dangerous, too!</script>"
Comment::Cell.(song).() #=> Title: &lt;script&gt;Dangerous&lt;/script&gt; by &lt;script&gt;Dangerous, too!&lt;/script&gt;
```

If you want to change the default decorator and use your own, you can just set `self.decorator_class` in your cell to your own decorator.


```ruby
class SongDecorator < Cell::ViewModel::Decoration::EscapeDecorator
  def published_at
    if datetime = super
      I18n.l(datetime)
    end
  end
end

class CommentCell < Cell::ViewModel
  include Decoration
  self.decorator_class = SongDecorator

  def show
    "Title: #{model.title} by #{model.artist.name} at #{model.published_at}"
  end
end
```

So this basic decorator pattern allows not only escaping but also reusable formatting of attribute values of your model for the view.
If you have one attribute, that you don't want to get escaped automatically you have to define it in your own decorator and call the model accessor directly.

```ruby
class SongDecorator < Cell::ViewModel::Decoration::EscapeDecorator
  def html_description
    @model.html_description
  end
end

class CommentCell < Cell::ViewModel
  include Decoration
  self.decorator_class = SongDecorator

  def show
    "Description: #{model.html_description}"
  end
end

song.html_description   #=> "<b>Yeah!</b>"
Comment::Cell.(song).() #=>"Description: <b>Yeah!</b>"
```

If you need you can also specify a `Twin` as decorator class.
If you want to instanciate the decorator by yourself simply do: `Cell::ViewModel::Decoration::EscapeDecorator.new(model)`


## Context Object

By default, every cell contains a context object. When [nesting cells](#nesting), this object gets passed in automatically. To add objects to the context, use the `:context` option.

```ruby
cell("comment", comment, context: { user: current_user })
```

You can read from the context object via the `context` method.

```ruby
def show
  context[:user] #=> <User ..>
  # ..
end
```

The context object is handy when dependencies need to be passed down (or up, when using layouts) a cell hierarchy.

Note that the context object gets `dup`ed when adding to it into nested cells. This is to prevent leaking nested state back into parent objects.

## Nesting

You can invoke cells in cells. This happens with the `cell` helper.

```ruby
def show
  html = cell(:comment_detail, model)
  # ..
end
```

The `cell` helper will automatically pass the context object to the nested cell.


## View Inheritance

Cells can inherit code from each other through Ruby's regular inheritance mechanisms.

```ruby
class CommentCell < Cell::ViewModel
end

class PostCell < CommentCell
end
```

Even cooler, `PostCell` will now inherit views from `CommentCell`.

```ruby
PostCell.prefixes #=> ["app/cells/post", "app/cells/comment"]
```

When views can't be found in the local `post` directory, they will be looked up in `comment`. This starts to become helpful when using [composed cells](#nesting).

If you only want to inherit views, not the entire class, use `::inherit_views`.

```ruby
class PostCell < Cell::ViewModel
  inherit_views Comment::Cell
end

PostCell.prefixes #=> ["app/cells/post", "app/cells/comment"]
```

## Builder

Often, it's good practice to replace decider code from views or classes by extracting it out into separate sub-cells. Or in case you want to render a polymorphic collection, builders come in handy.

Builders allow instantiating different cell classes for different models and options.

```ruby
class CommentCell < Cell::ViewModel
  include ::Cell::Builder

  builds do |model, options|
    if model.is_a?(Post)
      PostCell
    elsif model.is_a?(Comment)
      CommentCell
    end
  end
end
```
The `#cell` helper takes care of instantiating the right cell class for you.

```ruby
cell(:comment, Post.find(1)) #=> creates a PostCell.
```
This also works with collections.

```ruby
cell(:comment, collection: [@post, @comment]) #=> renders PostCell, then CommentCell.
```

Multiple calls to `::builds` will be ORed. If no block returns a class, the original class will be used (`CommentCell`). Builders are inherited.


## Caching

Cells allow you to cache per state. It's simple: the rendered result of a state method is cached and expired as you configure it.

To cache forever, don't configure anything

```ruby
class CartCell < Cell::Rails
  cache :show

  def show
    render
  end
```

This will run `#show` only once, after that the rendered view comes from the cache.


## Cache Options

Note that you can pass arbitrary options through to your cache store. Symbols are evaluated as instance methods, callable objects (e.g. lambdas) are evaluated in the cell instance context allowing you to call instance methods and access instance variables. All arguments passed to your state (e.g. via `render_cell`) are propagated to the block.

```ruby
cache :show, :expires_in => 10.minutes
```

If you need dynamic options evaluated at render-time, use a lambda.

```ruby
cache :show, :tags => lambda { |*args| tags }
```

If you don't like blocks, use instance methods instead.

```ruby
class CartCell < Cell::Rails
  cache :show, :tags => :cache_tags

  def cache_tags(*args)
    # do your magic..
  end
```

## Conditional Caching

The `:if` option lets you define a condition. If it doesn't return a true value, caching for that state is skipped.

```ruby
cache :show, :if => lambda { |*| has_changed? }
```

## Cache Keys

You can expand the state's cache key by appending a versioner block to the `::cache` call. This way you can expire state caches yourself.

```ruby
class CartCell < Cell::Rails
  cache :show do
    order.id
  end
```

The versioner block is executed in the cell instance context, allowing you to access all stakeholder objects you need to compute a cache key. The return value is appended to the state key: `"cells/cart/show/1"`.

As everywhere in Rails, you can also return an array.

```ruby
class CartCell < Cell::Rails
  cache :show do
    [id, options[:items].md5]
  end
```

Resulting in: `"cells/cart/show/1/0ecb1360644ce665a4ef"`.


## Debugging Cache

When caching is turned on, you might wanna see notifications. Just like a controller, Cells gives you the following notifications.

* `write_fragment.action_controller` for cache miss.
* `read_fragment.action_controller` for cache hits.

To activate notifications, include the `Notifications` module in your cell.

```ruby
class Comment::Cell < Cell::Rails
  include Cell::Caching::Notifications
```

## Cache Inheritance

Cache configuration is inherited to derived cells.

## Testing Caching

If you want to test it in `development`, you need to put `config.action_controller.perform_caching = true` in `development.rb` to see the effect.
