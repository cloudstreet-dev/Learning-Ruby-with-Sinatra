# Chapter 3: Routes, Requests, and Responses

## HTTP: The Language of the Web

Before we go deeper into Sinatra, we need to talk about HTTP. Not because I enjoy torturing you with protocol specifications, but because understanding HTTP is the difference between a developer who can build things and a developer who understands what they're building.

Rails lets you ignore HTTP. Sinatra doesn't. This is a feature, not a bug.

## What Actually Happens When You Visit a Website

You type `http://example.com/posts/42` into your browser. What happens?

1. **DNS lookup**: Your browser asks "Where is example.com?" and gets an IP address
2. **TCP connection**: Your browser opens a connection to that IP address on port 80 (or 443 for HTTPS)
3. **HTTP request**: Your browser sends text over that connection:
   ```
   GET /posts/42 HTTP/1.1
   Host: example.com
   User-Agent: Mozilla/5.0...
   Accept: text/html,application/xhtml+xml...
   ```
4. **Server processing**: The web server (that's your Sinatra app!) reads this text, figures out what to do, and generates a response
5. **HTTP response**: The server sends back text:
   ```
   HTTP/1.1 200 OK
   Content-Type: text/html
   Content-Length: 1234

   <html>...</html>
   ```
6. **Rendering**: Your browser reads the response and renders it

The entire web is just computers sending text to each other. HTTP is the format of that text.

## Anatomy of an HTTP Request

Every HTTP request has:

### 1. A Method (Verb)
- `GET` - "Give me a resource"
- `POST` - "Here's data to create something"
- `PUT` - "Here's data to update something"
- `DELETE` - "Delete this resource"
- `PATCH` - "Here's data to partially update something"
- `HEAD` - "Just give me the headers, not the body"
- `OPTIONS` - "What methods are allowed here?"

### 2. A Path
The part after the domain: `/posts/42`

### 3. Headers
Metadata about the request:
```
Host: example.com
User-Agent: Mozilla/5.0...
Accept: text/html
Cookie: session=abc123
Content-Type: application/json
```

### 4. A Body (Optional)
For POST, PUT, PATCH requests—the actual data being sent.

## Anatomy of an HTTP Response

Every HTTP response has:

### 1. A Status Code
- `200` - OK (success)
- `201` - Created (resource successfully created)
- `301` - Moved Permanently (redirect forever)
- `302` - Found (redirect temporarily)
- `400` - Bad Request (client screwed up)
- `401` - Unauthorized (need authentication)
- `403` - Forbidden (authenticated but not allowed)
- `404` - Not Found (resource doesn't exist)
- `500` - Internal Server Error (server screwed up)
- `503` - Service Unavailable (server is down/overloaded)

### 2. Headers
Metadata about the response:
```
Content-Type: text/html
Content-Length: 1234
Set-Cookie: session=xyz789
Cache-Control: max-age=3600
```

### 3. A Body (Optional)
The actual content—HTML, JSON, an image, etc.

## How Sinatra Handles Requests

When a request comes in, Sinatra:

1. Matches the HTTP method
2. Matches the path pattern
3. Extracts parameters
4. Executes the route block
5. Sends the return value as the response body

Here's what's happening under the hood:

```ruby
get '/posts/:id' do
  # At this point, Sinatra has already:
  # - Verified this is a GET request
  # - Matched the path pattern
  # - Extracted "42" from "/posts/42" into params[:id]

  "You requested post #{params[:id]}"
end
```

## Route Patterns: Getting Fancy

### Named Parameters

We've seen these:

```ruby
get '/hello/:name' do
  "Hello, #{params[:name]}!"
end
```

You can have multiple:

```ruby
get '/posts/:year/:month/:day/:slug' do
  year = params[:year]
  month = params[:month]
  day = params[:day]
  slug = params[:slug]

  "Post from #{year}-#{month}-#{day}: #{slug}"
end
```

### Wildcard (Splat) Parameters

Want to match everything after a certain point?

```ruby
get '/files/*' do
  file_path = params[:splat].first
  "You requested: #{file_path}"
end
```

Visit `/files/documents/2024/report.pdf` and `params[:splat]` will be `["documents/2024/report.pdf"]`.

Multiple splats:

```ruby
get '/*/between/*' do
  first = params[:splat][0]
  second = params[:splat][1]
  "First: #{first}, Second: #{second}"
end
```

Visit `/foo/between/bar` → "First: foo, Second: bar"

### Regular Expressions

For when you need precise control:

```ruby
# Only match numeric IDs
get %r{/posts/(\d+)} do
  post_id = params[:captures].first
  "Post ID: #{post_id}"
end
```

```ruby
# Match email-like patterns
get %r{/users/([\w]+@[\w]+\.[\w]+)} do
  email = params[:captures].first
  "User email: #{email}"
end
```

Be careful with regex routes—they can get unreadable fast.

### Optional Parameters

Use the `?` operator:

```ruby
get '/posts/?:id?' do
  if params[:id]
    "Showing post #{params[:id]}"
  else
    "Showing all posts"
  end
end
```

This matches both `/posts` and `/posts/42`.

### Route Conditions

Match routes based on more than just the path:

```ruby
# Only match if User-Agent contains "Mobile"
get '/home', :agent => /Mobile/ do
  "Mobile version"
end

get '/home' do
  "Desktop version"
end
```

```ruby
# Custom condition: only serve to certain IPs
set(:ip) { |value| condition { request.ip == value } }

get '/admin', :ip => '127.0.0.1' do
  "Admin panel (localhost only)"
end
```

### Route Order Matters

Sinatra matches routes in the order they're defined:

```ruby
get '/posts/new' do
  "New post form"
end

get '/posts/:id' do
  "Post #{params[:id]}"
end
```

This works. But reverse the order:

```ruby
get '/posts/:id' do
  "Post #{params[:id]}"
end

get '/posts/new' do
  "New post form"
end
```

Now `/posts/new` will match the first route with `id = "new"`. The second route is unreachable.

**Rule**: Specific routes before general routes.

## The Params Hash

`params` contains all the data from the request:

### URL Parameters

```ruby
get '/hello/:name' do
  params[:name]  # "Alice" from /hello/Alice
end
```

### Query Strings

```ruby
get '/search' do
  query = params[:query]
  page = params[:page] || 1

  "Searching for '#{query}', page #{page}"
end
```

Visit `/search?query=sinatra&page=2`

### POST Data

```ruby
post '/login' do
  username = params[:username]
  password = params[:password]

  # Don't actually do this—we'll cover authentication properly later
  if username == 'admin' && password == 'secret'
    "Logged in!"
  else
    "Invalid credentials"
  end
end
```

All in the same `params` hash. Convenient? Yes. Secure? Not particularly. We'll handle security later.

## The Request Object

Need more information about the request? Use the `request` object:

```ruby
get '/debug' do
  info = <<~INFO
    Method: #{request.request_method}
    Path: #{request.path}
    Query: #{request.query_string}
    IP: #{request.ip}
    User-Agent: #{request.user_agent}
    Accept: #{request.accept.first}
    Secure (HTTPS): #{request.secure?}
    Body: #{request.body.read}
  INFO

  content_type :text
  info
end
```

Visit this route and you'll see everything about the request.

### Useful Request Methods

```ruby
request.get?        # true if GET request
request.post?       # true if POST request
request.xhr?        # true if AJAX request
request.secure?     # true if HTTPS
request.path        # "/posts/42"
request.url         # "http://example.com/posts/42"
request.ip          # Client's IP address
request.user_agent  # Browser/client identifier
request.cookies     # Cookie hash
```

## The Response Object

You can also manipulate the response directly:

```ruby
get '/custom' do
  response['X-Custom-Header'] = 'Hello!'
  response.status = 201
  response.body = ['Custom response body']
end
```

But usually you'll use the shortcuts:

```ruby
get '/normal' do
  status 201
  headers 'X-Custom-Header' => 'Hello!'
  body 'Custom response body'
end
```

Even simpler:

```ruby
get '/simple' do
  'Just return a string and Sinatra handles the rest'
end
```

## Redirects

Send users somewhere else:

```ruby
get '/old-page' do
  redirect '/new-page'
end
```

By default, this uses status code 302 (temporary redirect). For permanent redirects:

```ruby
get '/old-page' do
  redirect '/new-page', 301
end
```

Redirect to external sites:

```ruby
get '/google' do
  redirect 'https://google.com'
end
```

## Halting and Passing

### Halt: Stop Processing Immediately

```ruby
get '/restricted' do
  halt 401, 'Access denied' unless authorized?

  'Secret content'
end
```

`halt` stops execution and returns the response immediately.

You can halt with a status code, body, and headers:

```ruby
halt 403, {'Content-Type' => 'text/plain'}, 'Forbidden'
```

### Pass: Skip to the Next Route

```ruby
get '/maybe' do
  pass if params[:skip]
  'First route'
end

get '/maybe' do
  'Second route'
end
```

Visit `/maybe` → "First route"
Visit `/maybe?skip=true` → "Second route"

## Filters: Before and After

Run code before or after route handlers:

```ruby
before do
  puts "Incoming request: #{request.path}"
  @start_time = Time.now
end

after do
  duration = Time.now - @start_time
  puts "Request took #{duration} seconds"
end

get '/' do
  'Hello'
end
```

Filters with path patterns:

```ruby
before '/admin/*' do
  halt 401 unless logged_in?
end
```

## Content Types Revisited

Sinatra defaults to `text/html`. Change it:

```ruby
get '/text' do
  content_type 'text/plain'
  'Plain text'
end

get '/json' do
  content_type :json
  '{"message": "Hello, JSON"}'
end

get '/xml' do
  content_type :xml
  '<message>Hello, XML</message>'
end
```

Sinatra recognizes symbols for common types:
- `:html` → `text/html`
- `:json` → `application/json`
- `:xml` → `application/xml`
- `:txt` → `text/plain`
- `:css` → `text/css`
- `:js` → `application/javascript`

## Attachments and Downloads

Make the browser download instead of display:

```ruby
get '/download' do
  attachment 'report.pdf'
  # return file contents
end
```

Or just serve a file:

```ruby
get '/file' do
  send_file '/path/to/file.pdf'
end
```

## Streaming Responses

For large responses, stream them:

```ruby
get '/stream' do
  stream do |out|
    1000.times do |i|
      out << "Line #{i}\n"
      sleep 0.01  # Simulate slow generation
    end
  end
end
```

The browser will receive data as it's generated, not all at once.

## A Real-World Example

Let's build a simple URL shortener to tie it all together:

```ruby
require 'sinatra'
require 'sinatra/reloader' if development?

# In-memory storage (replace with a database later)
$urls = {}
$counter = 0

# Homepage
get '/' do
  'URL Shortener - POST to /shorten with url parameter'
end

# Shorten a URL
post '/shorten' do
  url = params[:url]

  halt 400, 'URL required' if url.nil? || url.empty?

  $counter += 1
  code = $counter.to_s(36)  # Base-36 encoding
  $urls[code] = url

  content_type :json
  { short_code: code, url: "http://localhost:4567/#{code}" }.to_json
end

# Redirect to original URL
get '/:code' do
  code = params[:code]
  url = $urls[code]

  halt 404, 'Short URL not found' if url.nil?

  redirect url
end

# Debug: see all URLs
get '/admin/list' do
  content_type :json
  $urls.to_json
end
```

Test it with curl:

```bash
# Shorten a URL
curl -X POST http://localhost:4567/shorten -d "url=https://github.com"

# Response: {"short_code":"1","url":"http://localhost:4567/1"}

# Visit the short URL
curl -L http://localhost:4567/1
# Redirects to GitHub

# See all URLs
curl http://localhost:4567/admin/list
```

This demonstrates:
- Different routes for different purposes
- HTTP methods (GET, POST)
- Parameter handling
- Error handling with `halt`
- Redirects
- Content types (JSON)
- Status codes

## What We've Learned

HTTP isn't scary. It's just text. Sinatra makes it obvious what's happening at every step:

- Routes map HTTP methods + paths to code
- `params` contains all request data
- `request` and `response` objects give you fine control
- Status codes, headers, and content types matter
- Redirects, halts, and filters give you flow control

You now understand the request/response cycle better than most Rails developers. And that's not a dig at Rails developers—it's just that Rails hides this from you.

## Next Up

We've been returning plain text and raw HTML strings. That's not sustainable.

In [Chapter 4: Views and Templates](./04-templates-and-views.md), we'll learn how to separate presentation from logic and build actual web pages.

---

*"HTTP is the assembly language of the web. Learn it, and everything else makes sense."*
