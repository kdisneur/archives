---
layout: post
title: End to End test in JavaScript
tags:
  - javascript
  - rspec
  - ruby
  - capybara
---
When I have to work on a frontend application, I try to keep the logic outside of this application. When I can't,
I write [unit test with Jasmine](/2014-12-07-unit-test-angular-app-without-dependencies.html) but, usually, the frontend
application is more user interaction and all the complex part is done server side. So, writing unit tests could be just
called typo testing.

The API I rely on is a custom ruby API, fully tested, so I just have to test the user interactions.

## JavaScript frameworks

There are some frameworks allowing to test end to end user interactions:

* http://nightwatchjs.org
* https://github.com/angular/protractor/blob/master/README.md

But, the main concern I've with these frameworks is we can't `stub` external call easily. There are some projects trying to
achieve that but no one is really ready to go to production. Or, at least, I was not able to use them.

## Ruby frameworks

We have, in the ruby community, `rspec` and `capybara` to test this kind of behaviour. It's a really good duo where you can
express easily the user interaction (visit, click, fill fields,...). If you don't know them, I suggest you to read their
documentation.

Basically, you have to install `ruby` and configure `rspec` the following way:

```ruby
# spec_helper
require 'rspec'
require 'capybara/poltergeist'
require 'capybara/rspec'
require 'pry'

RSpec.configure do |config|
  config.mock_with :rspec
  config.include Capybara::DSL
  config.include AuthenticationHelper
end

Capybara.default_driver = :poltergeist
```

This code's snippet configures `rspec` to use `capybara` and `poltergeist` as a browser driver. This driver fully
supports `JavaScript` interactions.

### Create the server

When your `rspec` test suite includes `capybara`, the first thing `capybara` tries to do is to start a `rack` application.
If you're working with `rails` or `sinatra`, you would not have to much trouble because they are both based on `rack`, so
it's automatically configured for you.

In our case, we use a "pure" HTML, CSS and JS application powered by `gulp`, so we don't have any server to deliver our
files. We need to create this `rack` server ourself.

Hopefully, we can easily create a `rack` application to deliver static files:

```ruby
# spec_helper.rb
Capybara.app = Rack::Builder.new do
  run Proc.new { |environment|
    base_path  = './build'
    request    = Rack::Request.new(environment)
    index_file = File.join(base_path, request.path_info, 'index.html')
    request.path_info += 'index.html' if File.exists?(index_file)

    Rack::Directory.new(base_path).call(environment)
  }
end
```

The previous code creates a `rack` server, looking for files in your `./build` folder. Moreover, the 5th and 6th lines
add a small trick to deliver `index.html` files when you request for a directory.

The only downside with this setup is you have to run your build task yourself because the `rack` application will serve the files from your
build folder.

For example, I've to run `gulp build` before running my end to end tests.

### Write the tests

Here we are, we can start to write some tests like in the following example:

```ruby
# add_project_spec.rb
require_relative '../spec_helper'

describe 'Add project' do
  context 'when user is authenticated' do
    context 'when user click to import a project' do
      it 'imports the project' do
        visit '/application/?code=XXXX#/authorize'

        visit '/application/#/add-project'
        page.all('.m-add_project-organization')[0].click
        expect(page).to have_selector('.m-add_project-project--name')
        within('.m-add_project-project:first-child') do
          find('a').click
        end
        expect(page).to have_selector('.m-user_feedback_message')
        expect(page.find('.m-user_feedback_message').text).to eql('Project awesome-test-project has been successfully imported.')
      end
    end
  end
end
```

The tests work but it means your API has to be present and responsive. It's maybe not a problem for you and you could
stop at this step but it's not an option for me. I would like to have my tests running when:

* They are running on the CI and so, there is no local API to reply
* I use a third party API with a limit rate and I don't want to reach it.

### Save the API reply

It's now time to save the API response locally to use that the next time you run your tests (instead of calling the real
API). To do that, we often use `vcr` in the `ruby` community but this package allows to stub external API, only for
external request coming from ruby code. In our case, the calls come from an Ajax request, so it can't work.

Hopefully, a really nice gem does the job. The [puffing-billy](https://github.com/oesmith/puffing-billy) gem is able to
stub all your Ajax calls and stubs the request in a local file.

The configuration of `puffing-billy` is as simple as:

```ruby
# spec_helper.rb
Capybara.server_port = 59533

Billy.configure do |config|
  config.logger            = nil
  config.cache             = true
  config.ignore_cache_port = true
  config.persist_cache     = true
  config.cache_path        = 'tests/behaviours/cassettes/'
end

Capybara.default_driver = :poltergeist_billy
```

In the previous code sample we've just forced the value of the `capybara` server port. We have to do that because
`puffing-billy` saves everything locally for a specific host and port. But, `capybara` starts your `rack` application on
a new port each time you run your tests. The consequence is `puffing-billy` is not able to reuse the created cache files
and tries to do all the API call again.

Then, obviously, we configure `puffing-billy`. And, finally, we set a new `capybara` driver. `puffing-billy` comes with
it's own override of `poltergeist` and we have to use this one to be able to stub external calls.

In this step we have:

* Beautiful end to end tests
* No more useless API calls
* Tests ready to be run in isolation (locally or on a CI server)
* No worries to have about your third parties rate limit

Of course, sometimes, you should regenerate all these cached files, just to be sure the API continues to return the
good value. Personnally, I regenerate all my cached files after each feature.
