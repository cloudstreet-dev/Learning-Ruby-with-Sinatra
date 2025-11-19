# Chapter 1: Why Not Rails?

## The Rails Hegemony

Let's start with an uncomfortable truth: Ruby on Rails is one of the best things that ever happened to web development.

It pioneered MVC for the web. It championed convention over configuration. It made database migrations sane. It brought joy to developers drowning in Java EE. DHH literally changed how we build web applications.

But here's the thing about revolutions: they tend to replace one orthodoxy with another.

## The Problem with "The Rails Way"

Go to any Ruby job posting. 95% of them say "Ruby on Rails." Ask a newcomer how to build a web app in Ruby, and they'll answer "Rails." It's like asking how to get around a city and only being told about the subway system. Sure, it's great for many trips, but sometimes you just need to walk two blocks.

Rails has become such the default that we've forgotten to ask: **Do I actually need Rails for this?**

## When Rails is Overkill

Let's say you need to build:
- A webhook receiver for Stripe
- A simple API for a mobile app
- A microservice that processes images
- A status page that checks if services are up
- An internal tool with five routes

Do you really need:
- ActiveRecord with migrations?
- ActionMailer?
- ActiveJob?
- The asset pipeline?
- ActionCable?
- ActiveStorage?
- A 15-second boot time?

Probably not.

## The Learning Problem

Here's a conversation that happens daily:

**Junior Developer**: "I want to learn web development with Ruby."

**Senior Developer**: "Great! Learn Rails."

**Junior Developer**: "What's a route?"

**Senior Developer**: "It's in the `config/routes.rb` file. Just use `resources :posts` and you get seven routes automatically."

**Junior Developer**: "But what *is* a route?"

**Senior Developer**: "It maps URLs to controller actions."

**Junior Developer**: "What's a controller action?"

**Senior Developer**: "It's a method that Rails calls when a route is matched."

**Junior Developer**: "But how does Rails know when a route is matched?"

**Senior Developer**: "That's... uh... Rack middleware and... you know what, just trust the magic."

And that's how we create developers who can build apps but don't understand what they're building.

## Sinatra: Web Development Without Training Wheels

Sinatra is the opposite of Rails in the best possible way.

Here's a complete Sinatra application:

```ruby
require 'sinatra'

get '/' do
  'Hello, world!'
end
```

That's it. That's a web server. Run it with `ruby app.rb` and you've got a functioning web application.

Let's break down what this does:
1. Load the Sinatra library
2. When a GET request comes in for '/', return 'Hello, world!'

No generators. No scaffolding. No magic. Just code that does exactly what it says.

## What You Actually Learn

When you learn Sinatra, you're forced to understand:

### HTTP Fundamentals
Rails hides HTTP from you. Sinatra puts it front and center.

```ruby
get '/posts' do
  # Handles GET requests
end

post '/posts' do
  # Handles POST requests
end

put '/posts/:id' do
  # Handles PUT requests
end

delete '/posts/:id' do
  # Handles DELETE requests
end
```

Every route explicitly declares its HTTP method. You can't help but learn what GET, POST, PUT, and DELETE actually mean.

### The Request/Response Cycle
In Rails, you set instance variables in controllers and they magically appear in views. In Sinatra, you explicitly return responses:

```ruby
get '/hello/:name' do
  name = params[:name]
  "Hello, #{name}!"
end
```

Want to return JSON? You return JSON:

```ruby
get '/api/posts' do
  content_type :json
  { posts: Post.all }.to_json
end
```

Want to redirect? You redirect:

```ruby
post '/login' do
  if authenticated?
    redirect '/dashboard'
  else
    status 401
    "Authentication failed"
  end
end
```

Everything is explicit. You always know what's being sent back to the client.

### Middleware and Rack
Rails uses Rack, but you never see it. Sinatra *is* a Rack application. You'll learn what middleware is and how the entire Ruby web ecosystem is built on Rack.

```ruby
use Rack::Logger
use Rack::Session::Cookie, secret: 'my_secret'

get '/' do
  session[:count] ||= 0
  session[:count] += 1
  "You've visited #{session[:count]} times"
end
```

### Routing as a Concept
In Rails, routes are DSL magic in a config file. In Sinatra, routes are methods:

```ruby
get '/posts/:id' do
  @post = Post.find(params[:id])
  erb :show
end
```

This is a route. It's a method. When a GET request matches `/posts/:id`, this code runs. That's it. No magic.

## The Performance Argument

Let's talk cold hard numbers.

**Rails boot time**: 10-20 seconds (or more)
**Sinatra boot time**: < 1 second

**Rails memory footprint**: 100-200+ MB
**Sinatra memory footprint**: 10-20 MB

**Rails dependencies**: 50+ gems
**Sinatra dependencies**: 3 gems (rack, rack-protection, tilt)

For many applications, this matters. A lot.

Microservices? You can run 10 Sinatra apps in the memory of one Rails app.
Development? You can restart your Sinatra app before you finish typing `:wq` in Vim.
Testing? Your entire test suite can run faster than Rails boots.

## The "But Rails Scales" Argument

Here's a dirty secret: GitHub used to run on Rails. Twitter started on Rails. Shopify runs on Rails.

Know what they all have in common? They spent massive engineering effort fighting Rails' performance characteristics.

And here's another secret: Many of those companies are replacing parts of their Rails monoliths with... small, focused Sinatra apps (or similar microframeworks).

Sinatra scales. It scales easily. Because it does less, there's less to scale.

## When You Should Use Rails

Let's be fair: Rails is great for many things.

**Use Rails when you**:
- Need a full CRUD app with an admin panel in 24 hours
- Have a team that knows Rails inside and out
- Need mature solutions for common problems (background jobs, emails, file uploads)
- Want strong conventions that guide architectural decisions
- Are building a monolithic web application with complex domain models

Rails is a productivity multiplier for the right projects.

**Use Sinatra when you**:
- Need an API without the ceremony
- Want to understand web development fundamentals
- Value simplicity and explicitness over convention
- Need fast boot times and low memory usage
- Are building microservices
- Want to pick your own libraries for specific problems

## The Sinatra Advantage: Freedom

Rails is opinionated. That's its strength. It's also its weakness.

Sinatra has opinions too, but they're different:
- HTTP matters
- Explicit is better than implicit
- You should understand what your code does
- Pick the best tool for each job
- Small is beautiful

Rails gives you everything. Sinatra gives you nothing—and that's a gift.

With Sinatra, you choose:
- Your database library (Sequel? ActiveRecord? ROM? Raw SQL?)
- Your template engine (ERB? HAML? Slim? Liquid?)
- Your testing framework (RSpec? Minitest? Test::Unit?)
- Your deployment strategy
- Your asset management
- Your job queue
- Everything else

This sounds like more work. It is. But it's also educational, and often results in a better, leaner application.

## The Path Forward

This book will teach you Sinatra. Not as a Rails alternative, but as a gateway to understanding web development.

You'll learn:
- How HTTP actually works
- What REST really means
- How to structure applications without framework magic
- When to add libraries vs. writing your own code
- How to make architectural decisions

By the end, you'll be able to build production-ready web applications. You'll also understand what Rails is doing for you—making you a better developer regardless of which framework you choose.

## The Promise

Learn Sinatra, and you'll understand web development.

Learn Rails first, and you'll understand Rails.

One of these paths creates a better developer. I'll let you guess which one.

## Next Steps

Ready to write some code?

[Chapter 2: Hello World](./02-hello-world.md) awaits.

---

*"Perfection is achieved, not when there is nothing more to add, but when there is nothing left to take away."* — Antoine de Saint-Exupéry

Sinatra took that seriously.
