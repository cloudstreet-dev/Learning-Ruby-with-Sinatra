# Chapter 2: Hello World

## Five Minutes to a Web Server

Let's stop philosophizing and write some code.

By the end of this chapter, you'll have a working web application. Not a tutorial project that you'll throw away, but actual, deployable code that serves HTTP requests. And you'll understand every single line of it.

## Installation

First, install Sinatra:

```bash
gem install sinatra
```

That's it. No `rails new`, no generators, no configuration. Just a gem.

If you get a permission error, you might need to use `sudo` or set up rbenv/rvm. But you probably already know that if you're reading this book.

## Your First Web Server

Create a file called `app.rb`:

```ruby
require 'sinatra'

get '/' do
  'Hello, world!'
end
```

Run it:

```bash
ruby app.rb
```

You'll see:

```
[2025-11-18 10:30:45] INFO  WEBrick 1.8.1
[2025-11-18 10:30:45] INFO  ruby 3.2.0 (2022-12-25) [arm64-darwin23]
== Sinatra (v4.0.0) has taken the stage on 4567 for development
[2025-11-18 10:30:45] INFO  WEBrick::HTTPServer#start: pid=12345 port=4567
```

Open your browser to `http://localhost:4567`.

**Congratulations**. You're a web developer now.

## What Just Happened?

Let's break down those three lines:

```ruby
require 'sinatra'
```

This loads the Sinatra library. When Ruby loads Sinatra, it does something clever: it detects that you're in a "classic style" single-file app and automatically configures a web server for you.

```ruby
get '/' do
  'Hello, world!'
end
```

This defines a **route**. Specifically:
- `get` means "respond to HTTP GET requests"
- `'/'` is the path (the root of your website)
- The block (the `do...end` part) is what happens when someone visits that route
- The return value (`'Hello, world!'`) is sent back to the browser

When you run `ruby app.rb`, Sinatra:
1. Starts a web server (WEBrick by default)
2. Listens on port 4567
3. Waits for HTTP requests
4. Matches requests to your routes
5. Executes the matching route's block
6. Sends the return value back as the HTTP response

No magic. Just Ruby methods and blocks.

## Let's Get Fancy (But Not Too Fancy)

Let's add more routes:

```ruby
require 'sinatra'

get '/' do
  'Hello, world!'
end

get '/about' do
  'This is the about page!'
end

get '/hello/:name' do
  "Hello, #{params[:name]}!"
end

get '/time' do
  "The current time is #{Time.now}"
end
```

Restart your server (`Ctrl+C` to stop, `ruby app.rb` to start) and try:
- `http://localhost:4567/`
- `http://localhost:4567/about`
- `http://localhost:4567/hello/Alice`
- `http://localhost:4567/hello/Bob`
- `http://localhost:4567/time`

### Route Parameters

That `/hello/:name` route is interesting:

```ruby
get '/hello/:name' do
  "Hello, #{params[:name]}!"
end
```

The `:name` part is a **named parameter**. It's like a variable in your URL. When someone visits `/hello/Alice`, Sinatra:
1. Matches the route pattern
2. Captures "Alice" as the value of `:name`
3. Makes it available in `params[:name]`
4. Executes your block with that value

You can have multiple parameters:

```ruby
get '/hello/:first/:last' do
  "Hello, #{params[:first]} #{params[:last]}!"
end
```

Visit `/hello/Alice/Smith` and you'll get "Hello, Alice Smith!"

## HTTP Methods Matter

So far we've only used `get`. But the web has other HTTP methods, and they all mean something different:

```ruby
require 'sinatra'

# GET - Retrieve data
get '/posts' do
  'List of posts'
end

# POST - Create new data
post '/posts' do
  'Created a new post'
end

# PUT - Update existing data
put '/posts/:id' do
  "Updated post #{params[:id]}"
end

# DELETE - Delete data
delete '/posts/:id' do
  "Deleted post #{params[:id]}"
end
```

You can't easily test POST, PUT, and DELETE from a browser (browsers only send GET and POST by default), but we'll cover testing in a later chapter.

For now, just know: **different HTTP methods = different intentions**.

## Returning Different Content Types

By default, Sinatra returns HTML (technically `text/html`). But you can return anything:

```ruby
require 'sinatra'

# Return plain text
get '/text' do
  content_type 'text/plain'
  'This is plain text'
end

# Return JSON
get '/json' do
  content_type :json
  '{"message": "Hello, JSON!"}'
end

# Return HTML
get '/html' do
  content_type :html
  '<h1>Hello, HTML!</h1>'
end
```

The `content_type` method sets the `Content-Type` HTTP header, which tells the browser how to interpret the response.

## Status Codes

Every HTTP response has a status code. You've probably seen 404 (Not Found) and 500 (Internal Server Error). Sinatra defaults to 200 (OK), but you can set it explicitly:

```ruby
get '/success' do
  status 200
  'Everything is fine'
end

get '/created' do
  status 201
  'Resource created'
end

get '/unauthorized' do
  status 401
  'You shall not pass'
end

get '/not-found' do
  status 404
  'Nothing to see here'
end

get '/error' do
  status 500
  'Something went wrong'
end
```

These codes matter. They're how HTTP clients (browsers, curl, other apps) understand what happened.

## Handling 404s

What happens if you visit a route that doesn't exist? Try `http://localhost:4567/nonexistent`.

You'll get Sinatra's default 404 page. You can customize this:

```ruby
not_found do
  'Sorry, I have no idea what you're looking for.'
end
```

Now all 404s will show your custom message.

## Handling Errors

What if something goes wrong in your code?

```ruby
get '/boom' do
  1 / 0  # This will raise a ZeroDivisionError
end
```

Visit `/boom` and you'll see Sinatra's error page (in development mode). In production, users would see a generic error page. You can customize this:

```ruby
error do
  'Sorry, something went wrong: ' + env['sinatra.error'].message
end
```

## Auto-Reloading with Sinatra/Reloader

Tired of restarting your server every time you make a change?

Install the `sinatra-contrib` gem:

```bash
gem install sinatra-contrib
```

Update your app:

```ruby
require 'sinatra'
require 'sinatra/reloader' if development?

get '/' do
  'Hello, world!'
end
```

Now Sinatra will automatically reload your code when you change it. No more manual restarts.

## A Slightly Less Trivial Example

Let's build something marginally useful: a simple counter.

```ruby
require 'sinatra'
require 'sinatra/reloader' if development?

# In-memory counter (resets when server restarts)
$count = 0

get '/' do
  $count += 1
  "This page has been visited #{$count} times"
end

get '/reset' do
  $count = 0
  redirect '/'
end
```

This isn't production-ready (global variables are bad, mmkay?), but it demonstrates:
- Stateful applications
- The `redirect` method
- That Sinatra doesn't care how you manage state

Visit `/` a few times, then visit `/reset`.

## Configuration and Environments

Sinatra has three environments:
- `development` (default when running locally)
- `production` (what you'll use when deployed)
- `test` (for running tests)

You can configure behavior based on environment:

```ruby
require 'sinatra'

configure :development do
  require 'sinatra/reloader'
end

configure :production do
  set :show_exceptions, false
  set :dump_errors, false
end

get '/' do
  "Running in #{settings.environment} mode"
end
```

Run your app in different environments:

```bash
# Development (default)
ruby app.rb

# Production
RACK_ENV=production ruby app.rb

# Test
RACK_ENV=test ruby app.rb
```

## Modular vs. Classic Style

Everything we've written so far uses "classic" style—just define routes at the top level. This is great for small apps and learning.

For larger apps, you'll want "modular" style:

```ruby
require 'sinatra/base'

class MyApp < Sinatra::Base
  get '/' do
    'Hello from modular Sinatra!'
  end

  # In modular style, you must call run! explicitly
  run! if app_file == $0
end
```

The differences:
- **Classic**: Routes defined at top level, server starts automatically
- **Modular**: Routes defined in a class, more control, better for testing and mounting multiple apps

For now, stick with classic. We'll revisit modular style when we need it.

## What We've Learned

In less than 50 lines of code total, you've learned:
- How to install and run Sinatra
- What routes are and how they work
- How to handle URL parameters
- The different HTTP methods
- How to return different content types
- HTTP status codes
- Custom error handling
- Auto-reloading for development
- Environments and configuration

Not bad for Chapter 2.

## Exercise: Build Your Own

Before moving to the next chapter, build something yourself. Ideas:

1. **Personal landing page**: Multiple routes, links between them, show different information
2. **Magic 8-ball**: Visit `/ask/:question` and get a random answer
3. **Simple calculator**: Routes like `/add/5/3` or `/multiply/7/8`
4. **Motivational quotes**: Random quote on each page load

Don't overthink it. The goal is to get comfortable with routes, params, and returning responses.

## Next Up

You can build web applications now. They're not pretty (all that plain text), but they work.

In [Chapter 3: Routes, Requests, and Responses](./03-routing-and-http.md), we'll go deeper into HTTP, routing patterns, and how the request/response cycle actually works.

---

*"The best way to learn is to build something you'll actually use. The second best way is to build something silly and learn from it anyway."*
