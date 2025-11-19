# Chapter 6: Forms and User Input

## The Web is Read-Write

So far we've built read-only applications. Now let's make them interactive.

Forms are how users send data to your server. Understanding forms means understanding:
- How HTML forms work
- How browsers send data
- How to handle that data securely
- How to validate and sanitize user input

Rails form helpers hide this from you. Sinatra makes you write the HTML yourself. This is a good thing.

## A Basic HTML Form

The simplest form:

```html
<form method="POST" action="/posts">
  <input type="text" name="title" placeholder="Post title">
  <textarea name="body" placeholder="Post content"></textarea>
  <button type="submit">Create Post</button>
</form>
```

When submitted, the browser sends:
```
POST /posts HTTP/1.1
Content-Type: application/x-www-form-urlencoded

title=Hello+World&body=This+is+my+post
```

Sinatra parses this and makes it available in `params`:
```ruby
post '/posts' do
  params[:title]  # "Hello World"
  params[:body]   # "This is my post"
end
```

## Form Attributes

### Method

The HTTP method to use:
- `GET` - Data appears in URL, used for searches
- `POST` - Data in request body, used for creating/updating

```html
<!-- Search form (GET) -->
<form method="GET" action="/search">
  <input type="text" name="q">
  <button>Search</button>
</form>
```

Submitting sends: `GET /search?q=sinatra`

```html
<!-- Create form (POST) -->
<form method="POST" action="/posts">
  <input type="text" name="title">
  <button>Create</button>
</form>
```

Submitting sends: `POST /posts` with data in body.

### Action

Where to send the data:
```html
<form method="POST" action="/posts">  <!-- Goes to /posts -->
<form method="POST" action="/posts/42/comments">  <!-- Goes to /posts/42/comments -->
```

If omitted, submits to the current URL.

## Input Types

HTML5 gives us many input types:

```html
<form>
  <!-- Text inputs -->
  <input type="text" name="username">
  <input type="email" name="email">
  <input type="password" name="password">
  <input type="url" name="website">
  <input type="search" name="query">

  <!-- Numbers and dates -->
  <input type="number" name="age" min="0" max="120">
  <input type="range" name="rating" min="1" max="5">
  <input type="date" name="birthday">
  <input type="time" name="appointment">
  <input type="datetime-local" name="event_time">

  <!-- Choices -->
  <input type="checkbox" name="subscribe" value="yes">
  <input type="radio" name="size" value="small">
  <input type="radio" name="size" value="large">

  <!-- Files -->
  <input type="file" name="avatar">

  <!-- Multi-line text -->
  <textarea name="bio"></textarea>

  <!-- Dropdowns -->
  <select name="country">
    <option value="us">United States</option>
    <option value="uk">United Kingdom</option>
  </select>

  <!-- Buttons -->
  <button type="submit">Submit</button>
  <button type="reset">Reset</button>
  <button type="button">Just a button</button>
</form>
```

Modern browsers validate these automatically (e.g., email format, number ranges). But always validate on the server too.

## Handling Form Submissions

Let's build a complete "new post" form:

`app.rb`:
```ruby
require 'sinatra'
require 'sinatra/activerecord'

set :database, {adapter: 'sqlite3', database: 'blog.db'}

class Post < ActiveRecord::Base
  validates :title, presence: true, length: { minimum: 3 }
  validates :body, presence: true
end

# Show form
get '/posts/new' do
  @post = Post.new
  erb :new_post
end

# Handle submission
post '/posts' do
  @post = Post.new(
    title: params[:title],
    body: params[:body]
  )

  if @post.save
    redirect "/posts/#{@post.id}"
  else
    # Validation failed, show form again with errors
    erb :new_post
  end
end
```

`views/new_post.erb`:
```erb
<h1>New Post</h1>

<% if @post.errors.any? %>
  <div class="errors">
    <h3>Please fix these errors:</h3>
    <ul>
      <% @post.errors.full_messages.each do |msg| %>
        <li><%= msg %></li>
      <% end %>
    </ul>
  </div>
<% end %>

<form method="POST" action="/posts">
  <div>
    <label for="title">Title:</label>
    <input
      type="text"
      id="title"
      name="title"
      value="<%= @post.title %>"
      required
    >
  </div>

  <div>
    <label for="body">Body:</label>
    <textarea
      id="body"
      name="body"
      required
    ><%= @post.body %></textarea>
  </div>

  <button type="submit">Create Post</button>
</form>
```

Notice we pre-fill the form with `@post.title` and `@post.body`. This preserves user input when validation fails.

## PUT and DELETE: The Browser Limitation

HTML forms only support GET and POST. But REST uses PUT for updates and DELETE for deletions.

The workaround: use POST with a hidden field.

### Method Override

Enable it:
```ruby
require 'sinatra'
enable :method_override
```

Now you can fake PUT and DELETE:

```html
<!-- Edit form -->
<form method="POST" action="/posts/42">
  <input type="hidden" name="_method" value="PUT">
  <input type="text" name="title" value="Existing title">
  <button>Update</button>
</form>

<!-- Delete form -->
<form method="POST" action="/posts/42">
  <input type="hidden" name="_method" value="DELETE">
  <button>Delete</button>
</form>
```

Handle it:
```ruby
# Update
put '/posts/:id' do
  post = Post.find(params[:id])
  post.update(title: params[:title])
  redirect "/posts/#{post.id}"
end

# Delete
delete '/posts/:id' do
  Post.find(params[:id]).destroy
  redirect '/'
end
```

Sinatra sees `_method=PUT` and routes to the `put` handler.

## Nested Parameters

Want to organize params better?

```html
<form method="POST" action="/posts">
  <input type="text" name="post[title]">
  <input type="text" name="post[body]">
  <input type="text" name="post[author]">
  <button>Create</button>
</form>
```

Sinatra parses this as:
```ruby
params[:post]  # { "title" => "...", "body" => "...", "author" => "..." }
```

Use it:
```ruby
post '/posts' do
  @post = Post.new(params[:post])
  @post.save
  redirect '/'
end
```

This is cleaner and safer (we'll see why in a moment).

## Mass Assignment Protection

**Dangerous**:
```ruby
post '/posts' do
  Post.create(params[:post])
end
```

What if a malicious user submits:
```
post[title]=Hello&post[admin]=true
```

If your `Post` model has an `admin` column, you just made someone an admin.

**Solution**: Only permit specific attributes.

### ActiveRecord: Strong Parameters

Borrow from Rails:
```ruby
require 'sinatra'
require 'sinatra/activerecord'

helpers do
  def post_params
    params.fetch(:post, {}).slice('title', 'body', 'author')
  end
end

post '/posts' do
  @post = Post.new(post_params)
  @post.save
  redirect '/'
end
```

Or use a gem:
```bash
gem install sinatra-param
```

```ruby
require 'sinatra/param'

post '/posts' do
  param :title, String, required: true
  param :body,  String, required: true

  Post.create(title: params[:title], body: params[:body])
end
```

## Checkboxes and Multi-Value Inputs

### Single Checkbox

```html
<input type="checkbox" name="published" value="true">
```

When checked: `params[:published] = "true"`
When unchecked: `params[:published] = nil`

Handle it:
```ruby
post '/posts' do
  published = params[:published] == 'true'
  Post.create(title: params[:title], published: published)
end
```

### Multiple Checkboxes

```html
<input type="checkbox" name="tags[]" value="ruby">
<input type="checkbox" name="tags[]" value="sinatra">
<input type="checkbox" name="tags[]" value="web">
```

Checked boxes: `params[:tags] = ["ruby", "sinatra"]`

Handle it:
```ruby
post '/posts' do
  tags = params[:tags] || []
  Post.create(title: params[:title], tags: tags.join(','))
end
```

### Select with Multiple

```html
<select name="categories[]" multiple>
  <option value="tech">Tech</option>
  <option value="design">Design</option>
  <option value="business">Business</option>
</select>
```

Same as multiple checkboxes: `params[:categories] = ["tech", "design"]`

## File Uploads

Enable file uploads:

```html
<form method="POST" action="/upload" enctype="multipart/form-data">
  <input type="file" name="avatar">
  <button>Upload</button>
</form>
```

**Important**: `enctype="multipart/form-data"` is required for file uploads.

Handle it:
```ruby
post '/upload' do
  if params[:avatar]
    file = params[:avatar][:tempfile]
    filename = params[:avatar][:filename]

    # Save file
    File.open("./public/uploads/#{filename}", 'wb') do |f|
      f.write(file.read)
    end

    "Uploaded #{filename}"
  else
    "No file uploaded"
  end
end
```

`params[:avatar]` is a hash:
```ruby
{
  :filename => "photo.jpg",
  :type => "image/jpeg",
  :tempfile => #<Tempfile>
}
```

### Secure File Uploads

**Never trust filenames**. Users can upload `../../etc/passwd`.

Safe version:
```ruby
require 'securerandom'

post '/upload' do
  halt 400, 'No file' unless params[:avatar]

  file = params[:avatar][:tempfile]
  filename = params[:avatar][:filename]

  # Validate file type
  halt 400, 'Only images allowed' unless filename =~ /\.(jpg|jpeg|png|gif)$/i

  # Generate safe filename
  ext = File.extname(filename)
  safe_filename = "#{SecureRandom.uuid}#{ext}"

  # Save to uploads directory
  path = "./public/uploads/#{safe_filename}"
  File.open(path, 'wb') { |f| f.write(file.read) }

  "Uploaded as #{safe_filename}"
end
```

### Validate File Size

```ruby
post '/upload' do
  file = params[:avatar][:tempfile]

  # Max 5MB
  halt 400, 'File too large' if file.size > 5 * 1024 * 1024

  # ... save file
end
```

## CSRF Protection

**Cross-Site Request Forgery** is when an attacker tricks a user into submitting a form.

Example attack:
1. User logs into yoursite.com
2. User visits evilsite.com
3. evilsite.com has hidden form: `<form action="https://yoursite.com/delete-account" method="POST">`
4. Form auto-submits via JavaScript
5. User's account is deleted

### Rack::Protection

Sinatra includes CSRF protection by default (via Rack::Protection):

```ruby
require 'sinatra'
enable :sessions  # Required for CSRF protection

post '/delete-account' do
  # Protected automatically
end
```

For forms, include the CSRF token:

```erb
<form method="POST" action="/posts">
  <input type="hidden" name="authenticity_token" value="<%= csrf_token %>">
  <!-- form fields -->
</form>
```

Or use a helper:
```ruby
helpers do
  def csrf_tag
    '<input type="hidden" name="authenticity_token" value="' + csrf_token + '">'
  end
end
```

```erb
<form method="POST" action="/posts">
  <%== csrf_tag %>
  <!-- form fields -->
</form>
```

## Input Validation and Sanitization

Never trust user input.

### Validation

Check data is correct:

```ruby
post '/posts' do
  title = params[:title].to_s.strip
  body = params[:body].to_s.strip

  errors = []
  errors << "Title is required" if title.empty?
  errors << "Title too short" if title.length < 3
  errors << "Body is required" if body.empty?

  if errors.any?
    halt 400, errors.join(', ')
  end

  Post.create(title: title, body: body)
  redirect '/'
end
```

Better: use model validations (ActiveRecord does this).

### Sanitization

Remove dangerous content:

```ruby
require 'sanitize'

post '/posts' do
  title = Sanitize.fragment(params[:title])
  body = Sanitize.fragment(params[:body], Sanitize::Config::RELAXED)

  Post.create(title: title, body: body)
end
```

Or escape HTML:
```ruby
require 'rack/utils'

post '/posts' do
  title = Rack::Utils.escape_html(params[:title])
  Post.create(title: title)
end
```

ERB templates auto-escape by default (when using `<%= %>`), so usually you're safe.

## Flash Messages

Show messages after redirects:

```bash
gem install sinatra-flash
```

```ruby
require 'sinatra'
require 'sinatra/flash'

enable :sessions

post '/posts' do
  @post = Post.new(params[:post])

  if @post.save
    flash[:success] = 'Post created!'
    redirect "/posts/#{@post.id}"
  else
    flash.now[:error] = 'Failed to create post'
    erb :new_post
  end
end
```

In layout:
```erb
<% if flash[:success] %>
  <div class="alert success"><%= flash[:success] %></div>
<% end %>

<% if flash[:error] %>
  <div class="alert error"><%= flash[:error] %></div>
<% end %>
```

## A Complete CRUD Example

Putting it all together:

`app.rb`:
```ruby
require 'sinatra'
require 'sinatra/activerecord'
require 'sinatra/flash'
require 'sinatra/reloader' if development?

enable :sessions
enable :method_override

set :database, {adapter: 'sqlite3', database: 'blog.db'}

class Post < ActiveRecord::Base
  validates :title, presence: true, length: { minimum: 3, maximum: 100 }
  validates :body, presence: true
end

helpers do
  def post_params
    params.fetch(:post, {}).slice('title', 'body')
  end

  def csrf_tag
    '<input type="hidden" name="authenticity_token" value="' + csrf_token + '">'
  end
end

# Index
get '/' do
  @posts = Post.order(created_at: :desc)
  erb :index
end

# New
get '/posts/new' do
  @post = Post.new
  erb :new_post
end

# Create
post '/posts' do
  @post = Post.new(post_params)

  if @post.save
    flash[:success] = 'Post created successfully!'
    redirect "/posts/#{@post.id}"
  else
    flash.now[:error] = 'Failed to create post'
    erb :new_post
  end
end

# Show
get '/posts/:id' do
  @post = Post.find_by(id: params[:id])
  halt 404 unless @post
  erb :show_post
end

# Edit
get '/posts/:id/edit' do
  @post = Post.find(params[:id])
  erb :edit_post
end

# Update
put '/posts/:id' do
  @post = Post.find(params[:id])

  if @post.update(post_params)
    flash[:success] = 'Post updated successfully!'
    redirect "/posts/#{@post.id}"
  else
    flash.now[:error] = 'Failed to update post'
    erb :edit_post
  end
end

# Delete
delete '/posts/:id' do
  Post.find(params[:id]).destroy
  flash[:success] = 'Post deleted'
  redirect '/'
end
```

`views/new_post.erb`:
```erb
<h1>New Post</h1>
<%= erb :_post_form %>
```

`views/edit_post.erb`:
```erb
<h1>Edit Post</h1>
<%= erb :_post_form %>
```

`views/_post_form.erb`:
```erb
<% if @post.errors.any? %>
  <div class="errors">
    <ul>
      <% @post.errors.full_messages.each do |msg| %>
        <li><%= msg %></li>
      <% end %>
    </ul>
  </div>
<% end %>

<form method="POST" action="<%= @post.new_record? ? '/posts' : "/posts/#{@post.id}" %>">
  <%== csrf_tag %>

  <% unless @post.new_record? %>
    <input type="hidden" name="_method" value="PUT">
  <% end %>

  <div>
    <label for="title">Title:</label>
    <input
      type="text"
      id="title"
      name="post[title]"
      value="<%= @post.title %>"
      required
    >
  </div>

  <div>
    <label for="body">Body:</label>
    <textarea
      id="body"
      name="post[body]"
      required
    ><%= @post.body %></textarea>
  </div>

  <button type="submit"><%= @post.new_record? ? 'Create' : 'Update' %> Post</button>
</form>
```

## What We've Learned

Forms are the interface between users and your data:

- HTML forms are simple but powerful
- Always validate on the server (never trust the browser)
- Protect against mass assignment
- Sanitize user input
- Prevent CSRF attacks
- Use flash messages for feedback
- PUT and DELETE need method override

Rails generates forms for you. Sinatra makes you write them. By the end, you understand forms better than most Rails developers.

## Next Up

Users can create posts. But can they edit *other people's* posts? We need authentication.

In [Chapter 7: Sessions, Cookies, and Authentication](./07-sessions-and-auth.md), we'll learn how to identify users, keep them logged in, and control access.

---

*"A website without forms is a museum. With forms, it's a conversation. With proper validation, it's a civil conversation."*
