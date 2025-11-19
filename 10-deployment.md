# Chapter 10: Deployment and Production

## Getting Real

WEBrick on `localhost:4567` is not production.

This chapter is about taking your Sinatra app from your laptop to the internet, where real users can break it in ways you never imagined.

## Production Readiness Checklist

Before you deploy:

- [ ] Tests pass
- [ ] Environment variables configured
- [ ] Database configured for production
- [ ] Secrets not in code
- [ ] HTTPS enabled
- [ ] Logging configured
- [ ] Error handling implemented
- [ ] Production web server (not WEBrick)

Let's work through each.

## Environment Variables

**Never hardcode secrets**. Use environment variables.

Bad:
```ruby
set :session_secret, 'super_secret_key'
DB = Sequel.connect('postgres://localhost/mydb')
```

Good:
```ruby
set :session_secret, ENV.fetch('SESSION_SECRET')
DB = Sequel.connect(ENV.fetch('DATABASE_URL'))
```

Set them:
```bash
export SESSION_SECRET="your_secret_here"
export DATABASE_URL="postgres://user:pass@host/db"
```

Or use a `.env` file (with `dotenv` gem):

```bash
gem install dotenv
```

Create `.env`:
```
SESSION_SECRET=your_secret_here
DATABASE_URL=postgres://localhost/blog_dev
RACK_ENV=development
```

**IMPORTANT**: Add `.env` to `.gitignore`. Never commit secrets.

Load in app:
```ruby
require 'dotenv/load' if development?
```

For production, set variables in your deployment platform.

## Configuration by Environment

```ruby
require 'sinatra'

configure :development do
  require 'sinatra/reloader'
  set :show_exceptions, true
  set :logging, Logger::DEBUG
end

configure :production do
  set :show_exceptions, false
  set :dump_errors, false
  set :logging, Logger::INFO

  # Force HTTPS
  use Rack::SslEnforcer

  # Enable caching
  set :static_cache_control, [:public, max_age: 31536000]
end

configure :test do
  set :database, {adapter: 'sqlite3', database: 'test.db'}
end
```

Run in production:
```bash
RACK_ENV=production ruby app.rb
```

## Production Web Servers

WEBrick is slow and single-threaded. Use a real server.

### Puma (Recommended)

Fast, concurrent, battle-tested.

Install:
```bash
gem install puma
```

Create `config.ru`:
```ruby
require './app'
run Sinatra::Application
```

Run:
```bash
puma config.ru
```

Configure (`config/puma.rb`):
```ruby
# Number of threads
threads 1, 5

# Number of workers (processes)
workers ENV.fetch('WEB_CONCURRENCY', 2)

# Preload app for faster worker spawn
preload_app!

# Port
port ENV.fetch('PORT', 9292)

# Log
stdout_redirect('log/puma.stdout.log', 'log/puma.stderr.log', true)

# PID file
pidfile 'tmp/pids/puma.pid'

# State file
state_path 'tmp/pids/puma.state'

# Environment
environment ENV.fetch('RACK_ENV', 'development')

# Restart workers on code changes (development)
if ENV['RACK_ENV'] == 'development'
  plugin :tmp_restart
end
```

Run with config:
```bash
puma -C config/puma.rb
```

### Unicorn

Another option, simpler than Puma:

```bash
gem install unicorn
```

Create `config/unicorn.rb`:
```ruby
worker_processes 4
timeout 30
preload_app true

listen ENV.fetch('PORT', 8080)

before_fork do |server, worker|
  # Close database connections before forking
  ActiveRecord::Base.connection.disconnect! if defined?(ActiveRecord::Base)
end

after_fork do |server, worker|
  # Reconnect after forking
  ActiveRecord::Base.establish_connection if defined?(ActiveRecord::Base)
end
```

Run:
```bash
unicorn -c config/unicorn.rb
```

## Reverse Proxy with Nginx

In production, put Nginx in front of your app server:

```nginx
# /etc/nginx/sites-available/myapp
upstream myapp {
  server 127.0.0.1:9292;
}

server {
  listen 80;
  server_name example.com;

  # Redirect HTTP to HTTPS
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl http2;
  server_name example.com;

  # SSL certificates (use Let's Encrypt)
  ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  # Security headers
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-XSS-Protection "1; mode=block" always;

  # Serve static files directly
  location ~ ^/(images|javascript|js|css|flash|media|static)/ {
    root /var/www/myapp/public;
    expires 1y;
  }

  # Proxy to app
  location / {
    proxy_pass http://myapp;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

Enable:
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## Deployment Platforms

### Heroku (Easiest)

Perfect for getting started.

Install Heroku CLI:
```bash
brew install heroku/brew/heroku
```

Login:
```bash
heroku login
```

Create app:
```bash
heroku create myapp
```

Create `Procfile`:
```
web: bundle exec puma -C config/puma.rb
```

Create `Gemfile`:
```ruby
source 'https://rubygems.org'

gem 'sinatra'
gem 'sinatra-activerecord'
gem 'pg'  # Heroku uses PostgreSQL
gem 'puma'
gem 'rake'

group :development, :test do
  gem 'sqlite3'
  gem 'dotenv'
end
```

Install:
```bash
bundle install
```

Commit:
```bash
git init
git add .
git commit -m "Initial commit"
```

Deploy:
```bash
git push heroku main
```

Set environment variables:
```bash
heroku config:set SESSION_SECRET=your_secret
```

Run migrations:
```bash
heroku run rake db:migrate
```

Open:
```bash
heroku open
```

View logs:
```bash
heroku logs --tail
```

### Render (Modern Alternative)

Like Heroku but cheaper/free tier available.

1. Connect GitHub repo
2. Configure build command: `bundle install`
3. Configure start command: `bundle exec puma -C config/puma.rb`
4. Set environment variables in dashboard
5. Deploy

### Fly.io (Docker-Based)

Install:
```bash
brew install flyctl
```

Login:
```bash
fly auth login
```

Launch:
```bash
fly launch
```

This creates `fly.toml`:
```toml
app = "myapp"

[build]
  builder = "heroku/buildpacks:20"

[env]
  PORT = "8080"

[[services]]
  internal_port = 8080
  protocol = "tcp"

  [[services.ports]]
    handlers = ["http"]
    port = 80

  [[services.ports]]
    handlers = ["tls", "http"]
    port = 443
```

Deploy:
```bash
fly deploy
```

### VPS (DigitalOcean, Linode, AWS)

For full control:

1. **Create server** (Ubuntu 22.04 recommended)

2. **Install dependencies**:
```bash
sudo apt update
sudo apt install -y ruby ruby-dev build-essential nginx postgresql postgresql-contrib
```

3. **Install gems**:
```bash
sudo gem install bundler
cd /var/www/myapp
bundle install --deployment --without development test
```

4. **Setup database**:
```bash
sudo -u postgres createuser -s myapp
sudo -u postgres createdb myapp_production
```

5. **Configure systemd** (`/etc/systemd/system/myapp.service`):
```ini
[Unit]
Description=My Sinatra App
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/var/www/myapp
Environment="RACK_ENV=production"
Environment="DATABASE_URL=postgres://myapp:password@localhost/myapp_production"
ExecStart=/usr/local/bin/bundle exec puma -C config/puma.rb
Restart=always

[Install]
WantedBy=multi-user.target
```

6. **Start service**:
```bash
sudo systemctl enable myapp
sudo systemctl start myapp
sudo systemctl status myapp
```

7. **Configure Nginx** (see above)

## Database Migrations in Production

Never run migrations manually. Automate.

Add to `Rakefile`:
```ruby
namespace :db do
  desc 'Run migrations'
  task :migrate do
    require './app'
    ActiveRecord::MigrationContext.new('db/migrate').migrate
  end
end
```

Run on deploy:
```bash
# Heroku
heroku run rake db:migrate

# Manual
RACK_ENV=production bundle exec rake db:migrate
```

## Logging

Development: log everything.
Production: log what matters.

```ruby
require 'logger'

configure do
  if production?
    log_file = File.new('log/production.log', 'a+')
    log_file.sync = true
    set :logger, Logger.new(log_file)
  else
    set :logger, Logger.new(STDOUT)
  end
end

# Use it
before do
  logger.info "#{request.request_method} #{request.path}"
end

after do
  logger.info "Response: #{response.status}"
end

# In routes
get '/posts/:id' do
  logger.debug "Loading post #{params[:id]}"
  # ...
end

# Errors
error do
  logger.error "Error: #{env['sinatra.error']}"
  logger.error env['sinatra.error'].backtrace.join("\n")
end
```

Use a logging service:
- **Papertrail** (simple, hosted)
- **Loggly** (powerful, expensive)
- **ELK Stack** (self-hosted, complex)

## Error Tracking

Know when things break:

### Sentry

```bash
gem install sentry-ruby
```

```ruby
require 'sentry-ruby'

Sentry.init do |config|
  config.dsn = ENV['SENTRY_DSN']
  config.environment = ENV['RACK_ENV']
  config.traces_sample_rate = 0.1
end

use Sentry::Rack::CaptureExceptions

error do
  Sentry.capture_exception(env['sinatra.error'])
  erb :error
end
```

### Rollbar

```bash
gem install rollbar
```

```ruby
require 'rollbar'

Rollbar.configure do |config|
  config.access_token = ENV['ROLLBAR_TOKEN']
  config.environment = ENV['RACK_ENV']
end

use Rollbar::Middleware::Sinatra

error do
  Rollbar.error(env['sinatra.error'])
  erb :error
end
```

## Monitoring

### Application Monitoring

**New Relic**:
```bash
gem install newrelic_rpm
```

```ruby
require 'newrelic_rpm'
```

**Skylight**:
```bash
gem install skylight
```

```ruby
require 'skylight/sinatra'
```

### Server Monitoring

- **Uptime monitoring**: Pingdom, UptimeRobot
- **Server metrics**: Datadog, Grafana
- **Database**: pganalyze (PostgreSQL)

## Performance

### Caching

```ruby
require 'sinatra/cache'

configure do
  set :cache, Dalli::Client.new(ENV['MEMCACHIER_SERVERS'])
end

get '/posts/:id' do
  @post = cache.fetch("post:#{params[:id]}", expires_in: 3600) do
    Post.find(params[:id])
  end
  erb :post
end
```

Or HTTP caching:
```ruby
get '/posts/:id' do
  @post = Post.find(params[:id])

  last_modified @post.updated_at
  etag @post.cache_key

  erb :post
end
```

### Database Connection Pooling

```ruby
set :database, {
  adapter: 'postgresql',
  database: ENV['DB_NAME'],
  pool: 25,
  timeout: 5000
}
```

### Gzip Compression

```ruby
use Rack::Deflater
```

## Security Checklist

- [ ] HTTPS only (use SSL enforcer)
- [ ] Secure session cookies
- [ ] CSRF protection enabled
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (escape HTML)
- [ ] Rate limiting
- [ ] Security headers
- [ ] Dependency updates

Add security headers:
```ruby
use Rack::Protection

before do
  headers 'X-Frame-Options' => 'SAMEORIGIN'
  headers 'X-Content-Type-Options' => 'nosniff'
  headers 'X-XSS-Protection' => '1; mode=block'
  headers 'Strict-Transport-Security' => 'max-age=31536000; includeSubDomains'
  headers 'Content-Security-Policy' => "default-src 'self'"
end
```

## Continuous Deployment

Automate deployments with GitHub Actions.

`.github/workflows/deploy.yml`:
```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.2
          bundler-cache: true

      - name: Run tests
        run: bundle exec rspec

  deploy:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Deploy to Heroku
        uses: akhileshns/heroku-deploy@v3.12.12
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_app_name: "myapp"
          heroku_email: "you@example.com"
```

Now every push to `main` runs tests and deploys if they pass.

## Health Checks

Add a health endpoint:
```ruby
get '/health' do
  status = {
    status: 'ok',
    database: database_connected?,
    redis: redis_connected?,
    timestamp: Time.now.iso8601
  }

  if status.values.all? { |v| v == true || v.is_a?(String) }
    status 200
  else
    status 503
  end

  content_type :json
  status.to_json
end

def database_connected?
  ActiveRecord::Base.connection.active?
rescue
  false
end

def redis_connected?
  $redis.ping == 'PONG'
rescue
  false
end
```

Point monitoring services at `/health`.

## Zero-Downtime Deploys

### Use Process Managers

**Systemd** (Linux):
```bash
sudo systemctl reload myapp
```

**PM2** (Node.js ecosystem but works for Ruby):
```bash
pm2 start config.ru
pm2 reload myapp
```

### Blue-Green Deployment

Run two versions, switch traffic:
1. Deploy new version to "green" servers
2. Test green servers
3. Switch load balancer to green
4. Keep blue as rollback option

### Database Migrations

Make migrations backwards-compatible:

**Bad** (breaks during deploy):
```ruby
# Migration 1: Remove column
remove_column :posts, :old_field

# Migration 2: Add column
add_column :posts, :new_field, :string
```

**Good** (safe):
```ruby
# Deploy 1: Add new column
add_column :posts, :new_field, :string

# Deploy 2: Copy data
Post.update_all('new_field = old_field')

# Deploy 3: Use new field in code

# Deploy 4: Remove old column
remove_column :posts, :old_field
```

## Backups

**Never skip backups**.

### Automated Database Backups

Heroku:
```bash
heroku pg:backups:schedule --at '02:00 America/Los_Angeles'
```

PostgreSQL (cron):
```bash
# /etc/cron.daily/backup-db
#!/bin/bash
pg_dump myapp_production | gzip > /backups/myapp-$(date +%Y%m%d).sql.gz
find /backups -mtime +30 -delete  # Keep 30 days
```

Upload to S3:
```bash
aws s3 cp /backups/myapp-$(date +%Y%m%d).sql.gz s3://myapp-backups/
```

### Test Your Backups

A backup you haven't tested is Schrödinger's backup.

## The Deployment Checklist

Before every deploy:

1. [ ] Tests pass locally
2. [ ] Tests pass on CI
3. [ ] Database migrations are safe
4. [ ] Environment variables configured
5. [ ] Dependencies updated
6. [ ] Security scan passed
7. [ ] Changelog updated
8. [ ] Rollback plan ready

After deploy:

1. [ ] Health check passes
2. [ ] Logs look normal
3. [ ] Error rate hasn't spiked
4. [ ] Response times acceptable
5. [ ] Key features work

## What We've Learned

Production is different from development:

- **Environment variables** keep secrets safe
- **Production servers** (Puma, Unicorn) handle real traffic
- **Deployment platforms** (Heroku, Render, Fly.io) simplify hosting
- **Monitoring** tells you when things break
- **Logging** tells you why
- **Caching** makes things fast
- **Security** keeps attackers out
- **Backups** save your bacon

Rails hides deployment complexity with Capistrano and convention. Sinatra makes you understand what's actually happening.

## The End (and Beginning)

You've learned Sinatra. You understand:
- HTTP and how the web works
- Routing, requests, and responses
- Templates and views
- Databases and ORMs
- Forms and user input
- Sessions and authentication
- REST APIs
- Testing
- Deployment

You're not a Sinatra developer now. You're a **web developer** who happens to use Sinatra.

The best part? Everything you learned applies to other frameworks and languages. HTTP is HTTP. REST is REST. Databases are databases.

Rails developers know Rails. You know the web.

Now go build something.

---

*"The deployment is the moment of truth. Everything before is just practice."*

## Appendix: Resources

### Official Documentation
- [Sinatra Documentation](http://sinatrarb.com/)
- [Sinatra Recipes](http://recipes.sinatrarb.com/)

### Recommended Gems
- **sinatra-contrib**: Extensions and utilities
- **sinatra-activerecord**: ActiveRecord integration
- **rack-protection**: Security middleware
- **bcrypt**: Password hashing
- **puma**: Production server
- **dotenv**: Environment variables

### Further Reading
- "Sinatra: Up and Running" by Alan Harris & Konstantin Haase
- "Build Awesome Command-Line Applications in Ruby 2" by David Copeland
- "The Pragmatic Programmer" by Hunt & Thomas

### Communities
- [Sinatra GitHub Discussions](https://github.com/sinatra/sinatra/discussions)
- [Ruby Discord](https://discord.gg/ruby)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/sinatra)

---

## Acknowledgments

This book wouldn't exist without:
- **Frank Sinatra** - For doing it his way
- **Blake Mizerany** - Original Sinatra creator
- **Konstantin Haase** - Long-time maintainer
- **The Sinatra community** - For keeping it simple

And you, for choosing to learn the hard (better) way.

---

**Remember**: Rails developers are just Sinatra developers who got comfortable.

Stay uncomfortable. Keep learning. Build things.

*End of "Learning Ruby with Sinatra"*
