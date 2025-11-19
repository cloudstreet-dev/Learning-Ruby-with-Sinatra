# Chapter 9: Testing Sinatra Applications

## Why Test?

Let me guess your objections:
- "Testing is slow"
- "My code is simple, I don't need tests"
- "I'll just test manually"
- "Tests are for enterprises, not my side project"

I've heard them all. I've made all those arguments. I was wrong.

Here's the truth: **Untested code is legacy code the moment you write it**.

Tests aren't about finding bugs (though they do). They're about:
- **Confidence** - Change code without fear
- **Documentation** - Tests show how code should be used
- **Design** - Hard-to-test code is usually bad code
- **Speed** - Automated tests are faster than manual testing
- **Sanity** - Sleep better knowing your app works

## Test Frameworks

Ruby has several:

### Minitest (Lightweight)

Built into Ruby. No dependencies.

```ruby
require 'minitest/autorun'

class MyTest < Minitest::Test
  def test_addition
    assert_equal 4, 2 + 2
  end
end
```

### RSpec (Expressive)

More popular. Better readability.

```ruby
require 'rspec'

describe 'Math' do
  it 'adds numbers correctly' do
    expect(2 + 2).to eq(4)
  end
end
```

We'll use RSpec because it's common and readable. But Minitest is great too.

## Setting Up RSpec

Install:
```bash
gem install rspec
gem install rack-test
```

Initialize:
```bash
rspec --init
```

This creates:
- `.rspec` - Configuration
- `spec/spec_helper.rb` - Test setup

Update `spec/spec_helper.rb`:
```ruby
ENV['RACK_ENV'] = 'test'

require_relative '../app'
require 'rspec'
require 'rack/test'

RSpec.configure do |config|
  config.include Rack::Test::Methods

  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end

  config.shared_context_metadata_behavior = :apply_to_host_groups
end
```

## Your First Test

Create `spec/app_spec.rb`:
```ruby
require 'spec_helper'

def app
  Sinatra::Application
end

describe 'Homepage' do
  it 'returns 200 OK' do
    get '/'
    expect(last_response).to be_ok
  end

  it 'contains welcome text' do
    get '/'
    expect(last_response.body).to include('Welcome')
  end
end
```

Run:
```bash
rspec
```

You should see:
```
..

Finished in 0.01234 seconds
2 examples, 0 failures
```

Congratulations. You're now a test-driven developer.

## Testing Routes

```ruby
describe 'GET /posts' do
  it 'returns all posts' do
    get '/posts'

    expect(last_response).to be_ok
    expect(last_response.content_type).to include('text/html')
  end

  it 'displays post titles' do
    Post.create(title: 'Test Post', body: 'Content')

    get '/posts'

    expect(last_response.body).to include('Test Post')
  end
end

describe 'GET /posts/:id' do
  it 'returns a specific post' do
    post = Post.create(title: 'Specific', body: 'Content')

    get "/posts/#{post.id}"

    expect(last_response).to be_ok
    expect(last_response.body).to include('Specific')
  end

  it 'returns 404 for non-existent post' do
    get '/posts/99999'

    expect(last_response.status).to eq(404)
  end
end

describe 'POST /posts' do
  it 'creates a new post' do
    expect {
      post '/posts', post: { title: 'New', body: 'Content' }
    }.to change { Post.count }.by(1)

    expect(last_response.status).to eq(302)  # Redirect
  end

  it 'rejects invalid post' do
    expect {
      post '/posts', post: { title: '', body: '' }
    }.not_to change { Post.count }

    expect(last_response).not_to be_redirect
  end
end

describe 'DELETE /posts/:id' do
  it 'deletes a post' do
    post = Post.create(title: 'Delete Me', body: 'Content')

    expect {
      delete "/posts/#{post.id}"
    }.to change { Post.count }.by(-1)
  end
end
```

## Testing JSON APIs

```ruby
describe 'API' do
  before do
    header 'Content-Type', 'application/json'
  end

  describe 'GET /api/posts' do
    it 'returns JSON' do
      get '/api/posts'

      expect(last_response.content_type).to include('application/json')
    end

    it 'returns posts array' do
      Post.create(title: 'API Test', body: 'Content')

      get '/api/posts'

      data = JSON.parse(last_response.body)
      expect(data['posts']).to be_an(Array)
      expect(data['posts'].first['title']).to eq('API Test')
    end
  end

  describe 'POST /api/posts' do
    it 'creates a post from JSON' do
      expect {
        post '/api/posts', {title: 'New', body: 'Content'}.to_json
      }.to change { Post.count }.by(1)

      expect(last_response.status).to eq(201)

      data = JSON.parse(last_response.body)
      expect(data['title']).to eq('New')
    end

    it 'returns 422 for invalid data' do
      post '/api/posts', {title: ''}.to_json

      expect(last_response.status).to eq(422)

      data = JSON.parse(last_response.body)
      expect(data['error']).to be_present
    end
  end
end
```

## Testing Authentication

```ruby
describe 'Authentication' do
  let(:user) { User.create(email: 'test@example.com', password: 'password123') }

  describe 'POST /login' do
    it 'logs in with valid credentials' do
      post '/login', email: user.email, password: 'password123'

      expect(last_response).to be_redirect
      expect(last_response.location).to include('/dashboard')
    end

    it 'rejects invalid credentials' do
      post '/login', email: user.email, password: 'wrong'

      expect(last_response).not_to be_redirect
      expect(last_response.body).to include('Invalid')
    end

    it 'sets session on successful login' do
      post '/login', email: user.email, password: 'password123'

      expect(last_request.env['rack.session'][:user_id]).to eq(user.id)
    end
  end

  describe 'Protected routes' do
    it 'redirects when not logged in' do
      get '/dashboard'

      expect(last_response).to be_redirect
      expect(last_response.location).to include('/login')
    end

    it 'allows access when logged in' do
      # Log in by setting session directly
      env 'rack.session', { user_id: user.id }

      get '/dashboard'

      expect(last_response).to be_ok
    end
  end

  describe 'GET /logout' do
    it 'clears session' do
      env 'rack.session', { user_id: user.id }

      get '/logout'

      expect(last_request.env['rack.session'][:user_id]).to be_nil
    end
  end
end
```

## Database Setup for Tests

### Use a Test Database

Update `app.rb`:
```ruby
configure :test do
  set :database, {adapter: 'sqlite3', database: 'test.db'}
end
```

### Clean Database Between Tests

Install:
```bash
gem install database_cleaner
```

Update `spec/spec_helper.rb`:
```ruby
require 'database_cleaner/active_record'

RSpec.configure do |config|
  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end

  config.around(:each) do |example|
    DatabaseCleaner.cleaning do
      example.run
    end
  end
end
```

Now each test starts with a clean database.

## Factories for Test Data

Don't manually create test data. Use factories.

Install:
```bash
gem install factory_bot
```

Create `spec/factories.rb`:
```ruby
require 'factory_bot'

FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password { 'password123' }
    name { 'Test User' }

    trait :admin do
      admin { true }
    end
  end

  factory :post do
    sequence(:title) { |n| "Post #{n}" }
    body { 'Post content here' }
    association :author, factory: :user
  end
end
```

Include in `spec/spec_helper.rb`:
```ruby
require_relative 'factories'

RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
end
```

Use in tests:
```ruby
describe 'Posts' do
  it 'displays user posts' do
    user = create(:user)
    post1 = create(:post, author: user, title: 'First')
    post2 = create(:post, author: user, title: 'Second')

    get "/users/#{user.id}/posts"

    expect(last_response.body).to include('First')
    expect(last_response.body).to include('Second')
  end

  it 'creates admin user' do
    admin = create(:user, :admin)

    expect(admin.admin?).to be true
  end
end
```

Factories are:
- Faster to write than manual setup
- More maintainable
- Self-documenting

## Testing Models

Test business logic in isolation:

```ruby
describe Post do
  describe 'validations' do
    it 'requires title' do
      post = Post.new(body: 'Content')
      expect(post).not_to be_valid
      expect(post.errors[:title]).to include("can't be blank")
    end

    it 'requires body' do
      post = Post.new(title: 'Title')
      expect(post).not_to be_valid
    end

    it 'is valid with all attributes' do
      post = Post.new(title: 'Title', body: 'Body')
      expect(post).to be_valid
    end
  end

  describe '#published?' do
    it 'returns true for published posts' do
      post = create(:post, published_at: Time.now)
      expect(post.published?).to be true
    end

    it 'returns false for unpublished posts' do
      post = create(:post, published_at: nil)
      expect(post.published?).to be false
    end
  end

  describe '#word_count' do
    it 'counts words in body' do
      post = create(:post, body: 'One two three')
      expect(post.word_count).to eq(3)
    end
  end
end
```

## Mocking and Stubbing

Don't hit external services in tests. Mock them.

```ruby
describe 'External API' do
  it 'fetches weather data' do
    # Stub HTTP request
    stub_request(:get, 'https://api.weather.com/current')
      .to_return(body: {temp: 72}.to_json)

    get '/weather'

    expect(last_response.body).to include('72')
  end
end
```

Install WebMock:
```bash
gem install webmock
```

Add to `spec/spec_helper.rb`:
```ruby
require 'webmock/rspec'

RSpec.configure do |config|
  # Disable real HTTP requests
  WebMock.disable_net_connect!(allow_localhost: true)
end
```

## Testing Helpers

```ruby
describe 'Helpers' do
  include MyApp::Helpers  # If using modular style

  describe '#format_date' do
    it 'formats dates correctly' do
      date = Time.new(2025, 1, 15)
      expect(format_date(date)).to eq('January 15, 2025')
    end
  end

  describe '#truncate' do
    it 'truncates long text' do
      text = 'a' * 200
      expect(truncate(text, 50)).to have_attributes(length: 53)  # 50 + '...'
    end

    it 'leaves short text alone' do
      text = 'short'
      expect(truncate(text, 50)).to eq('short')
    end
  end
end
```

## Test Coverage

How much of your code is tested?

Install:
```bash
gem install simplecov
```

Add to top of `spec/spec_helper.rb`:
```ruby
require 'simplecov'
SimpleCov.start do
  add_filter '/spec/'
end
```

Run tests:
```bash
rspec
```

Check coverage:
```bash
open coverage/index.html
```

**Aim for 80%+ coverage**. 100% is overkill. Less than 70% is risky.

## Test Organization

```
spec/
  spec_helper.rb
  factories.rb
  models/
    user_spec.rb
    post_spec.rb
  routes/
    posts_spec.rb
    auth_spec.rb
    api_spec.rb
  helpers/
    formatting_spec.rb
  integration/
    user_signup_spec.rb
```

Keep related tests together.

## TDD: Test-Driven Development

Write tests first, then code. Sounds backwards. It's not.

### The TDD Cycle

1. **Red** - Write a failing test
2. **Green** - Write minimal code to pass
3. **Refactor** - Clean up code

Example: Building a tag system.

#### 1. Red: Write the test

```ruby
describe Post do
  describe '#tags' do
    it 'returns tags as array' do
      post = create(:post, tag_list: 'ruby, sinatra, web')
      expect(post.tags).to eq(['ruby', 'sinatra', 'web'])
    end
  end
end
```

Run: **FAILS** (method doesn't exist)

#### 2. Green: Make it pass

```ruby
class Post < ActiveRecord::Base
  def tags
    tag_list.split(',').map(&:strip)
  end
end
```

Run: **PASSES**

#### 3. Refactor: Improve

```ruby
class Post < ActiveRecord::Base
  def tags
    return [] if tag_list.blank?
    tag_list.split(',').map(&:strip).reject(&:blank?)
  end
end
```

Add edge case tests:
```ruby
it 'handles empty tag list' do
  post = create(:post, tag_list: '')
  expect(post.tags).to eq([])
end

it 'handles nil tag list' do
  post = create(:post, tag_list: nil)
  expect(post.tags).to eq([])
end
```

### Benefits of TDD

- Ensures your code is testable
- Prevents over-engineering
- Catches edge cases early
- Provides instant feedback

## Common Testing Patterns

### Let and Before

```ruby
describe 'Posts' do
  let(:user) { create(:user) }
  let(:post) { create(:post, author: user) }

  before do
    # Runs before each test
    env 'rack.session', { user_id: user.id }
  end

  it 'displays post' do
    get "/posts/#{post.id}"
    expect(last_response).to be_ok
  end

  it 'allows editing own post' do
    get "/posts/#{post.id}/edit"
    expect(last_response).to be_ok
  end
end
```

`let` is lazy (evaluated when first used).
`let!` is eager (evaluated before test).

### Shared Examples

```ruby
shared_examples 'requires authentication' do
  it 'redirects to login' do
    get path
    expect(last_response).to be_redirect
    expect(last_response.location).to include('/login')
  end
end

describe 'Protected routes' do
  it_behaves_like 'requires authentication' do
    let(:path) { '/dashboard' }
  end

  it_behaves_like 'requires authentication' do
    let(:path) { '/posts/new' }
  end
end
```

### Contexts

```ruby
describe 'POST /posts' do
  context 'when logged in' do
    before { env 'rack.session', { user_id: user.id } }

    it 'creates post' do
      expect {
        post '/posts', post: { title: 'New', body: 'Content' }
      }.to change { Post.count }.by(1)
    end
  end

  context 'when not logged in' do
    it 'redirects to login' do
      post '/posts', post: { title: 'New', body: 'Content' }
      expect(last_response).to be_redirect
    end
  end
end
```

## Integration Tests

Test full user workflows:

```ruby
describe 'User signup flow' do
  it 'allows new user to sign up and create post' do
    # Visit signup page
    get '/signup'
    expect(last_response).to be_ok

    # Submit signup form
    post '/signup', email: 'new@example.com', password: 'password123'
    expect(last_response).to be_redirect
    follow_redirect!

    # Should be logged in and on dashboard
    expect(last_response.body).to include('Dashboard')

    # Create a post
    get '/posts/new'
    expect(last_response).to be_ok

    post '/posts', post: { title: 'First Post', body: 'Content' }
    expect(last_response).to be_redirect
    follow_redirect!

    # Post should appear
    expect(last_response.body).to include('First Post')

    # Logout
    get '/logout'
    expect(last_response).to be_redirect

    # Should no longer be able to create posts
    get '/posts/new'
    expect(last_response).to be_redirect
    expect(last_response.location).to include('/login')
  end
end
```

## What We've Learned

Testing isn't optional. It's how you build confidence:

- **RSpec** provides readable test syntax
- **Rack::Test** lets you test HTTP interactions
- **FactoryBot** creates test data easily
- **DatabaseCleaner** keeps tests isolated
- **SimpleCov** shows what's tested
- **TDD** improves design
- **Integration tests** verify workflows

Rails developers often skip testing because it's "too hard" with all the magic. Sinatra developers have no excuse—testing is straightforward.

## Next Up

Your app works. Your tests pass. Time to share it with the world.

In [Chapter 10: Deployment and Production](./10-deployment.md), we'll learn how to deploy Sinatra apps, configure production environments, and keep them running.

---

*"Code without tests is broken by design."* — Someone who learned the hard way
