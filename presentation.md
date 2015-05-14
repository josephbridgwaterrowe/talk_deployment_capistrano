class: center, middle

# Deployment Strategies

???

Painless deployment is fun.

Automating it for continuous deployment is even more fun.

---

# Some Deployment Options

- Capistrano

- Container (Heroku, Docker)

- GitHub

???

Heroku provides options to automate deployment from GitHub and Dropbox

The Heroku GitHub auto deploys are now out of beta and they have added
Pull Request deploys to new apps (in beta).

What other options are there?

---

# Capistrano

- Most popular deployment tool for Rails.

- Simple to configure and use.

- Libraries for integration with other common libraries.
  - rails
  - rvm
  - sidekiq
  - puma
  - etc.

- Extensible
  - Hook into the deployment flow
  - It's just Ruby code

???

Assuming that capistrano is the most popular deployment framework for Rails apps.

Libraries for integration hook into the deployment flow and add additional
deployment steps.

Deployment flow:

```
deploy:starting    - start a deployment, make sure everything is ready
deploy:started     - started hook (for custom tasks)
deploy:updating    - update server(s) with a new release
deploy:updated     - updated hook
deploy:publishing  - publish the new release
deploy:published   - published hook
deploy:finishing   - finish the deployment, clean up everything
deploy:finished    - finished hook
```

How many people are using Capistrano?

---

# Install

```ruby
# Gemfile
group :development do
  gem 'capistrano', '3.4.0'
  gem 'capistrano-bundler'
  gem 'capistrano-rails'
end
```

- cap install (v3) or Capify (v2)

```bash
mkdir -p config/deploy
create config/deploy.rb
create config/deploy/staging.rb
create config/deploy/production.rb
mkdir -p lib/capistrano/tasks
Capified
```

```bash
# .gitignore
config/deploy.rb
config/deploy/*.rb
```

???

This talk uses v3 of Capistrano.

We ignore the deployment configuration files and provide example files, for
example deploy.example.rb without any sensitive information.

 - repo name
 - user name

For automated solutions (like heaven) I imagine that this would have to
be provided by environment variables.

---

# Configuration

```ruby
# /config/deploy.example.rb
lock '3.4.0'

set :branch, 'master'
set :deploy_to, '/var/www/ourapp'
set :keep_releases, 5
set :linked_files, %w{config/application.yml config/database.yml}
set :linked_dirs,
    %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system}
set :normalize_asset_timestamps,
    %w{public/images public/javascripts public/stylesheets}
set :repo_url, 'git@github.com:ourorg/ourapp.git'

namespace :deploy do
  after :publishing, :link_www
  after :publishing, :schedule
  after :publishing, :upload_assets
  after :publishing, :restart
end
```

???

This is our opinionated recipe, making some assumptions on the target environment.

For example

 - App servers are deployed to a (passwordless) user called deploy,
 - Web server assets are deployed to a (passwordless) user called www, see below.

Deployments are always deployed to /var/www/app_name.

We create a /var/www/current to link to /var/www/app_name so that we can
use a generic unicorn init.d configuration.

---

# More configuration

```ruby
# /config/deploy/production.example.rb
server 'app01.ourapp.com',
       user: 'deploy',
       port: 22,
       roles: %w{app db web}
server 'app02.ourapp.com',
       user: 'deploy',
       port: 22,
       roles: %w{app db web}

set :web_servers,
    %w{web01.ourapp.com web02.ourapp.com}
```
```ruby
# /lib/capistrano/tasks/link_www.rake
namespace :deploy do
  task :link_www do
    on roles(:app) do
      within release_path do
        execute :rm, '-f', '/var/www/current'
        execute :ln, '-s', release_path, '/var/www/current'
      end
    end
  end
end
```

???

Common application code path to simplify unicorn configuration, single
init.d script.

---

# Even more configuration

```ruby
# /lib/capistrano/tasks/upload_assets.rake
namespace :deploy do
  task :upload_assets do
    run_locally do
      execute 'RAILS_ENV=production', :bundle, :exec, :rake, 'assets:precompile'
    end

    web_servers = fetch(:web_servers)
    web_servers.each do |web_server|
      puts "Deploying to web server: #{web_server}"

      run_locally do
        execute :ssh, "www@#{web_server}", "mkdir -p #{deploy_to}/current"
        execute :rsync, "-av ./public www@#{web_server}:#{deploy_to}/current"
      end
    end

    run_locally do; execute :rm, '-rf public/assets'; end
  end
end
```

???

Push compiled assets to all our web servers.

---

# Check

```bash
cap production deploy:check
```

```bash
# snipped previous log lines
DEBUG [30e3e742] Command:
      [ -f /var/www/contract_app/shared/config/application.yml ]
DEBUG [30e3e742] Finished in 0.005 seconds with exit status 1 (failed).
ERROR linked file /var/www/contract_app/shared/config/application.yml
      does not exist on app10-prd-gsh.westernmilling.com
(Backtrace restricted to imported tasks)
```

- Create application.yml, database.yml, etc.

```bash
# snipped previous log lines
DEBUG [c189ed73] Command:
                 [ -f /var/www/contract_app/shared/config/application.yml ]
DEBUG [c189ed73] Finished in 0.005 seconds with exit status 0 (successful).
```

???

- Check the deployment environment
  - files and folders
  - permissions
  - user

---

# Deploy

```bash
cap production deploy
```

---
class: center, middle

# Thank you
