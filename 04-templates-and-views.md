# Chapter 4: Templates and Views

## The Problem with String Concatenation

Let's be honest: returning strings from routes is fine for APIs, but terrible for HTML:

```ruby
get '/users/:id' do
  user = find_user(params[:id])
  "<html><body><h1>#{user.name}</h1><p>#{user.email}</p></body></html>"
end
```

This is awful for many reasons:
- Hard to read
- Hard to maintain
- No syntax highlighting
- HTML escaping issues
- Designers can't help you
- You can't reuse layouts

There's a better way.

## Templates: Separating Logic from Presentation

Templates let you write HTML (or other formats) in separate files with placeholders for dynamic content.

Instead of:
```ruby
get '/' do
  "<h1>Hello, #{@name}!</h1>"
end
```

You write:
```ruby
get '/' do
  @name = "Alice"
  erb :index
end
```

And create `views/index.erb`:
```html
<h1>Hello, <%= @name %>!</h1>
```

Sinatra looks for templates in the `views/` directory by default.

## ERB: Embedded Ruby

ERB (Embedded Ruby) is Ruby's default template engine. It's built into Ruby, so no extra gems needed.

### Basic ERB Syntax

```erb
<!-- Output a value -->
<h1>Hello, <%= @name %>!</h1>

<!-- Execute code without output -->
<% if logged_in? %>
  <p>Welcome back!</p>
<% else %>
  <p>Please log in.</p>
<% end %>

<!-- Loops -->
<ul>
  <% @posts.each do |post| %>
    <li><%= post.title %></li>
  <% end %>
</ul>
```

Two tags:
- `<%= %>` - Execute Ruby and output the result
- `<% %>` - Execute Ruby without output

### Your First Template

Create `views/hello.erb`:
```erb
<!DOCTYPE html>
<html>
<head>
  <title>Hello</title>
</head>
<body>
  <h1>Hello, <%= @name %>!</h1>
  <p>The time is <%= Time.now.strftime('%I:%M %p') %></p>
</body>
</html>
```

Use it:
```ruby
require 'sinatra'

get '/hello/:name' do
  @name = params[:name]
  erb :hello
end
```

Visit `/hello/Alice` and you'll see a proper HTML page.

### Layouts: DRY Up Your HTML

Every page needs the same HTML boilerplate. Let's not repeat it.

Create `views/layout.erb`:
```erb
<!DOCTYPE html>
<html>
<head>
  <title><%= @title || "My Site" %></title>
  <link rel="stylesheet" href="/style.css">
</head>
<body>
  <header>
    <h1>My Sinatra Site</h1>
    <nav>
      <a href="/">Home</a>
      <a href="/about">About</a>
    </nav>
  </header>

  <main>
    <%= yield %>
  </main>

  <footer>
    <p>&copy; 2025 My Site</p>
  </footer>
</body>
</html>
```

The `<%= yield %>` is where your page content goes.

Now your templates can be simple:

`views/home.erb`:
```erb
<h2>Welcome Home</h2>
<p>This is the home page.</p>
```

`views/about.erb`:
```erb
<h2>About Us</h2>
<p>We build things with Sinatra.</p>
```

Routes:
```ruby
get '/' do
  erb :home
end

get '/about' do
  erb :about
end
```

Sinatra automatically uses `views/layout.erb` as the layout.

To disable the layout:
```ruby
get '/plain' do
  erb :plain, :layout => false
end
```

To use a different layout:
```ruby
get '/admin' do
  erb :admin_dashboard, :layout => :admin_layout
end
```

### Partials: Reusable Chunks

Want to reuse HTML snippets across pages?

Create `views/_post.erb`:
```erb
<article class="post">
  <h3><%= post.title %></h3>
  <p><%= post.excerpt %></p>
  <a href="/posts/<%= post.id %>">Read more</a>
</article>
```

Use it:
```erb
<% @posts.each do |post| %>
  <%= erb :_post, :locals => { post: post } %>
<% end %>
```

Convention: prefix partials with `_`.

### ERB in Your Routes (Inline Templates)

For quick prototypes, you can embed templates in your Ruby file:

```ruby
require 'sinatra'

get '/' do
  @name = "Alice"
  erb :index
end

__END__

@@index
<h1>Hello, <%= @name %>!</h1>
```

Everything after `__END__` is templates. Start each with `@@template_name`.

This is handy for single-file apps, but gets messy fast.

## HAML: The Elegant Alternative

ERB is fine, but it's still full of angle brackets. HAML (HTML Abstraction Markup Language) is cleaner:

Install it:
```bash
gem install haml
```

Use it:
```ruby
require 'sinatra'
require 'haml'

get '/' do
  @name = "Alice"
  haml :index
end
```

Create `views/index.haml`:
```haml
%h1 Hello, #{@name}!
%p The time is #{Time.now.strftime('%I:%M %p')}
```

### HAML Syntax

```haml
# Tags
%h1 Heading
%p Paragraph
%div.class#id Content

# CSS classes and IDs
%div.container
  %h1#title My Title
  %p.intro This is an intro

# Shortcuts for divs (div is assumed)
.container
  #title My Title
  .intro This is an intro

# Attributes
%a{href: '/about', title: 'About page'} About
%img{src: '/logo.png', alt: 'Logo'}

# Ruby interpolation
%h1= @title
%p= @description

# Ruby code (no output)
- if logged_in?
  %p Welcome back!
- else
  %p Please log in

# Loops
%ul
  - @items.each do |item|
    %li= item.name
```

A complete HAML layout (`views/layout.haml`):
```haml
!!!
%html
  %head
    %title= @title || "My Site"
    %link{rel: 'stylesheet', href: '/style.css'}
  %body
    %header
      %h1 My Sinatra Site
      %nav
        %a{href: '/'} Home
        %a{href: '/about'} About

    %main
      = yield

    %footer
      %p &copy; 2025 My Site
```

HAML is:
- More concise (less typing)
- Cleaner (no closing tags)
- Indentation-based (like Python)

But it's also:
- One more thing to learn
- Less familiar to non-Rubyists
- Harder to copy/paste HTML from the web

Pick your preference. I like HAML for personal projects, ERB for teams.

## Slim: The Minimalist

Want even less syntax? Try Slim:

```bash
gem install slim
```

```ruby
require 'sinatra'
require 'slim'

get '/' do
  slim :index
end
```

`views/index.slim`:
```slim
h1 Hello, #{@name}!
p The time is #{Time.now.strftime('%I:%M %p')}

- if @logged_in
  p Welcome back!
- else
  p Please log in

ul
  - @items.each do |item|
    li = item.name
```

Slim is like HAML without the `%`. It's fast and minimal. But it's also less common, so fewer tutorials and Stack Overflow answers.

## Template Helpers

Keep logic out of templates by defining helper methods:

```ruby
require 'sinatra'

helpers do
  def format_date(date)
    date.strftime('%B %d, %Y')
  end

  def logged_in?
    !session[:user_id].nil?
  end

  def current_user
    @current_user ||= User.find(session[:user_id]) if logged_in?
  end

  def markdown(text)
    # Render markdown to HTML (requires a markdown gem)
    RDiscount.new(text).to_html
  end
end

get '/posts/:id' do
  @post = Post.find(params[:id])
  erb :post
end
```

`views/post.erb`:
```erb
<article>
  <h1><%= @post.title %></h1>
  <time><%= format_date(@post.created_at) %></time>

  <% if logged_in? && current_user.can_edit?(@post) %>
    <a href="/posts/<%= @post.id %>/edit">Edit</a>
  <% end %>

  <div class="content">
    <%= markdown(@post.body) %>
  </div>
</article>
```

Helpers keep templates clean and logic reusable.

## Escaping HTML

By default, ERB/HAML/Slim escape HTML to prevent XSS attacks:

```ruby
get '/' do
  @user_input = "<script>alert('XSS')</script>"
  erb :index
end
```

```erb
<p><%= @user_input %></p>
```

This renders as:
```html
<p>&lt;script&gt;alert('XSS')&lt;/script&gt;</p>
```

The browser displays the script tag as text instead of executing it.

To output raw HTML (dangerous!):
```erb
<p><%== @trusted_html %></p>
```

Only do this with content you control.

## Static Files

Templates are for dynamic content. For static files (CSS, JS, images), use the `public/` directory:

```
public/
  style.css
  app.js
  logo.png
```

These are automatically served:
- `http://localhost:4567/style.css`
- `http://localhost:4567/app.js`
- `http://localhost:4567/logo.png`

In your templates:
```html
<link rel="stylesheet" href="/style.css">
<script src="/app.js"></script>
<img src="/logo.png" alt="Logo">
```

No route needed. Sinatra handles it.

## A Real Example: A Blog

Let's build a simple blog to tie everything together.

`app.rb`:
```ruby
require 'sinatra'
require 'sinatra/reloader' if development?

# Fake database (we'll use a real one later)
Post = Struct.new(:id, :title, :body, :created_at)

$posts = [
  Post.new(1, "First Post", "This is my first post.", Time.now - 86400),
  Post.new(2, "Second Post", "Another brilliant post.", Time.now - 3600),
  Post.new(3, "Third Post", "I'm on a roll.", Time.now)
]

helpers do
  def format_date(time)
    time.strftime('%B %d, %Y at %I:%M %p')
  end

  def truncate(text, length = 100)
    text.length > length ? text[0...length] + '...' : text
  end
end

get '/' do
  @posts = $posts.sort_by(&:created_at).reverse
  erb :index
end

get '/posts/:id' do
  @post = $posts.find { |p| p.id == params[:id].to_i }
  halt 404, erb(:not_found) unless @post
  erb :post
end

not_found do
  erb :not_found
end
```

`views/layout.erb`:
```erb
<!DOCTYPE html>
<html>
<head>
  <title><%= @title || "My Blog" %></title>
  <link rel="stylesheet" href="/style.css">
</head>
<body>
  <header>
    <h1><a href="/">My Blog</a></h1>
    <p>Thoughts on Sinatra and stuff</p>
  </header>

  <main>
    <%= yield %>
  </main>

  <footer>
    <p>&copy; 2025 | Built with Sinatra</p>
  </footer>
</body>
</html>
```

`views/index.erb`:
```erb
<h2>Recent Posts</h2>

<% @posts.each do |post| %>
  <article class="post-summary">
    <h3><a href="/posts/<%= post.id %>"><%= post.title %></a></h3>
    <time><%= format_date(post.created_at) %></time>
    <p><%= truncate(post.body) %></p>
    <a href="/posts/<%= post.id %>">Read more &rarr;</a>
  </article>
<% end %>
```

`views/post.erb`:
```erb
<article class="post">
  <h2><%= @post.title %></h2>
  <time><%= format_date(@post.created_at) %></time>
  <div class="post-body">
    <%= @post.body %>
  </div>
  <p><a href="/">&larr; Back to all posts</a></p>
</article>
```

`views/not_found.erb`:
```erb
<h2>404 - Not Found</h2>
<p>Sorry, that page doesn't exist.</p>
<p><a href="/">Go home</a></p>
```

`public/style.css`:
```css
body {
  font-family: Georgia, serif;
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  line-height: 1.6;
}

header {
  border-bottom: 2px solid #333;
  margin-bottom: 2rem;
}

header h1 {
  margin: 0;
  font-size: 2.5rem;
}

header h1 a {
  color: #333;
  text-decoration: none;
}

header p {
  color: #666;
  margin: 0.5rem 0 1rem 0;
}

.post-summary {
  border-bottom: 1px solid #ddd;
  padding-bottom: 1.5rem;
  margin-bottom: 1.5rem;
}

.post-summary h3 {
  margin: 0 0 0.5rem 0;
}

.post-summary time {
  color: #666;
  font-size: 0.9rem;
}

.post h2 {
  margin-bottom: 0.5rem;
}

.post time {
  color: #666;
  font-size: 0.9rem;
}

.post-body {
  margin: 2rem 0;
}

footer {
  border-top: 2px solid #333;
  margin-top: 3rem;
  padding-top: 1rem;
  text-align: center;
  color: #666;
}
```

Run it, and you have a functional blog. It's not fancy, but it works.

## What We've Learned

Templates separate presentation from logic:

- **ERB** is Ruby's default (familiar, angle-bracket-heavy)
- **HAML** is cleaner (indentation-based, no closing tags)
- **Slim** is minimal (fastest, least syntax)
- **Layouts** eliminate repetition
- **Partials** promote reuse
- **Helpers** keep logic out of templates
- **Static files** go in `public/`

You can now build actual web pages, not just return strings.

## Next Up

Our blog has posts, but they're hardcoded. Time to add a database.

In [Chapter 5: Working with Data](./05-data-and-databases.md), we'll learn how to persist data, work with databases, and build CRUD applications.

---

*"A template is worth a thousand string concatenations."*
