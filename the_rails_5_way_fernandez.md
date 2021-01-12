# The Rails 5 Way
*Obie Fernandez*

# Chapter 1: Rails Configuration and Environments

## Bundler
* `bundle install` updates all dependencies named in your Gemfile to the latest versions that do not conflict with other dependencies.

### Gem Locking
* Once a lock file is created, Bunder will only load specific versions of gem that you were using at the moment the Gemfile was locked.

### Packaging Gems
* Package all gems in the `vendor/cache` directory using `bundle pacakge`.
* Run `bundle install --local` installs gems from the package rather than connecting to third-party sources.
* Non-rails scripts are executed with `bundle exec` to get a properly initialized RubyGems environment.

### Binstubs
* A script containing an executable that runs in the contect of the bundle.
  * No need to prefix with `bundle exec`.
* Default stubs are:
  * `bin/bundle`
  * `bin/rails`
  * `bin/rake`
  * `bin/setup`
  * `bin/spring`
  * `bin/update`
* To add a binstub of a commonly used executable: `bundle binstubs some-gem-name`.
  * Creates a binstub in the `bin` folder.

## Startup Scripts
* Three files set up the entire Rails stack.

### `config/environment.rb`
* Loads `application.rb`, thedn runs initializer scripts.

### `config/boot.rb`
* Required by `application.rb` to set up Bundler and load paths for RubyGems.
* `Bundler.setup` adds all gems to the Rails load path but doesn't require them, i.e. "lazy loading".

### `config/application.rb`
* Loads Rails gems and gems for the current Rails.env and configures the application.
* Changes to these files require a server restart.
* By replacing `require rails/all`, you can cherry-pick only components needs by your application.
* The create of a module specifically for your application lays a foundation for running multiple Rails applications in the same executable Ruby process.

## Default Initializers
* `config/initializers` contains default initializer scripts.
* Add your own initializer scripts to this dir to expand default initialization behaviour.

### `config/initializers/application_controller_renderer.rb`
* Templates are rendered in a context with supplementary data. E.g. if a view template needs to create a URL, how will it know what hostname to use? Set those defaults here.

### `config/initializers/assets.rb`
* For customizing the behaviour of the Asset Pipeline.

### `config/initializers/backtrace_silencers.rb`
* Eliminates unhelpful lines from stack traces. E.g. libraries that you don't need to see traces for.
* Provides a mechanism to temporarily remove all silencers for debugging.

### `config/initializers/cookies_serializer.rb`
* Controls how Rails handles serialization to strings and back when setting cookies.
* It's advisable to only store strings and number in cookies, otherwise the conversion will have to be handled manually when reading the values on subsequent requests.

### `config/initializers/filter_parameter_logging.rb`
* Specify request parameter to redact from log files.

### `config/initializers/inflections.rb`
* Rails' `Inflector` class handles pluralization.
  * Lives inside Active Support `inflections.rb`.
* The initializer overrides default behaviour of `Inflector`.

### `config/initializer/mime_types.rb`
* Register MIME types that aren't supported by default.

### `config/new_framework_defaults.rb`
* Contains migration options to help migration to Rails 5 from earlier versions.
* Not present when creating new projects with Rails 5+.

### `config/initializers/session_store.rb`
* Rails session cookies are encrypted using an encypted cookie store.
* `session_store.rb` configures the session store of the application by setting its session store type and key.
* Session cookies are signed using `secret_key_base` set in `config/secrets.yml`.

### `config/initializers/wrap_parameters.rb`
* When submitting JSON to a controller, Rails wraps the parameters into a nested hash with the controller's name as the key so that the controller can treat JavaScript clients and HTML forms identically.
  ```ruby
  { "title": "The Rails 5 Way", "article" => {"title": "The Rails 5 Way"}}
  ```

## Other Common Initializers

### Generator Default Settings
* Used to override assumptions Rails generators make about your toolchain. E.g. swap template_engine or test_framework.

### Load Path Modifications
* By default, Rails looks in standard directories, such as those under `app/`. This is referred to as the load path.
* Add other directories to the load path using:
  ```Ruby
  config.autoload_paths += %W(#{config.root}/extras)
  ```

### Schema Dumper
* Every time you run a test, Rails dumps the schema of your development database and copies it to the test database using an auto-generated `schema.rb` script.
* Reverting to SQL dumping may be necesasry if you use constraints or db-specific column types, for exampe.
  ```Ruby
  config.active_record.schema_format = :sql
  ```

### Console
* Create console helpers be supplying a block to be evaluated when the Rails environment is loaded via the terminal.
  ```Ruby
  console do
    def angus
      User.find_by(email: "angus@gmail.com")
    end
  end
  ```

## Spring Application Preloader
* Spring is a preloader which keeps your application process running in the background during development. Without it, you would have to boot Rails from scratch every time you ran a test or rake task.
* Spring monitors folders `config` and `initializers` for changes and automatically restarts the application.
  * Also restarts if any gem dependencies are changed.
* Edit which files Spring monitors for changes in `config/spring.rb`.

## Development Mode
* Configuration settings specified in `config/environments` override those set in initializers.

### Automatic Class Reloading
* `config.cache_classes`: controls hot reloading. When `true`, Rails uses Ruby's `require` statement for class loading. When `false`, it uses `load`.
  * `require` executes a file and caches it. `load` always executes a file again.
* If Rails encounters a class or module that is not already defined, it guesses which files it should require:
  * If the class or module it not nested, insert and underscore between the constant's names and require a file of this name:
  ```Ruby
  EstimationCalculator => require "estimation_calculator"
  ```
  * If nested, insert underscores and require a file in the corresponding set of subdirectories:
  ```Ruby
  MacGyver::SwissArmyKnife => require "mac_gyver/swiss_army_knife"
  ```
  * You should rarely need to explicitly load Ruby code in your Rails applications if you follow these naming conventions.

### Eager Load
* `config.eager_load`: During development, project code and libraries are lazy loaded as they're needed.
  * Set to `true` in production. Provides a performance increase to web servers that copy on write.

### Error Reports
* Requests from localhost generator error messages that include debugging information such as line number and backtrace.
* `config.consider_all_requests_local` displays developer error messages even when the machine is remote.

### Caching
* The only time you want caching in development mode is when actually testing caching.
* Caching behaviour is determined by the presence of `tmp/caching-dev.txt` in addition to `config.action_controller.perform_caching`.

### Action Mailer Settings
* `config.action_mailer.raise_delivery_errors`: Rails assumes you don't want Action Mailer to raise delivery exceptions in development.
  * Toggle this to `true` if you want to send mail in development mode.
* Setting `config.action_mailer.perform_deliveries = false` means no delivery attempt is performed, but you can still see the mail in the log file to check that it looks correct.

### Assets Debug Mode
* In development, JavaScript and CSS are served separately in the order they were specified in their respective manifest files.
* `config.assets.debug = false` tells Sprokets to concatenate and preprocess all assets.
* Development mode momits logger output for asset requests. Turned on with `config.assets.quiet = false`.

### Missing Translations
* Rails views normally just print the key for missing translations. If you want an error, set `config.action_view.raise_on_missing_translations`.

## Test Mode
* `config.action_mailer.delivery_method = :test` accumulates sent emails in the `ActionMailer::Base.deliveries array`.

## Custom Environments
* You can create additional environments by cloning one of the existing environment files in `config/environments`, e.g. staging and deployment envs.
  * A triage env might use normal env settings for development mode but point its db connection to a production database.

## Production Mode

### Assets
* Assets are precompiled by the Asset Pipeline. All files included in `application.js` and `application.css` asset manifests are compressed and concatenated into files of the same name in `public/assets`.
* If an asset is requested that is not in `public/assets`, Rails throws an exception.
  * `config.assets.compile = true` enables live asset compilation as a fallback.
* To include additional files in precompilation, use `config.assets.precompile += %w(filename)`.

## Configuring a Database
* An old best practice is not to store `config/database.yml` in version control.
* Since Rails 4.1, you can configure Active Record with the env var `DATABASE_URL`, which is a connection string. This allows each developer (and the production env) to maintain their own copy of `config/database.yml` that is not stored in version control.

## Configuring Application Secrets
* `config/secrets.yml` is used to store sensitive data, such as access keys and passwords required for external APIs.
* Rails requires that `secret_key_base` is set for each environment.
* Replaced with `credentials.yml.enc` and `master.key` in Rails 5.2.
  * `credentials` is encrypted and safe to check into version control.
  * `master.key` must not be checked into version control.
  * All production secret values should be specified as environment variables.

## Logging
* Most programming contexts in Rails (models, controllers, view templates) have a `logger` attribute, which holds a reference to a logger conforming to the interface of `Log4r` or the default Ruby `Logger` class.
* A global logger can be accessed anywhere with `Rails.logger`.
* To create a new logger: `logger = Logger.new(STDOUT)`.

### Rails Log Files
* The `log` folder holds three log files corresponding to each of the environments.
* To clear large log files: `rake log:clear`.

### Tagged Logging
* To add tagged informated to logs, pass an array of one or more method names that respond to the `request` object to the `config.log_tags` configuration setting. E.g. `config.log_tags([:subdomain])`.
* Times reported by the logger have low accuracy. It's hard to measure the timing of something from within itself.
  * The percentage of time spent rendering vs. the database will not always sum to 100%.
* **Identification of N+1 select problems**: watch out for many separate SELECT statements with the only difference being the value of the primary key.
* Database access during rendering is bad practice, but there are many ways it can creep in (e.g. triggered by lazy loading of associations).

### `Rails::Subscriber.colorize_logging`
* Tells Rails whether to use ANSI codes to colorize logging.
* May complicate things if using syslog.


# Chapter 2: Routing

## The `routes.rb` file
* At runtime, the block given to `Rails.application.routes.draw` is evaluated in an instance of of `ActionDispatch::Routing::Mapper`.
* The routing system finds a pattern match for a URL it's trying to recognize or a parameters match for a URL it's trying to generate.
  * Traverses the routes in the order they're defined.

### Constraining Request Methods
* To constrain a route to more than one HTTP method, pass `:via` and array of verbs:
  ```Ruby
  match "products/:id" => "products#show", via: [:get, :post]
  ```
* Providing the deprecated `:any` options doesn't error; it sets the route to respond to a non-existent `ANY` HTTP method.

### Segment Keys
* Symbols like `:id` within route patterns are segment keys.
* It's possible to hardcode parameters into route definitions that don't have an effect on matching but are passed along with `params`.
  ```Ruby
  get "products/special" => "products#show", special: "true"
  ```

### Optional Segment Keys
* Parentheses are used to define optional segment keys:
  ```Ruby
  match ":controller(/:action(/:id(.:format)))", via: :any
  ```

### Defining Defaults
* Define defaults by supplying a hash for the `:defaults` option.
  ```Ruby
  get "photos/:id", to: "photos#show", defaults: {format: "jpg"}
  ```
  * Applies even to parameters not specified as dynamic segments of the route itself.
* For security reasons, you can't override defaults by changing the values in the `params` object.

### Redirect routes
* Code a redirect directly into a route definition:
  ```Ruby
  get "/foo", to: redirect("/bar")
  ```
* String interpolation can be used to relay parameters to the redirect argument:
  ```Ruby
  get "docs/:article", to: redirect("/wiki/%{article}")
  ```
* `redirect` can take a block, which receives the request params as its argument.
  ```Ruby
  match "/api/v1/:api",
    to: redirect { |params| "/api/v2/#{params[:api].pluralize}" },
    via: [:get, :post]
  ```
  * `do end` syntax for the block won't work, as the block would be passed to `match` instead of `redirect`.
* `redirect` accepts params such as `:status`.
  * Rails uses 301 status for redirects by default.
  * All options that work with a call to `url_for` work with a redirect (e.g. `:host`, `:port`, etc.
  * The `:path` parameter in conjunction with a wildcard and interpolation enables you to supply only the parts of the URL that need to change:
  ```Ruby
  get "stores/:name", to: redirect(status: 302, path: "/%{name}")
  get "stores/:name(*all)", to: redirect(status: 302, path: "/%{name}%{all}")
  ```
 * To encapsulate commonly used redirection code, redirect can accept an object that responds to a method `call`, which takes `params` and `request` as args and returns a `string`.
  ```Ruby
  get "accounts/name" => redirect(SubdomainRedirector.new("api"))
  ```
  * If you return a URL without a preceding slash, the URL is prefixed with the current `SCRIPT_NAME` environment variable. This is typically `/`, but may be different in a mounted engine or where the application is deploy to a subdirectory of a website.

### The Format Segment
* `.:format` matches a literal dot and a `format` segment key, e.g. `.json`.
* `respond_to` allows you to write actions to return different results according to the requested format.
  ```Ruby
  def show
    @product = Product.find(params[:id])
    respond_to do |format|
      format.html
      format.json { render json: @product.to_json }
      format.any
    end
  end
  ```
  * Requesting a format that isn't specified responds `406 Not Acceptable`.

### Routes as Rack Endpoints
* The value of the `:to` option is a Rack Endpoint:
  ```Ruby
  get "/hello", to: proc { |env| [200, {}, ["Hello, world"]] }
  ```
* The router is loosely coupled to controllers. The shorthand `items#show` syntax relies on the `action` method of the controller classes to return a Rack endpoint that executes the requested action.
  ```Ruby
  >> ItemsController.action(:show)
  => #Proc:...
  ```

### Accept Header
* Branching on a `repond_to` can also be triggered by setting the `Accept` header in the request.
  * There is no need to add the `.:format` segment too.
  * Difficult to get working correctly in the real world due to client/browser inconsistencies.

### Segment Key Constraints
* The `:constraint` option is used to match on the nature of the segment keys.
  ```Ruby
  get ":controller/show/:id" => :show, constraints: {:id => /\d+/}
  get ":controller/show/:id" => :show_error
  ```
* Implicit anchoring:
  * `/\d+/` doesn't match "foo32bar" because Rails implictly anchors it at both ends.
  * Adding explicit enchors `\A` and `\z` causes exceptions.
* For more powerful constraint checking, you can get full access to the `request` object by passing a block or any other object that responds to `call` to `:constraints`:
  ```Ruby
  get "records/:id" => "records#protected",
    constraints: proc { |req| req.params[:id].to_i < 100 }
  ```

### The Root Route
* Specifies what should happen when someone connects to the root of your website or routing namespace.
  ```Ruby
  root to: "welcome#index"
  root "user_sessions#new"
  ```
* Any static contect in the public directory hierarchy matching the URL scheme that you come up with results in the static content being served up by the web server instead of triggering the routing rules.

## Route Globbing
* For a route that matches everything after the second URI component, blog the route with an asterisk.
  ```Ruby
  get "items/list/*specs", controller: "items", action: "list"
  ```

## Named Routes

### Creating a Named Route
* `get "help" => "help#index", as: "help"`
* Test named routes in the console using teh special `app` object:
  ```Ruby
  >> app.clients_path
  => "/clients"
  ```
* Named routes save effort when generating URLs by bypassing the matching process.

### `name_path` versus `name_url`
* Named routes create at least two url helper methods: `_url` and `_path`.
* Redirects should specify a full URL.
* The Rails way is to prefer `_path` in most other situations.
  * Using `_path` in mailer templates is deprecated; use `_url` instead.

### Argument Sugar
* 
  ```Ruby
  get "auction/:auction_id/item/:id" => "items#show", as: "item"
  link_to "Auction of #{item.name}", item_path(auction, item)
  => /auction/5/item/11
  ```
  * Rails infers the ids of bother auction and item by calling `to_param` on any non-hash argument passed to named route helpers.
    * Args must be provided in the order in which their ids occur in the route's pattern string.
    * To use a description or slug rather than an id, override `to_param` on the relevant model:
      ```Ruby
      def to_param
        description.parameterize
      end
      ```
    * You'll need to make provisions to relocate the object from its new URL param, e.g. as a separate database column for `slug`.
    * Avoid using IDs in URLs: competitors can see how many records you have, and numeric consecutive IDs allow people to write spiders to steal your content. It's a window into your database.

## Scoping Routing Rules
* To reduce repetition of path and controller names:
  ```Ruby
  scope path: "/auctions", controller: :auctions do
    get "new" => :new
    get "edit/:id" => :edit
    get "pause/:id" => :pause
  end
  ```

### Controller
* The following definitions are identical:
  ```Ruby
  scope controller: :auctions do
  scope :auctions do
  ```
* The `controller` method can be used instead of `scope` as syntactic sugar:
  ```Ruby
  controller :actions do
  ```

### Path Prefix
* The `scope` method accepts a `:path` option, or it can interpret a string as its first parameter to mean a path prefix:
  ```Ruby
  scope path: "/auctions" do
  scope "/auctions" do
  ```
* `:path` can also use symbols:
  ```Ruby
  scope :auctions, :archived do # routes to /auctions/archived
  ```

### Name Prefix
* Name prefixes modify how named route URL helper methods are generated:
  ```Ruby
  scope :auctions, as: "admin" do
    get "new" => :new, as: "new_auction" # generates admin_new_auction_url`
  end
  ```

### Namespaces
* `namespace` rolls module, name prefix and path prefix settings into one declaration:
  ```Ruby
  namespace :auctions do
    get "new" => :new
    get "edit/:id" => :edit
    post "pauses/:id" => :pause
  end
  ```

### Bundling Constraints
* If you find yourself repeating similar segment key contraints in related routes, you can bundle them together using the `:constraints` option of the `scope` method:
  ```Ruby
  scope controller: :auctions, constraints: {:id => /\d+/} do
    get "edit/:id" => :edit
    post "pause/:id" => :pause
  end
  ```
* To effect only a subset of routes in a scope:
  ```Ruby
  scope path: "/auctions", controller: :auctions do
    get "new" => :new
    constraints id: /\d+/ do
      get "edit/:id" => :edit
      post "pause/:id" => :pause
    end
  end
  ```
* To enable module reuse, supply `constraints` with an object that has a `matches?` method:
  ```Ruby
  class DateFormatConstraint
    def self.matches?(request)
      request.params[:date] =~ /A\d{4}-\d\d-\d\d\z/
    end
  end

  constraints(DateFormatConstraint) do
    get "since/:date" => :since
  end
  ```
  * If malformed, reponds with 404 instead of causing an exception.

### Direct Routes
* From Rails 5.1, you can create custom URL helpers that link to a static URL:
  ```Ruby
  direct(:apple) { "http://www.apple.com" }
  >> apple_url
  => "http://www.apple.com"
  ```

## Listing Routes
* `rails routes` outputs all application routes.
* `rails routes -c products` filters by a controller parameter.
* Visit `/rails/info/routes` in development to see a complete list of routes.


# Chapter 3: REST

## Resources and Representations
* The REST style characterizes communication between system components as a series of requests to which the responses are representations of resources.

## The Standard RESTful Controller Actions
* The default request method is GET.
* In a `form_tag` or `form_for` call, the POST method with be used automatically.
* When you need to, you can specify a request method along wit hthe URL generated by the named route:
  ```Ruby
  link_to "Delete", auction_path(auction), method: :delete
  ```
  * Depending on the helper method (e.g. `form_for`), you might have to put the method in a nested hash:
    ```Ruby
    form_for "auction", url: auction_path(auction), html: {method: :patch} do |f|
    ```
    * Not normally required – Rails automatically uses POST or PATCH as appropriate if you pass an object to form helpers.

### The PATCH and DELETE Cheat
* A PATCH or DELETE request is actually a POST request with a hidden field called `_method` set to "patch" or "delete".

### Limiting Routes Generated
* `except` and `only` options limit the routes generated by `resources`.

## Singular Resource Routes
* A singleton resource at the top level of your routes can be appropriate when there's only one resource of its type for the user, e.g. a profile or session: `resource :profile`.
* Generates all routes apart from `index`.

## Nested Routes
*
  ```Ruby
  resources :auctions do
    resources :bids
  end
  ```
  * Creates auction resources, but also creates nested RESTful routes for bids: `auction_bids_url`, `new_auction_bid_url`, etc.
  * Requires an auction to be passed whenever bid helpers are called:
    ```Ruby
    link_to "See all bids", auction_bids_path(auction)
    ```
* You can nest to any depth. Each level adds one to the number of arguments you have to supply to the nested route helpers.
* Instead of specifying the route to be used in a view helper, you can pass an object:
  ```Ruby
  link_to "Delete this bid", [auction, bid], method: :delete
  ```
  * Since the object is an array, Rails infers that the route is nested and identifies the correct helper based on the order and class names of the objects.

### REST Controller Mappings
* Routes are mapped to controllers based on the name of the resource.

### Considerations
* Nested resources make it easy to enforce permissions and contect-based constraints. A nested resource should only be accessible in the context of its parent resource, and it's easy to enforce that in code based on the way you load the nested resource using the parent's Active Record association.
  ```Ruby
  auction = Auction.find(params[:auction_id])
  bid = auction.bids.find(params[:id])
  ```

### Deep Nesting?
* The helper methods for routes nested more than two levels deep become long and unwieldy.
* As a rule of thumb, limit to two levels of nesting.

### Shallow Routes
* The `shallow` option can be used to shorten the URLs of nested routes by leaving of the parent component:
  ```Ruby
  resources :auctions, shallow: true do
    resources :bids do
      resources :comments
    end
  end
  ```
    * Produces routes like `/bids/:bid_id/comments` rather than `/auctions/:auction_id/bids/:bid_id/comments`.

## Routing Concerns
* `config/routes.rb` can get repetitous when nested routes are shared across multiple resources. The `concern` method can encapsulate shared behaviour across routes.
  ```Ruby
  concern :commentable do
    resources :comments
  end
  
  concern :image_attachable do
    resources :image_attachments, only: :index
  end

  resources :auctions, concerns: [:commentable, :image_attachable] do
    resources :bids
  end

  resources :bids, concerns: :commentable
  ```

### RESTFUL Route Customizations
* Customize the default set of RESTful routes with `member` for routes that operate on a single resource and `collection` for routes that operate on collections:
  ```Ruby
  resources :auctions do
    resources :bids do
      member do
        get :retract
      end

      collection do
        match :terminate, via: [:get, :post]
      end
    end
  end
  ```
* Simplified form:
  ```Ruby
  resources :auctions do
    match :retract, via: [:get, :post], on: :member
    match :terminate, via: [:get, :post], on: :collection
  end
  ```

### Custom Action Names
* The `path_names` option allows you to specify alternative name mappings:
  ```Ruby
  resources :projects, path_names: {new: "nuevo", edit: "cambiar"}
  ```
* URLs change but the names of the generated helper methods do not.

### Mapping to a Different Controller
* Use the `:controller` option:
  ```Ruby
  resources :photos, controller: "images"
  ```

### Routes for New Resources
* Specify routes that only apply to resources that have not yet been saved:
  ```Ruby
  resources :reports do
    new do
      post :preview
    end
  end
  ```
  * Results in the route `preview_new_report`.

### Considerations for Extra Routes
* It is typically better not to add additional actions and stick to the RESTful defaults by considering what should actually become a separate resource with a controller of its own.

## Controller-Only Resources
* For cases where the resources you're representing can be encapsulated in a controller but not a model. E.g. `resource :session`.

## Different Representations of Resources
* As a client of REST services, you don't actually retrieve a resource from a server; you retrieve a representation of a resource.

### Responders Gem
* The Responders Gem offers a more concise way to respond to a variety of formats in the same controller.
  ```Ruby
  class AuctionsController < ApplicationController
    respond_to :html, :xml, :json
    def index
      @auctions = Auction.all
      respond_with(@auctions)
    end
  end
  ```
* When a request comes in, the responder attempts the following (given a `.json` extension on the URL):
  1. Attempt to render an associated view with a `.json` extension.
  2. If no view exists, call `to_json` on the object passed to `responds_with`.
  3. If the object doesn't respond to `to_json`, call `to_format` on it.
* For nested and namespaced resources, you must pass dependencies to the `respond_with` method:
  ```Ruby
  respond_with(@user, :managed, @client)
  ```

### Formatted Named Routes
* To generate a link to an alternative representation of a resource, pass an extra argument to the named route helper:
  ```Ruby
  link_to "XML version of this auction", auction_path(@auction, :xml)
  ```

## The RESTful Action Set

### Index
* It is better to eliminate conditional branching in controller actions where possible. The more dependence on server-side state we can eliminate (and instead accomplish using only the request and router) the better. For example:
  ```Ruby
  resources :auctions do
    resources :bids do
      get :manage, on: :collection
    end
  end

  resources :bids

  class BidsControllers < ApplicationController
    before_action :check_authorization, only: :manage

    def index
      @bids = Bid.all
    end

    def manage
      @bids = auction.bids
    end

    protected

    def auction
      @auction ||= Auction.find(params[:auction_id])
    end

    def check_authorization
      auction.authorized?(current_user)
    end
  end
  ```
  * Note, if these are truly different resources, they should be given their own controller. There will likely be other actions that need to be authorized and scoped to the current user.

### Show
* As with index actions, it's good to make show actions as public as possible and offload the administrative and privileged views onto either a different controller or a different action.

### Destroy
* `DELETE` submissions are dangerous. It is hard to trigger them accidentally (e.g. by a crawler). When you specify a `DELETE` method, JavaScript that submits the form is bound to the delete link, along with a `rel="nofollow"` attribute on the link. Bots don't submit forms and shouldn't follow `nofollow` links.

### `new` and `create`
* The new action doesn't do much and often does nothing. It can even be left out and Rails will still figure out which view to render, but the controller will need a helper method:
  ```Ruby
  protected

  def auction
    @auction ||= current_user.auctions.build(params[:auction])
  end
  helper_method :auction
  ```

### `edit` and `update`
* `form_for` checks whether the object passed to it has been persisted and automaticallly applies the `PATCH` method as appropriate.


# Chapter 4: Working with Controllers

## Rack
* Rack is a modular interface for handling web requests. It abstracts the handling of HTTP requests and responses into a single `call` method.
  ```Ruby
  class HelloWorld
    def call(env)
      [200, {"Content-Type" => "text/plain"}, ["Hello, world!"]]
    end
  end
  ```
* An HTTP request invokes the call method and passes in a hash of environment (request) variables and returns a three-element array consisting of the status, a hash of response headers and the body of the response.
  * Similar to CGI (Common Gateway Interface), a specification dating back to the 1990's about how a web server should communicate with executable scripts and programs to serve dynamic content.
* Classes that satisfy Rack's call interface can be chained together as middleware filters.
  * Rack itself includes filter classes that do things such as logging and exception handling.
* Much of Action Controller is implemented as Rack middleware modules.
  * `rake middleware`

### Configuring Your Middleware Stack
* Your application object allows you to access and manipulate the Rack middleware stack during initialization, via `config.middleware`:
  ```Ruby
  module Example
    class Application < Rails::Application
      ...
      config.middleware.use Rack::ShowStatus
    end
  end
  ```
* Rack middleware classes need to have an explicit initializer method, even if they don't require runtime arguments.
* The methods of `config.middleware` give you control over the order in which the middleware stack is configured:
  * `insert_after(existing_mw, new_mw, args)`
  * `insert_before(existing_mw, new_mw, args)`
  * `delete(mw)`
  * `swap(existing_mw, new_mw, args)`: Swaps a specified middleware from the stack with a new class.
  * `use(new_mw, args)`: Adds the desired middleware to the end of the middleware stack.

## Action Dispatch: Where It All Begins
* As of Rails 3, dispatching of requests was extracted into its own sub-component of Action Pack called Action Dispatch. It contains classes that interface the rest of the controller system to Rack.

### Request Handling
* The entry point to a request is an instance of `ActionDispatch::Routing::RouteSet`, the object on which you call `draw` at the top of `config/routes.rb`.
* The route set chooses the rule that matches, and calls its Rack endpoint.
  * A route like `get 'foo', to: 'foo#index'` has a dispatcher instance associated to it, whose call method ends up executing `FooController.action(:index).call`.
* Prior to Rails 4, the RouteSet would convert its path data into an array of regular expressions and iterate through it to try to match the given URLs. This was slow.
  * Nowadays, the core routing module in ActionDispatch is called Journey.
  * Journey uses a generalized transition graph (GTG) and non-deterministic finite automata (NFA) to match routes.
* The route RouteSet can call any other type of Rack endpoint, like a Sinatra app, a redirect macro or a bare lambda.
  * In these cases, no dispatcher is involved.

### Getting Intimate with the Dispatcher
* To see everything contained in the `ActionDispatch::Response` object returned from `call`:
  `y AppName::Application.call(env)`
    * `y` prints output as YAML.


## Render unto View...
* You don't actually need to define a controller action as long as you have a template that matches the action name.
* To reload the Rails console: `reload!`.
  * Instances of Active Record objects will also need to be reloaded using their individual `reload` methods.

### When in Doubt, Render
* At the end of every controller action, if nothing else is specified, the default behaviour is to render the template whose name matches the name of the controller and action.

### Explicit Rendering
* A controller action can render a differnent template by calling `render` explicitly.
* Capture a call to render without sending it to the browser with `render_to_string`.

### Rendering Another Action's Template
* `render action: "new"`
  * Renders the template of the new action. Does not execute the new method.

### Rendering a Different Template Altogether
* `render template: "/products/index.html.haml"`
  * It's not necessary to explicitly pass a hash with `:template` because it is the default option.
* All of the following work when called from `ProductsController`:
  ```Ruby
  render "/products/index.html.haml"
  render "products/index.html.haml"
  render "products/index.html"
  render "products/index"
  render "index"
  render :index
  ```
* The `:template` option only works with a pth relative to the template root (`app/views`).
  * Access templates outside the application using `render file: "path"`.
    * Beware of security issues when using unsantized paths from user input.

### Rendering a Partial Template
* `render partial: "product"` renders `app/views/products/_product.html.haml`
* `render partial: "shared/product` renders `app/views/shared/_product.html.haml`
* If you pass an object to `render :partial`, Rails will use its class name to find a partial to render:
  * `render partial: @product`
  * `render @product`
  * `render "product"`
* Parital rendering from a controller is mostly used in conjunction with XHR calls that need to dynamically update segments of an already displayed page.

### Rendering HTML
* It's poor practice, but you can send an HTML string back to the browser using the `:html` option:
  `render html: "<strong>Not Found</strong>".html_safe`
    * Without `html_safe`, Rails will complain about a potential security vulnerability.

### Rendering Inline Template Code
* Occasionally, you may want to send a snippet of HTML generated using template code that's too small to merit its own partial.
  * `render inline: "%span.foo #{@foot.name}", type: "haml"`
  * Contentious – violates separation of concerns between MVC layers.
  * Rails treats `:inline` code exactly like a view template.
  * Default type is `ERb`.

### Accessing Helpers in the Controller
* Access view helpers in the controller via the `helpers` method of the base controller class.
  * Returns a module containing all the helper methods available to the view.
  *
    ```Ruby
    module UsersHelper
      def full_name(user)
        user.first_name + user.last_name
      end
    end

    class UsersController < ApplicationController
      def update
        @user = User.find(params[:id])
        if @user.update(user_params)
          notice = "#{helpers.full_name(@user)} is successfully updated"
          redirect_to user_path(@user), notice: notice
        else
          render :edit
        end
      end
    end
    ```
  * Using helpers inside controllers is occasionally very useful for generating dynamic flash messages.

### Rendering JavaScript
* Rails can execute arbitrary JavaScript expressions in your browser:
  `render js: "alert('Hello, world!')"`
  * Sent to the browser with a MIME type of text/javascript.

### Renering Text
* `render plain: "Submission accepted"`.

### Rendering Raw Body Output
* Send raw content back to the browser without setting any content type using the `:body` option.
* Use only if you don't care about content type.
  * Defaults to `text/html`.
* `:plain` or `:html` is usually more appropriate.

### Rendering Other Types of Structured Data
* `:json`:
  * Active Record has built-in support for converting to JSON: `render json: @record`.
    * As long as the parameter responds to `to_json`, Rails will call it for you.
    * Additional options passed to `render :json` are included in the invocation of `to_json`, e.g. `render json: @projects, include: :tasks`
* `:xml`:
  * Active Record has built-in support for converting to XML: `render xml: @record`.
    * As long as the parameter responds to `to_xml`, Rails will call it for you.
    * Additional options passed to `render :xml` are included in the invocation of `to_xml`, e.g. `render xml: @projects, include: :tasks`

### Default Rendering Policies
* If a template exists for the controller action, but not in the right format (or variant, etc.), then an `ActionController::UnknownFormat` is raised.
  * Typically results in "204 No Content" response
  * Often this happens because you forgot to end your method with a call to `redirect_to`.
  * Many browsers ignore 204 and do nothing, making missing templates hard to diagnose.

### Rendering Nothing
* Indicate your intention to render nothing by using the `head` method, which takes a status code: `head :ok`.
  * `:head` takes a hash of headers to be included with the response:
    `head :created, location: auction_path(@auction)`.
    * The Location header indicates the URL to redirect the page to. It only has meaning when served with `3xx` or `201` status code.

### Rendering Options
* `:content_type`:
  * Rails doesn't validate the format of the MIME identifier you pass to the `:content_type` option.
* `:layout`:
  * Specify whether you want a layout template to be rendered by passing a boolean or the name of a template:
    ```Ruby
    render layout: false
    render layout: "login"
    ```
    * `true` raises an `ArgumentError`.
* `:status`:
  * Sets the response status.

## Additonal Layout Options
* To reuse layouts for multiple actions, specify layout options at the controller class level:
  ```Ruby
  class EventController < ActionController::Base
    layout "events", only: [:index, :new]
    layout "global", expect: [:index, :new]
  end
  ```

## Redirecting
* `redirect_to`: tells the user agent (the browser) to perform a new request for a different URL.
* HTTP 1.0 has two redirect status codes: 301 Moved Permanently and 302 Moved Temporarily.
  * A permanent redirect meant that the user agent should forget about the old URL and use the new one from now on, updating any references it might have kept.
  * A temporary redirect was one-time only
* Problematic for POST requests: what method should be used for the redirect request?
  * Temporary redirects were used for both redirecting to a view of resource that had just been modified in the original POST request and also for redirecting the entire original POST request to a new URL that would take care of it.
* HTTP 1.1 introduced 303 See Other and 207 Temporary Redirect.
  * 303 See Other tells the user agent to perform a GET request, regardless of what the original verb was.
  * 307 Temporary Redirect uses the same verb as the original request.
* Most browsers handle 302 Moved Temporarily and 303 See Other identically, and Rails still uses 302 in `redirect_to`
* To force a 307 redirect, e.g. to continue processing a POST request in a different action, assign a path to `response.header["Location"]` and render with `render status: 307`.

### The `redirect_to` Method
* redirect_to(target, response_status = {})
* `target` can be one of several things:
  * Hash: the URL is generated by calling `url_for` with the hash.
  * Active Record object: the URL is generated by calling `url_for` with the hash.
  * String starting with protocol like http://: Used directly as the target URL.
  * String without a protocol: The current protocol and host is prepended to the argument.
* Flash messages can be assigned as part of the redirection.
  * There are accessors for the commonly used flash names `alert` and `notice`, as well as a general-purpose flash bucket:
    ```Ruby
    redirect_to post_url(@post), alert: "Watch it!"
    redirect_to post_url(@post), status: :found, notice: "Pay attention"
    redirect_to post_url(@post), status: 301, flash: { updated_post_id: @post.id }
    redirect_to :atom, alert: "Something serious happened"
    ```
  * Register your own flash types with `ActionController::Flash.add_flash_types`:
    ```Ruby
    class ApplicationController
      add_flash_types :error
    end
    ```
* Render statements don't halt execution of the controller action method.
  * To prevent `DoubleRenderError`, explicitly call return after `redirect_to` or `render`.

### The `redirect_back` method
* `redirect_back` returns the suer to the page they just came from.
* The location to return to comes from the `HTTP_REFERER` header.
  * Not guaranteed to be set, so you must provide a `:fallback_location` parameter:
    `redirect_back fallback_location: root_path`

## Controller/View Communication
* Rails implements controller-to-view data handoffs through instance variables.
  * When the template is rendered, the contect is an instance of `ActionView::Base`. That instance does not have access to those of the controller object.
  * Rails loops through the controller object's varaibles and creates a matching instance variable for the view object.
  * To avoid this sharing of data, see the Decent Exposure library.

## Action Callbacks
* Action callbacks enable controllers to run shared pre- and post-processing code for their actions.
* Make action callback methods protected or private, otherwise they might be callable as public actions on your controller via the default route.
  * Prior to Rails 5, you could declare that a method should never be dipatched with `hide_action`.
* Action callbacks have accessto `request`, `reponse` and all the instance varaibles of the controller, and can set instance variables used by the requested controller action.

### Action Callback Inheritance
* Controller inheritance hierarchies share action callbacks downward.
  * To ensure an action callback runs no matter what, define it in `ApplicationController`, from which all other controllers inherit.
    ```Ruby
    class ApplicationController < ActionController::Base
      after_action :compress
    ```
* Subclasses can add or skip action callbacks without affecting the superclass.
* If an action callback calls `render` or `redirect_to`, subsequent action callbacks are never called.
  * Known as halting the action callback chain.

### Action Callback Types
* Three forms: method reference (symbol), external class, or block.
* Action Callback Classes:
  * Using an external class makes generic callbacks reuable (e.g. output compression).
  * The name of the class method should match the type of callback desired (before, after, around).
  *
    ```Ruby
    class OutputCompressionActionCallback
      def self.after(controller)
        controller.response.body = compress(controller.response.body)
      end
    end

    class NewspaperController < ActionController::Base
      after_action OutputCompressionActionCallback
    end
    ```
  * The method of the action callback class is passed the controller instance. It gets full access to the controller and can manipulate it as it sees fit.
* Inline Method:
  *
    ```Ruby
    class WeblogController < ApplicationController::Base
      before_action do
        redirect_to new_user_session_path unless authenticated?
      end
    end
    ```
  * The block is executed in the context of the controller instance using `instance_eval`.
    * Has access to both reponse and request objects and convenience methods for params, sessions, template and assigns.

### Action Callback Chain Ordering
* `before_action` and `after_action` appends the specified callbacks to the existing chain.
* `prepend_before_action` and `prepend_after_action` prepend the callbacks to the chain.

### Around Action Callbacks
* Execute both before and after the wrapped action.
* Within the action callback, `yield` runs the controleller action.
* To use a block as an `around_action`, pass a blck taking both the controller and the action parameters as args:
  ```Ruby
  around_action do |controller, action|
    logger.debug "before #{controller.action_name}"
    action.call
    logger.debug "after #{controller.action_name}"
  end
  ```
  * You can't call `yield` from blocks, so explictly invoke `call` on the action parameter.

### Action Callback Chain Skipping
*
  ```Ruby
  class SignupController < ApplicationController
    skip_before_action :authenticate
    skip_after_action :method_name
  end
  ```

### Action Callback Conditions
* Action callbacks can be limited to specific actions with `:only` and `:except` options.

### Action Callback Chain Halting
* Around action callbacks halt the request unless the action block is called. I.e. if it returns before yielding, action callbacks after will not be run.

## Streaming

### ActionController::Live
* The `ActionController::Live` module is a controller mixin that enables controller actions to stream data to the client.
* Adds an I/O-like interface object called `stream` to the `reponse` object.
  * Calling `write` on `repsonse.stream` immediately sends the written data to the client.
  * `close` explicitly closes the stream.
*
  ```Ruby
  class StreamingController < ApplicationController
    include ActionController::Live

    # Streams about 180 MB of generated data to the browser.
    def stream
      response.headers["Content-Type"] = "text/event-stream"
      10_000_000.times do |i|
        response.stream.write "Event #{i} just happened\n"
      end
    ensure
      reponse.stream.close
    end
  end
  ```
* All actions executed from `ActionController::Live`-enabled controllers are run in a separate thread. The controller action code must be threadsafe.
* A concurrent Ruby webserver, such as Puma, is required.
* Headers must be added to the response before anything is written to the client.
* Streams must be closed when finished, otherwise a socket may be left open indefinitely.

### Support for EventSource
* The EventSource API presents a simple alternative to XHR polling and web socks, with the limitation that the client can only listen to updates; it cannot publish anything.
* Ideal for updating read-only views like dashboards, social media updates, etc.
* Due to incomplete browser support, the EventSource JavaScrupt library is recommended over using the API directly.
* EventSource and live streaming (together with Redis) make it trivial to implement things like chat:
  ```Ruby
  class ChatChannelController < ApplicationController
    include ActionController::Live

    def show
      response.headers["Content-Type"] = "text/event-stream"
      redis = Redis.new
      redis.psubscribe("channel-#{params[:id]}:*") do |on|
        on.pmessage do |subscription, event, data|
          response.stream.write "data: #{data}\n\n"
        end
      end
    resuce IOError
      # Client disconnected
    ensure
      redis.quit
      reponse.stream.close
    end
  end
  ```
  ```JavaScript
  source = new EventSource(chatChannelUrl(id));
  source.addEventListener("message", function(e) {
    appendChatMessage(e.data)
  });
  ```

### Steaming Templates
* Streaming views enables views to be rendered as they are processed, including only running Active Record scoped queries when they are needed.
* Normal rendering order is reversed: layout is rendered first, then each part of the template is processed.
* Enabled with `render stream: true`.
  * Can only be used to stream view templates.

### Streaming Buffers and Files
* `send_data(data, options = {})`
  * Send textual or binary data in a buffer to the user as a named file.
  * Options:
    * `:filename`: suggest a filename for the browser to use.
    * `:type`: defaults to `application/octet-stream`.
    * `:disposition`: `inline` or `attachment` (download - default).
    * `:status`: HTTP status.
* `send_file(path, options = {})`
  * Send an existing file to the client using `Rack::SendFile` middleware.
  * Intercepts the response and replaces it with a webserver-specific `X-Sendfile` header.
  * The webserver is responsible for writing the file contents to the client, not Rails (webservers are optimized for these tasks).
  * Options:
    * `:filename`: suggest a filename for the browser to use. Defaults to `File.basename(path)`.
    * `:type`: defaults to `application/octet-stream`.
    * `:disposition`: `inline` or `attachment` (download - default).
    * `:status`: HTTP status.
    * `:url_based_filename`: Determines whether the browser should guess the filename from the URL, which is necessary for I18n filenames on certain browsers.
      * Overridden by `:filename`.
  * Security considerations:
    * The `send_file` methods can be used to read any file accessible to the user running the Rails server processed.
    * Sanitize any path parameter derived from user input.
  * There are few reasons to serve static files through Rails. Unless you are protecting content, it is recommended to cache thefile after sending it. The easiest way is to copy a newly generated file to the `public` directory after sending it:
    ```Ruby
    public_dir = File.join(Rails.root, "public", controller_path)
    FileUtils.mkdir_p(public_dir)
    FileUtils.cp(filename, File.join(public_dir, filename))
    ```
    * All subsequent views of the resource will be served by the webserver.

## Variants
* Variants provide the ability to render different HTML, JSON and XML templates based on various criteria.
*
  ```Ruby
  class ApplicationController < ActionController::Base
    before_action :set_variant

    protected

    def set_variant
      request.variant = :mobile if request.user_agent =~ /iPhone/i
    end
  end
  ```
* In a controller action, we can explicitly respond to variants like any other format:
  ```Ruby
  class PostsController < ApplicationController
    def index
      ...
      respond_to do |format|
        format.html do |html|
          html.mobile do # renders app/views/posts/index.html+mobile.haml
            @mobile_only_variable = true
          end
        end
      end
    end
  end
  ```
* If not `respond_to` block is delcared within your action, Action Pack will automatically render the correct variant template if one exists in your views directory.
* Variants can be used for many things, such as rolling out features to a certain group of users or even A/B testing a template.


# Chapter 5: Working with Active Record
* The Active Record pattern maps one domain class to one database table, and one instance of that class to each row of that database.

## The Basics
* Models are subclasses of `ApplicationRecord`.
* The Annotate gem can maintain schema information in a comment at the top of each model.
* Association methods (e.g. `has_many`) are class methods of Active Record's `Base` class which are executed in the context of the model class, adding attributes that are subsequently available to model instances.

### Setting Names Manually
* `table_name` and `primary_key` enable you to set the name of each feature:
  ```Ruby
  class Client < ApplicationRecord
    self.table_name = "CLIENT"
    self.primary_key = "CID"
  end
  ```

## Defining Attributes
* At runtime, the Active Record model class reads its attributes directly from the database defintion.
* Rails 5's Attributes API makes it easy to set defaults in the model rather than in migrations, keeping domain logic centralized:
  ```Ruby
  class TimesheetEntry < ApplicationRecord
    attribute :category, :string, default: "n/a"
  end
  ```
* Rails does not use instance variables to store model attributes.
  * `read_attribute` and `write_attribute` can be used to override default accessors.
    ```Ruby
    def category
      read_attribute(:category) || "n/a"
    end

    def message=(txt)
      write_attribute(:message, txt + " in bed")
    end
    ```
  * Attributes can also be accessed with hash notation.
    ```Ruby
    def category
      self[:category] || "n/a"
    end
    ```

## CRUD: Creating, Reading, Updating, Deleting

### Creating New Active Record Model Instances
* Use `new_record?` or `persisted?` to determine if an Active Record object is saved.
* Active Record constructors take an optional block, which can be used to do additional initialization and is executed after any passed-in attributes are set on the instance.
  ```Ruby
  c = Client.new do |client|
    client.name = "Angus
  end
  ```
* `create` creates a new instance, persists it to the database and returns it in one operation.

### Reading Active Record Objects
* `ActiveRecord::RecordNotFound` is raised when a record with the requested ID can't be found.
* `first_or_initialize` looks for the first instance of a record with the specified properties, and, if not found, initializes (but doesn't save) a new instance with the provided parameters.
* `find` understands arrays of IDs, and raises `RecordNotFound` if it can't find all of them.
* `take` returns the first N objects from the table, relying on the database implementation's ordering.

### Reading and Writing Attributes
* Use symbols when the string is a name for something and a string when it's a value.
* `attributes` returns a hash with each attribute and its corresponding value as returned by `read_attribute`.
  * `attributes` doesn't invoke custom readers when accessing its values.
* Pass a hash to `attributes=` to bulk update an object's attributes.
  * `attributes=` does invoke custom writers.

### Accessing and Manipulating Attributes Before They Are Typecast
* Active Record connection adapters are the classes that implement database-specific behaviour.
  * They fetch results as strings, which Rails converts.
* To manipulate raw attribute data, use `<attribute_name>_before_type_cast` in the model.

### Reloading
* The `reload` method does a query to the database and resets the attributes of an Active Record object.
  * You can pass the options than can be passed to `find`, such as `lock: true`

### Cloning
* `clone` produces a shallow copy.
* No associates are copied.
  * Accessing associations will re-query the database.

### The Query Cache
* The query cache is a hash stored on the current thread, one for every active database connection.
  * Most Rails processes have one DB connection.
* If the same SQL is used again, the cached result set is used to generate a new set of model objects.
* Save and delete operations clear the cache.
* Clear the cache manually with `Model.connection.clear_query_cache`.

### Updating
* Update can take a single id or a list of ids.
* The update **class** method invokes validation first, but returns the object whether or not validation passes.
  * Check validity with `valid?`.
* It is simpler to use the update instance method, which returns a boolean.

### Updating By Condition
* The class method `update_all` takes two parameters: the change to be made and the match conditions, expressed as part of a where clause.
  * Returns number of records updated.
  * `Project.update_all({manager: "Ron Campbell"}), technology: "Rails")

### Updating Specific Attributes
* `update_attribute` updates a single attribute and saves the record **without running validations**.
* `update_column` and `update_columns` take key-value pairs, skips validations, skips callbacks and does not bump the `updated_at` timestamp.
* Attribute writers are automatically created for associations. For example, on a model which `has_many :users`, `user_ids` can be used to set all user IDs at once.

### Saving Without Updating Timestamp
* `save(touch: false)` doesn't bump the `updated_at` timestamp.

### Convenience Updaters
* `increment`, `decrement` and `toggle` do what they suggest.
* The bang variant of each calls `update_attribute` instead of `update`.

### Touching Records
* 
  ```Ruby
  user.touch # sets updated_at to now without firsting callbacks or validation
  user.touch(:viewed_at) # sets viewed_at and updated_at to now
  ```
* Touching a child record in a relation affects the parent's `updated_at` too.

### Readonly Attributes
* Read-only attributes must be set before the record is persisted:
  ```Ruby
  class Customer < ApplicationRecord
    attr_readonly :social_security_number
  end
  ```
* Trying to update a read-only attribute fails silently.
* List read-only attributes with `readonly_attributes`.

### Deleting and Destroying
* The `destroy` methods removes the object from the database and prevents you from modifying it again with `freeze`.
* Saving a destroyed object fails silently.
  * Check for destruction with `destroyed?`.
* With `destroy!`, `ActiveRecord::RecordNotDestroyed` is raised if the record can't be destroyed.
* `destroy` and `delete` can be called as class methods, passing an ID or array of IDs.
* `delete` uses SQL directly and does not load any instances.
  * Faster, but doesn't trigger `before_destroy` callbacks or the destruction of dependent associatons.

## Database Locking

### Optimistic Locking
* Optimistic locking is the strategy of detecting and resolving collisions if they occur.
* Recommended in situations where collisions are infrequent.
* Database records are never actually locked.
* To enable, add a `lock_version` column with a default value of `0`.
* If the same record is loaded as two different model instances and saved differently, the first instance will win the update. The second will cause an `ActiveRecord::StaleObjectError`
* The initializer setting `Rails.application.config.active_record.locking_column` controls the name of `lock_version`.
* Handling `StaleObjectError`:
  * At minimum, let the user know why their update failed:
    ```Ruby
    def update
      timesheet = Timesheet.find(params[:id])
      timesheet.update(params[:timesheet])
      # redirect somewhere
    rescue ActiveRecord::StaleObjectError
      redirect_to [:edit, timesheet], flash: { error: "Timesheet was modified while you were editing it." }
    end
    ```
* Disadvantages:
  * Update operations are slower as the lock version must be checked.
  * Potential for bad UX, as the user finds out only after the save attempt.

### Pessimistic Locking
* Works in conjunction with transactions:
  ```Ruby
  Timesheet.transaction do
    t = Timesheet.lock.first
    t.approved = true
    t.save!
  end
  ```
* Calling `lock!` on a model instance calls `reload(lock: true)` behind the scenes.
  * Discards unsaved changes.
  * To unlock, call `lock!(false)`.
* Pessimistic locking happens at the database level.
  * SELECT statement generated will have a `FOR UPDATE` (or similar) clause added.
* Easier to implement, but can lead to situations where one Rails process is waiting on another to release a database lock and not serving requests.
  * Less of an issue in Rails than on other platforms as a database transaction is never persisted across more than one HTTP request.
* Keep pessimistic-locking transactions small and fast-executing.
  * Beware situations where there are many users competing for access to a record that takes a long time to update.

## Querying
* `ActiveRecord::Relation`: a chainable object that is lazy evaluated against the database only when the actual records are needed.
* Active Record's querying and relationship behaviour are implemented using the Arel relational algebra gem (part of Rails).

### `where(*conditions)`
* Parameters are automatically sanitized to prevent SQL injection.
* An `IN` clause is created if you associated an array of values with a particular key.
* The string form can be used for statements that don't involve data originating outside your app.
  * Most useful for `LIKE` comparisons, greater/less than, and SQL functions not part of ActiveRecord, like querying into Hstore and JSON columns in Postgres.
* Dates, booleans and arrays are coerced into SQL representations automatically.
* Bind variables:
  * Name bind variables by replacing `?` with a symbol and supplying a hash with keys that correspond to the symbols.
    ```Ruby
    Product.where("name = :name", name: "Angus")
    ```
* `nil` conditions:
  * The first example does not work, but the second does:
    ```Ruby
    User.where("email = ?", nil)
    # => SELECT * FROM users WHERE (email = NULL)

    User.where(email: nil)
    # => SELECT * FROM users WHERE (users.email IS NULL)
    ```
  * In Postgres, `NULL` is equal to nothing, not even itself.

### `order(*clauses)`
* The SQL spec defaults to ascending order if the asc/desc option is ommitted.
  * Note, there is no ordering prescribed by the SQL standard if an `ORDER BY` clause is omitted.
* The value of the `:order` option is not validated by Rails.
  * You can pass any code that is understood by the underlying database, at the expense of portability. E.g. `Timesheet.order("RANDOM()")`.
    * Note: Randomly ordering large databases performs terribly on most databases.
    * A performant way to get a random record in Ruby: `Timesheet.limit(1).offset(rand(Timesheet.count)).first`.

### `take(number)` and `skip(number)`
* `take`/`limit` sets a limit on the number of records returns.
* `skip`/`offset` must be chained to `take`/`limit` and specifies the number of rows to skip in the result set (0-indexed).
* The Kaminari gem is recommended for pagination.

### `select(*clauses)`
* By default, Active Record generates `SELECT * FROM` clauses, but this can be changed:
  ```Ruby
    b = BillableWeek.select("mon_hrs + tues_hrs AS two_day_total").first
    b.two_day_total
    # => 16
  ```
* Any columns not specified by the query will not be present on the resulting objects.

### `from(*tables)`
* `from` modifies the table name portion of the SQL statements generated by Active Record.
  * Providing a custom value to include extra tables for joins or to reference a view or subquery:
    `Topic.select("title").from(Topic.approved)"`
  * Overriding generated aliases with your own:
    `Topic.select("a.title").from(Topic.approved, :a)`

### `group(*args)`
* `group` specifies a `GROUP BY` clause to add to the query.
* Generally combined with `select`, since valid SQL requires that all selected columns in a grouped `SELECT` be either aggregate functions or columns.
  * Aggregate columns may sometimes be strings if Active Record doesn't try to typecast them. Use `to_i` or `to_f`.
*
  ```Ruby
  users = Account.select("name, SUM(cash) as money").group("name").to_a
  ```

### distinct
* Serves the function of `DISTINCT`.

### having(*clauses)
* Performs the function of SQL `HAVING` on group queries.

### eager_load(*associations)
* Grabs all data, including associations, in a single query using `LEFT OUTER JOIN` and loads it into memory.

### preloads(*associations)
* Uses separate queries to load first the models and then the requested associations for all the models at once, avoiding a separate query for each individual model's associations.
*
  ```Ruby
  User.preload(:auctions).to_a
  # User Load (0.1ms) SELECT "users".* FROM "users"
  # Auction Load (0.2ms) SELECT "auctions".* FROM "auctions"
  # WHERE "auctions"."user_id" IN (1, 2)
  ```

### includes(*associations)
* Eliminates N+1 queries by letting you specify which associations to load in advance.
* When no `WHERE` clause is supplied, invokes `preloads` to load associations.
* When a `WHERE` is supplied, delegates to `eager_load`, which uses an (expensive) `LEFT OUTER JOIN`.
*
  ```Ruby
  # First-degree association
  users = User.where(login: "mack").includes(:billable_weeks)

  # Second-degree association
  clients = Client.includes(users: [:avatar])

  # Second- and third-degree associations
  Client.includes(users: [:avatar, {timesheets: :billable_weeks}])
  ```

### references(*table_names)
* Used to indicate that a related table is referenced by some part of the SQL expression under construction.
* Only needed when Active Record can't figure out what to join on its own, e.g.:
  ```Ruby
  User.includes(:auctions).where("auctions.name = ?", "Lumina")
  # => ActiveRecord::StatementInvalid: no such column: auctions.name

  User.includes(:auctions).where("auctions.name = ?", "Lumina").references(:auctions)
  # Works!
  ```
* A better alternative to `references` is to use the hash syntax for `where`:
  ```Ruby
  User.includes(:auctions).where(auctions: {name: "Lumina"})
  ```
* `includes` and its sister methods work with association names, but `references` requires actual table names.

### joins(expression)
* Uses an `INNER JOIN` to join the specified tables together.
* Does not eager load, so is not suitable for use where the joined tables must be accessed.
* Accepts nested hash notation or a string:
  ```Ruby
  User.joins(:auctions)
  User.joins(auctions: [:bids])
  Buyer.select(:*, "count(carts.id) as cart_count")
    .joins("LEFT OUTER JOIN carts ON carts.buyer_id = buyers.id")
    .group("buyers.id")
  ```

### left_outer_join
* Used when a `LEFT OUTER JOIN` is needed for querying purposes alone, in which case it is more intention-revealing than `includes`.
* Aliased to `left_join`.

### `find_or_create_by(attributes, &block)
* Finds the first record using the relation and the given attributes. If none are found, saves a new record using the attributes and the where clause values of the relation.
  ```Ruby
  User.active.find_or_create_by(first_name: "Buster")
  # => Finds or creates an active user called Buster.
  ```
* `create_with` specifies attributes to using in creation but not the query.
  ```Ruby
  User.create_with(active: true).find_or_create_by(first_name: "Buster")
  # => Creates an active user called Buster if no user of any status called Buster exists.
  ```
* Newly created records will be passed to the block for modification before being saved.
* Not atomic. Executes a `SELECT`, then executes an `INSERT` if nothing matches the query.
  * If there are other threads or processes, a race condition could result in two similar records being saved.
  * In the case of similar records with unique constraints being created concurrently, you can catch the resulting exception and retry:
    ```Ruby
    begin
      CreditAccount.transaction(requires_new: true) do
        CreditAccount.find_or_create_by(user_id: user.id)
      end
    rescue ActiveRecord::RecordNotUnique
      retry
    end
    ```
* Returns an unsaved record if validation fails.
  * `find_or_create_by!` raises an exception.

### `find_or_initialize_by(attributes, &block)
* Like `find_or_create_by` but calls `new` rather than `create` and returns an unsaved record.

### `reset`
* The `next` access of a relation hits the database again instead of using cached values.
  * Contrast with `reload`, which hits the database again immediately.

### `explain`
* Runs `EXPLAIN` on the query triggered by this relation and returns the result as a string.

### `extending(*modules, &block)`
* Specifies one or more modules with methods that will extend the scope.
* Returns a relation for further chaining.
*
  ```Ruby
  module Pagination
    def page(number)
      # pagination code
    end
  end

  scope = Model.all.extending(Pagination)
  scope.page(params[:page])
  ```
* Takes a block which acts as an anonymous module:
  ```Ruby
  scope = Model.all.extending(Pagination) do
    def per_page(number)
      # pagination code goes here
    end
  end
  ```

### `exists?`
* Takes arguments of `find` but returns a boolean indicating whether the query has results.

### `none`
* A chainable relation that causes a query to return `ActiveRecord::NullRelation` (an implementation of the Null Object pattern).
* Used in instances where you have a method that returns a relation, but there is a condition in which you don't want the database to be queried.
  * The rest of the chained conditions will still work, removing the need to check whether the object you're working with is a relation.
  *
    ```Ruby
    def visible
      case role
      when :reviewer
        Post.published
      when :bad_user
        Post.none
      end
    end
    ```

### `lock`
* Specifies locking settings for the query.

### `readonly`
* Marks returns objects as readonly.

### `reorder`
* Replaces any existing order defined on a relation.

### `reverse_order`
* Reverses an existing order clause.

### `rewhere(conditions)`
* Changes a previously set where condition for a given attribute instead of appending to that condition.
* Rarely used.
* In the console, `where_values_hash` tells you what conditions are in effect for a given scope or relation.

### `scoping(&block)`
* Defines a scope for all queries in the provided block.
  ```Ruby
  Comment.where(post_id: 1).scoping do
    Comment.first
  end
  # => SELECT * FROM comments WHERE comments.post_id = 1 LIMIT 1
  ```

### `unscope(*args)`
* Useful for removing an unwanted relation without reconstructing the entire relation chain.
  * To remove an order clause:
    ```Ruby
    Member.order("name DESC").unscope(:order)
    ```
  * To remove specific `:where` values, pass a hash:
    ```Ruby
    Member.where(name: "Tyrion", active: true).unscope(where: :name)
    ```

### merge(other)
* Merges in the conditions from another relation or array.
* Returns the intersection of both sets of conditions.
*
  ```Ruby
  Post.recent.joins(:comments).merge(Comment.where(editor_pick: true))
  ```
* `other` can also be a `Proc`, whose evaluation context is the relation you're merging into:
  ```Ruby
  editor_pick = -> { where(editor_pick: true) }
  post.comments.latest.merge(editor_pick)
  ```
  * Useful when extract shared logic to reuse across contexts.

### or(other)
* Generates `OR` expressions.
*
  ```Ruby
  Member.where(name: "Tyrion").or(Member.where(family: "Lannister").first)
  ```

### load
* Loads the relation from the database and returns it.
* Used in extremely rare cases where it is necessary to load a relation during its construction.

### cache_key
* Returns a key used to identify records fetched by this query.

## Ignoring Columns
* To make columns invisible to Active Record (e.g. legacy tables that are inconveniently named):
  ```Ruby
  class User < ApplicationRecord
    self.ignore_columns = %w(associations)
  end

## Connections to Multiple Databases in Different Models
* Connections are created via `ActiveRecord::Base.establish_connection`.
* Connections are retrived via `ActiveRecord::Base.connection`.
* All classes inheriting from `ActiveRecord::Base` will use this connection by default.
* To configure per-class databases, create the database entry in `database.yml`, then call `establish_connnection` from the model.
  * For subclasses of the model to use the new DB, the model itself must be marked as an abstract class, or Rails will consider the subclasses to be using single-table inheritance (STI).
  *
    ```Ruby
    class LegacyProjectBase < ApplicationRecord
      establish_connection :legacy_database
      self.abstract_class = true
      ...
    ```
* Rails keeps database connections in a pool inside the `ActiveRecord::Base` class instance.
  * It is a hash indexed by the Active Record class.
* When a connection is needed, the `retrive_connection` method walks up the class hierarchy from the model to `ActiveRecord::Base` until a matching connection is found.

## Using the Database Connection Directly
* Sometimes useful for custom scripts and ad-hoc testing.
  ```Ruby
  def execute_sql_file(path)
    File.read(path).split(";").each do |sql|
      begin
        ActiveRecord::Base.connection.execute("#{sql}\n") unless sql.blank?
      resuce ActiveRecord::StatementInvalid
        $stderr.puts "warning: #{$!}"
      end
    end
  end
  ```
* To execute methods directly on the underlying database drive library, using `raw_connection`:
  `rc = ActiveRecord.Base.connection.raw_connection`.

### The `DatabaseStatements` Module
* `ActiveRecord::ConnectionAdapters::DatabaseStatements` mixes a number of methods into the connection object that make it possible to work with the database directly instead of using Active Record models.
* The following methods take place on the connection object.
  * `begin_db_transaction()`: Begins a transaction and turns off Active Record's autocommitting behaviour.
  * `commit_db_transaction()`: Commits the transaction (and turns on autocommitting).
  * `delete(sql_statement)`: Executes an SQL `DELETE` statement and returns the number of row affected.
  * `execute(sql_statement)`: Executes the SQL provided.
    * This method is abstract in `DatabaseStatements` and is overridden by the database adapter.
    * The result set object corresponds to the adapter in use.
  * `insert(sql_statement)`: Executes an SQL `INSERT` and returns the last autogenerated ID.
  * `reset_sequence!(table, column, sequence = nil)`: Updates the named sequence to the maximum value of the specified table's column (Postgres and Oracle).
  * `rollback_db_transaction()`: Rolls back the currently active transaction (and turns on autocommitting).
    * Called automatically when a transaction block raises an exception or returns false.
  * `select_all(sql_statement)`: Returns an array of record hashes with the column names as keys and column values and values.
  * `select_one(sql_statement)`: Returns only the first row of the result set as a hash with column names and keys and column values as values.
    * Does not add a limit clause automatically. Consider adding one.
  * `select_value(sql_statement)`: Works like `select_one` but returns only the first column value of the first row of the result set.
  * `select_values(sql_statement)`: Returns an array of the values of the first column in all rows of the result set.
  * `update(sql_statement)`: Executes the statement and returns the number of rows affected. Works exactly like `delete`.

### Other Connection Methods
* `active?`: Indicates whether the connection is active and ready to perform queries.
* `adapter_name`: Returns the human-readable name of the adapter.
* `disconnect!` and `reconnect!`: Closes the active connection or closes and opens a new one in its place.
* `raw_connection`: Provides access to the underlying database connection. Useful when you need to execute a statement using features of the driver that aren't exposed in Active Record (e.g. function creation).
* `supports_count_distinct?`: `true` for all adapters but SQLite.
* `supports_migrations?`
* `tables`: Lists the tables in the underlying schema, including those that aren't exposed as Active Record models, like `schema_info` and `sessions`.
* `verify!(timeout)`: Lazily verify the connection, calling `active?` only if it hasn't been called for `timeout` seconds.

## Custom SQL Queries
* Active Record's `find_by_sql` class method takes an SQL query string and returns an array of Active Record objects based on the results.
  * Predates `Relation`.
* Using SQL directly reduces database portability.

### Preventing SQL Injection Attacks
* Rails sanitizes SQL, provided you parameterize your query.
* Active Record executes queries using the `connection.select_all` method, iterating over the resulting array of hashes, and invoking your model's `initialize` method for each row in the result set.

## Other Configuration Options
* The following options can be configured on `Rails.application.config.active_record` in an initializer:
  * `default_timezone`: Tells Rails whether to use `Time.local` or `Time.utc` when pulling dates and times from the database.
    * Defaults to `:local`.
  * `logger`: Accepts a logger conforming to the interface of `Log4r` or the default Ruby `Logger` class,which is passed to any new database connections made.
    * Retrieved by called `logger` on an Active Record model class or instance.
    * Set to `nil` to disable logging.
  * `primary_key`: Used to toggle `:uuid`.
  * `schema_format`: Specifies that format to use when dumping the database schema with rake tasks.
    * Use `:sql` to dump as SQL, or `:ruby` to dump as an `ActiveRecord::Schema` file.
      * Using `:sql` reduces portability.
  * `schema_migrations_table_name`: Set a string to be used as the name of the schema migrations table.
  * `store_full_sti_class`: Specifies whether Active Record should store the full constant name including namespace when using single-table inheritance (STI).
  * `warn_on_records_fetched_greater_than`: Warns if queries fetch too many results.


# Chapter 6: Active Record Migrations

## Creating Migrations
* `rails generate migration NAME [field[:type][:index]...] [options]`.
* The model and scaffolding generators create migrations unless you specific `--skip-migration`.

### Generator Magic
* If the migration name is of the form `CreateXXX` followed by a list of column names and types, a migration creating the table XXX with the columns listed will be generated.
* If the migration name is of the form `AddXXXToYYY` or `RemoveXXXFromYYY` followed by a list of column names and types then a migration containing the appropriate `add_column` and `remove_column` statements will be created.
* Create indexes on new columns with `rails g migration AddPartNumberToProducts part_number:string:index`.
* Join tables are generated if `JoinTable` is part of the name:
  `rails g migration CreateJoinTableCustomerProduct customer product`

### Sequencing
* A record of migrations that have been run is kept in the `schema_migrations` table maintained by Rails.

### Rolling Back
* To roll back to an earlier version of the schema, pass a version number to roll back to:
  `rails db:migrate VERSION=2016123170510`.

### Redo
* `rails db:migrate:redo` rolls back one step and re-migrates in one command.

### Reversible Operations
* The `reversible` acts similiarly to older `up` and `down` methods.
*
  ```Ruby
  def change
    reversible do |dir|
      dir.up do
        execute <<-END
          CREATE FUNCTION add(integer, integer) RETURNS integer
            AS "select $1 + $2;"
            LANGUAGE SQL
            IMMUTABLE
            RETURNS NULL ON NULL INPUT;
        END
      end

      dir.down do
        execute "DROP FUNCTION add"
      end
    end
  end
  ```
  * The migrations API doesn't know anything about customs functions, so we have to talk to the database connection directly with `execute`.

### Irreversible Operations
* Migrations that are destructive should raise an `ActiveRecord::IrreversibleMigration` exception in their `reversible` down block.
* E.g., an integer field can be converted to a string, but can't be converted back again:
  ```Ruby
  def change
    reversible do |dir|
      dir.up do
        change_column :clients, :phone, :string
      end

      dir.down { raise ActiveRecord::IrreversibleMigration }
    end
  end
  ```

### `create_table(name, options, &block)
* Assumes we want an autoincrementing, integer-typed primary key.
  * `id` doesn't need to be specified in the migration to create the table.
* To disable the primary key:
  ```Ruby
  create_table :ingredients_recipes, id: false do |t|
  ...
  end
  ```
* To rename the primary key:
  ```Ruby
  create_table :ingredients_recipes, id: :clients_id do |t|
    t.column :name, :string
    ...
  ```
* Options:
  * `force: true` tells the migration to drop the table being created if it already exists
  * `:options` allows you to append database-specific instructions to the SQL statement, e.g. character set, collation, etc.
  * `temporary: true` creates a temporary table that only exists during the current connection to the database.
* You can remove old migration files to keep `db/migrate` manageable. The database should be recreated with `db:reset`/`db:load`, to load directly from the schema.

### `create_join_table(*table_names)
* `create_join_table(:ingredients, :recipes)`
  * Creates a table named `ingredients_recipes` with no primary key.
* Options:
  * `:table_name`: specify a custom name for the join table.
  * `:column_options`: extra options to append to the foreign key column definitions. E.g., using UUIDs instead of integers.
  * `:options, :temporary and :force`: Equivalent to those of `create_table`.

### API Reference
* The following methods are available in the context of `create_table` and `change_table`:
  * `change(column_name, type, options = {})`: Changes the column's definition according to the new options.
  * `change_default(column_name, default)`: Set a new default value for the column.
  * `column(column_name, type, options = {})`: Adds a new column to the table.
    * Shorthand: call by type: `t.string(:first_name)`.
  * `index(column_name, options = {})`: Adds a new index. `column_name` can be a symbol or an array of symbols.
    * `:name`: Override the default index name.
    * Supports Postgres partial indices: `add_index(:clients, :status, where: "active")`.
    * Supports Postgres expression indices with optional operator classes:
      ```Ruby
      def change
        add_index(
          :users,
          "lower(last_name) varchar_pattern_ops",
          name: "index_users_on_name_unique",
          unique: true
        )
      end
      ```
        * `varchar_pattern_ops` is useful for fields you know you will be doing `LIKE` or regex queries on. It changes the way the index analyses the column from being based on a locale-specifc collation to a character-by-character B-tree.
          * Be careful to also create normal indices if needed.
  * `belongs_to(*args)` and `references(*args)`: Aliases that add a foreign key column to a model.
    * Optionally adsd a `_type` column if `:polymorphic` is true.
    * Rails 5 automatically creates an index for each foreign key. Pass `index: false` to disable.
  * `remove(*column_names)`: Removes the specified columns.
  * `remove_index(options = {})`: Removes an index using either the `:column` or `:name` option.
  * `remove_references(*args)` and `remove_belongs_to(*args)`: Removes a reference. Removes a `type` column if `polymorphic: true` is passed.
  * `remove_timestamps`: Almost never used.
  * `rename(old_column_name, new_column_name)`: Renames a column.
  * `revert(migration_name, &block)`: Used to revert a migration from within another migration file. Also takes a block of directives to reverse on execution.
    * `revert CreateProductsMigration`.
  * `timestamps`: Adds Active Record-maintained `created_at` and `updated_at` columns.
    * Timestamps are automatically `NOT NULL` as of Rails 5.

## Defining Columns
* Columns can be added using either the column method inside the block of a `create_table` statement, or with the `add_column` method.

### Native Database Column Types
* Each connection adapter class has a `native_database_types` hash which establishes the mapping from Active Record column types to the database-native types.

### Column Options
* `default: value`: Sets a default to be used as the initial value of the column.
* `limit: size`: Adds a size parameter to string, text, binary, or integer columns.
  * Limits for strings refer to number of characters. Limits for others specify bytes.
* `null: false`: Adds a `NOT NULL` constraint.
* `index: true`: Adds an ordinary generated index to the column.
* `comment: text`: Adds a comment for the column that will be visible in `schema.rb` and certain kinds of database management software.
* Decimal precision:
  * `precision: number`: The total number of digits in the number.
  * `scale: number`: The number of digits to the right of the decimal point.

### Column Type Gotchas
* `:binary`: Storing binary can cause significant performance problems when the record is loaded into memory.
  * Use `select` to avoid loading the binary attribute.
* `:boolean`: Rails handles the mapping between `true` and `false` and the database implementation of the same. Always use the Ruby values.
* `:datetime` and `:timestamp`: The Ruby class that Rails maps to `datetime` and `timestamp` columns is `Time`.
  * In 32-bit environment, `Time` doesn't work for dates before 1902.
  * Rails falls back to `DateTime` if necessary, but it is slower that `Time`, being written in Ruby, not C.
* `:text`: Text columns can slow down query performance. It may be best to put `text` columns of performance critical routes in separate tables.

### Preserving Custom Data Types
* If a database-specific data type is critical to your project, use `config.active_record.schema_format = :sql` in `config/application.rb` to make Rails dump the native SQL DDL format rather than cross-platform Ruby code.

### "Magic" Timestamp Columns
* Rails will automatically timestamp create and updated operations if the table has `created_at/on` or `updated_at/on` columns.
* Turn off auto timestamping globally in an initializer with `ActiveRecord::Base.record_timestamps = false`.

### More Command-Line Magic
* Column type modifiers can be passed directly on the command line, enclosed by curly braces following the field type:
  `rails g migration AddDetailsToProducts price:decimal{5,2} supplier:references{polymorphic}.

## Transactions
* Turn off transactions for a particular migration with the `disable_dll_transaction!` class method of `ActiveRecord::Migration`.

## Data Migration
* Data migrations (changing the form or structure of the data in the database) should typically rely on raw SQL using the `execute` method of the database connection.

### Using SQL
* 
  ```Ruby
  class CombineNumberInPhones < ActiveRecord::Migration
    def change
      add_column :phones, :number, :string

      reversible do |dir|
        dir.up do
          execute "UPDATE phones SET number = CONCAT(area_code, prefix, suffix)"
        end

        dir.down { raise ActiveRecord::IrreversibleMigration }
      end

      remove_column :phones, :area_code
      remove_column :phones, :prefix
      remove_column :phones, :suffix
    end
  end
  ```
* The equivalent Ruby code is slower and less future-proof, and the structure of `Phone` will continue to evolve after this migration is written.

### Migration Models
* You can declare an Active Record model inside a migration script that is namespaced to that migration class and won't interfere with the actual model of that name.
* This avoids using the model directly. As your schema evolves, older migrations that use model classes break down and become ususable (e.g. if certain attributes which were present when the migration was written are removed).
* However, migration models can be hard to debug. Raw SQL is recommended for data migrations.
*
  ```Ruby
  class HashPasswordOnUsers < ActiveRecord::Migration
    class User < ActiveRecord::Base
    end

    def change
      reversible do |dir|
        dir.up do
          add_column :users, :hashed_password, :string
          User.reset_column_information
          User.find_each do |user|
            user.hashed_password = Digest::SHA1.hexdigest(user.password)
            user.save!
          end
          remove_column :users, :password
        end

        dir.down { raise ActiveRecord::IrreversibleMigration }
      end
    end
  end
  ```
  * `reset_column_information` is used when performing a data migration immediately after a table migration because Active Record caches column information on the first request.

## Database Schema
* `db/schema.rb` is generated every time you migrate and reflects the latest status of your database schema.
* To recreate your database on another server, use `db:schema:load`, don't run migrations from scratch.

## Database Seeding
* Use the bang version of create methods, otherwise you won't know if there are errors in the seed file.
  * Can also use `first_or_create` to make seeding idempotent, or call `Model.delete_all` prior to seeding.1
* Use the `seed.rb` file for data that is essential to all environments.
  * For dummy data only used on development/staging, create custom rake tasks under `lib/tasks`.

## Database-Related Tasks
* `db:forward` and `db:rollback`: migrate forward/back one migration.
* `db:migrate`: run all migrations, or migration up/down to the specified `VERSION` env var.
  * The version is specified as the timestamp portion of the filename.
* `db:migrate:down`: Invoke the down method of the specified migration only.
* `db:migrate:up`: Invoke the up method of the specified migration only.
* `db:migrate:redo`: Executes the down method of the latest migration, immediately followed by its `up` method.
  * Typically used for correct a mistake in the up method or to test that a migration is working.
* `db:migrate:reset`: Reset the DB for the current environment using migrations rather than the schema.
* `db:migrate:status`: Prints the status of all existing migrations.
* `db:reset` and `db:setup`: Creates the database for the current environment, loads the schema from `schema.rb`, then loads seed data. `reset` drops the database first.
* `db:schema:dump`: Creates a `db/schema.rb` file that can be portably used against any DB supported by ActiveRecord.
  * Happens automatically any time you migrate.
* `db:schema:load`: Loads `db/schema.rb` into the database for the current environment.
* `db:seed`: Loads the seed data from `db/seed.rb`.
* `db:structure:dump`: Dumps the database structure into an SQL file containing from DDL code in a format corresponding to your database driver.
* `db:test:prepare`: Loads the test schema by doing a `db:schema:dump` from development and `db:schema:load` into test.
  * Stnadard spec-related Rake tasks run `db:test:prepare` automatically.
* `db:version`: Returns the timestamp of the latest migration to be run.


# Chapter 7: Active Record Associations

## The Association Hierarchy
* `ActiveRecord::Associations::CollectionProxy` acts like a middleman between the object that owns the association and the actual associated object.

## One-to-Many Relationships
* 
  ```Ruby
  class User < ActiveRecord::Base
    has_many :timesheets
  end

  class Timesheet < ActiveRecord::Base
    belongs_to :user
  end
  ```
* Don't create associations that have the same name as instance methods of `ActiveRecord::Base`, e.g. `attributes` and `connection`. The association adds a method of that name to the model, overriding the inherited method.

### Adding Associated Objects to a Collection
* Appending an object to a `:has_many` collection automatically saves that object unless the parent object is not yet stored in the database.

## Belongs to Associations
* The `belongs_to` class method expresses a relationship from one Active Record object to a signle associated object for which it has a foreign key attribute.
  * I.e. the thing that calls `belongs_to` has the foreign key.

### Methods
* Reloading:
  * To explicitly reload and association, call `reload_<association_name>` on the parent.
* Building and creating related objects via the association:
  * `build_<association_name>` doesn't save the new object.
  * `create_<association_name>` does save the object.

### Options
* `autosave`: whether to automatically save the owning record whenever this record is saved.
  * Defaults to `false`.
  * If `true`, always save the associated object or destroy it if marked for destruction.
  * By default, associated objects are only saved if they are new records.
* `class_name`:
  * To create a `Timesheet` that both belongs to a `User` and must be approved by a different `User`:
    ```Ruby
    class Timesheet < ApplicationRecord::Base
      belongs_to :approver, class_name: "User"
      belongs_to :user
    ...
    ```
* `counter_cache`: makes Rails automatically update a counter field on the associated object with the number of belonging objets.
  * If `true`, the method is `<belonging_obj>s_count`.
  * Alternatively, supply your own name as a symbol: `counter_cache: :number_of_children`.
  * Useful if you expect a significant proportion of your association collections to be empty. When the counter cache is 0, Rails won't try to query the database.
  * The value of the counter cache column must be set to 0 by default in the database.
    * If the counter cache is altered on the database side, reset the stale value in Active Record with `Model.reset_counters(value, :association_name)`.
* `dependent`: Specifies that the associated owner record should be destroyed or deleted from the database.
  * `:destroy` causes the depedent's callbacks to fire; `:delete` does not.
* `foreign_key`: Specifies the name of the foreign key column that should be used to find the associated object.
  * Inferred by Rails if the name of the column is `<association>_id`.
  * `belongs_to :administrator, foreign_key: "admin_user_id"`
* `inverse_of`: Explicitly delcares the name of the inverse association in a bi-directional relationship. Enables Rails to return the same instance of an object in memory, no matter what side of the relationship it's accessed from.
  * Largely inferred from Rails >= 4.
    * Can't be inferred in cases where the options `:conditions`, `:through` and `:foreign_key` are passed to the association. Should be added explicitly in these circumstances.
  * Check whether `inverse_of` is set correctly with `Model.reflect_on_association(:posts).has_inverse?`.
* `polymorphic`: Polymorphic relations allow the related object to be one of several classes, such as `Business` or `Person`, but collected under a single association name, e.g. `Contacts`.
  * The class name of the related object is stored in the database along with its foreign key.
  * Any compatible model in the system can fill the role of the polymorphic association.
  *
    ```Ruby
    create_table :comments do |t|
      t.text :body
      t.references :subject, polymorphic: true

      # references can be used as a shortcut for the following two statements
      # t.integer :subject_id
      # t.string :subject_type

      t.timestamps
    end
    
    class Comment < ActiveRecord::Base
      belongs_to :subject, polymorphic: true
    end

    class ExpenseReport < ActiveRecord::Base
      belongs_to :user
      has_many :comments, as: :subject
    end

    class Timesheet < ActiveRecord::Base
      belongs_to :user
      has_many :comments, as: :subject
    end
    ```
* `primary_key`: Almost never needed. Specify a target other than the associated table's primary key to use as the foreign_key.
* `required`: `belongs_to` associations automatically add validation requiring the associated record to be present. If you're modeling an optional `belongs_to`, set this to false.
* `touch`: If `true`, touches the owning records `updated_at` timestamp when the associated record is saved.
  * Supply a specific timesstammppp column with `column_name`.
* `validate`: Tells Active Record to validate the owner record, but only in circumstances wherre it would normally save the owner record.

### Scopes
* Scopes work in much the same way as they do for Active Record queries:
  ```Ruby
  class Timesheet < ActiveRecord::Base
    belongs_to :approver, -> { where(approver: true) }, class_name: "User"
  end
  ```

## Has Many Associations
* `has_many` associations enable you to define a relationship in which one model has many other models that belong to it.
  ```Ruby
  class User < ActiveRecord::Base
    has_many :timesheets
  end
  ```

### Methods
* `has_many` association proxies are wrappers around a Ruby array and have all of a normal array's methods.
* Named scopes and all of `ActiveRecord::Base`'s class methods are also available.
* `<<(*records)` and `create(attributes = {})`: add one or more associated object depending on whether they're passed an array.
  * Both trigger `:before_add` and `:after_add` callbacks.
  * `<<` returns the association proxy, which enables chaining.
    * Returns false if validation fails.
  * `create` returns the new instance created.
* `average(column_name, options = {})`: Convenience wrapper for `calculate(:average, ...)`.
* `build(attributes={}, &block)`: Effectively an alias for new, which should be used instead.
  * Can be passed an array of hashes and will build each one, but this would normally be accomplished by setting `accepts_nested_attributes_for` on the owning class.
* `calculate(operation, column_name, options = {})`: Provides aggregate (`:sum`, `:average`, `:minimum` and `:maximum`) values within the scope of associated records.
* `clear`: A chainable version of `delete_all`.
* `count(column_name=nil, options={})`: counts all associated records in the database.
  * `:column_name`: count on a column instead of generrrating `COUNT(*)` in the resulting SQL.
  * If the `:counter_sql` option is set for the association, it will be used for the query.
  * Specify custom SQL with `:finder_sql`.
* `create(attributes, &block)` and `create!(attributes, &block)`: Creates a new record with its foreign key attribute set correctly, adds the new record to the association collection and then saves it in one method call.
  * The owning record must already be saved to use `create`, otherwise `ActiveRecord::RecordNotSaved` will be raised.
* `delete(*records)` and `delete_all`: used to sever associations by clearing the foreign key of the associated record.
  * `:dependent` is set to `:nullify` by default. If set to `:delete` or `:destroy`, then the associated records will be removed from the database.
* `destroy(*records)` and `destroy_all`: used to remove associations from the database.
  * There are load issues using `destroy_all` with large collections, since many objects will be loaded into memory at once.
* `ids`: Returns an array of primary keys for the associated objects by hitting the database with a pluck operation.
* `include?(record)`: Checks to see if the supplied record exists in the association collection and that it still exists in the database.
* `length`: Returns the size of the collection by loading it (if necessary) and calling `size` on the array.
* `maximum(column_name, options = {})`: Convenience wrapper for `calculate(:maximum, ...)`.
* `minimum(column_name, options = {})`: Convenience wrapper for `calculate(:minimum, ...)`.
* `pluck(*column_names)`: Returns an array of  attribute values.
* `replace(other_array)`: Replaces the collection of records currently inside the proxy with `other_array`.
  * Deletes objects that exist in the current collection, but not in `other_array`, and inserting (using `concat`) objects that don't exist in the current collection but do exist in `other_array`.
* `size`: If the collection has already been loaded, or its owner object has never been saved, returns the size of the underlying array. Otherwise, a `SELECT COUNT(*)` query gets the size of the collection without loading objects.
  * The query is bounded to the `:limit` option of the association, if set.
  * If a `:counter_cache` option is set on the association, its value is used instead of a query.
* `sum(column_name, options = {})`: Convenience wrapper for `calculate(:sum, ...)`.
* `uniq`: Iterates over the target collection and populates an `Array` with the unique values present, as determined by the `id` attribute.

### Options
* Largely identical to `:belongs_to` associations, but includes also:
  * `after_add` and `before_add`: Called after/before a record is added to the collection with `<<`.
    * Not called on `create`.
    * Can take an array of lambdas or symbols.
    * Raising an exception in the `before` callback chain stops the object being added to the collection:
      ```Ruby
      has_many :unchangeable_posts,
               class_name: "Post",
               before_add: ->(owner, record) { raise "You can't add a post!" }
      ```
  * `after_remove` and `before_remove`: Called after/before a record is removed from the collection with the `delete` method.
    * Can take an array of lambdas or symbols.
  * `as`: Specifies the polymorphic `belongs_to` association to use on the related class.
    * Raising an exception in the `before` callback chain stops the object being removed from the collection.

### Scopes
* Scopes work in much the same way as they do for Active Record queries:
  ```Ruby
  has_many :pending_comments, -> { where(approved: true) }, class_name: "Comment" 
  ```

## Many-to-Many Relationships
* `has_and_belongs_to_many` establishes a link between two associated models via an intermediate join table which doesn't have a primary key.
  * Now effectively deprecated.

## `has_many :through`
* Enable you to specify a one-to-many relationship indirectly via an intermediate join table. You can specify more than one such relaationship via the same table, which makes it a replacement for `has_and_belongs_to_many`. The advantage is that the join table contains fully fledged model objects complete with primary keys and anciallary data. Join models work just like all your other models.
* `has_many_through` is  always used in conjunction with a normal `has_many` association.
  ```Ruby
  class Timesheet < ActiveRecord::Base
    has_many :billable_weeks
    has_many :clients, through: :billable_weeks
  end
* Aggregating `has_many` or `has_one` associations on a join model:
  ```Ruby
  class Grandparent < ApplicationRecord
    has_many :parents
    has_many :grandchildren, through: :parents, source: :children
  end

  class Parent < ActiveRecord::Base
    belongs_to :grandparent
    has_many :children
  end
  ```
  * You can't append or create new records through aggregated collections.
    * E.g. If you tried to create a new child record on `grandparent.grandchildren`, Rails wouldn't know what parent to assign the child to.
* Non-aggregating `has_many :through` associations work in almost exactly the same way as `has_many` associations.
  * Appending an object will save the object as expected, and Rails will manage the join model for you by creating a new instance with default values.
  * Appending to a non-aggregating `has_many :through` association with `<<`, Active Record will create a new join model even if one already exists for the models being joined.
    * Add a `validates_uniqueness_of` constraint on the join model to prevent this.
      `validates_uniqueness_of :client_id, scope: :timesheet_id`

### Options
* The options for `has_many :through` are the same as for `has_many` – `:through` is an option on `has_many`!
* Some options change when used with `:through`:
  * `:class_name` and `:foreign_key` are no longer valid, since they are implied from the target association on the join model.
  * `:source`: Specifies which association to use on the associated class.
    * Active Record assumes that the target is the singular or plural version of the `:has_many` association name.
  * `:source_type` is needed to establish a `has_many :through` to a polymorphic `belongs_to` association on the join model.
    ```Ruby
    class ClientContact < ActiveRecord::Base
      belongs_to :client
      belongs_to :contact, polymorphic: true
    end

    class Person < ActiveRecord::Base
      has_many :client_contacts, as: :contact
    end

    class Business < ActiveRecord::Base
      has_many :client_contacts, as: :contact
    end

    Client.first.contacts # Won't work because Active Record would have to be aware of all classes that could be a contact.

    # To make this work:
    class Client < ActiveRecord::Base
      has_many :client_contacts
      has_many :people, through: :client_contacts, source: :contact, source_type: :person
      has_many :businesses, through: :client_contacts, source: :contact, source_type: :business
    end

### Unique Association Objects
* The `distinct` scope method tells the association to include only unique objects.
  * Useful when using `has_many :through` since two different join model objects could easily reference the same related object.
  * E.g. in the case where more than one `billable_week` may be linked to the same `timesheet`:
    ```Ruby
    class Client < ActiveRecord::Base
      has_many :timesheets, -> { distinct }, through: :billable_weeks
    end

## One-to-One Relationships

### `has_one`
* Works exactly like `has_many`, but a `LIMIT 1` clause is applied.
  * Reminder: the record with `belongs_to` on it contains the foreign key.
* When you associate a new record to an object that already `has_one` associated record, the original association is severed, but the record is not deleted from the database.
  * Set `dependent: :destroy` to force deletion when dissociated from a `has_one` relationship.

### Using `has_one` together with `has_many`
* `has_one` can be used to single out one record of significance alongside an established `has_many` relationship:
  ```Ruby
  class User < ActiveRecord::Base
    has_many :timesheets
    has_one :latest_sheet, -> { order("created_at DESC") }, class_name: "Timesheet"
  end
  ```
  * It is not necessary to add another `belongs_to` to `:timesheets`.

### Options & Scopes
* Similar to `has_many`.

## Working with Unsaved Objects and Associations

### One-to-One Associations
* Assigning a new object to a `belongs_to` association does not save the parent ot the associated object.
  E.g. `profile.avatar = avatar`
* Assining a new object to a `has_one` association automatically saved that object (and the object being replaced, if there is one) so that their foreign key fields are updated.
  E.g. `avatar.profile = profile`.
* When saves fail for any of the objects being updated, the operation returns `false` and the assignment is cancelled.

### Collections
* Adding a new object to `has_many` and `has_and_belongs_to_many` collections automatically saves it unless the owner of the collection is not yet stored in the database (since there would be no foreign key value to save).
* If objects being added to the collection fail to save properly, the operation returns false.
* To add an object without automatically saving it, use the collection's `new` or `build` methods.
* If the parent has `autosave` set to `true`, members of the collection will be saved when the parent is saved.
  * The `accepts_nested_attributes_for` method results in the associated collection's `autosave` option being turned on.

### Deletion
* Associations with autosave on can have their records deleted when an inverse record is saved.
  * Enables records on both sides of the association to get persisted within the same transacation.
  * Handled through `mark_for_destruction`.
  * 
    ```Ruby
    class User < ActiveRecord::Base
      has_many :timesheets, autosave: true
    end

    user = User.where(name: "Durran")
    user.timesheets.closed.each(&:mark_for_destruction)
    user.save
    ```
    * If the transaction fails and the associated records do not get deleted, they are still marked for destruction.
      * Use `reload` to clear the flag if necessary.

## Association Extensions
* The proxy objects that handle access to associations can be extended with your own custom finders and factory methods.
*
  ```Ruby
  class Account < ActiveRecord::Base
    has_many :people do
      def named(full_name)
        first_name, last_name = full_name.split(" ", 2)
        where(first_name: first_name, last_name: last_name).first_or_create
      end
    end
  end

  person = account.people.named("Angus Morrison")
  ```
* To share the same extensions between many associations, use an extesion module.
    ```Ruby
    module ByNameExtension
      def named(full_name)
        first_name, last_name = full_name.split(" ", 2)
        where(first_name: first_name, last_name: last_name).first_or_create
      end
    end

    class Account < ActiveRecord::Base
      has_many :people, -> { extending(ByNameExtension) }
    end
    ```
* If you don't need to reuse the extension logic with more than one model, use class methods instead.
  * Class methods are automatically available on `has_many` associations.

## The `CollectionProxy` Class
* The parent of all association proxies.

### Owner, Reflection and Target
* The object that holds the association is known as the `@owner`.
* The associated object (or array of objects) is the `@target`.
* Metadata about the association itself is found in `@reflection`, which is an instance of `ActiveRecord::Reflection::AssociationReflection`.
  * Contains all of the configuration options for the association.
* The proxy delegates unknown methods to `@target` via Ruby's built-in `method_missing` hook.
  * This is how a `has_many` association proxy, which inherits from `Array`, has all of `Array`'s methods.


# Chapter 8: Validations

## Finding Errors
* Errors for every Active Record model can be found under the `errors` attribute.
* When you call `valid?`, a series of steps is taken to find errors:
  1. Clear the `errors` collection.
  2. Run validations.
  3. Return whether the model's `errors` collection is empty.
* Mark an object as invalid by adding to its `errors` collection using its `add` methods.

## The Simple Declarative Validations

### `validates_absence_of`
* Ensures the specified attributes are blank using the `blank?` method.

### `validates_acceptance_of`
* Used to validate checkbox-style agreeeement to T&Cs, etc.
* A database column matching the attribute declared in the validation is not required.
  * Store the value only if you need to keep track of whether the user has accepted the terms.
* The `:accept` option changes the value that represents acceptance.
  * Default is "1".
  *
    ```Ruby
    class Cancellation < ActiveRecord::Base
      validates_acceptance_of :account_cancellation, accept: "YES"
    end
    ```

### `validates_associated`
* Used to ensure all associated objects are valid on save.
* Works on any kind of association.
* Superceded by the default `validate: true` on `has_many` associations.
  * CAUTION: Setting `validate: true` on a `belongs_to` can cause infinite loops because of this.
* Will not fail if the association is `nil`.
  * Requires conjunction with `validates_presence_of`.

### `validates_confirmation_of`
* Used to validate double-entry confirmation fields.
* Creates a virtual attribute for the confirmation value and compares the two attributes to make sure they match.
* The interface used to set values for the model would need to include extra text fields with a `_confirmation` suffix.

### `validates_each`
* Takes an array of attribute names to check and a block to check them against.
  ```Ruby
  class Invoice < ActiveRecord::Base
    validates_each :supplier_id, :purchase_order do |record, attr, value|
      record.errors.add(attr) unless PurchasingSystem.validate(attr, value)
    end
  end
  ```

### `validates_format_of`
* Pass one or more attributes to check and a regular expression as the `:with` option.

### `validates_inclusion_of` and `validates_exclusion_of`
* Take a variable number of attribute names and an `:in` option, and check to make sure the value of the attribute is included/excluded from the enumerable passed as `:in`.

### `validates_length_of`
* Takes a variety of options to specify length constraints for a given attribute.
* Options: `:minimum`, `:maximum`, `:within`, `:is`.
* Error message options:
  * `:too_long`, `:too_short`, `:wrong_length`.
  * `#{count}` can be used as a placeholder for the number corresponding to the constraint.
  *
    ```Ruby
    class Account < ActiveRecord::Base
      validates_length_of :account_number, is: 16,
                          wrong_length: "should be #{count} characters long"
    ```

### `validates_numericality_of`
* Ensures an attribute only holds a numeric value.
* `:only_integer` specifies that the number can only be an integer.
* Options: `:even`, `:odd`, `:equal_to`, `:greater_than`, `:greater_than_or_equal_to`, `:less_than`, `:less_than_or_equal_to`, `:other_than`.
* Infinity and special float values:
  * Infinity, positive infinity, negative infinity, positive zero, negative zero and NaN are considered numbers by `validates_numericality_of` and databases that support the IEEE 754 standard.

### `validates_presence_of`
* Checks whether the attribute is blank using `blank?`
* Don't use for boolean attributes like checkboxes; use `validates_acceptance_of` instead.
* The value `false` is considered blank. To make sure only `true` or `false` values are set on your attribute, use: `validates_inclusion_of :protected, in: [true, false]`.
* Validating the presence of associated objects:
  * To ensure that an association is present, pass `validates_presence_of` its foreign key.
    * Fails when both parent and child are unsaved because the foreign key is blank.
  * This is a valid use case for actual foreign-key constraints in the database.
    * Triggers an Active Record exception when the constraint is violated.

### `validates_uniqueness_of`
* Ensures the value of an attribute is unique for all models of the same type.
  * Does not add a unique constraint at the database level.
  * Constructs and executes a query for the matching record in the database.
* The `:scope` option allows additional attributes to be used to determine uniqueness:
  `validates_uniqueness_of :line_two, scope: [:line_one, :city, :zip]`
  * `:case_sensitive` can be specified for textual attributes.
* Can be used to validate that all items in an array bound for a Postgres array column are unique.
* This validation is not foolproof due to the race condition between the `SELECT` query that checks for duplicates and the `INSERT` or `UPDATE` that persists the record.
  * Could generate an Active Record exception, so be prepared to handle it.
  * Use a unique index constraint in the database if the column absolutely must be unique.
* Limit Constraint Lookup
  * Specify criteria that constrain a uniqueness validation against a set of records by setting the `:conditions` option.
    ```Ruby
    class Article < ActiveRecord::Base
      validates_uniqueness_of :title,
        conditions: -> { where.not(published_at: nil) }
    end
    ```

### `validates_with`
* Used to run reusable validation classes.
* To implement a custom validator, extend `ActiveRecord::Validator` and implement the `validate` method.
  * The record being validated is available as `record`.
  * Manipulate the record's `errors` hash to log validation errors.
*
  ```Ruby
  class EmailValidator < ActiveRecord::Validator
    def validate
      record.errors[:email] << "is not valid" unless
        record.email =~ /email regex/
    end
  end

  class Account < ActiveRecord::Base
    validates_with EmailValidator
  end
  ```
* To make sure the validator isn't constrained to use a specific attribute name, you can access validation options at runtime via the `options` hash:
  ```Ruby
  class EmailValidator << ActiveRecord::Validator
    def validate
      email_field = options[:attr]
      record.errors[email_field] << "is not valid" unless
        record.send(email_field) =~ /email regex/
    end
  end

  class Account < ActiveRecord::Base
    validates_with EmailValidator, attr: :email
  end
  ```

### Record Invalid
* Whenever you use bang operations and validation fails, be prepared to rescue `ActiveRecord::RecordInvalid`.

## Common Validation Options
* `allow_blank` and `allow_nil`
* `if` and `unless`
* `message`: Overrides the default error message format described in the English locale file in Active Model.
  * `model`, `attribute`, `value` and `count` are always available for interpolation in error messages.
* `on`: Limit validation runs to only `:create` or `:save`.
* `strict`: Causes an `ActiveModel::StrictValidationFailed` exception to be raised when a model is invalid.

## Conditional Validation
* `:if` and `:unless` options determine at runtime whether the validation should be run.
* Accepts symbols (best performance), strings (to `eval`; worst performance) and procs (`instance_eval`'d; most elegant one-liners) as conditions:
  `validates_presence_of :approver, if: -> { approved? && !legacy? }`

### Usage and Considerations
* A primary use case for conditional validations is when a object can validly be persited in more than one state.
  ```Ruby
  validates_presence_of :password, if: :password_required?
  validates_presence_of :password_confirmation, if: :password_required?

  
  def password_required?
    encrypted_password.blank? || !password.blank?
  end
  ```
* Use `with_options` to DRY up the above:
  ```Ruby
  with_options if: :password_required? do |user| 
    validates_presence_of :password, if: :password_required?
    validates_presence_of :password_confirmation, if: :password_required?
  end
  ```
  * Mixed into `Object` by Active Support.

### Validation Contexts
* Declare a validation an pass any symbol as the name of a validation context to the `:on` option.
  * Invoked only when `record.valid?(context_name)` is called explicitly.
  * 
    ```Ruby
    class Report < ActiveRecord::Base
      validates_presence_of :name, on: :publish
    end

    class ReportsController < ApplicationController
      def publish
        if report.valid?(:publish)
          redirect_to(report, notice: "Report published")
        else
          flash.now.alert = "Can't publish unnamed reports"
          render :show
        end
      end
    end
    ```

## Short-Form Validation
* The `validates` method identifies an attribute and accepts options that correspond to each of the validators above, letting you group all the validations that aplly to a single field together.
*
 ```Ruby
  validates :username, presence: true,
    format: { with: /[A-Za-z0-9]/ },
    length: { minimum: 3 },
    uniqueness: true
  ```

## Custom Validation Techniques

### Custom Validation Macros
* Add custom validation macros available to all model classes by extending `ActiveModel::EachValidator` and implementing `validate_each`.
* The class name is infered from the symbol provided to `validates` (`:like`).
* You can receive options from `validates` by adding an initializer to the validator class.
*
  ```Ruby
  class LikeValidator < ActiveModel::EachValidator
    def initialize(options)
      @with = options[:with]
      super
    end

    def validate_each(record, attribute, value)
      unless value[:with]
        record.errors.add(attribute, "does not appear to be like #{@with}")
      end
    end
  end

  class Report < ActiveRecord::Base
    validates :name, like: { with: "Report" }
  end
  ```

### Create a Custom Validator Class
* Inherit from `ActiveModel` validator and implement `validate`, which takes the record to validate.
* 
  ```Ruby
  class MyValidator < ActiveModel::Validator
    def validate(record)
      record.errors[:base] << "FAIL" unless # some logic
    end
  end

  class Report < ActiveRecord::Base
    validates_with MyValidator
  end
  ```

### Add a `validate` Method to Your Model
* An older technique that is recommended against because it adds complexity and is harder to test in isolation.
*
  ```Ruby
  class Example < ActiveRecord::Base
    def validate
      if total != (attr1 + attr2 + attr3)
        errors[:total] << "doesn't add up"
      end
    end
  end
  ```
* When adding errors to a particular attribute you use a sentence fragment, but when adding errors to `errors[:base]`, you use an entire sentence.
  * The method used to expose error messages to the user, `full_messages_for`, takes the name of the attribute plus whatever errors have been added to it and turns it into a sentence with Active Support's `to_sentence` method.
* The return value of a custom validation method is not used.

## Working with the Errors Hash
* `errors[:base] = msg`: Adds an error message related to the overall object state itself and not any particular attribute.
  * Should be complete sentences.
* `errors[:attribute] = msg`: Adds an error message related to a particular attribute.
  * Should be a sentence fragment that will have the capitalized name of the attribute added to it.
* `clear`: Clears the state of the `Errors` object.
* `full_messages_for(:attribute)`: Return full human-readable error messages.

## Testing Validations with Shoulda
* The Shoulda matchers library contains matchers designed to easily test validations.
*
  ```Ruby
  describe Post do
    it { should validate_uniqueness_of(:title) }
  end
  ```


# Chapter 9: Advanced Active Record

## Scopes
* Enable you to define and chain query criteria in a declarative and reusable manner.
* These implementations are equivalent:
  ```Ruby
  class Timesheet < ActiveRecord::Base
    scope submitted, -> { where(submitted: true) }

    def self.submitted
      where(submitted: true)
    end
  end
  ```

### Scopes and Joins
* The `join` method can be used to create cross-model scopes. E.g. returning the users who have timesheets submitted less than 7 days ago:
  ```Ruby
  scope :tardy, -> { 
    joins(:timesheets).
    where("timesheets.submitted_at <= ?", 7.days.ago).
    group("users.id")
  }

### Scope Combinations
* The example above violates the single responsibility principle by using Timesheet logic in the User class. `merge` can be used to avoid this:
  ```Ruby
  class Timesheet
    scope :last, -> { where("timesheet.submitted_at <= ?", 7.days.ago) }
    ...
  
  class User
    scope :tardy, -> { joins(:timesheets).group("users.id").merge(Timesheet.late) }
    ...
  ```

### Default Scopes
* The scope to use when no scope is specified.
  ```Ruby
  default_scope { where(status: "open") }
  ```
* Default scopes also get applied to models when creating or building them. New models of the class in the above example will be created with `status: "open"`.
* To switch of the default scope for a query, use `unscope`:
  ```Ruby
  Timesheet.unscoped.order("submitted_at DESC").to_a
  ```

### Using Scopes for CRUD
* Scopes including a `where` caluse using hashed conditions will populate attributes of objects built from them:
  ```Ruby
  scope :perfect, -> { submitted.where(total_hours: 40) }
  ts = Timesheet.perfect.build
  ts.total_hours
  # => 40
  ```

## Callbacks

### One-Liners
* Blocks passed to callbacks are evaluated with `instance_eval`, so the context is the record itself.
* `before_destroy { logger.info "Destroying" }`.

### Protected or Private
* Except when using a block, the access level for callback methods should always be protected or private.

### Matched `before/after` Callbacks
* List of callbacks:
  * `before_validation`
  * `after_validation`
  * `before_save`
  * `around_save`
  * `after_save`
  * `before_create`
  * `after_create`
  * `before_destroy`
  * `around_destroy`
  * `after_destroy`
  * `after_commit`
  * `after_rollback`
  * `after_touch`
* Callbacks can be limited to specific Active Record life cycles with `:on`.
  * E.g. `on: [:create, :update]`
* These methods don't trigger callbacks:
  * `decrement`
  * `decrement_counter`
  * `delete`
  * `delete_all`
  * `increment`
  * `increment_counter`
  * `toggle`
  * `update_column`
  * `update_all`
  * `update_counters`

### Halting Execution
* To explicitly halt the callback chain, use `throw(:abort)`.

### Callback Usages
* Cleaning up attribute formatting with `before_validation`:
  ```Ruby
  class CreditCard < ActiveRecord::Base
    before_validation on: :create do
      # Strip non-digits
      self.number = number.gsub(/[^0-9]/, "")
    end
  end
* Geocoding with `before_save`:
  ```Ruby
  class Address < ActiveRecord::Base
    before_save :geocode
    ...

    def geocode
      result = Geocoder.coordinates(to_s)
      if result.present?
        self.latitude = result.first
        self.longitude = result.last
      else
        errors[:base] << "Geocoding failed. Please check address."
        throw(:abort)
      end
    end
  end
  ```
* Non-destructive deletion with `before_destroy`:
  ```Ruby
  class Account < ActiveRecord::Base
    before_destroy do
      self.update_attribute(:deleted_at, Time.current)
      throw(:abort)
    end
  end
  ```
  * A real implementation would need to modify all finders to included `deleted_at` is `NULL` conditons to prevent deleted records from showing up in the application. Use the gem `destroyed_at` instead.
* Cleaning up associated files with `after_destroy`:
  ```Ruby
  def destroy_attached_files
    Paperclip.log("Deleting attachments.")
    each_attachment do |name, attachment|
      attachment.send(:flush_delete)
    end
  end
  ```

### Special Callbacks: `after_initialize` and `after_find`
* `after_initialize` is invoked whenever a new Active Record model is instantiated (either from scratch or the database).
* `after_find` is invoked whenever Active Record loads a model object from the database.
  * Called before `after_initialize`
* To run code the first time a model is ever instantiated:
  ```Ruby
  after_initialize do
    if new_record?
      ...
    end
  end
  ```

### Callback Classes
*
  ```Ruby
  class MarkDeleted
    def self.before_destroy(model)
      ...
    end
  end

  class Auditor
    def initialize(audit_log)
      @audit_log = audit_log
    end

    def after_create(model)
      ...
    end

    def after_update(model)
      ...
    end
  end

  class Account < ActiveRecord::Base
    after_create Auditor.new(DEFAULT_AUDIT_LOG)
    after_update Auditor.new(DEFAULT_AUDIT_LOG)
    before_destroy MarkDeleted
  end
  ```
* To create macro-style methods for repetitive logic, you can add class methods to `ActiveRecord::Base` in `lib/core_ext/active_record_base.rb`.
  ```Ruby
  class ActiveRecord::Base
    def self.acts_as_audited(audit_log=DEFAULT_AUDIT_LOG)
      auditor = Auditor.new(audit_log)
      after_create auditor
      after_destroy auditor
    end
  end

  class Account < ActiveRecord::Base
    acts_as_audited
  end
  ```

## Attributes API
* Enables declaration of types and default values for attributes.
* Overrides Active Record's standard type casting and gives control over how values are converted to and from SQL.
* Type coercion: taking form-inputted values from the user and transforming them into representations that are more useful to our code.
  * E.g. multi-part date fragments from date input into a usable `Date` object.
*
  ```Ruby
  class Event < ApplicationRecord
    attribute :repeats, :boolean, default: false
    attribute :repeates_end, :date
  end
  ```
### The `attribute` Method
* Takes a `name`, `cast_type` and options:
  * `default`: The default value to use. Overrided defaults set via the database.
    * Defaults to `nil` if not specified.
  * `array`: If using Postgres, specifies that the type should be an array of `cast_type`.
  * `range`: If using Postgres, specifies that the type should be a range. Supply a Ruby `Range` object.

### Custom Types
*
  ```Ruby
  class Inquiry < ActiveRecord::Type::String
    def type
      :inquiry
    end

    def cast(value)
      super.inquiry
    end
  end

  class Event < ApplicationRecord
    attribute :repeats, Inquiry.new
  end
  ```
* By default, `cast` is invoked on both setting and getting values from the database.
* To only affect setting behaviour, override `deserialize` for the getting and leave `cast` for setting only.
* If `serialize` is implemented, it will be invoked as part of generating SQL queries.

### Registering New Attribute Types
* If a custom type will be reused, register it globally in an initializer:
  ```Ruby
  # config/initializers/custom_attribute_types.rb
  ActiveRecord::Type.register :inquiry, Inquiry.new
  ```
* Use the `adapter: :adapter_name` option if the custom type works only with a specific database adapter.
* Working with money: use the `Money-Rails` gem.