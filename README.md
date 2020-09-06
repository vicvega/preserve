# Preserve [![Gem version](https://img.shields.io/gem/v/preserve)](https://rubygems.org/gems/preserve) [![Build status](https://img.shields.io/travis/pienkowb/preserve/develop)](https://travis-ci.org/pienkowb/preserve) [![Coverage status](https://img.shields.io/coveralls/github/pienkowb/preserve/develop)](https://coveralls.io/github/pienkowb/preserve) [![Maintainability status](https://img.shields.io/codeclimate/maintainability/pienkowb/preserve)](https://codeclimate.com/github/pienkowb/preserve)

Preserve is a Rails plugin which stores selected parameters in the session to make them available in subsequent requests.

## Installation

Add the following line to your application's Gemfile:

```ruby
gem 'preserve'
```

Install the gem with Bundler:

```
$ bundle install
```

Or do it manually by running:

```
$ gem install preserve
```

## Usage

For the sole purpose of this example, let's assume we have a Rails application with a controller showing all parameters sent in a request.

```ruby
class ParametersController < ApplicationController
  def index
    render json: parameters
  end

  private

  def parameters
    params.except(:controller, :action)
  end
end
```

Routes are declared as following:

```ruby
Rails.application.routes.draw do
  resources :parameters, only: :index
end
```

Let's start the application and test its behavior using [cURL](https://curl.haxx.se/).
The whole concept is based on the session, so in order for this to work, the cookie engine must be enabled (hence the `-c` and `-b` options).

In the first request, the `per_page` parameter is set to 20.

```
$ curl -c cookies http://localhost:3000/parameters?per_page=20
```
```json
{"per_page":"20"}
```

As expected, the application returns the parameter and its value.

The next request is sent without any parameters.

```
$ curl -b cookies http://localhost:3000/parameters
```
```json
{}
```

Obviously, the `per_page` parameter is no longer available.

Now, let's call the `preserve` macro inside the `ParametersController` class with the parameter name as an argument.

```ruby
class ParametersController < ApplicationController
  preserve :per_page

  # ...
end
```

Sending the same two requests again gives a different result.

```
$ curl -c cookies http://localhost:3000/parameters?per_page=20
```
```json
{"per_page":"20"}
```

```
$ curl -b cookies http://localhost:3000/parameters
```
```json
{"per_page":"20"}
```

This time, the `per_page` parameter is still available when the second request is made, even though it wasn't sent particularly in that request.

### Multiple arguments

If more than one parameter needs to be preserved within the same controller, its name can be passed as a succeeding argument to the `preserve` method.

```ruby
preserve :page, :per_page
```

### Action restrictions

Limiting functionality provided by the gem to a certain set of controller actions can be achieved by applying the `only` (or `except`) option.

```ruby
preserve :per_page, only: :index
```

It behaves exactly like the `only` (or `except`) option of an [Action Controller filter](https://guides.rubyonrails.org/action_controller_overview.html#filters).

### Application-wide parameters

When there's a need to store a parameter used across the whole application, the `preserve` macro should be called inside the `ApplicationController`.

```ruby
class ApplicationController < ActionController::Base
  preserve :locale
end
```

In more complex scenarios, controller inheritance can be utilized to further adjust the scope.

### Blank parameter values

Normally, when a parameter value is an empty string, it is overwritten by a value stored in the session.
To change this behavior, you can use the `allow_blank` option.

```ruby
preserve :status, allow_blank: true
```

As a result, the parameter value would be restored from the session only when the parameter was not sent in a request.

### Setting a session key prefix

By default, parameter values are stored in the session with a key that consists of a controller name and parameter name (e.g. `users_order` for the `order` parameter in the `UsersController`).

In most cases such combination results in a unique session key, but there might be a situation when it's necessary to add a prefix in order to avoid conflicts with a session key that is already in use.
It can be done by passing the `prefix` option.

```ruby
class UsersController < ApplicationController
  preserve :order, prefix: 'preserved'
end
```

From now on, the parameter will be stored in the session with the `preserved_users_order` key.
