# EndpointFlux
A simple way to organise API endpoints

✍✍✍ **Feedback is welcome** ✅✅✅

💚💚💚 **Stars are welcome** 💚💚💚

Telegram group  https://t.me/endpoint_flux

![Logo](https://svgshare.com/i/PQo.svg)

[![Gem Version](https://badge.fury.io/rb/endpoint-flux.svg)](http://badge.fury.io/rb/endpoint-flux)

[![CircleCI](https://circleci.com/gh/pavelkvasnikov/endpoint-flux.svg?style=svg)](https://app.circleci.com/pipelines/github/pavelkvasnikov/endpoint-flux)

[![Maintainability](https://api.codeclimate.com/v1/badges/a99a88d28ad37a79dbf6/maintainability)](https://codeclimate.com/github/codeclimate/codeclimate/maintainability)

[![Known Vulnerabilities](https://snyk.io/test/github/pavelkvasnikov/endpoint-flux/badge.svg?targetFile=Gemfile.lock)](https://snyk.io/test/github/pavelkvasnikov/endpoint-flux?targetFile=Gemfile.lock)

[![codecov](https://codecov.io/gh/pavelkvasnikov/endpoint-flux/branch/master/graph/badge.svg)](https://codecov.io/gh/pavelkvasnikov/endpoint-flux)

![dependabot](https://badgen.net/dependabot/pavelkvasnikov/endpoint-flux/?icon=dependabot)

## Index
- [Projects code organisation](#projects-code-organisation)
- [Usage](#usage)
  - [Installation](#installation)
  - [Configuration](#configuration)  
  - [Endpoints](#endpoints)  
  - [Routing](#routing)
  - [Controllers](#controllers)  
  - [Middlewares](#middlewares)
      - [Authenticator](#authenticator)
      - [Authorizator](#authorizator)      
      - [Policy](#policy)
      - [Validator](#validator)
      - [Decorator](#decorator)
  - [Decorators](#decorators)
  - [Validations](#validations)  
  - [Services](#services)
  - [Exceptions](#exceptions)
  - [Response helpers](#response-helpers)  
- [Contributing](CONTRIBUTING.md)
  - [Maintainers](https://github.com/resolving/endpoint-flux/graphs/contributors)
- [License](#license)


## Projects code organisation

EndpointFlux offers you a new file and logic organisation in Ruby applications.

```
app
├── controllers
│   ├── users_controller.rb
├── endpoint_flux
│   ├── decorators
│   │   ├── users
│   │   │   ├── project.rb
│   │   ├── user.rb
│   │   ├── ...
│   ├── endpoints
│   │   ├── users
│   │       ├── create.rb
│   │       ├── update.rb
│   │       ├── ...
│   ├── middlewares
│   │   ├── authenticator
│   │   │   ├── default.rb
│   │   ├── authorizator
│   │   │   ├── default.rb
│   │   ├── decorator
│   │   │   ├── paginate.rb
│   │   │   ├── representable.rb
│   │   │   ├── ...
│   │   ├── policy
│   │   │   ├── comment.rb
│   │   │   ├── ...
│   │   ├── validator
│   │       ├── inline.rb
│   ├── services
│   │   ├── auth.rb
│   │   ├── ...
│   ├── validations
│   │   ├── predicates
│   │   │   ├── base.rb
│   │   │   ├── date.rb
│   │   │   ├── ...
│   │   ├── base.rb
│   │   ├── error.rb
│   │   ├── user.rb
```


## Usage

### Installation
Add this line to your application's Gemfile:

```ruby
gem 'endpoint-flux', require: 'endpoint_flux'
```

For a latest versvion
```ruby
gem 'endpoint-flux', git: 'https://github.com/pavelkvasnikov/endpoint-flux.git', require: 'endpoint_flux'
```

And then execute:
```bash
$ bundle install
```

Or install it yourself as:
```bash
$ gem install endpoint-flux
```

Run rake task to generate project structure (currently available in a latest version)
```bash
$ rails endpoint_flux:init
```
It will add a new dir `app/endpoint_flux` and new initializer `config/initializers/endpoint_flux.rb`

### Configuration

You can initialize the EndpointFlux before using and can specify a bunch of params as you need. Locate it in 
`config/initializers/endpoint_flux.rb`.

With the `EndpointFlux.config.middlewares_namespaces` directive you can specify the location of middleware. 
```ruby
EndpointFlux.config.middlewares_namespaces << 'middlewares'
```

To specify the default middleware that will be used for each response, if nothing specified in Endpoint class, use 
`EndpointFlux.config.default_middlewares` directive. For example:
```ruby
EndpointFlux.config.default_middlewares :validator, :inline
```
Where `:validator` is a middleware and `:inline` is a class that defined in 
`app/endpoint_flux/middlewares/validator/inline.rb`. More details [here](#validator).

With `EndpointFlux.config.rescue_from` you can specify how to handle the custom exceptions that would be raised in 
Application. For example: 
 ```ruby
not_found_errors = [ActiveRecord::RecordNotFound]
EndpointFlux.config.rescue_from(not_found_errors) do |_, attrs, _|
  attrs[1].body = EndpointFlux::Exceptions::NotFound.new.to_hash
  attrs
end
 ```

Also you can specify interceptor that would be called before processing each response
```ruby
EndpointFlux.config.interceptor do |attrs|
  Rails.root.join('maintenance.txt').exist? &&
    raise(EndpointFlux::Exceptions::ServiceUnavailable)

  attrs
end 
```

And if you need you can define your own methods like this:
```ruby
EndpointFlux::Endpoint.class_eval do
  define_method(:raise_validation_error) do |errors|
    raise EndpointFlux::Exceptions::Validation, errors
  end
end
```

Config example:
```ruby
# config/initializers/endpoint_flux.rb
require 'endpoint_flux'

EndpointFlux.config.middlewares_namespaces << 'middlewares'

EndpointFlux.config.default_middlewares :authenticator, :default
EndpointFlux.config.default_middlewares :authorizator, :default
EndpointFlux.config.default_middlewares :validator, :inline
EndpointFlux.config.default_middlewares :policy,    :skip
EndpointFlux.config.default_middlewares :decorator, :skip

not_found_errors = [ActiveRecord::RecordNotFound]
EndpointFlux.config.rescue_from(not_found_errors) do |_, attrs, _|
  attrs[1].body = EndpointFlux::Exceptions::NotFound.new.to_hash
  attrs
end

EndpointFlux.config.interceptor do |attrs|
  Rails.root.join('maintenance.txt').exist? &&
    raise(EndpointFlux::Exceptions::ServiceUnavailable)

  attrs
end

EndpointFlux::Endpoint.class_eval do
  define_method(:raise_validation_error) do |errors|
    raise EndpointFlux::Exceptions::Validation, errors
  end
end
```

### Routing

EndpointFlux has Rails helper - 
[`present`](https://github.com/resolving/endpoint-flux/blob/master/lib/endpoint_flux/rails/concerns/endpoint_controller.rb), 
which integrates with Rails controllers. So, you can use the default Rails routing system to define routes.

```ruby
Rails.application.routes.draw do
  resources :users
end
```

```ruby
class UsersController < ApplicationController
  def index
    present 'users/index' # it dispatches to Endpoints::Users::Index endpoint class.
  end  
end
```

Or if you're using it in not Rails application, you can implement the middleware for providing
such data to Endpoints namespace, for example:

```ruby
class BaseHandler
  def process(msg, options, namespace)
    params   = JSON.parse(msg)
    action   = options[:headers]['action']
    endpoint = endpoint_for("#{namespace}/#{action}")

    _, response = endpoint.perform(request_object(params))

    response.body
  end
  
  private

  def endpoint_for(namespace)
    if ::EndpointFlux.config.endpoints_namespace
      ::EndpointFlux.config.endpoints_namespace + '/' + namespace
    else
      namespace
    end.camelize.constantize
  end

  def request_object(params)
    ::EndpointFlux::Request.new(headers: {}, params: params.to_h.deep_symbolize_keys!)
  end
end
```


### Controllers

Controllers are simple endpoints for HTTP. They don't have any business logic and just dispatch to an endpoint class.

```ruby
class UsersController < ApplicationController
  def index
    present 'users/index' # it dispatches to Endpoints::Users::Index endpoint class.
  end  
end
```
Finally `present` method renders `response.body` in JSON format (`render json: response.body`).  
 

### Endpoints

Endpoints encapsulate business logic and it's a central part of applications architecture. It can be used in any Ruby 
application like Rails, Sinatra and etc. 
It's a simple coordinator between all layers needed to get the job done. Endpoint needs for defining and implementing 
steps for the processing data for response. It uses middlewares for data processing that receives the arguments from 
the caller and returns the array `[request, response]` with `response` that contains `body` and `headers` for API.

```ruby
# app/endpoint_flux/endpoints/users/comments/index.rb
module Endpoints
  module Users
    module Comments
      module Index
        include EndpointFlux::Endpoint

        policy :user
        policy :comments

        validator :inline do
          required(:user_id).value(:number?)
        end

        process do |request, response|
          response.body[:comments] = request.scope

          [request, response]
        end

        decorator :add_status, 200        
        decorator :representable, decorator: :comment, collection?: true, wrapped_in: :comments
      end
    end
  end
end
```



### Middlewares

EndpointFlux has 6 types of predefined middlewares. They will be called in the strong defined order - `authenticator, 
authorizator, validator, policy, process, decorator`. Where `process` should be defined inside the endpoint class for 
the request processing.

Also you can add your own middleware class to this flow or change the order. It's possible in two ways: 

* Inside the custom endpoint class, for example:  
```ruby
# app/endpoint_flux/endpoints/users/index.rb

module Endpoints
  module Users
    module Index
      include EndpointFlux::Endpoint
      # define new flow with `new_middleware` only for this endpoint
      flow %i[authenticator authorizator validator policy process new_middleware decorator]
      
      authorizator :skip
      #...
      process do |request, response|
      # ... some actions
        [request, response]
      end
      
      # define the middleware
      new_middleware :default 
      
      decorator :add_status, 200
    end
  end
end      
```

* Globally in EndpointFlux config section that will affect all endpoints, for example:
```ruby
# config/initializers/endpoint_flux.rb
# ...
# define default value for new middleware
EndpointFlux.config.default_middlewares :new_middleware, :default 
# Global change the middlewares order flow by adding a new one `new_middleware`
EndpointFlux.config.flow(%i[authenticator authorizator validator policy process new_middleware decorator]) 
```

Middleware class definition should contains `self.perform(*args)` method and returns the `[request, response]` as a result
```ruby
# app/endpoint_flux/middlewares/new_middleware/default.rb
module Middlewares
  module NewMiddleware
    module Default
      def self.perform(request, response, _)
        # ... some actions
        [request, response]
      end
    end
  end
end
```

We have implemented the default middlewares that could be used to skip it without any changes to data. It's located 
[here](https://github.com/resolving/endpoint-flux/tree/master/lib/endpoint_flux/middlewares)
 

#### Authenticator

Here you can implement your authenticate system. For example you can user the [JWT gem](https://github.com/jwt/ruby-jwt)
Locate it in `app/endpoint_flux/middlewares/authenticator` folder.
Also you can skip this middleware in Endpoint class by `authenticator :skip` directive


#### Authorizator

Here you can implement your authorization system and check the user permissions according to the user role.
Locate it in `app/endpoint_flux/middlewares/authorizator` folder.
Also you can skip this middleware in the Endpoint class by `authorizator :skip` directive



#### Policy

Here you implement different policy scopes and use them inside the Endpoint class. And also you can chain it to each other by
calling in special order. Locate it in `app/endpoint_flux/middlewares/policy` folder.

For example: 
* User policy 
```ruby
# app/endpoint_flux/middlewares/policy/user.rb
module Middlewares
  module Policy
    module User
      def self.perform(request, response, _)
        request.scope = ::User.find(request.params[:user_id])

        [request, response]
      end
    end
  end
end
```

* Comments policy 
```ruby
# app/endpoint_flux/middlewares/policy/comments.rb
module Middlewares
  module Policy
    module Comments
      def self.perform(request, response, _)
        raise 'scope must be set' unless request.scope
        raise 'scope must be User' unless request.scope.class.name == 'User'

        request.scope = ::Comment.where(user_id: request.scope.id)

        [request, response]
      end
    end
  end
end
```

And usage inside the Endpoint class:
```ruby
# app/endpoint_flux/endpoints/users/comments/index.rb
module Endpoints
  module Users
    module Comments
      module Index
        include EndpointFlux::Endpoint

        policy :user  # get user scope
        policy :comments # get users comments scope

        validator :inline do
          required(:user_id).value(:number?)
        end

        process do |request, response|
          response.body[:comments] = request.scope

          [request, response]
        end

        decorator :add_status, 200        
        decorator :representable, decorator: :comment, collection?: true, wrapped_in: :comments
      end
    end
  end
end
```


#### Validator

Here you can implement validation system for request params using the [Dry validation gem](http://dry-rb.org/gems/dry-validation)
or another libraries. Locate it in `app/endpoint_flux/middlewares/validator` folder.
Also you can skip this middleware in the Endpoint class by `validator :empty` directive

```ruby
# app/endpoint_flux/middlewares/validator/inline.rb
module Middlewares
  module Validator
    module Inline
      def self.perform(request, response, _options, &block)
        validation = ::Services::Validation(&block).call(request.params)
        unless validation.success?
          raise ::EndpointFlux::Exceptions::Validation, validation.messages
        end
        request.params = validation.result

        [request, response]
      end
    end
  end
end
```

Just declare the schema block inside the endpoint to provide it to middleware 
```ruby
# app/endpoint_flux/endpoints/users/create.rb
module Endpoints
  module Users
    module Create      
      include EndpointFlux::Endpoint

      authenticator :skip
      authorizator :skip
      
      validator :inline do
        required(:user).schema do
          required(:email).value(:str?, :email?)
          required(:password).value(:str?, :password?)
        end        
      end

      process do |request, response|
        # some actions ... like calling checking for user uniqueness, Mailer Sidekiq workers, token generation and etc.
  
        response.body[:user] = ::User.create(request.params[:user])
        
        # ...

        [request, response]
      end

      decorator :add_status, 200        
      decorator :representable, decorator: :user      
    end
  end
end
```


#### Decorator

Here you can implement a decorator system for representing the response.
Locate it in `app/endpoint_flux/middlewares/decorator` folder. You can call it inside the endpoint class by using 
directive like this `decorator :representable, decorator: :user`, where `:representable` it's your decorators class name 
and `decorator: :user` it's a custom params as you wish (in this situation specialising to use User decorator for 
representing data).  
For example 

```ruby
# app/endpoint_flux/middlewares/decorator/representable.rb
module Middlewares
  module Decorator
    module Representable
      def self.perform(request, response, options)
        resource_name = options[:decorator]
        resource      = response.body[resource_name]

        response.body[resource_name] = ::Services::Decorator.call(resource, options) if resource

        [request, response]
      end
    end
  end
end      
```

You can add a custom status to the response body by using directive `decorator :add_status, {status_number}`, 
for example `decorator :add_status, 200`.
Also you can skip this middleware in the Endpoint class by `decorator :skip` directive



### Decorators

Endpoint can use representers from `app/endpoint_flux/decorators` to serialize and parse JSON and XML documents for APIs.
For example you can use 
[Representable gem](http://trailblazer.to/gems/representable), it maps representation documents from and to Ruby objects
and includes JSON, XML and YAML support, plain properties and compositions.
You can define the decorator schema class in `app/endpoint_flux/decorators` folder and specify it inside of the 
endpoint class by providing as params for `decorator` directive, 
for example  `decorator :representable, decorator: :user`

```ruby
# app/endpoint_flux/decorators/user.rb
module Decorators
  class User < Representable::Decorator
    include Representable::JSON

    property :id
    property :name
    property :email    
    property :role, exec_context: :decorator   
    
    property :updated_at
    property :created_at

    def role
      represented.role.name
    end
  end
end    
```


### Validations

In `app/endpoint_flux/validations` you can locate a custom validation classes and use them with Validator middleware.


### Services

You can move some business logic from endpoints to service object and locate it here `app/endpoint_flux/services`.


### Exceptions

You can use EndpointFlux predefined Exceptions for you business logic, for example 
`raise ::EndpointFlux::Exceptions::Validation`.
They defined in 
[lib/endpoint_flux/exceptions](https://github.com/resolving/endpoint-flux/tree/master/lib/endpoint_flux/exceptions)

The list of exceptions:
* `Forbidden`
* `NotFound`
* `ServiceUnavailable`
* `Unauthorized`
* `Validation`


### Response helpers

If needs you can use the response helpers to check the response body status such as `success?`, `invalid?` or 
you can define your own helpers in that way. You can use it with an instance of the `EndpointFlux::Response` class.
They defined in 
[lib/endpoint_flux/response.rb](https://github.com/resolving/endpoint-flux/blob/master/lib/endpoint_flux/response.rb)

The list of helpers:
* `success?`
* `invalid?`
* `forbidden?`
* `unauthorized?`
* `not_found?`


## [Contributing](CONTRIBUTING.md)

### [Maintainers](https://github.com/resolving/endpoint-flux/graphs/contributors)


## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
