# Chapter 7: Sessions, Cookies, and Authentication

## The Stateless Web Problem

HTTP is stateless. Each request is independent, with no memory of previous requests.

User logs in → Server says "OK, you're logged in"
User requests another page → Server says "Who are you again?"

This is a problem. We need to remember users across requests.

The solution: **sessions** and **cookies**.

## Cookies: The Browser's Memory

A cookie is a small piece of data the server sends to the browser, and the browser sends back with every subsequent request.

Server says:
```
Set-Cookie: user_id=42; Path=/; HttpOnly
```

Browser remembers, and on every request sends:
```
Cookie: user_id=42
```

Now the server can identify the user.

## Enabling Sessions in Sinatra

Sinatra makes sessions easy:

```ruby
require 'sinatra'

enable :sessions

get '/' do
  session[:count] ||= 0
  session[:count] += 1
  "You've visited #{session[:count]} times"
end
```

Behind the scenes, Sinatra:
1. Generates a unique session ID
2. Stores it in a cookie
3. Keeps session data server-side (in memory by default)

Visit `/` repeatedly and the count increases.

## Session Configuration

### Secret Key (Important!)

By default, Sinatra uses a random secret. This resets when you restart the server, logging everyone out.

Set a persistent secret:

```ruby
enable :sessions
set :session_secret, 'your_very_secret_random_string_here'
```

Generate a secure secret:
```bash
ruby -e "require 'securerandom'; puts SecureRandom.hex(64)"
```

**Never commit this to git**. Use environment variables:

```ruby
enable :sessions
set :session_secret, ENV.fetch('SESSION_SECRET')
```

Set it:
```bash
export SESSION_SECRET="your_secret_here"
```

### Session Options

```ruby
use Rack::Session::Cookie,
  secret: ENV.fetch('SESSION_SECRET'),
  expire_after: 86400,  # 24 hours in seconds
  same_site: :lax,      # CSRF protection
  secure: production?,  # HTTPS only in production
  httponly: true        # Not accessible via JavaScript
```

## Cookie vs. Session Storage

### Cookie-Based (default)

Session data stored in the cookie itself:
- **Pros**: Stateless, works with load balancers, no server memory needed
- **Cons**: Limited to 4KB, visible to client (even if encrypted)

```ruby
use Rack::Session::Cookie, secret: 'secret'
```

### Server-Side Storage

Session data stored on server, only ID in cookie:

**Memory** (default, but lost on restart):
```ruby
enable :sessions
```

**Memcache** (for production with multiple servers):
```ruby
require 'rack-session'
require 'dalli'

use Rack::Session::Memcache,
  memcache_server: 'localhost:11211',
  secret: ENV.fetch('SESSION_SECRET')
```

**Redis** (popular choice):
```ruby
require 'rack/session/redis'

use Rack::Session::Redis,
  redis_server: { url: ENV.fetch('REDIS_URL') },
  secret: ENV.fetch('SESSION_SECRET')
```

**Database** (via ActiveRecord):
```ruby
require 'activerecord-session_store'

use ActiveRecord::SessionStore,
  secret: ENV.fetch('SESSION_SECRET')
```

Choose based on your needs:
- **Development**: Default (memory)
- **Single-server production**: Redis or database
- **Multi-server production**: Redis or Memcache

## Using Sessions

### Storing Data

```ruby
post '/login' do
  # After authentication...
  session[:user_id] = user.id
  session[:username] = user.username
  session[:logged_in_at] = Time.now
  redirect '/dashboard'
end
```

### Reading Data

```ruby
get '/dashboard' do
  halt 401 unless session[:user_id]

  @user_id = session[:user_id]
  @username = session[:username]
  erb :dashboard
end
```

### Deleting Data

```ruby
get '/logout' do
  session.clear
  redirect '/'
end
```

Or delete specific keys:
```ruby
session.delete(:user_id)
```

## Building Authentication

Let's build a real authentication system.

### The User Model

```ruby
require 'bcrypt'

class User < ActiveRecord::Base
  include BCrypt

  validates :email, presence: true, uniqueness: true
  validates :password, presence: true, length: { minimum: 8 }, if: :password

  def password
    @password ||= Password.new(password_hash) if password_hash
  end

  def password=(new_password)
    @password = new_password
    self.password_hash = Password.create(new_password)
  end

  def authenticate(test_password)
    password == test_password
  end
end
```

Migration:
```ruby
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :email, null: false
      t.string :password_hash, null: false
      t.timestamps
    end

    add_index :users, :email, unique: true
  end
end
```

**Never store passwords in plain text**. BCrypt handles secure hashing.

### Registration

```ruby
get '/signup' do
  @user = User.new
  erb :signup
end

post '/signup' do
  @user = User.new(
    email: params[:email],
    password: params[:password]
  )

  if @user.save
    session[:user_id] = @user.id
    flash[:success] = 'Account created!'
    redirect '/dashboard'
  else
    flash.now[:error] = 'Failed to create account'
    erb :signup
  end
end
```

`views/signup.erb`:
```erb
<h1>Sign Up</h1>

<form method="POST" action="/signup">
  <%== csrf_tag %>

  <div>
    <label for="email">Email:</label>
    <input
      type="email"
      id="email"
      name="email"
      value="<%= @user.email %>"
      required
    >
  </div>

  <div>
    <label for="password">Password:</label>
    <input
      type="password"
      id="password"
      name="password"
      minlength="8"
      required
    >
  </div>

  <button type="submit">Sign Up</button>
</form>

<p>Already have an account? <a href="/login">Log in</a></p>
```

### Login

```ruby
get '/login' do
  erb :login
end

post '/login' do
  user = User.find_by(email: params[:email])

  if user && user.authenticate(params[:password])
    session[:user_id] = user.id
    flash[:success] = 'Logged in!'
    redirect params[:return_to] || '/dashboard'
  else
    flash.now[:error] = 'Invalid email or password'
    erb :login
  end
end
```

`views/login.erb`:
```erb
<h1>Log In</h1>

<form method="POST" action="/login">
  <%== csrf_tag %>

  <div>
    <label for="email">Email:</label>
    <input type="email" id="email" name="email" required>
  </div>

  <div>
    <label for="password">Password:</label>
    <input type="password" id="password" name="password" required>
  </div>

  <button type="submit">Log In</button>
</form>

<p>Don't have an account? <a href="/signup">Sign up</a></p>
```

### Logout

```ruby
get '/logout' do
  session.clear
  flash[:success] = 'Logged out'
  redirect '/'
end
```

## Authentication Helpers

Keep authentication logic DRY:

```ruby
helpers do
  def current_user
    @current_user ||= User.find_by(id: session[:user_id]) if session[:user_id]
  end

  def logged_in?
    !current_user.nil?
  end

  def require_login
    unless logged_in?
      flash[:error] = 'Please log in'
      redirect "/login?return_to=#{request.path}"
    end
  end

  def require_logout
    if logged_in?
      flash[:error] = 'Already logged in'
      redirect '/dashboard'
    end
  end
end
```

Use them:

```ruby
get '/dashboard' do
  require_login
  @user = current_user
  erb :dashboard
end

get '/login' do
  require_logout
  erb :login
end

before '/admin/*' do
  require_login
  halt 403 unless current_user.admin?
end
```

In templates:

```erb
<header>
  <% if logged_in? %>
    <p>Welcome, <%= current_user.email %>!</p>
    <a href="/logout">Log out</a>
  <% else %>
    <a href="/login">Log in</a>
    <a href="/signup">Sign up</a>
  <% end %>
</header>
```

## Remember Me

Keep users logged in longer:

```ruby
post '/login' do
  user = User.find_by(email: params[:email])

  if user && user.authenticate(params[:password])
    session[:user_id] = user.id

    if params[:remember_me]
      # Extend session to 30 days
      response.set_cookie('user_id',
        value: user.id,
        max_age: 30 * 24 * 60 * 60,
        httponly: true,
        secure: production?
      )
    end

    redirect '/dashboard'
  else
    flash.now[:error] = 'Invalid credentials'
    erb :login
  end
end
```

Check the cookie:
```ruby
helpers do
  def current_user
    @current_user ||= begin
      user_id = session[:user_id] || cookies['user_id']
      User.find_by(id: user_id) if user_id
    end
  end
end
```

Better: use a random token instead of user ID in the cookie.

## Token-Based Authentication

For "remember me" or password reset, use secure tokens:

Add to User model:
```ruby
class User < ActiveRecord::Base
  # ... existing code ...

  def generate_token(purpose)
    token = SecureRandom.urlsafe_base64(32)
    self.update("#{purpose}_token" => token, "#{purpose}_token_expires_at" => 24.hours.from_now)
    token
  end

  def token_valid?(purpose, token)
    return false if token.nil?
    stored_token = send("#{purpose}_token")
    expires_at = send("#{purpose}_token_expires_at")

    stored_token == token && expires_at && expires_at > Time.now
  end
end
```

Migration:
```ruby
class AddTokensToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :remember_token, :string
    add_column :users, :remember_token_expires_at, :datetime
    add_column :users, :reset_token, :string
    add_column :users, :reset_token_expires_at, :datetime

    add_index :users, :remember_token
    add_index :users, :reset_token
  end
end
```

Remember me:
```ruby
post '/login' do
  user = User.find_by(email: params[:email])

  if user && user.authenticate(params[:password])
    session[:user_id] = user.id

    if params[:remember_me]
      token = user.generate_token(:remember)
      response.set_cookie('remember_token',
        value: token,
        max_age: 30 * 24 * 60 * 60,
        httponly: true,
        secure: production?
      )
    end

    redirect '/dashboard'
  end
end
```

Check remember token:
```ruby
helpers do
  def current_user
    @current_user ||= begin
      if session[:user_id]
        User.find_by(id: session[:user_id])
      elsif cookies['remember_token']
        user = User.joins(:remember_token).find_by(remember_token: cookies['remember_token'])
        session[:user_id] = user.id if user&.token_valid?(:remember, cookies['remember_token'])
        user
      end
    end
  end
end
```

## Password Reset

```ruby
get '/forgot-password' do
  erb :forgot_password
end

post '/forgot-password' do
  user = User.find_by(email: params[:email])

  if user
    token = user.generate_token(:reset)
    # Send email with reset link
    # PasswordResetMailer.send_reset_email(user, token)
    flash[:success] = 'Check your email for reset instructions'
  else
    flash[:error] = 'Email not found'
  end

  redirect '/login'
end

get '/reset-password/:token' do
  @token = params[:token]
  @user = User.find_by(reset_token: @token)

  halt 404 unless @user&.token_valid?(:reset, @token)

  erb :reset_password
end

post '/reset-password' do
  user = User.find_by(reset_token: params[:token])

  halt 404 unless user&.token_valid?(:reset, params[:token])

  if user.update(password: params[:password])
    user.update(reset_token: nil, reset_token_expires_at: nil)
    session[:user_id] = user.id
    flash[:success] = 'Password reset successfully'
    redirect '/dashboard'
  else
    flash.now[:error] = 'Failed to reset password'
    erb :reset_password
  end
end
```

## OAuth with OmniAuth

Let users log in with GitHub, Google, etc.

Install:
```bash
gem install omniauth
gem install omniauth-github
```

Configure:
```ruby
require 'omniauth'
require 'omniauth-github'

use OmniAuth::Builder do
  provider :github,
    ENV['GITHUB_CLIENT_ID'],
    ENV['GITHUB_CLIENT_SECRET'],
    scope: 'user:email'
end
```

Routes:
```ruby
get '/auth/:provider/callback' do
  auth = request.env['omniauth.auth']

  user = User.find_or_create_by(email: auth.info.email) do |u|
    u.name = auth.info.name
    u.oauth_provider = auth.provider
    u.oauth_uid = auth.uid
    u.password = SecureRandom.hex(32)  # Random password they'll never use
  end

  session[:user_id] = user.id
  flash[:success] = 'Logged in with GitHub!'
  redirect '/dashboard'
end

get '/auth/failure' do
  flash[:error] = "Authentication failed: #{params[:message]}"
  redirect '/login'
end
```

Add to login page:
```erb
<a href="/auth/github">Log in with GitHub</a>
```

## Two-Factor Authentication (2FA)

For extra security:

```bash
gem install rotp
gem install rqrcode
```

Add to User model:
```ruby
class User < ActiveRecord::Base
  def generate_otp_secret
    self.otp_secret = ROTP::Base32.random
    save
  end

  def otp_qr_code
    provisioning_uri = ROTP::TOTP.new(otp_secret, issuer: 'My App').provisioning_uri(email)
    RQRCode::QRCode.new(provisioning_uri).as_svg(module_size: 4)
  end

  def verify_otp(code)
    ROTP::TOTP.new(otp_secret).verify(code, drift_behind: 30)
  end
end
```

Enable 2FA:
```ruby
get '/settings/2fa/enable' do
  require_login
  current_user.generate_otp_secret
  @qr_code = current_user.otp_qr_code
  erb :enable_2fa
end

post '/settings/2fa/enable' do
  require_login

  if current_user.verify_otp(params[:code])
    current_user.update(otp_enabled: true)
    flash[:success] = '2FA enabled'
    redirect '/settings'
  else
    flash.now[:error] = 'Invalid code'
    erb :enable_2fa
  end
end
```

Check 2FA on login:
```ruby
post '/login' do
  user = User.find_by(email: params[:email])

  if user && user.authenticate(params[:password])
    if user.otp_enabled?
      session[:pending_user_id] = user.id
      redirect '/login/2fa'
    else
      session[:user_id] = user.id
      redirect '/dashboard'
    end
  else
    flash.now[:error] = 'Invalid credentials'
    erb :login
  end
end

get '/login/2fa' do
  halt 401 unless session[:pending_user_id]
  erb :login_2fa
end

post '/login/2fa' do
  user = User.find(session[:pending_user_id])

  if user.verify_otp(params[:code])
    session.delete(:pending_user_id)
    session[:user_id] = user.id
    redirect '/dashboard'
  else
    flash.now[:error] = 'Invalid code'
    erb :login_2fa
  end
end
```

## Rate Limiting

Prevent brute-force attacks:

```bash
gem install rack-attack
```

```ruby
require 'rack/attack'

use Rack::Attack

Rack::Attack.throttle('login/email', limit: 5, period: 60) do |req|
  if req.path == '/login' && req.post?
    req.params['email']
  end
end

Rack::Attack.blocklist('block bad IPs') do |req|
  # Block IPs that hit /admin without auth
  if req.path.start_with?('/admin') && !req.session[:user_id]
    Rack::Attack::Allow2Ban.filter(req.ip, maxretry: 3, findtime: 60, bantime: 3600)
  end
end
```

Handle blocks:
```ruby
Rack::Attack.blocklisted_response = lambda do |env|
  [429, {'Content-Type' => 'text/plain'}, ['Too many requests. Try again later.']]
end
```

## Security Best Practices

### 1. Always Use HTTPS in Production

```ruby
configure :production do
  use Rack::SslEnforcer
end
```

### 2. Set Secure Cookie Flags

```ruby
set :sessions,
  httponly: true,   # No JavaScript access
  secure: production?,  # HTTPS only
  same_site: :lax   # CSRF protection
```

### 3. Validate on Server

Never trust client-side validation alone.

### 4. Use Strong Passwords

```ruby
class User < ActiveRecord::Base
  validates :password, length: { minimum: 12 },
    format: {
      with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
      message: 'must include uppercase, lowercase, and number'
    }
end
```

### 5. Hash Passwords with BCrypt

Already covered. Never store plain text.

### 6. Prevent Timing Attacks

```ruby
def authenticate(email, password)
  user = User.find_by(email: email)

  # Compare hashes even if user not found (prevents timing attacks)
  if user
    user.authenticate(password) ? user : nil
  else
    BCrypt::Password.create('dummy')  # Fake work
    nil
  end
end
```

### 7. Log Security Events

```ruby
post '/login' do
  user = User.find_by(email: params[:email])

  if user && user.authenticate(params[:password])
    logger.info "Successful login: #{user.email} from #{request.ip}"
    session[:user_id] = user.id
  else
    logger.warn "Failed login attempt: #{params[:email]} from #{request.ip}"
  end
end
```

## What We've Learned

Authentication in Sinatra requires understanding the fundamentals:

- **Cookies** store data in the browser
- **Sessions** maintain state across requests
- **BCrypt** securely hashes passwords
- **Tokens** enable "remember me" and password resets
- **OAuth** lets users authenticate with third parties
- **2FA** adds extra security
- **Rate limiting** prevents abuse

Rails gives you Devise. Sinatra makes you build it. The second approach teaches you how authentication actually works.

## Next Up

We've built a full web application with authentication. But what about APIs?

In [Chapter 8: Building a RESTful API](./08-restful-api.md), we'll learn how to build JSON APIs, handle API authentication, and design REST endpoints.

---

*"Sessions are just cookies that forgot they're cookies."*
