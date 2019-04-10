# Swagger::Docs::Rails
copied from [@swagger-docs](https://github.com/richhollis/swagger-docs)

Generates swagger-ui json files for rails apps with APIs. You add the swagger DSL to your controller classes and then run one rake task to generate the json files.

[gem]: https://rubygems.org/gems/swagger-docs

## Swagger Version Specification Support

This project supports elements of the v2.0 swagger specification. only simple function v2.0 specification. not yet realize senior function, but is enough to generate api docs for rails project, and just need to declare paraneters then anything other is same with a rails project. I'm open to accepting a PR on  this. Contact me if you are interested in helping with that effort - thanks!

## Example usage

Here is an extract of the DSL from a user controller API class:

```ruby
swagger_controller :users, "User Management"

swagger_api :index do
  summary "Fetches all User items"
  notes "This lists all the active users"
  param :query, :page, :integer, :required, "Page number"
end
def index
  # do anything like you do in rails
  render json: {status: 200, msg: 'ok', data: {page_num: params[:page]}}
end

swagger_api :create do
  summary "create a user"
  param :query, :nickname, :string, :required, "nickname"
  param_list :query, :gender, :string, :required, "gender", [ "male", "female" ]
  param :query, :email, :string, :required, "email"
  param :query, :phone, :string, :required, "phone"
  response :unauthorized
  response :not_acceptable
end
def create
  # do anything like you do in rails
  render json: {
    status: 200,
    msg: 'ok',
    data: {
      nickname: params[:nickname],
      gender: params[:gender],
      email: params[:email],
      phone: params[:phone]
    }
  }
end
```

## Installation

Add this line to your application's Gemfile:

    gem 'swagger-doc-rails'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install swagger-doc-rails

## Usage

### Create Initializer

Create an initializer in config/initializers (e.g. swagger_docs.rb) and define your APIs:

```ruby
Swagger::Docs::Config.register_apis({
  "1.0" => {
    # the extension used for the API
    :api_extension_type => :json,
    # the output location where your .json files are written to
    :api_file_path => "public",
    # the URL base path to your API
    :host => "localhost:3000",
    # if you want to delete all .json files at each generation
    :clean_directory => true,
    # # Ability to setup base controller for each api version. Api::V1::SomeController for example.
    :parent_controller => Api::ApplicationController,

    :controller_base_path => 'api/',
    # # add custom attributes to api-docs
    :attributes => {
      :info => {
        "title" => "Swagger test",
        "version" => '1.0.0',
        "description" => "siple description.",
      }
    }
  }
})
```

#### Configuration options

The following table shows all the current configuration options and their defaults. The default will be used if you don't supply your own value.

<table>
<thead>
<tr>
<th>Option</th>
<th>Description</th>
<th>Default</th>
</tr>
</thead>
<tbody>

<tr>
<td><b>api_extension_type</b></td>
<td>The extension, if necessary, used for your API - e.g. :json or :xml </td>
<td>nil</td>
</tr>

<tr>
<td><b>api_file_path</b></td>
<td>The output file path where generated swagger-docs files are written to. </td>
<td>public/</td>
</tr>

<tr>
<td><b>base_path</b></td>
<td>The URI base path for your API - e.g. api.somedomain.com</td>
<td>/</td>
</tr>

<tr>
<td><b>base_api_controller / base_api_controllers</b></td>
<td>The base controller class your project uses; it or its subclasses will be where you call swagger_controller and swagger_api. An array of base controller classes may be provided.</td>
<td>ActionController::Base</td>
</tr>

<tr>
<td><b>clean_directory</b></td>
<td>When generating swagger-docs files this option specifies if the api_file_path should be cleaned first. This means that all files will be deleted in the output directory first before any files are generated.</td>
<td>false</td>
</tr>

<tr>
<td><b>formatting</b></td>
<td>Specifies which formatting method to apply to the JSON that is written. Available options: :none, :pretty</td>
<td>:pretty</td>
</tr>

<tr>
<td><b>camelize_model_properties</b></td>
<td>Camelizes property names of models. For example, a property name called first_name would be converted to firstName.</td>
<td>true</td>
</tr>

<tr>
<td><b>parent_controller</b></td>
<td>Assign a different controller to use for the configuration</td>
<td></td>
</tr>

</tbody>
</table>

#### Custom resource paths`(PR #126)

```ruby
class Api::V1::UsersController < ApplicationController

  swagger_controller :users, "User Management", resource_path: "/some/where"
```

### DRYing up common documentation

Suppose you have a header or a parameter that must be present on several controllers and methods. Instead of duplicating it on all the controllers you can do this on your API base controller:

```ruby
class Api::BaseController < ActionController::Base
  class << self
    Swagger::Docs::Generator::set_real_methods

    def inherited(subclass)
      super
      subclass.class_eval do
        setup_basic_api_documentation
      end
    end

    private
    def setup_basic_api_documentation
      [:index, :show, :create, :update, :delete].each do |api_action|
        swagger_api api_action do
          param :header, 'Authentication-Token', :string, :required, 'Authentication token'
        end
      end
    end
  end
end
```

### Run rake task to generate docs

```
rake swagger:docs
```

Swagger-ui JSON files should now be present in your api_file_path (e.g. ./public/api/)

#### Additional logging for generation failures

Errors aren't displayed by default. To see all error messages use the ```SD_LOG_LEVEL``` environment variable when running the rake task:

```
SD_LOG_LEVEL=1 rake swagger:docs
```

Currently only constantize errors are shown.

Errors are written to ```$stderr```. Error logging methods can be found in ```Config``` and can be overridden for custom behaviour.

### Advanced Customization

#### Inheriting from a custom Api controller

By default swagger-docs is applied to controllers inheriting from ApplicationController.
If this is not the case for your application, use this snippet in your initializer
_before_ calling Swagger::Docs::Config#register_apis(...).

```ruby
class Swagger::Docs::Config
  def self.base_api_controller; Api::ApiController end
end
```

#### Custom route discovery for supporting Rails Engines

By default, swagger-docs finds controllers by traversing routes in `Rails.application`.
To override this, you can customize the `base_application` config in an initializer:

```ruby
class Swagger::Docs::Config
  def self.base_application; Api::Engine end
end
```

If you want swagger to find controllers in `Rails.application` and/or multiple
engines you can override `base_application` to return an array.

```ruby
class Swagger::Docs::Config
  def self.base_application; [Rails.application, Api::Engine, SomeOther::Engine] end
end
```

Or, if you prefer you can override `base_applications` for this purpose. The plural
`base_applications` takes precedence over `base_application` and MUST return an
array.

```ruby
class Swagger::Docs::Config
  def self.base_applications; [Rails.application, Api::Engine, SomeOther::Engine] end
end
```

#### Transforming the `path` variable

Swagger allows a distinction between the API documentation server and the hosted API
server through the `path` variable (see [Swagger: No server Integrations](https://github.com/wordnik/swagger-core/wiki/No-server-Integrations)). To override the default swagger-docs behavior, you can provide a `transform_path`
class method in your initializer:

```ruby
class Swagger::Docs::Config
  def self.transform_path(path, api_version)
    "http://example.com/api-docs/#{api_version}/#{path}"
  end
end
```

The transformation will be applied to all API `path` values in the generated `api-docs.json` file.

#### Precompile

It is best-practice *not* to keep documentation in version control. An easy way
to integrate swagger-docs into a conventional deployment setup (e.g. capistrano,
chef, or opsworks) is to piggyback on the 'assets:precompile' rake task. And don't forget
to add your api documentation directory to .gitignore in this case.

```ruby
#Rakefile or lib/task/precompile_overrides.rake
namespace :assets do
  task :precompile do
    Rake::Task['assets:precompile'].invoke
    Rake::Task['swagger:docs'].invoke
  end
end
```

### Output files

api-docs.json output:

```json
{
  "swagger": "2.0",
  "host": "localhost:3000",
  "basePath": "/api/",
  "schemes": [
    "http"
  ],
  "paths": {
    "/users": {
      "get": {
        "summary": "Fetches all User items",
        "description": "User Management",
        "parameters": [
          {
            "name": "page",
            "in": "query",
            "required": true,
            "description": "Page number",
            "type": "integer"
          },
          {
            "name": "nested_id",
            "in": "query",
            "required": true,
            "description": "Team Id",
            "type": "integer"
          }
        ],
        "responses": {
          "200": {
            "description": "User Management",
            "schema": {
            }
          }
        }
      }
    },
    "/users/": {
      "post": {
        "summary": "创建一个新用户",
        "description": "User Management",
        "parameters": [
          {
            "name": "nickname",
            "in": "query",
            "required": true,
            "description": "昵称",
            "type": "string"
          },
          {
            "name": "gender",
            "in": "query",
            "required": true,
            "description": "性别",
            "type": "string"
          },
          {
            "name": "email",
            "in": "query",
            "required": true,
            "description": "邮箱",
            "type": "string"
          },
          {
            "name": "phone",
            "in": "query",
            "required": true,
            "description": "手机",
            "type": "string"
          }
        ],
        "responses": {
          "200": {
            "description": "User Management",
            "schema": {
               "type": "pbject",
               "items": {
                 "properties": {
                   "nickname": {
                       "type": "string"
                   },
                   "gender": {
                       "type": "string"
                   },
                  "email": {
                       "type": "string"
                   },
                  "phone": {
                       "type": "string"
                   }
                 }
               }
            }
          }
        }
      }
    },
    "/users/{id}": {
      "get": {
        "summary": "Fetches a single User item",
        "description": "User Management",
        "parameters": [
          {
            "name": "id",
            "in": "path",
            "required": false,
            "description": "User Id",
            "type": "integer"
          }
        ],
        "responses": {
          "200": {
            "description": "User Management",
            "schema": {
            }
          }
        }
      }
    },
  },
  "info": {
    "title": "Swagger test",
    "version": "1.0.0",
    "description": "simple description."
  }
}

```

### integration swagger-ui
```sh
  cd app_name/public
  git clone https://github.com/swagger-api/swagger-ui.git
  vim swagger-ui/dist/index.html
  # change url to http://localhost:3000/api-docs.json (any url of yours)
  window.swaggerUi = new SwaggerUi({
    url: "http://localhost:3000/api-docs.json",
    dom_id: "swagger-ui-container",
    supportedSubmitMethods: ['get', 'post', 'put', 'delete'],
    ... ... ...})
```
open your browser and type http://localhost:3000/swagger-ui/dist/index.html, then you can see the
swagger-ui home page which lists your apis

## Thanks to our contributors

Welcome contributors for making swagger-doc-railss even better.

## Related Projects

**[@swagger-docs](https://github.com/richhollis/swagger-docs)**

**[@swagger-ui](https://github.com/swagger-api/swagger-ui.git)**

## Contributing

When raising a Pull Request please ensure that you have provided good test coverage for the request you are making.

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
