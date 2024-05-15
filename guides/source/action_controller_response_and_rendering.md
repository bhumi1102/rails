**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON https://guides.rubyonrails.org.**

Action Controller Response and Rendering
========================================

This guide covers the handoff between Action Controller and Action View.

After reading this guide, you will know:

* How the Controller interacts with the View to create a response.
* How to use rendering methods such as `render` and `redirect_to`.

--------------------------------------------------------------------------------

Overview: How the Pieces Fit Together
-------------------------------------

This guide focuses on the interaction between Controllers and Views in the Model-View-Controller (MVC) pattern. In Rails, [Action Controller](action_controller_overview.html) is responsible for orchestrating the process of handling an HTTP request and composing a response. The Controller layer first hands off data access logic to the Model layer. Then, when it's time to send a response back to the client, the Controller layer hands things off to the View layer. This guide focuses on the handoff between the Controller and the View layers.

The Controller to View interaction has two parts. The first part involves the Controller deciding what type of response to send and using an appropriate method to create that response. The second part is about finding the correct layout and wrapping the response in that layout, when the response is a full-blown view.

Rendering Views by Convention
-----------------------------

By default, controllers in Rails automatically render views with names that match controller action names and correspond to [CRUD verbs and routes](routing.html#crud-verbs-and-actions).

NOTE: Default rendering of views that match controller action names is an excellent example of the ["convention over configuration"](https://rubyonrails.org/doctrine#convention-over-configuration) technique that Rails promotes.

For example, if you have an empty Controller class `BooksController`:

```ruby
class BooksController < ApplicationController
end
```

And the following in your routes file:

```ruby
resources :books
```

And you have a view file `index.html.erb` at the default location `app/views/books/` with this:

```html+erb
<h1>Books are coming soon!</h1>
```

When you navigate to `/books`, Rails will automatically find and render `app/views/books/index.html.erb`. And you will see "Books are coming soon!" on your screen. This is because Rails is using naming conventions for routes, controller actions, and view files as well as the view file location to find and render the `index` view in response to navigating to `/books`.

This is not magic. It is "convention over configuration" in action. You can imagine that the following code implicitly exists in `BooksController` class:

```ruby
# We do not explicitly need to write the boilerplate index action.
class BooksController < ApplicationController
  def index
    render :index
  end
end
```

But we do not need write the above boilerplate `index` action. You'd define the `index` method once you need to do anything other than `render :index`. If you create a `Book` model and add the following index action to `BooksController`:

```ruby
class BooksController < ApplicationController
  def index
    @books = Book.all
  end
end
```

This will find and render the same `index.html.erb` file. Note that we still do not need to have an explicit `render` statement at the end of the `index` action.

If you do not explicitly render something at the end of a controller action, Rails will automatically look for the `action_name.html.erb` view in the controller's view path and render it. So in this case, Rails will render the `app/views/books/index.html.erb` file.

Now that we have a `Book` model and `@books` instance variable available we would modify the `index` view to display more details about all the books. Something like this:

```html+erb
<h1>Listing Books</h1>

<table>
  <thead>
    <tr>
      <th>Title</th>
      <th>Content</th>
      <th colspan="3"></th>
    </tr>
  </thead>

  <tbody>
    <% @books.each do |book| %>
      <tr>
        <td><%= book.title %></td>
        <td><%= book.content %></td>
        <td><%= link_to "Show", book %></td>
        <td><%= link_to "Edit", edit_book_path(book) %></td>
        <td><%= link_to "Destroy", book, data: { turbo_method: :delete, turbo_confirm: "Are you sure?" } %></td>
      </tr>
    <% end %>
  </tbody>
</table>

<br>

<%= link_to "New book", new_book_path %>
```

NOTE: The actual rendering is done by nested classes of the module [`ActionView::Template::Handlers`](https://api.rubyonrails.org/classes/ActionView/Template/Handlers.html). This guide does not dig into that process, but it's important to know that the file extension, such as `html.erb`, controls the choice of template handler.

So far, we have described how controller actions render responses implicitly. Now, let's see how to explicitly create more elaborate responses from controller actions.

From the controller's point of view, there are three ways to create an HTTP response:

* Call [`render`][controller.render] to create a full response to send back to the browser. More examples in [Creating Responses Using `render` section](#creating-responses-using-render).
* Call [`redirect_to`][] to send an HTTP redirect status code to the browser. More examples in [Creating Responses Using `redirect-to` section](#creating-responses-using-redirect-to)
* Call [`head`][] to create an HTTP header only response. More in [this section](#building-header-only-responses-using-head).

[controller.render]: https://api.rubyonrails.org/classes/ActionController/Rendering.html#method-i-render
[`redirect_to`]: https://api.rubyonrails.org/classes/ActionController/Redirecting.html#method-i-redirect_to
[`head`]: https://api.rubyonrails.org/classes/ActionController/Head.html#method-i-head

The following sections cover creating responses with each of the three options. As well as the difference between render and redirect.

Creating Responses Using `render`
---------------------------------

This section describes the various way in which you can customize the behavior of `render`. The controller's [`render`][controller.render] method does the heavy lifting of constructing a response to HTTP requests and sending your application's content to the client.

You can render the default view for a controller action, or a specific view template, or a file, or inline code, or nothing at all. You can render text, JSON, or XML. You can specify the content type or HTTP status of the rendered response as well.

TIP: If you want to see the exact results of a call to `render` without needing to inspect it in a browser, you can call `render_to_string`. This method takes exactly the same options as `render`, but it returns a string instead of sending a response back to the browser.

### Rendering an Action's View

If you want to render the view that corresponds to a different template within the same controller, you can use `render` with the name of the view:

```ruby
def update
  @book = Book.find(params[:id])
  if @book.update(book_params)
    redirect_to(@book)
  else
    render "edit"
  end
end
```

Calling the `update` action in this controller will render the `edit.html.erb` template, in the `else` clause if the call to `@book.update` fails. The `edit.html.erb` file is assumed to belong to the same controller.

If you prefer, you can use a symbol instead of a string to specify the action to render:

```ruby
def update
  @book = Book.find(params[:id])
  if @book.update(book_params)
    redirect_to(@book)
  else
    render :edit, status: :unprocessable_entity
  end
end
```

### Rendering an Action's Template from Another Controller

What if you want to render a template from an entirely different controller from the one that contains the action code?

You can also do that with `render`. It accepts a path, relative to `app/views`, of the template you want to render. For example, if you have `some_action` in a `DifferentController` that lives in `app/controllers/`, you can render `show.html.erb` template in `app/views/books` this way:

```ruby
# In `app/controllers/different_controller.rb`
def some_action
  render "books/show"
end
```

Rails knows that this view belongs to a different controller because of the embedded slash character in the string. If you want to be explicit, you can use the `:template` option:

```ruby
render template: "books/show"
```

The above two ways of rendering (rendering the template of another action in the same controller, and rendering the template of an action in a different controller) are actually variants of the same operation.

In fact, in the `BooksController` class, inside of the update action where we want to render the edit template if the book does not update successfully, all of the following render calls would all render the `edit.html.erb` template in the `views/books` directory:

```ruby
render :edit
render action: :edit
render "edit"
render action: "edit"
render "books/edit"
render template: "books/edit"
```

Which one you use is really a matter of style and convention, but the rule of thumb is to use the simplest one that makes sense for the code you are writing.

### Using `render` with `:inline`

It is possible to use the `render` method without a view at all. There is an `:inline` option that takes an ERB string as part of the method call. For example, this is valid:

```ruby
render inline: "<% products.each do |p| %><p><%= p.name %></p><% end %>"
```

Rails applications do not typically render ERB inline in strings like this. You might do this for a quick demonstration like `render inline: "<h1>Hello World</h1>"`.

By default, inline rendering uses ERB. You can force it to use Builder instead with the `:type` option:

```ruby
render inline: "xml.p {'Horrid coding practice!'}", type: :builder
```

WARNING: There is rarely a good reason to use this option in production code. Mixing ERB into your controllers mixes controllers and views in MVC. It will make it harder for other developers to follow the logic of your project. Use a separate ERB file instead.

### Rendering Text

You can send plain text - with no markup at all - back to the browser by using
the `:plain` option to `render`:

```ruby
render plain: "OK"
```

TIP: Rendering pure text is most useful when you're responding to Ajax or web
service requests that are expecting something other than proper HTML.

NOTE: By default, if you use the `:plain` option, the text is rendered without
using the current layout. If you want Rails to put the text into the current
layout, you need to add the `layout: true` option and use the `.text.erb`
extension for the layout file.

### Rendering HTML

You can send an HTML string back to the browser by using the `:html` option to
`render`:

```ruby
render html: helpers.tag.strong('Not Found')
```

TIP: This is useful when you're rendering a small snippet of HTML code.
However, you might want to consider moving it to a template file if the markup
is complex.

NOTE: When using `html:` option, HTML entities will be escaped if the string is not composed with `html_safe`-aware APIs.

### Rendering JSON

JSON is a JavaScript data format used by many Ajax libraries. Rails has built-in support for converting objects to JSON and rendering that JSON back to the browser:

```ruby
render json: @product
```

TIP: You don't need to call `to_json` on the object that you want to render. If you use the `:json` option, `render` will automatically call `to_json` for you.

### Rendering XML

Rails also has built-in support for converting objects to XML and rendering that XML back to the caller:

```ruby
render xml: @product
```

TIP: You don't need to call `to_xml` on the object that you want to render. If you use the `:xml` option, `render` will automatically call `to_xml` for you.

### Rendering Vanilla JavaScript

Rails can render vanilla JavaScript:

```ruby
render js: "alert('Hello Rails');"
```

This will send the supplied string to the browser with a MIME type of `text/javascript`.

### Rendering Raw Body

You can send a raw content back to the browser, without setting any content
type, by using the `:body` option to `render`:

```ruby
render body: "raw"
```

TIP: This option should be used only if you don't care about the content type of
the response. Using `:plain` or `:html` might be more appropriate most of the
time.

NOTE: Unless overridden, your response returned from this render option will be
`text/plain`, as that is the default content type of Action Dispatch response.

### Rendering Raw File

Rails can render a raw file from an absolute path. This is useful for
conditionally rendering static files like error pages.

```ruby
render file: "#{Rails.root}/public/404.html", layout: false
```

This renders the raw file (it doesn't support ERB or other handlers). By
default it is rendered within the current layout.

WARNING: Using the `:file` option in combination with user input can lead to
security problems since an attacker could use this action to access security
sensitive files in your file system.

TIP: [`send_file`][] is often a faster and better option if a layout isn't required.

[`send_file`]: https://api.rubyonrails.org/v7.0.4.2/classes/ActionController/DataStreaming.html#method-i-send_file

### Rendering Objects

Rails can render objects responding to `#render_in`. The format can be controlled by defining `#format` on the object.

```ruby
class Greeting
  def render_in(view_context)
    view_context.render html: "Hello, World"
  end

  def format
    :html
  end
end

render Greeting.new
# => "Hello World"
```

This calls `render_in` on the provided object with the current view context. You can also provide the object by using the `:renderable` option to `render`:

```ruby
render renderable: Greeting.new
# => "Hello World"
```

### Options for `render`

Calls to the [`render`][controller.render] method generally accept six options:

* `:content_type`
* `:layout`
* `:location`
* `:status`
* `:formats`
* `:variants`

This section provides examples of using each of these options.

#### The `:content_type` Option

By default, Rails will serve the results of a rendering operation with the MIME content-type of `text/html` (or `application/json` if you use the `:json` option, or `application/xml` for the `:xml` option.). If you need to change the content-type, you can do so by setting the `:content_type` option:

```ruby
render template: "feed", content_type: "application/rss"
```

#### The `:layout` Option

With most of the options to `render`, the rendered content is displayed as part of the current layout. This guide covers more about [finding](#) and [using layouts](#) in the sections below.

If you need to use a layout other than the current layout Rails uses by default, you can use the `:layout` option to specify a file to use as the layout for the current action:

```ruby
render layout: "special_layout"
```

You can also tell Rails to render with no layout at all:

```ruby
render layout: false
```

#### The `:location` Option

You can use the `:location` option to set the HTTP `Location` header:

```ruby
render xml: photo, location: photo_url(photo)
```

#### The `:status` Option

Rails will automatically generate a response with the correct HTTP status code (in most cases, this is `200 OK`). You can use the `:status` option to change this:

```ruby
render status: 500
render status: :forbidden
```

Rails understands both numeric status codes and the corresponding symbols shown below.

| Response Class      | HTTP Status Code | Symbol                           |
| ------------------- | ---------------- | -------------------------------- |
| **Informational**   | 100              | :continue                        |
|                     | 101              | :switching_protocols             |
|                     | 102              | :processing                      |
| **Success**         | 200              | :ok                              |
|                     | 201              | :created                         |
|                     | 202              | :accepted                        |
|                     | 203              | :non_authoritative_information   |
|                     | 204              | :no_content                      |
|                     | 205              | :reset_content                   |
|                     | 206              | :partial_content                 |
|                     | 207              | :multi_status                    |
|                     | 208              | :already_reported                |
|                     | 226              | :im_used                         |
| **Redirection**     | 300              | :multiple_choices                |
|                     | 301              | :moved_permanently               |
|                     | 302              | :found                           |
|                     | 303              | :see_other                       |
|                     | 304              | :not_modified                    |
|                     | 305              | :use_proxy                       |
|                     | 307              | :temporary_redirect              |
|                     | 308              | :permanent_redirect              |
| **Client Error**    | 400              | :bad_request                     |
|                     | 401              | :unauthorized                    |
|                     | 402              | :payment_required                |
|                     | 403              | :forbidden                       |
|                     | 404              | :not_found                       |
|                     | 405              | :method_not_allowed              |
|                     | 406              | :not_acceptable                  |
|                     | 407              | :proxy_authentication_required   |
|                     | 408              | :request_timeout                 |
|                     | 409              | :conflict                        |
|                     | 410              | :gone                            |
|                     | 411              | :length_required                 |
|                     | 412              | :precondition_failed             |
|                     | 413              | :payload_too_large               |
|                     | 414              | :uri_too_long                    |
|                     | 415              | :unsupported_media_type          |
|                     | 416              | :range_not_satisfiable           |
|                     | 417              | :expectation_failed              |
|                     | 421              | :misdirected_request             |
|                     | 422              | :unprocessable_entity            |
|                     | 423              | :locked                          |
|                     | 424              | :failed_dependency               |
|                     | 426              | :upgrade_required                |
|                     | 428              | :precondition_required           |
|                     | 429              | :too_many_requests               |
|                     | 431              | :request_header_fields_too_large |
|                     | 451              | :unavailable_for_legal_reasons   |
| **Server Error**    | 500              | :internal_server_error           |
|                     | 501              | :not_implemented                 |
|                     | 502              | :bad_gateway                     |
|                     | 503              | :service_unavailable             |
|                     | 504              | :gateway_timeout                 |
|                     | 505              | :http_version_not_supported      |
|                     | 506              | :variant_also_negotiates         |
|                     | 507              | :insufficient_storage            |
|                     | 508              | :loop_detected                   |
|                     | 510              | :not_extended                    |
|                     | 511              | :network_authentication_required |

NOTE:  If you try to render content along with a non-content status code
(100-199, 204, 205, or 304), it will be dropped from the response.

#### The `:formats` Option

Rails uses the format specified in the request (or `:html` by default). You can
change this passing the `:formats` option with a symbol or an array:

```ruby
render formats: :xml
render formats: [:json, :xml]
```

If a template with the specified format does not exist an `ActionView::MissingTemplate` error is raised.

#### The `:variants` Option

The `:variants` options tells Rails to look for template variations of the same
format. You can specify a list of variants by passing the `:variants` option
with a symbol or an array. For example:

```ruby
# called in BookController#index
render variants: [:mobile, :desktop]
```

For the above variants Rails will look for the following set of templates and use the first that exists.

- `app/views/book/index.html+mobile.erb`
- `app/views/book/index.html+desktop.erb`
- `app/views/book/index.html.erb`

If a template with the specified format does not exist an `ActionView::MissingTemplate` error is raised.

Instead of setting the variant on the render call you may also set it on the request object in your controller action.

```ruby
def index
  request.variant = determine_variant
end

private
  def determine_variant
    variant = nil
    # some code to determine the variant(s) to use
    variant = :mobile if session[:use_mobile]

    variant
  end
```

Creating Responses using `redirect_to`
-------------------------------------

Another way to create an HTTP response from a controller action is to use the `redirect_to` method. This will send an HTTP redirect response back to the client, instructing the client to send a new request to a different URL.

The `redirect_to` method is typically used to redirect users to a different page after performing some action, such as after submitting a form or after completing a sign-in process.

For example, after creating a book we can `redirect_to` that book's `show` page:

```ruby
def create
  @book = Book.new(book_params)

  if @book.save
    redirect_to book_url(@book), notice: "Book was successfully created."
  else
    render :new, status: :unprocessable_entity
  end
end
```

If the `@book` was created successfully we use `redirect_to` to tell the client to make a new HTTP request to that book's show page URL (e.g.`books/42`). If you look at the "network" tab of a browser's dev tools, you will see this second request.

There is also a [`redirect_back`][] method that returns the user to the page
they just came from. The location is pulled from the `HTTP_REFERER` header.
Since this header is not guaranteed to be set, you must provide the
`fallback_location`.

```ruby
redirect_back(fallback_location: root_path)
```

NOTE: `redirect_to` and `redirect_back` do not immediately stop the execution of the controller method, they simply set HTTP responses. So statements *after* `redirect_to` in our controller get executed. To terminate the execution of the method immediately after the `redirect_to`, you need to use `return`. For example `redirect_to book_url(@book) and return`

[`redirect_back`]: https://api.rubyonrails.org/classes/ActionController/Redirecting.html#method-i-redirect_back

### Getting a Different Redirect Status Code

Rails uses HTTP status code 302, a temporary redirect, when you call `redirect_to`. If you'd like to set a different status code, perhaps 301, a permanent redirect, you can use the `:status` option:

```ruby
redirect_to books_path, status: 301
```

Just like the `:status` option for `render`, `:status` for `redirect_to` accepts both numeric and symbolic values.

The Difference Between `render` and `redirect_to`
------------------------------------------------

As we have seen above, the `redirect_to` method is used to send an HTTP redirect response instructing the client's browser to make a new request to a different URL. And the `render` method is used to render a view template from the current action. As we saw, `render` has many options that allow it to render views from different actions or even different controllers.

So what is the difference between `render` and `redirect_to`? Consider this example:

```ruby
# This code does not work as expected.
def index
  @books = Book.all
end

def show
  @book = Book.find_by(id: params[:id])
  if @book.nil?
    render action: "index"
  end
end
```

In the `show` action, if a book with a given `id` is not found, we attempt to show the user all the books by rendering the `index` action with `render action: "index"`. This does not make sense. Here's why:

The `render` call will find the `index.html.erb` file and attempt to render it. However, the `index` view expects a `@books` instance variable to be set. Since we are coming from the `show` action, while trying to render the `index` view, the `@books` instance variable will *not* be set. Note that `render action: "index"` does *not* execute the code in the `index` controller action before rendering the view, it just attempts to render the view specified.

This is the difference between `render` and `redirect_to`. The `redirect_to` method ends the current action, the browser makes a brand new HTTP request, which in turn, execute the appropriate controller action.

The above example updated to use `redirect_to` instead of `render`:

```ruby
# This code will work. But is not typically what we do.
def index
  @books = Book.all
end

def show
  @book = Book.find_by(id: params[:id])
  if @book.nil?
    redirect_to action: :index
  end
end
```

With this code, the browser will make a new request for the index page, the code in the `index` method will run, which will set the `@books` instance variable, then the `index.html.erb` view will be rendered. And it will all work as expected.

Note that the above example is for demonstration. It is not typical to render all books from a `show` action for a given book. If a resource is not found, it is reasonable to return a HTTP 404 Not Found status code when `@book.nil` is true.

### Avoiding Double Render Errors

A call to `render` method in a controller action does not stop execution of that action. Therefore, if you have multiple `render` statements in a code path, you will get the error message "Can only render or redirect once per action".

Here's an example that will trigger this error:

```ruby
def show
  @book = Book.find(params[:id])
  if @book.ebook?
    render action: "ebook_show"
  end
  render action: "regular_show"
end
```

If `@book.ebook?` is true, the `ebook_show` view will render. However, the code execution continues after the `if` and Rails will see the second `render` and throw the "double render" error at the `render action: "regular_show"` line.

A fix is simple: it's to ensure that there is only one `render` per code path. You *could* add a `return` after the first `render` to force the controller action to stop execution like this:

```ruby
# This is a possible fix but a better solution is below.
def show
  @book = Book.find(params[:id])
  if @book.ebook?
    render action: "ebook_show"
    return
  end
  render action: "regular_show"
end
```

But this is not as readable. A better solution is to have a single `render` per code path. Here is the updated `show` action:

```ruby
# A better solution for "double render" error.
def show
  @book = Book.find(params[:id])
  if @book.ebook?
    render action: "ebook_show"
  else
    render action: "regular_show"
  end
end
```

Note that the implicit `render` at the end of the controller action does not cause a "double render" error. Rails detects if a `render` has already been called for a controller action, so the following will work fine:

```ruby
def show
  @book = Book.find(params[:id])
  if @book.ebook?
    render action: "ebook_show"
  end
end
```

This will render the `ebook_show` view if `@book.ebook?` is true. Otherwise default rendering the `show` view at the end of the `show` action.

Building Header-Only Responses Using `head`
------------------------------------------

Rails controller actions can also use the [`head`][] method to send a response with HTTP headers only (no body) to the client.

The `head` method accepts an [HTTP status code](#the-status-option) as a number or symbol. And an options hash with header names and values.

For example, a header only response with `400 Bad Request`:

```ruby
head :bad_request
```

This would produce the following header:

```http
HTTP/1.1 400 Bad Request
Connection: close
Date: Sun, 24 Jan 2010 12:15:53 GMT
Transfer-Encoding: chunked
Content-Type: text/html; charset=utf-8
X-Runtime: 0.013483
Set-Cookie: _blog_session=...snip...; path=/; HttpOnly
Cache-Control: no-cache
```

Or a response with the HTTP `Location` header set:

```ruby
head :created, location: photo_path(@photo)
```

Which would produce:

```http
HTTP/1.1 201 Created
Connection: close
Date: Sun, 24 Jan 2010 12:16:44 GMT
Transfer-Encoding: chunked
Location: /photos/1
Content-Type: text/html; charset=utf-8
X-Runtime: 0.083496
Set-Cookie: _blog_session=...snip...; path=/; HttpOnly
Cache-Control: no-cache
```