---
layout: post
title: Elixir on Circle-CI
tags:
  - elixir
  - erlang
  - CI
---
[Circle-CI](https://circleci.com) is an awesome continuous integration service which supports lots of languages out of
the box. For most of the projects it can detect the technologies used and run the tests automatically. You don't have
anything to configure.

Unfortunately Circle-CI doesn't support Elixir yet but, compared to other CI, it gives you full power. You can customise
each step of your build and even install your own packages on your container.

So, to make Circle-CI able to run Elixir tests, the first step is to add a `circle.yml` file at the root of your project.

```yaml
machine:
  services:
    - redis
dependencies:
  pre:
    - wget http://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb && sudo dpkg -i erlang-solutions_1.0_all.deb
    - sudo apt-get update
    - sudo apt-get install elixir
    - yes | mix deps.get
    - mix local.rebar
test:
  override:
    - mix test
```

As you can see the file is really easy, we just add the erlang solutions package, we update `apt` sources and then, we
install Elixir. Pretty simple.
As you can see I also added a redis service on the machine because I need it on my project. A list of all supported
services is available in the [Circle-CI documentation](https://circleci.com/docs/environments#databases).

Now we should be able to run the tests and see if they work well.

![Slow Circle-CI build](/assets/images/elixir-on-circleci/first-build.png)

The build takes 3 minutes 43 to run the tests. It's not too bad, but we could improve the build speed using
the caching feature from Circle-CI. This feature allows you to cache specific folders so we can run some commands only
when it's necessary.
This feature seems really useful but that's also mean we can't rely on `apt-get` anymore. We will have to install the
languages ourself.

To do that we can use an [agnostic version manager: asdf](https://github.com/HashNuke/asdf). This version manager is able
to install, through plugins, a specific version of Erlang and Elixir.

First, add a new file in your project root named `.tool-versions`. This file is required by `asdf` in order to install
the version you need.

```
erlang 17.5
elixir 1.0.4
```

Then, update your `circle.yml`:

```yaml
machine:
  environment:
    PATH: "$HOME/.asdf/bin:$HOME/.asdf/shims:$PATH"
  services:
    - redis
dependencies:
  cache_directories:
    - ~/.asdf
  pre:
    - if ! asdf | grep version; then git clone https://github.com/HashNuke/asdf.git ~/.asdf; fi
    - asdf plugin-add erlang https://github.com/HashNuke/asdf-erlang.git
    - asdf plugin-add elixir https://github.com/HashNuke/asdf-elixir.git
    - asdf install
    - yes | mix deps.get
    - yes | mix local.rebar
test:
  override:
    - mix test
```

The new version of our `circle.yml` looks a bit more complex but is definitively not. What we do is just:

* Installing `asdf` if the binary is not present
* Adding Erlang and Elixir plugins to `asdf` if they are not yet installed
* Installing, based on your `.tool-versions` file, a specific version of Erlang and Elixir if they are not yet installed
* Adding the `asdf` folder which contains Erlang and Elixir binaries to the cache
* Updating the `PATH` environment variable to add `asdf` paths

Now we should have a build much more slower (9 minutes 06):

![Slow Circle-CI build](/assets/images/elixir-on-circleci/build-slow.png)

The build is really slow the first time because it has to install Erlang from sources (7 minutes 30) but once it's done,
this step will be cached for the next builds and really fast until you have to update Erlang again.

Moreover, this solution lets you choose the exact version of Erlang and Elixir you want your tests to run against.

The Erlang release cycle is 2 versions a year so, I guess, it's still valuable to waste 8 minutes twice a year if that's
speed up a bit your tests and let you choose the version of Erlang and Elixir you want.

Elixir releases are more frequent but, because the plugin use a precompiled version of elixir, the installation is
really fast.

Below the last version running with cache:

![Fast Circle-CI build](/assets/images/elixir-on-circleci/build-fast.png)

We end up with a CI, a bit faster (1 minute 51 ~ 2 minutes saved) and much more flexible: supporting all the Erlang and
Elixir versions.

If we want to go a bit further with the cache usage, we could also cache the Elixir compile result to compile
only the change with the previous version of master instead of compile all the project everytime. I'm not really sure
it's a good idea because it could lead to a false positive: build working on CI but compilation failing on a new server.
As much as possible we shouldn't use the Circle-CI cache feature for our code. We shoud use it only, and if necessary,
for dependencies.

You have now all cards in hand to follow the Circle-CI baseline: "ship better code, faster".

_Special thanks to [@a_laibe](https://twitter.com/a_laibe), [@sreid5](https://twitter.com/sreid5) and
[@HashNuke](https://twitter.com/HashNuke) for their reviews and feedbacks._
