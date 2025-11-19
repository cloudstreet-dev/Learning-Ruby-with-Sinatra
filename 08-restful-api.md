# Chapter 8: Building a RESTful API

## APIs: The Invisible Web

Most modern applications are actually two applications:
1. The frontend (browser, mobile app, CLI)
2. The backend API (data and logic)

Your Sinatra app can be both. Or just the API. Or consume other APIs.

This chapter is about building JSON APIs that mobile apps, JavaScript frontends, and other services can consume.

## What is REST?

**REST** (Representational State Transfer) is an architectural style for APIs. It uses:
- HTTP methods to indicate actions
- URLs to identify resources
- Status codes to indicate results
- Standard formats (usually JSON) for data

### RESTful Routes

For a `posts` resource:

| Method | Path | Action | Description |
|--------|------|--------|-------------|
| GET | /posts | index | List all posts |
| GET | /posts/:id | show | Get one post |
| POST | /posts | create | Create post |
| PUT | /posts/:id | update | Update entire post |
| PATCH | /posts/:id | update | Update parts of post |
| DELETE | /posts/:id | destroy | Delete post |

This is REST. Same routes, different representations (HTML vs JSON).

## Your First API Endpoint

```ruby
require 'sinatra'
require 'sinatra/activerecord'
require 'json'

set :database, {adapter: 'sqlite3', database: 'blog.db'}

class Post < ActiveRecord::Base
end

# API: List all posts
get '/api/posts' do
  content_type :json

  posts = Post.all
  posts.to_json
end

# API: Get one post
get '/api/posts/:id' do
  content_type :json

  post = Post.find_by(id: params[:id])
  halt 404, {error: 'Not found'}.to_json unless post

  post.to_json
end
```

Test it:
```bash
curl http://localhost:4567/api/posts
curl http://localhost:4567/api/posts/1
```

## Proper JSON Responses

### Set Content-Type

Always:
```ruby
content_type :json
```

Or use a before filter:
```ruby
before '/api/*' do
  content_type :json
end
```

### Return Proper Status Codes

```ruby
get '/api/posts/:id' do
  post = Post.find_by(id: params[:id])

  if post
    status 200
    post.to_json
  else
    status 404
    {error: 'Post not found'}.to_json
  end
end
```

Common codes:
- **200 OK** - Success
- **201 Created** - Resource created
- **204 No Content** - Success but no body (often for DELETE)
- **400 Bad Request** - Invalid data
- **401 Unauthorized** - Need authentication
- **403 Forbidden** - Authenticated but not allowed
- **404 Not Found** - Resource doesn't exist
- **422 Unprocessable Entity** - Validation failed
- **500 Internal Server Error** - Server error

### Structure Error Responses

Be consistent:
```ruby
def json_error(message, status = 400)
  halt status, {
    error: message,
    timestamp: Time.now.iso8601
  }.to_json
end

post '/api/posts' do
  post = Post.new(params)

  if post.save
    status 201
    post.to_json
  else
    json_error(post.errors.full_messages.join(', '), 422)
  end
end
```

## A Complete RESTful API

```ruby
require 'sinatra'
require 'sinatra/activerecord'
require 'sinatra/reloader' if development?
require 'json'

set :database, {adapter: 'sqlite3', database: 'blog.db'}

class Post < ActiveRecord::Base
  validates :title, presence: true
  validates :body, presence: true
end

# Helpers
helpers do
  def json_params
    JSON.parse(request.body.read)
  rescue JSON::ParserError
    halt 400, {error: 'Invalid JSON'}.to_json
  end

  def json_error(message, status = 400)
    halt status, {error: message}.to_json
  end
end

# Set JSON content type for all API routes
before '/api/*' do
  content_type :json
end

# Index - GET /api/posts
get '/api/posts' do
  posts = Post.order(created_at: :desc)

  # Pagination
  page = (params[:page] || 1).to_i
  per_page = (params[:per_page] || 20).to_i
  posts = posts.limit(per_page).offset((page - 1) * per_page)

  {
    posts: posts,
    page: page,
    per_page: per_page,
    total: Post.count
  }.to_json
end

# Show - GET /api/posts/:id
get '/api/posts/:id' do
  post = Post.find_by(id: params[:id])
  json_error('Post not found', 404) unless post

  post.to_json
end

# Create - POST /api/posts
post '/api/posts' do
  data = json_params
  post = Post.new(data)

  if post.save
    status 201
    post.to_json
  else
    json_error(post.errors.full_messages.join(', '), 422)
  end
end

# Update - PUT/PATCH /api/posts/:id
['/api/posts/:id'].each do |path|
  put path do
    post = Post.find_by(id: params[:id])
    json_error('Post not found', 404) unless post

    data = json_params
    if post.update(data)
      post.to_json
    else
      json_error(post.errors.full_messages.join(', '), 422)
    end
  end

  patch path do
    # Same as PUT for simple cases
    pass
  end
end

# Delete - DELETE /api/posts/:id
delete '/api/posts/:id' do
  post = Post.find_by(id: params[:id])
  json_error('Post not found', 404) unless post

  post.destroy
  status 204  # No content
  nil
end
```

Test it:
```bash
# List posts
curl http://localhost:4567/api/posts

# Get one post
curl http://localhost:4567/api/posts/1

# Create post
curl -X POST http://localhost:4567/api/posts \
  -H "Content-Type: application/json" \
  -d '{"title":"API Post","body":"Created via API"}'

# Update post
curl -X PUT http://localhost:4567/api/posts/1 \
  -H "Content-Type: application/json" \
  -d '{"title":"Updated Title"}'

# Delete post
curl -X DELETE http://localhost:4567/api/posts/1
```

## Custom JSON Serialization

Don't expose everything:

```ruby
class Post < ActiveRecord::Base
  def as_json(options = {})
    {
      id: id,
      title: title,
      body: body,
      author: author&.name,
      created_at: created_at.iso8601,
      url: "/posts/#{id}"
    }
  end
end
```

Or use a gem:
```bash
gem install active_model_serializers
```

```ruby
class PostSerializer < ActiveModel::Serializer
  attributes :id, :title, :body, :created_at
  belongs_to :author

  def created_at
    object.created_at.iso8601
  end
end

get '/api/posts/:id' do
  post = Post.find(params[:id])
  PostSerializer.new(post).to_json
end
```

## Versioning Your API

APIs should be versioned so you can make breaking changes:

### URL Versioning (Simple)

```ruby
namespace '/api/v1' do
  get '/posts' do
    # v1 implementation
  end
end

namespace '/api/v2' do
  get '/posts' do
    # v2 implementation (different format)
  end
end
```

Requires `sinatra-contrib`:
```bash
gem install sinatra-contrib
```

```ruby
require 'sinatra/namespace'
```

### Header Versioning (RESTful)

```ruby
before '/api/*' do
  @api_version = request.env['HTTP_ACCEPT']&.match(/version=(\d+)/)[1] || '1'
end

get '/api/posts' do
  case @api_version
  when '1'
    # v1 format
  when '2'
    # v2 format
  end
end
```

Request:
```bash
curl -H "Accept: application/json; version=2" http://localhost:4567/api/posts
```

## API Authentication

### 1. API Keys (Simple)

Generate for each user:
```ruby
class User < ActiveRecord::Base
  before_create :generate_api_key

  def generate_api_key
    self.api_key = SecureRandom.hex(32)
  end
end
```

Require it:
```ruby
helpers do
  def authenticate_api!
    key = request.env['HTTP_AUTHORIZATION']&.sub('Bearer ', '')
    @current_user = User.find_by(api_key: key)

    json_error('Unauthorized', 401) unless @current_user
  end
end

before '/api/*' do
  authenticate_api!
end
```

Use it:
```bash
curl -H "Authorization: Bearer your_api_key_here" http://localhost:4567/api/posts
```

### 2. JWT (JSON Web Tokens)

More secure, includes expiration:

```bash
gem install jwt
```

```ruby
require 'jwt'

SECRET_KEY = ENV.fetch('JWT_SECRET')

helpers do
  def generate_jwt(user)
    payload = {
      user_id: user.id,
      exp: Time.now.to_i + 3600  # 1 hour
    }
    JWT.encode(payload, SECRET_KEY, 'HS256')
  end

  def decode_jwt(token)
    JWT.decode(token, SECRET_KEY, true, algorithm: 'HS256').first
  rescue JWT::DecodeError
    nil
  end

  def authenticate_api!
    token = request.env['HTTP_AUTHORIZATION']&.sub('Bearer ', '')
    payload = decode_jwt(token)

    json_error('Unauthorized', 401) unless payload

    @current_user = User.find_by(id: payload['user_id'])
    json_error('Unauthorized', 401) unless @current_user
  end
end

# Login to get JWT
post '/api/auth/login' do
  user = User.find_by(email: json_params['email'])

  if user && user.authenticate(json_params['password'])
    {token: generate_jwt(user)}.to_json
  else
    json_error('Invalid credentials', 401)
  end
end

# Protected endpoint
before '/api/posts*' do
  authenticate_api!
end
```

Use it:
```bash
# Login
TOKEN=$(curl -X POST http://localhost:4567/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password"}' \
  | jq -r '.token')

# Use token
curl -H "Authorization: Bearer $TOKEN" http://localhost:4567/api/posts
```

### 3. OAuth 2.0 (Complex, but Standard)

For third-party API access:

```bash
gem install oauth2
```

Implementation is complex. Consider using a service like Auth0 or implementing with the `doorkeeper` gem.

## Rate Limiting

Protect your API:

```ruby
require 'rack/attack'

use Rack::Attack

Rack::Attack.throttle('api/ip', limit: 100, period: 60) do |req|
  req.ip if req.path.start_with?('/api')
end

Rack::Attack.throttle('api/token', limit: 1000, period: 60) do |req|
  if req.path.start_with?('/api')
    req.env['HTTP_AUTHORIZATION']
  end
end
```

Return rate limit info:
```ruby
after '/api/*' do
  response.headers['X-RateLimit-Limit'] = '100'
  response.headers['X-RateLimit-Remaining'] = '95'  # Calculate based on actual usage
  response.headers['X-RateLimit-Reset'] = (Time.now + 60).to_i.to_s
end
```

## CORS (Cross-Origin Resource Sharing)

Allow JavaScript from other domains to call your API:

```ruby
require 'sinatra/cross_origin'

configure do
  enable :cross_origin
end

before do
  response.headers['Access-Control-Allow-Origin'] = '*'
  response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, PATCH, DELETE, OPTIONS'
  response.headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
end

# Handle preflight requests
options '*' do
  status 200
end
```

Or use `rack-cors`:
```ruby
require 'rack/cors'

use Rack::Cors do
  allow do
    origins '*'  # or specific domains: 'example.com', 'app.example.com'
    resource '/api/*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options]
  end
end
```

## Webhooks

Let other services call your API when events happen:

```ruby
class Webhook < ActiveRecord::Base
  # url, event, secret
end

def trigger_webhooks(event, data)
  Webhook.where(event: event).each do |webhook|
    Thread.new do
      HTTParty.post(webhook.url,
        body: {
          event: event,
          data: data,
          signature: OpenSSL::HMAC.hexdigest('SHA256', webhook.secret, data.to_json)
        }.to_json,
        headers: {'Content-Type' => 'application/json'}
      )
    end
  end
end

post '/api/posts' do
  post = Post.create(json_params)

  if post.persisted?
    trigger_webhooks('post.created', post)
    status 201
    post.to_json
  else
    json_error(post.errors.full_messages.join(', '), 422)
  end
end
```

## API Documentation

Document your API. Clients need to know how to use it.

### Manual Documentation

```ruby
get '/api/docs' do
  markdown :api_docs
end
```

`views/api_docs.md`:
````markdown
# API Documentation

## Authentication

Include your API key in the Authorization header:
```
Authorization: Bearer YOUR_API_KEY
```

## Endpoints

### List Posts

```
GET /api/posts
```

Query parameters:
- `page` (integer) - Page number (default: 1)
- `per_page` (integer) - Items per page (default: 20)

Response:
```json
{
  "posts": [...],
  "page": 1,
  "per_page": 20,
  "total": 100
}
```

### Create Post

```
POST /api/posts
```

Body:
```json
{
  "title": "Post Title",
  "body": "Post content"
}
```

Response: 201 Created
````

### Automatic Documentation

Use Swagger/OpenAPI:

```bash
gem install sinatra-swagger
```

```ruby
require 'sinatra/swagger'

class API < Sinatra::Base
  register Sinatra::Swagger

  swagger_root do
    key :swagger, '2.0'
    info do
      key :version, '1.0.0'
      key :title, 'My API'
      key :description, 'A Sinatra API'
    end
  end

  swagger_path '/posts' do
    operation :get do
      key :description, 'Returns all posts'
      key :operationId, 'findPosts'
      key :produces, ['application/json']
      parameter do
        key :name, :page
        key :in, :query
        key :description, 'Page number'
        key :required, false
        key :type, :integer
      end
    end
  end

  get '/posts' do
    # ...
  end
end
```

## Testing Your API

```ruby
# test/api_test.rb
require 'minitest/autorun'
require 'rack/test'
require_relative '../app'

class APITest < Minitest::Test
  include Rack::Test::Methods

  def app
    Sinatra::Application
  end

  def test_list_posts
    get '/api/posts'
    assert last_response.ok?
    assert_equal 'application/json', last_response.content_type

    data = JSON.parse(last_response.body)
    assert data.key?('posts')
  end

  def test_create_post
    post '/api/posts',
      {title: 'Test', body: 'Content'}.to_json,
      'CONTENT_TYPE' => 'application/json'

    assert_equal 201, last_response.status

    data = JSON.parse(last_response.body)
    assert_equal 'Test', data['title']
  end

  def test_create_invalid_post
    post '/api/posts',
      {title: ''}.to_json,
      'CONTENT_TYPE' => 'application/json'

    assert_equal 422, last_response.status
  end

  def test_unauthorized
    get '/api/posts'
    assert_equal 401, last_response.status
  end
end
```

Run tests:
```bash
ruby test/api_test.rb
```

## Best Practices

### 1. Use Plural Nouns

`/api/posts` not `/api/post`

### 2. Nest Resources Logically

```
GET /api/posts/1/comments
POST /api/posts/1/comments
DELETE /api/posts/1/comments/5
```

But don't nest too deep:
```
# Bad
/api/users/1/posts/2/comments/3/likes/4

# Better
/api/likes/4
```

### 3. Filter, Sort, Search via Query Params

```
GET /api/posts?status=published&sort=created_at&order=desc
GET /api/posts?search=sinatra
```

### 4. Return Full Objects on Create/Update

Don't make clients do another GET.

### 5. Support PATCH for Partial Updates

```ruby
patch '/api/posts/:id' do
  post = Post.find(params[:id])
  data = json_params

  # Only update provided fields
  post.update(data.slice('title', 'body', 'published'))
  post.to_json
end
```

### 6. Include Related Resources

```
GET /api/posts/1?include=author,comments
```

```ruby
get '/api/posts/:id' do
  post = Post.find(params[:id])
  includes = params[:include]&.split(',') || []

  json = post.as_json

  json[:author] = post.author.as_json if includes.include?('author')
  json[:comments] = post.comments.as_json if includes.include?('comments')

  json.to_json
end
```

### 7. Version from Day One

Even if it's just `/api/v1`.

### 8. Log Everything

```ruby
require 'logger'

configure do
  set :logger, Logger.new(STDOUT)
end

before '/api/*' do
  logger.info "#{request.request_method} #{request.path} from #{request.ip}"
end

after '/api/*' do
  logger.info "Response: #{response.status}"
end
```

## What We've Learned

Building APIs with Sinatra is straightforward:

- **REST** is a convention, not a framework requirement
- **JSON** is the lingua franca of APIs
- **Status codes** communicate meaning
- **Authentication** can be API keys, JWT, or OAuth
- **Rate limiting** protects your server
- **CORS** enables cross-domain requests
- **Versioning** future-proofs your API
- **Documentation** makes your API usable

Rails gives you `rails-api`. Sinatra gives you control and understanding.

## Next Up

You can build applications and APIs. But how do you know they work?

In [Chapter 9: Testing Sinatra Applications](./09-testing.md), we'll learn how to write tests, why they matter, and how to build confidence in your code.

---

*"An API without documentation is a puzzle. An API without tests is a time bomb."*
