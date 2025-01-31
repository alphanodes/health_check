# health_check gem

[![Tests](https://github.com/AlphaNodes/health_check/workflows/Tests/badge.svg)](https://github.com/alphanodes/health_check/actions/workflows/tests.yml) [![Run Linters](https://github.com/alphanodes/health_check/workflows/Run%20Linters/badge.svg)](https://github.com/alphanodes/health_check/actions/workflows/linters.yml)

Simple health check of Rails 7.x and 8.x apps for use with Pingdom, NewRelic, EngineYard etc.

The basic goal is to quickly check that rails is up and running and that it has access to correctly configured resources (database, email gateway)

health_check provides various monitoring URIs, for example:

```console
curl localhost:3000/health_check
success

curl localhost:3000/health_check/all.json
{"healthy":true,"message":"success"}

curl localhost:3000/health_check/database_cache_migration.xml
<?xml version="1.0" encoding="UTF-8"?>
<hash>
  <healthy type="boolean">true</healthy>
  <message>success</message>
</hash>
```

You may also issue POST calls instead of GET to these urls.

On failure (detected by health_check) a 500 http status is returned with a simple explanation of the failure
(if include_error_in_response_body is true)

```console
curl localhost:3000/health_check/fail
health_check failed: invalid argument to health_test.
```

The health_check controller disables sessions for versions that eagerly load sessions.

## Checks

- standard (default) - site, database and migrations checks are run plus email if ActionMailer is defined and it is not using the default configuration
- all / full - all checks are run (can be overriden in config block)
- cache - checks that a value can be written to the cache
- custom - runs checks added via config.add_custom_check
- database - checks that the current migration level can be read from the database
- email - basic check of email - :test returns true, :sendmail checks file is present and  executable, :smtp sends HELO command to server and checks response
- migration - checks that the database migration level matches that in db/migrations
- rabbitmq - RabbitMQ Health Check
- redis / redis-if-present - checks Redis connectivity
- resque-redis / resque-redis-if-present - checks Resque connectivity to Redis
- s3 / s3-if-present - checks proper permissions to s3 buckets
- sidekiq-redis / sidekiq-redis-if-present - checks Sidekiq connectivity to Redis
- elasticsearch / elasticsearch-if-present - checks Elasticsearch connectivity
- site - checks rails is running sufficiently to render text

Some checks have a *-if-present form, which only runs the check if the corresponding library has been required.

The email gateway is not checked unless the smtp settings have been changed.
Specify full or include email in the list of checks to verify the smtp settings
(eg use 127.0.0.1 instead of localhost).

Note: rails also checks migrations by default in development mode and throws an `ActiveRecord::PendingMigrationError` exception (http error 500) if there is an error

## Installation

Add the following line to Gemfile (after the rails gems are listed)

```ruby
gem 'health_check'
```

And then execute

```console
bundle
```

Or install it yourself as:

```console
gem install health_check
```

## Configuration

To change the configuration of health_check, create a file `config/initializers/health_check.rb` and add a configuration block like:

```ruby
HealthCheck.setup do |config|

  # uri prefix (no leading slash)
  config.uri = 'health_check'

  # Text output upon success
  config.success = 'success'

  # Text output upon failure
  config.failure = 'health_check failed'

  # Disable the error message to prevent /health_check from leaking
  # sensitive information
  config.include_error_in_response_body = false

  # Log level (success or failure message with error details is sent to rails log unless this is set to nil)
  config.log_level = 'info'

  # Timeout in seconds used when checking smtp server
  config.smtp_timeout = 30.0

  # http status code used when plain text error message is output
  # Set to 200 if you want your want to distinguish between partial (text does not include success) and
  # total failure of rails application (http status of 500 etc)

  config.http_status_for_error_text = 500

  # http status code used when an error object is output (json or xml)
  # Set to 200 if you want to distinguish between partial (healthy property == false) and
  # total failure of rails application (http status of 500 etc)

  config.http_status_for_error_object = 500

  # bucket names to test connectivity - required only if s3 check used, access permissions can be mixed
  config.buckets = {'bucket_name' => [:R, :W, :D]}

  # You can customize which checks happen on a standard health check, eg to set an explicit list use:
  config.standard_checks = [ 'database', 'migrations', 'custom' ]

  # Or to exclude one check:
  config.standard_checks -= [ 'emailconf' ]

  # You can set what tests are run with the 'full' or 'all' parameter
  config.full_checks = ['database', 'migrations', 'custom', 'email', 'cache', 'redis', 'resque-redis', 'sidekiq-redis', 's3']

  # Add one or more custom checks that return a blank string if ok, or an error message if there is an error
  config.add_custom_check do
    CustomHealthCheck.perform_check # any code that returns blank on success and non blank string upon failure
  end

  # Add another custom check with a name, so you can call just specific custom checks. This can also be run using
  # the standard 'custom' check.
  # You can define multiple tests under the same name - they will be run one after the other.
  config.add_custom_check('sometest') do
    CustomHealthCheck.perform_another_check # any code that returns blank on success and non blank string upon failure
  end

  # max-age of response in seconds
  # cache-control is public when max_age > 1 and basic_auth_username is not set
  # You can force private without authentication for longer max_age by
  # setting basic_auth_username but not basic_auth_password
  config.max_age = 1

  # Protect health endpoints with basic auth
  # These default to nil and the endpoint is not protected
  config.basic_auth_username = 'my_username'
  config.basic_auth_password = 'my_password'

  # Whitelist requesting IPs by a list of IP and/or CIDR ranges, either IPv4 or IPv6 (uses IPAddr.include? method to check)
  # Defaults to blank which allows any IP
  config.origin_ip_whitelist = %w(123.123.123.123 10.11.12.0/24 2400:cb00::/32)

  # Use ActionDispatch::Request's remote_ip method when behind a proxy to pick up the real remote IP for origin_ip_whitelist check
  # Otherwise uses Rack::Request's ip method (the default, and always used by Middleware), which is more susceptable to spoofing
  # See https://stackoverflow.com/questions/10997005/whats-the-difference-between-request-remote-ip-and-request-ip-in-rails
  config.accept_proxied_requests = false

  # http status code used when the ip is not allowed for the request
  config.http_status_for_ip_whitelist_error = 403

  # rabbitmq
  config.rabbitmq_config = {}

  # When redis url/password is non-standard
  config.redis_url = 'redis_url' # default ENV['REDIS_URL']
  # Only included if set, as url can optionally include passwords as well
  config.redis_password = 'redis_password' # default ENV['REDIS_PASSWORD']

  # Failure Hooks to do something more ...
  # checks lists the checks requested
  config.on_failure do |checks, msg|
    # log msg somewhere
  end

  config.on_success do |checks|
    # flag that everything is well
  end
end
```

You may call add_custom_check multiple times with different tests. These tests will be included in the default list ("standard").

If you have a catchall route then add the following line above the catch all route (in `config/routes.rb`):

```ruby
health_check_routes
```

### Installing As Middleware

Install health_check as middleware if you want to sometimes ignore exceptions from later parts of the Rails middleware stack,
eg DB connection errors from QueryCache. The "middleware" check will fail if you have not installed health_check as middleware.

To install health_check as middleware add the following line to the config/application.rb:

```ruby
config.middleware.insert_after Rails::Rack::Logger, HealthCheck::MiddlewareHealthcheck
```

Note: health_check is installed as a full rails engine even if it has been installed as middleware. This is so the
remaining checks continue to run through the complete rails stack.

You can also adjust what checks are run from middleware, eg if you want to exclude the checking of the database etc, then set

```ruby
config.middleware_checks = ['middleware', 'standard', 'custom']
config.standard_checks = ['middleware', 'custom']
```

Middleware checks are run first, and then full stack checks.
When installed as middleware, exceptions thrown when running the full stack tests are formatted in the standard way.

## Uptime Monitoring

Use a website monitoring service to check the url regularly for the word "success" (without the quotes) rather than just a 200 http status so
that any substitution of a different server or generic information page should also be reported as an error.

If an error is encounted, the text "health_check failed: some error message/s" will be returned and the http status will be 500.

See

- Pingdom Website Monitoring - https://www.pingdom.com
- NewRelic Availability Monitoring - http://newrelic.com/docs/features/availability-monitoring-faq
- Engine Yard's guide - https://support.cloud.engineyard.com/entries/20996821-monitor-application-uptime (although the guide is based on fitter_happier plugin it will also work with this gem)
- Nagios check_http (with -s success) - https://www.nagios-plugins.org/doc/man/check_http.html
- Any other montoring service that can be set to check for the word success in the text returned from a url

### Requesting Json and XML responses

Health_check will respond with an encoded hash object if json or xml is requested.
Either set the HTTP Accept header or append .json or .xml to the url.

The hash contains two keys:

- healthy - true if requested checks pass (boolean)
- message - text message ("success" or error message)

The following commands

```console
curl -v localhost:3000/health_check.json
curl -v localhost:3000/health_check/email.json
curl -v -H "Accept: application/json" localhost:3000/health_check
```

Will return a result with Content-Type: application/json and body like:

```json
{"healthy":true,"message":"success"}
```

These following commands

```console
curl -v localhost:3000/health_check.xml
curl -v localhost:3000/health_check/migration_cache.xml
curl -v -H "Accept: text/xml" localhost:3000/health_check/cache
```

Will return a result with Content-Type: application/xml and body like:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<hash>
  <healthy type="boolean">true</healthy>
  <message>success</message>
</hash>
```

See https://github.com/ianheggie/health_check/wiki/Ajax-Example for an Ajax example

## Silencing log output

It is recomended that you use silencer, lograge or one of the other log filtering gems.

For example, with lograge use the following to exclude health_check from being logged:

```ruby
config.lograge.ignore_actions = ["HealthCheck::HealthCheckController#index"]
```

Likewise you will probably want to exclude health_check from monitoring systems like newrelic.

## Caching

Cache-control is set with
- public if max_age is > 1 and basic_auth_username is not set (otherwise private)
- no-cache
- must-revalidate
- max-age (default 1)

Last-modified is set to the current time (rounded down to a multiple of max_age when max_age > 1)

## Known Issues

- See https://github.com/ianheggie/health_check/issues
- No inline documentation for methods
- <b>rvm gemsets breaks the test</b> - specifically <tt>rvm use 1.9.3</tt> works but <tt>rvm gemset use ruby-1.9.3-p385@health_check --create</tt> triggers a "Could not find gem 'coffee-rails (~> 3.2.1) ruby' in the gems available on this machine." error in the last call to bundle (installing health_check as a gem via a path into the temp railsapp)

## Similar projects

- fitter_happier plugin by atmos - plugin with similar goals, but not compatible with uptime, and does not check email gateway
- HealthBit - inspired by this gem but with a fresh start as a simpler rack only application, no travis CI tests (yet?) but looks interesting.

## Manual testing

The instructions have been changed to using a vagrant virtual box for consistent results.

Install vagrant 1.9.7 or later and virtual_box or other local virtual machine provider.  Add the vagrant plugin called vbguest.

```console
vagrant plugin install vagrant-vbguest
```

Create a temp directory for throw away testing, and clone the health_check gem into it

```console
mkdir -p ~/tmp
cd ~/tmp
git clone https://github.com/ianheggie/health_check.git ~/tmp/health_check
```

The Vagrantfile includes provisioning rules to install chruby (ruby version control),
ruby-build will also be installed and run to build various rubies under /opt/rubies.

Use <tt>vagrant ssh</tt> to connect to the virtual box and run tests.

The test script will package up and install the gem under a temporary path, create a dummy rails app configured for sqlite,
install the gem, and then run up tests against the server.
This will require TCP port 3456 to be free.

Cd to the checked out health_check directory and then run the test as follows:

```console
cd ~/tmp/health_check

vagrant up   # this will also run vagrant provision and take some time
              # chruby and various ruby versions will be installed

vagrant ssh

cd /vagrant  # the current directory on your host is mounted here on the virtual machine

chruby 2.2.2 # or some other ruby version (run chruby with no arguments to see the current list)

test/test_with_railsapp

exit        # from virtual machine when finished
```

The script will first call `test/setup_railsapp` to setup a rails app with health_check installed and then
run up the rails server and perform veraious tests.

The script `test/setup_railsapp` will prompt you for which gemfile under test you wish to use to install the appropriate rails version, and then
setup tmp/railsapp accordingly.

The command `rake test` will also launch these tests, except it cannot install the bundler and rake gems if they are missing first (unlike test/test_with_railsapp)

## Copyright

Copyright (c) 2010-2021 Ian Heggie, released under the MIT license.
See MIT-LICENSE for details.

## Contributors

Thanks go to the various people who have given feedback and suggestions via the issues list and pull requests.

### Contributing

Use gem versions for stable releases, or github branch / commits for development versions:
- for Rails 5.x and 6.x use feature branched off master [master](https://github.com/ianheggie/health_check/tree/master) for development;
- for Rails 4.x use feature branches off the [rails4](https://github.com/ianheggie/health_check/tree/rails4) stable branch for development;
- for Rails 3.x use feature branches off the [rails3](https://github.com/ianheggie/health_check/tree/rails3) stable branch for development;
- for Rails 2.3 use feature branches off the [rails2.3](https://github.com/ianheggie/health_check/tree/rails2.3) stable branch for development;

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Create a test that confirms your changes work
4. Update README.rdoc to explain enhancements, or add succinct comment in code when fixing bugs
5. Commit your changes (`git commit -am 'Add some feature'`)
6. Push to the branch (`git push origin my-new-feature`)
7. Create new Pull Request (Code with BDD tests and documentation are highly favoured)

<em>Feedback welcome! Especially with suggested replacement code, tests and documentation</em>
