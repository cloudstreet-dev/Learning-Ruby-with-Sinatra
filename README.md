# Learning Ruby with Sinatra

## Why This Book Exists

Everyone uses Ruby on Rails. That's the problem.

Rails is an incredible framework that revolutionized web development. But somewhere along the way, "Ruby web development" became synonymous with "Rails," and that's done a disservice to both newcomers and the Ruby ecosystem itself.

This book argues for a different path: **Sinatra**.

## Why Sinatra, Not Rails?

### Rails is a Black Box

Rails gives you 50,000 lines of framework code before you write a single line of your own. It's "convention over configuration" until you need to configure something, at which point you're Googling for hours trying to understand the magic happening behind the scenes.

Sinatra is **158 lines of code**. You can read the entire source in an afternoon and actually understand what's happening.

### You Don't Need a Framework to Learn a Framework

Rails teaches you Rails. Sinatra teaches you web development.

When you learn Sinatra, you learn:
- How HTTP actually works
- What routes really are
- How request/response cycles function
- Why REST matters
- What middleware does

Rails abstracts all of this away. That's great when you're experienced. It's terrible when you're learning.

### Most Apps Don't Need Rails

Need an API? A webhook receiver? A simple web service? A microservice? Rails is bringing a freight train to a bicycle race.

Sinatra boots in milliseconds, has minimal dependencies, and doesn't make assumptions about your database, your ORM, your asset pipeline, your JavaScript framework, or your preferred brand of coffee.

### Sinatra Makes You a Better Rails Developer

Even if you end up using Rails (and you might!), understanding Sinatra will make you a better Rails developer. You'll understand what Rails is doing under the hood, when to fight the framework, and when to embrace it.

## What You'll Learn

This book takes you from zero to production-ready web applications using nothing but Sinatra and your wits.

### Table of Contents

1. **[Why Not Rails?](./01-why-not-rails.md)**
   A manifesto for the unconvinced

2. **[Hello World: Your First Sinatra App](./02-hello-world.md)**
   From `gem install` to running server in 5 minutes

3. **[Routes, Requests, and Responses](./03-routing-and-http.md)**
   Understanding HTTP without the magic

4. **[Views and Templates](./04-templates-and-views.md)**
   Making things pretty with ERB, HAML, and friends

5. **[Working with Data](./05-data-and-databases.md)**
   Databases, ORMs, and when not to use either

6. **[Forms and User Input](./06-forms-and-params.md)**
   Handling data without Rails form helpers

7. **[Sessions, Cookies, and Authentication](./07-sessions-and-auth.md)**
   Keeping users logged in without Devise

8. **[Building a RESTful API](./08-restful-api.md)**
   JSON, status codes, and API design

9. **[Testing Sinatra Applications](./09-testing.md)**
   RSpec, Rack::Test, and why tests matter

10. **[Deployment and Production](./10-deployment.md)**
    Getting your app into the real world

## Who This Book Is For

- **Beginners** who want to learn web development without drowning in framework magic
- **Rails developers** who want to understand what's happening under the hood
- **API developers** who don't need the full Rails stack
- **Anyone** who's tired of 10-second boot times

## What You Need

- Basic Ruby knowledge (variables, methods, classes)
- A computer with Ruby installed
- A text editor
- Curiosity about how the web actually works

## The Philosophy

This book has three core principles:

1. **Understand, don't memorize**: We'll explain *why* things work, not just *how*
2. **Simple beats complex**: If we can solve it with 10 lines instead of 100, we will
3. **Real-world ready**: Every example could be (and in some cases, is) production code

## A Note on Humor

Web development is serious business. This book occasionally isn't. If you find a joke that doesn't land, just `git blame` it and send me a pull request with something funnier.

## Let's Begin

Ready to learn web development the Sinatra way?

Start with [Chapter 1: Why Not Rails?](./01-why-not-rails.md)

---

*This book is a work in progress. Found a bug? Have a suggestion? The code is cleaner than the prose? Open an issue or submit a PR.*

## License

MIT License - Because knowledge should be free, and also because I can't afford a lawyer to write a custom license.

---

**Remember**: Rails developers are just Sinatra developers who got comfortable.

Let's keep you uncomfortable for a little while longer.
