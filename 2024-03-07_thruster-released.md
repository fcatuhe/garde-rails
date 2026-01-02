# Thruster is now open source

**Author:** Kevin McConnell, Programmer
**Published:** March 7, 2024
**Source:** <https://dev.37signals.com/thruster-released/>

*A minimal HTTP/2 proxy for easy, production-ready Rails deployments.*

---

We’ve just released [Thruster](https://github.com/basecamp/thruster) as open source!

Thruster is a minimal HTTP/2 proxy server that we wrote to make it easier to serve a Rails application with great performance and security.
It runs alongside your existing Puma process, and adds:
- HTTP/2 support
- SSL (via [Let’s Encrypt](https://letsencrypt.org), for automatic certificate management)
- HTTP caching, for public assets
- Efficient static file serving via `X-Sendfile` and compression

Puma already does a great job of serving Rails applications.
Thruster is designed to give it an extra little boost, so you’ll have all you need for your application to perform well on the open Internet.

---

## Why we built Thruster

The idea for Thruster came about when we started working on the [ONCE](https://once.com/) project last year.
We needed a way to package Rails applications that customers could run simply, by themselves.
The applications should require very little effort to set up, and no ongoing maintenance, but they still need to be fast and secure.

We realized that – out of the box – Rails with Puma actually gets you pretty close, but on its own it’s not quite enough.
To reach the performance you’d expect, you’d typically need to deploy Puma along with additional web servers, CDNs, and the like.
You usually end up with a few moving parts to set up and maintain.

At a certain scale that’s still the right thing to do, but for a lot of applications, it feels like more complexity than you should need.
And for our ONCE project, since our customers would be running the software themselves, it was more complexity than we could afford.

So we designed Thruster to be a simple, zero-config answer to those missing pieces we weren’t yet getting from Rails & Puma.
Something that we could just drop in to a project, but otherwise not have to think about.

Although we initially designed it for the ONCE project, we’ve since found it useful when deploying other Rails applications too.

---

## How Thruster works

Thruster wraps your Puma process so that you don’t have to worry about running multiple processes or configuring them to know about each other.

Typically you’d start your Rails application with something like:

```
rails server
```

To run the same command with Thruster, you’d use:

```
thrust rails server
```

This starts the Thruster process, which in turn starts your Rails application.
If your application quits, Thruster will quit too.
Their lifetimes are tied together.
This is particularly useful when running in a container environment, since you can simply prefix your existing `CMD` with `thrust`, and still get the same container restart behavior that you had before.

To enable automatic SSL, you just need to tell Thruster which domain it should accept traffic for.
You do that with the `SSL_DOMAIN` environment variable.
For example:

```
SSL_DOMAIN=example.com thrust rails server
```

As long as you have a valid DNS record pointing that domain to your application, Thruster will take care of provisioning and renewing the SSL certificates as needed.

You can also use Thruster in an environment where you only need some of its features.
For example, if you already have SSL termination taken care of, you can still use Thruster for its caching.
Or if you’re already using a CDN, you can still use Thruster to get rapid serving of private static files via `X-Sendfile`.

You can install Thruster by adding [its gem](https://rubygems.org/gems/thruster) to your Gemfile, and you can dive into the code on [GitHub](https://github.com/basecamp/thruster).

We hope you find it useful!
