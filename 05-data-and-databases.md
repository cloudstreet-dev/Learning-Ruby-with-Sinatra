# Chapter 5: Working with Data and Databases

## The Persistence Problem

Our blog from Chapter 4 had a fatal flaw: restart the server and all your posts disappear.

Global variables (`$posts`) live in memory. Memory is wiped when the process ends. We need **persistence**—data that survives server restarts, crashes, and deployments.

Enter databases.

## Your Options

Unlike Rails, which assumes you want ActiveRecord and a relational database, Sinatra doesn't care. You can use:

- **SQLite** - File-based SQL database (great for dev/small apps)
- **PostgreSQL** - Production-grade SQL database
- **MySQL** - Another production SQL database
- **MongoDB** - NoSQL document database
- **Redis** - In-memory key-value store
- **Flat files** - JSON, YAML, CSV (don't do this for production)
- **Anything else** - Sinatra doesn't judge

For this chapter, we'll use SQLite (simple) and PostgreSQL (production-ready).

## ORMs vs. Raw SQL

You can also choose how to talk to your database:

- **Raw SQL** - Write SQL queries yourself
- **ActiveRecord** - Rails' ORM (you can use it without Rails)
- **Sequel** - Lighter, more flexible ORM
- **ROM (Ruby Object Mapper)** - Functional approach

Rails developers only know ActiveRecord. You get to choose.

## Starting Simple: SQLite

SQLite is perfect for learning:
- No server to run (just a file)
- Built into Ruby
- Production-ready for low-traffic apps
- Same SQL syntax as PostgreSQL/MySQL

Install the gem:
```bash
gem install sqlite3
```

## Using Raw SQL

Let's start with raw SQL so you understand what's happening:

```ruby
require 'sinatra'
require 'sqlite3'

# Open database connection
DB = SQLite3::Database.new('blog.db')
DB.results_as_hash = true  # Return rows as hashes

# Create table if it doesn't exist
DB.execute <<-SQL
  CREATE TABLE IF NOT EXISTS posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  );
SQL

# List all posts
get '/' do
  @posts = DB.execute('SELECT * FROM posts ORDER BY created_at DESC')
  erb :index
end

# Show single post
get '/posts/:id' do
  @post = DB.execute('SELECT * FROM posts WHERE id = ?', params[:id]).first
  halt 404 unless @post
  erb :post
end

# Create new post (we'll make a form later)
post '/posts' do
  DB.execute(
    'INSERT INTO posts (title, body) VALUES (?, ?)',
    params[:title],
    params[:body]
  )
  redirect '/'
end

# Delete post
delete '/posts/:id' do
  DB.execute('DELETE FROM posts WHERE id = ?', params[:id])
  redirect '/'
end
```

**Important**: Always use `?` placeholders for user input. This prevents SQL injection.

**Bad** (vulnerable to SQL injection):
```ruby
DB.execute("SELECT * FROM posts WHERE id = #{params[:id]}")
```

**Good**:
```ruby
DB.execute('SELECT * FROM posts WHERE id = ?', params[:id])
```

## Introducing ActiveRecord

Raw SQL is educational, but gets tedious. Let's use ActiveRecord—yes, the same one from Rails.

Install it:
```bash
gem install activerecord
gem install sqlite3
```

Create `app.rb`:
```ruby
require 'sinatra'
require 'sinatra/activerecord'

# Database configuration
set :database, {adapter: 'sqlite3', database: 'blog.db'}

# Define models
class Post < ActiveRecord::Base
  validates :title, presence: true
  validates :body, presence: true
end

# Routes
get '/' do
  @posts = Post.order(created_at: :desc)
  erb :index
end

get '/posts/:id' do
  @post = Post.find_by(id: params[:id])
  halt 404 unless @post
  erb :post
end

post '/posts' do
  post = Post.new(title: params[:title], body: params[:body])
  if post.save
    redirect '/'
  else
    @errors = post.errors.full_messages
    erb :new_post
  end
end

delete '/posts/:id' do
  post = Post.find(params[:id])
  post.destroy
  redirect '/'
end
```

Much cleaner! But we need to create the database table.

## Migrations with ActiveRecord

Create a `Rakefile`:
```ruby
require 'sinatra/activerecord/rake'
require './app'
```

Create `db/migrate/001_create_posts.rb`:
```ruby
class CreatePosts < ActiveRecord::Migration[7.0]
  def change
    create_table :posts do |t|
      t.string :title, null: false
      t.text :body, null: false
      t.timestamps
    end
  end
end
```

Run migrations:
```bash
rake db:migrate
```

This creates `blog.db` with the `posts` table.

**Useful commands**:
```bash
rake db:migrate          # Run pending migrations
rake db:rollback         # Undo last migration
rake db:migrate:status   # See migration status
rake db:create          # Create database
rake db:drop            # Delete database
```

## ActiveRecord Basics

### Creating Records

```ruby
# Method 1: new + save
post = Post.new(title: 'Hello', body: 'World')
post.save

# Method 2: create (new + save)
Post.create(title: 'Hello', body: 'World')

# Method 3: create! (raises exception on failure)
Post.create!(title: 'Hello', body: 'World')
```

### Reading Records

```ruby
# Find by ID
post = Post.find(1)  # Raises exception if not found
post = Post.find_by(id: 1)  # Returns nil if not found

# Find by attribute
post = Post.find_by(title: 'Hello')

# Get all records
posts = Post.all

# Ordering
posts = Post.order(created_at: :desc)
posts = Post.order('created_at DESC')

# Limiting
posts = Post.limit(10)

# Chaining
posts = Post.where(published: true)
           .order(created_at: :desc)
           .limit(10)

# Counting
count = Post.count
count = Post.where(published: true).count

# First and last
post = Post.first
post = Post.last
```

### Updating Records

```ruby
# Method 1: update attributes
post = Post.find(1)
post.update(title: 'New Title')

# Method 2: set and save
post = Post.find(1)
post.title = 'New Title'
post.save

# Update all matching records
Post.where(published: false).update_all(published: true)
```

### Deleting Records

```ruby
# Delete specific record
post = Post.find(1)
post.destroy

# Delete all matching records
Post.where(draft: true).destroy_all

# Delete all records
Post.destroy_all
```

## Validations

Ensure data integrity:

```ruby
class Post < ActiveRecord::Base
  validates :title, presence: true, length: { minimum: 3, maximum: 100 }
  validates :body, presence: true
  validates :slug, uniqueness: true, format: { with: /\A[a-z0-9-]+\z/ }
end
```

Check validity:
```ruby
post = Post.new(title: 'Hi')
post.valid?  # false
post.errors.full_messages  # ["Title is too short (minimum is 3 characters)"]

post.save   # Returns false (doesn't save)
post.save!  # Raises ActiveRecord::RecordInvalid exception
```

## Associations

Link models together:

```ruby
class Post < ActiveRecord::Base
  has_many :comments
  belongs_to :author, class_name: 'User'
end

class Comment < ActiveRecord::Base
  belongs_to :post
end

class User < ActiveRecord::Base
  has_many :posts, foreign_key: 'author_id'
end
```

Migrations for associations:

```ruby
class CreateComments < ActiveRecord::Migration[7.0]
  def change
    create_table :comments do |t|
      t.references :post, foreign_key: true
      t.string :author
      t.text :body
      t.timestamps
    end
  end
end

class AddAuthorToPosts < ActiveRecord::Migration[7.0]
  def change
    add_reference :posts, :author, foreign_key: { to_table: :users }
  end
end
```

Usage:
```ruby
# Get post's comments
post = Post.find(1)
comments = post.comments

# Create associated comment
post.comments.create(author: 'Alice', body: 'Great post!')

# Get comment's post
comment = Comment.find(1)
post = comment.post

# Eager loading (avoid N+1 queries)
posts = Post.includes(:comments).all
```

## Introducing Sequel

Sequel is a lighter, more flexible alternative to ActiveRecord:

```bash
gem install sequel
gem install sqlite3
```

```ruby
require 'sinatra'
require 'sequel'

# Connect to database
DB = Sequel.connect('sqlite://blog.db')

# Create table
DB.create_table? :posts do
  primary_key :id
  String :title, null: false
  Text :body, null: false
  DateTime :created_at
  DateTime :updated_at
end

# Define model
class Post < Sequel::Model
  plugin :timestamps, update_on_create: true
end

# Routes
get '/' do
  @posts = Post.reverse_order(:created_at).all
  erb :index
end

get '/posts/:id' do
  @post = Post[params[:id]]
  halt 404 unless @post
  erb :post
end

post '/posts' do
  Post.create(title: params[:title], body: params[:body])
  redirect '/'
end
```

Sequel is:
- Faster than ActiveRecord
- More functional (chainable queries)
- Better documentation
- Less magic

But it's also:
- Less common (fewer tutorials)
- Different API (coming from Rails is harder)

Pick your preference. I like Sequel for new projects, ActiveRecord when working with Rails developers.

## Query Building

### ActiveRecord

```ruby
# Where clauses
Post.where(published: true)
Post.where('views > ?', 1000)
Post.where('created_at > ?', 1.week.ago)

# Chaining
Post.where(published: true)
    .where('views > ?', 1000)
    .order(created_at: :desc)
    .limit(10)

# Or conditions
Post.where(published: true).or(Post.where(featured: true))

# Not
Post.where.not(published: true)

# Pluck (get just one column)
titles = Post.pluck(:title)  # ["First Post", "Second Post"]
```

### Sequel

```ruby
# Where clauses
Post.where(published: true)
Post.where { views > 1000 }
Post.where { created_at > Date.today - 7 }

# Chaining
Post.where(published: true)
    .where { views > 1000 }
    .reverse_order(:created_at)
    .limit(10)

# Or conditions
Post.where(Sequel.|({published: true}, {featured: true}))

# Not
Post.exclude(published: true)

# Select columns
titles = Post.select_map(:title)
```

Sequel's block syntax is lovely:
```ruby
Post.where { (views > 1000) & (published =~ true) }
```

## Switching to PostgreSQL

SQLite is great for dev, but production apps usually need PostgreSQL.

Install:
```bash
gem install pg
```

**ActiveRecord**:
```ruby
set :database, {
  adapter: 'postgresql',
  host: 'localhost',
  database: 'blog_production',
  username: 'your_username',
  password: 'your_password'
}
```

**Sequel**:
```ruby
DB = Sequel.connect(
  adapter: 'postgres',
  host: 'localhost',
  database: 'blog_production',
  user: 'your_username',
  password: 'your_password'
)
```

Or use environment variables (better for security):
```ruby
DB = Sequel.connect(ENV['DATABASE_URL'])
```

Set `DATABASE_URL`:
```bash
export DATABASE_URL="postgres://user:password@localhost/blog_production"
```

The code stays the same. That's the beauty of ORMs.

## A Complete CRUD Example

Let's build a full blog with create, read, update, delete:

`app.rb`:
```ruby
require 'sinatra'
require 'sinatra/activerecord'
require 'sinatra/reloader' if development?

set :database, {adapter: 'sqlite3', database: 'blog.db'}

class Post < ActiveRecord::Base
  validates :title, presence: true, length: { minimum: 3 }
  validates :body, presence: true

  def formatted_date
    created_at.strftime('%B %d, %Y')
  end
end

# List posts
get '/' do
  @posts = Post.order(created_at: :desc)
  erb :index
end

# New post form
get '/posts/new' do
  @post = Post.new
  erb :new
end

# Create post
post '/posts' do
  @post = Post.new(params[:post])
  if @post.save
    redirect "/posts/#{@post.id}"
  else
    erb :new
  end
end

# Show post
get '/posts/:id' do
  @post = Post.find_by(id: params[:id])
  halt 404 unless @post
  erb :show
end

# Edit post form
get '/posts/:id/edit' do
  @post = Post.find(params[:id])
  erb :edit
end

# Update post
put '/posts/:id' do
  @post = Post.find(params[:id])
  if @post.update(params[:post])
    redirect "/posts/#{@post.id}"
  else
    erb :edit
  end
end

# Delete post
delete '/posts/:id' do
  Post.find(params[:id]).destroy
  redirect '/'
end
```

We'll add the forms in the next chapter.

## Database Best Practices

### Use Connection Pooling

For production:
```ruby
set :database, {
  adapter: 'postgresql',
  database: 'blog_production',
  pool: 5,  # Connection pool size
  timeout: 5000
}
```

### Use Transactions

For operations that must all succeed or all fail:

```ruby
ActiveRecord::Base.transaction do
  post = Post.create!(title: 'New Post', body: 'Content')
  post.comments.create!(author: 'System', body: 'First!')
end
```

If any operation fails, everything rolls back.

### Avoid N+1 Queries

**Bad** (N+1 query problem):
```ruby
posts = Post.all
posts.each do |post|
  puts post.comments.count  # Queries database for EACH post
end
```

**Good** (eager loading):
```ruby
posts = Post.includes(:comments).all
posts.each do |post|
  puts post.comments.count  # Already loaded
end
```

### Index Your Queries

In migrations:
```ruby
add_index :posts, :slug, unique: true
add_index :posts, :created_at
add_index :comments, :post_id
```

Speeds up queries by orders of magnitude.

## When Not to Use a Database

Sometimes you don't need a database:

- **Configuration**: Use YAML or environment variables
- **Temporary data**: Use memory or Redis
- **Sessions**: Use encrypted cookies
- **Cache**: Use Redis or Memcached
- **Static content**: Use flat files

Don't use a database for everything just because Rails does.

## What We've Learned

Sinatra doesn't force an ORM on you:

- **Raw SQL**: Full control, more typing
- **ActiveRecord**: Familiar, heavy, lots of magic
- **Sequel**: Clean, fast, functional
- **SQLite**: Great for dev and small apps
- **PostgreSQL**: Production-ready
- **Migrations**: Version control for your database schema
- **Validations**: Data integrity
- **Associations**: Link models together

Pick the tools that fit your needs, not what a framework dictates.

## Next Up

We have a database. We can store posts. But we need a way for users to create them.

In [Chapter 6: Forms and User Input](./06-forms-and-params.md), we'll build forms, handle user input, and make our blog interactive.

---

*"Premature optimization is the root of all evil, but so is choosing MongoDB for your blog."* — Not Knuth, but probably still true
