---
layout: post
title: "How I migrated from Heroku to Digital Ocean with Chef and Capistrano"
date: 2013-03-07 11:24
comments: true
categories: [deployment, chef, capistrano, unicorn, nginx, rails]
---

UPDATE:

* Removed ElasticSearch and MongoDB recipes since they were not so useful for this tutorial.
* Added unicorn.rb
* Added ssh authentication step
* Added file paths

I've always loved deploying to Heroku. The simplicity of a git push let me focus on developing my applications which is what I really care about. However, both because of the [scandal about the routing system] [1] and because I wanted to expand my skill set by entering the sysadmin land, at [Responsa] [8] I decided to migrate to a VPS solution.

At this point I had three choices to make:

1. Hosting provider
2. Technology stack
3. Deploy strategy

## Provider

Many hackers I follow were recommending [Digital Ocean] [7] so I gave it a try. I must say I was very impressed with the simplicity and power of their dashboard, so I decided to use it.

I immediately changed my root password

```
passwd
```

Copied over my ssh key with

```
ssh-copy-id root@$IP
```

And disabled password access setting `PasswordAuthentication no` in `/etc/ssh/sshd_config`

## Technology

The decision of the web server was also quick. I wanted to achieve 0 downtime deployments so Github use of [Unicorn + Nginx] [2] jumped to my mind.

## Deploy strategy

This is where things got a little bit complicated. Disclaimer: I'm not a Linux/Unix pro, so many system administration practices where unknown to me prior to this week. Having said that, It was clear to me that the community is very fragmented. There were so many solutions to the same problems and so many scripts! After digging, trying and failing miserably I settled on the stack that caused me the least suffering:

1. Chef solo and Knife for the machine provisioning
2. Capistrano for the deployment

## Chef

[Chef] [3] is a provisioning tool written in Ruby. Its DSL is very expressive and powerful. The community is full of useful cookbooks that ease the setup of common services, however it seemed to lack a way to handle community cookbooks. This is where [Librarian Chef] [4] comes in. I just had to write a Cheffile with all the dependencies and I was done.

```ruby
# Cheffile
#!/usr/bin/env ruby
#^syntax detection

site 'http://community.opscode.com/api/v1'

cookbook 'libqt4',
  :git => 'https://github.com/phlipper/chef-libqt4'

cookbook 'nodejs'
cookbook 'nginx'
cookbook 'runit'
cookbook 'java'
cookbook 'imagemagick'
cookbook 'vim'
cookbook 'ruby_build', :git => 'git://github.com/fnichol/chef-ruby_build.git'
cookbook 'rbenv', :git => 'git://github.com/fnichol/chef-rbenv.git'
cookbook 'redis', :git => 'git://github.com/cassianoleal/chef-redis.git'
cookbook 'memcached'
```

To bootstrap the machine with Chef and Ruby many people where using custom Knife templates that were not working for me. Some installed ruby with RVM, others with rbenv. In the end I found [Knife Solo] [5] that solved all my problems. With one command after the initialization I could install Chef AND run all my recipes to install Ruby and every other service I needed.

```
knife solo init
knife solo bootstrap root@$IP node.json
```

Librarian and Knife Solo forced me to use a specific project structure:

```
mychefrepo/
├── cookbooks
├── site-cookbooks
├── Cheffile
├── Cheffile.lock
└── node.json
```

The node.json contains the run list of recipes:

```json
{
  "user": {
    "name": "deployer",
    "password": $PASSWORD
  },
  "environment": "production",
  "server_name": "goresponsa.com",
  "deploy_to": "/var/www/responsa",
  "ruby-version": "1.9.3-p286",
  "run_list": [
    "recipe[vim]",
    "recipe[libqt4]",
    "recipe[imagemagick]",
    "recipe[java]",
    "recipe[redis::source]",
    "recipe[memcached]",
    "recipe[nodejs]",
    "recipe[ruby_build]",
    "recipe[rbenv::system]",
    "recipe[runit]",
    "recipe[nginx]",
    "recipe[main]"
  ]
}
```

All recipes except the "main" one are taken from community cookbooks.

The main recipe contains machine/application specific setup:

```ruby
# chef/site-cookbooks/main/recipes/default.rb

# setup

rbenv_ruby node['ruby-version']
rbenv_global node['ruby-version']

rbenv_gem 'bundler'

group 'admin' do
  gid 420
end

user node[:user][:name] do
  password node[:user][:password]
  gid 'admin'
  home "/home/#{node[:user][:name]}"
  shell '/bin/bash'
  supports :manage_home => true
end

directory "#{node[:deploy_to]}/tmp/sockets" do
  owner node[:user][:name]
  group 'admin'
  recursive true
end

# certificates

directory "#{node[:deploy_to]}/certificate" do
  owner node[:user][:name]
  recursive true
end

cookbook_file "#{node[:deploy_to]}/certificate/#{node[:environment]}.crt" do
  source "#{node[:environment]}.crt"
  action :create_if_missing
end

cookbook_file "#{node[:deploy_to]}/certificate/#{node[:environment]}.key" do
  source "#{node[:environment]}.key"
  action :create_if_missing
end

# configuration

template '/etc/nginx/sites-enabled/default' do
  source 'nginx.erb'
  owner 'root'
  group 'root'
  mode 0644
  notifies :restart, 'service[nginx]'
end

["sv", "service"].each do |dir|
  directory "/home/#{node[:user][:name]}/#{dir}" do
    owner node[:user][:name]
    group 'admin'
    recursive true
  end
end

runit_service "runsvdir-#{node[:user][:name]}" do
  default_logger true
end

runit_service 'responsa' do
  sv_dir "/home/#{node[:user][:name]}/sv"
  service_dir "/home/#{node[:user][:name]}/service"
  owner node[:user][:name]
  group 'admin'
  restart_command '2'
  restart_on_update false
  default_logger true
end

service 'nginx'
```

I'm using runit to manage the unicorn service that is declared in a template file:

```
# chef/site-cookbooks/main/templates/default/sv-runsvdir-deployer-run.erb

#!/bin/sh
exec 2>&1
exec chpst -u deployer runsvdir /home/deployer/service
```


```
# chef/site-cookbooks/main/templates/default/sv-responsa-run.erb

#!/bin/bash
exec 2>&1

<% unicorn_command = @options[:unicorn_command] || 'unicorn_rails' -%>

#
# Since unicorn creates a new pid on restart/reload, it needs a little extra love to
# manage with runit. Instead of managing unicorn directly, we simply trap signal calls
# to the service and redirect them to unicorn directly.

function is_unicorn_alive {
    set +e
    if [ -n $1 ] && kill -0 $1 >/dev/null 2>&1; then
        echo "yes"
    fi
    set -e
}

echo "Service PID: $$"

CUR_PID_FILE=/var/www/responsa/shared/pids/unicorn.pid
OLD_PID_FILE=$CUR_PID_FILE.oldbin

if [ -e $OLD_PID_FILE ]; then
    OLD_PID=$(cat $OLD_PID_FILE)
    echo "Waiting for existing master ($OLD_PID) to exit"
    while [ -n "$(is_unicorn_alive $OLD_PID)" ]; do
        /bin/echo -n '.'
        sleep 2
    done
fi

if [ -e $CUR_PID_FILE ]; then
    CUR_PID=$(cat $CUR_PID_FILE)
    if [ -n "$(is_unicorn_alive $CUR_PID)" ]; then
        echo "Unicorn master already running. PID: $CUR_PID"
        RUNNING=true
    fi
fi

if [ ! $RUNNING ]; then
    echo "Starting unicorn"
    cd /var/www/responsa/current
    export PATH="/usr/local/rbenv/shims:/usr/local/rbenv/bin:$PATH"
    # You need to daemonize the unicorn process, http://unicorn.bogomips.org/unicorn_rails_1.html
    bundle exec <%= unicorn_command %> -c config/unicorn.rb -E <%= @options[:environment] || 'staging' %> -D
    sleep 3
    CUR_PID=$(cat $CUR_PID_FILE)
fi

function restart {
    echo "Initialize new master with USR2"
    kill -USR2 $CUR_PID
    # Make runit restart to pick up new unicorn pid
    sleep 2
    echo "Restarting service to capture new pid"
    exit
}

function graceful_shutdown {
    echo "Initializing graceful shutdown"
    kill -QUIT $CUR_PID
}

function unicorn_interrupted {
    echo "Unicorn process interrupted. Possibly a runit thing?"
}

trap restart HUP QUIT USR2 INT
trap graceful_shutdown TERM KILL
trap unicorn_interrupted ALRM

echo "Waiting for current master to die. PID: ($CUR_PID)"
while [ -n "$(is_unicorn_alive $CUR_PID)" ]; do
    /bin/echo -n '.'
    sleep 2
done
echo "You've killed a unicorn!"
```

Nginx is used as a reverse proxy:

```
# chef/site-cookbooks/main/templates/default/nginx.erb

upstream unicorn {
  server unix:/var/www/responsa/tmp/sockets/responsa.sock fail_timeout=0;
}

server {
  listen 80;
  listen 443 default ssl;
  server_name <%= node[:server_name] %>;
  root /var/www/responsa/current/public;
  # set far-future expiration headers on static content
  expires max;

  server_tokens off;

  # ssl                  on;
  ssl_certificate      <%= "/var/www/responsa/certificate/#{node[:environment]}.crt" %>;
  ssl_certificate_key  <%= "/var/www/responsa/certificate/#{node[:environment]}.key" %>;

  ssl_session_timeout  5m;

  ssl_protocols  SSLv2 SSLv3 TLSv1;
  ssl_ciphers  HIGH:!aNULL:!MD5;
  ssl_prefer_server_ciphers   on;

  # set up the rails servers as a virtual location for use later
  location @unicorn {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP  $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_intercept_errors on;
    proxy_redirect off;
    proxy_pass http://unicorn;
    expires off;
  }

  location / {
    try_files $uri @unicorn;
  }

  # error_page 500 502 503 504 /500.html;
}
```

And here's the unicorn configuration file:

```ruby
# config/unicorn.rb

rails_env = ENV['RAILS_ENV'] || 'production'

worker_processes (rails_env == 'production' ? 6 : 3)

preload_app true

# Restart any workers that haven't responded in 30 seconds
timeout 30

working_directory '/var/www/responsa/current'

# Listen on a Unix data socket
pid '/var/www/responsa/shared/pids/unicorn.pid'
listen "/var/www/responsa/tmp/sockets/responsa.sock", :backlog => 2048

stderr_path '/var/www/responsa/shared/log/unicorn.log'
stdout_path '/var/www/responsa/shared/log/unicorn.log'

before_exec do |server|
  ENV["BUNDLE_GEMFILE"] = "/var/www/responsa/current/Gemfile"
end

before_fork do |server, worker|
  ##
  # When sent a USR2, Unicorn will suffix its pidfile with .oldbin and
  # immediately start loading up a new version of itself (loaded with a new
  # version of our app). When this new Unicorn is completely loaded
  # it will begin spawning workers. The first worker spawned will check to
  # see if an .oldbin pidfile exists. If so, this means we've just booted up
  # a new Unicorn and need to tell the old one that it can now die. To do so
  # we send it a QUIT.
  #
  # Using this method we get 0 downtime deploys.

  old_pid = '/var/www/responsa/shared/pids/unicorn.pid.oldbin'

  if File.exists?(old_pid) && server.pid != old_pid
    begin
      Process.kill("QUIT", File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
      # someone else did our job for us
    end
  end
end
```

## Capistrano

After setting up the machine I created a snapshot on Digital Ocean, in case I had to restart from scratch.

Time to deploy! Capistrano was an easy choice.

Using Capistrano multistage I set up the production script

```ruby
# config/deploy/production.rb

set :server_ip, $MY_IP
server server_ip, :app, :web, :primary => true
set :rails_env, 'production'
set :branch, 'master'
```

This is used in combo with the deploy script:

```ruby
# config/deploy.rb

require 'bundler/capistrano'
require 'sidekiq/capistrano'
require 'capistrano/ext/multistage'

set :stages, %w(production staging)
set :default_stage, 'staging'

default_run_options[:pty] = true
ssh_options[:forward_agent] = true

set :application, 'responsa'
set :repository,  $PATH_TO_GITHUB_REPO
set :deploy_to, "/var/www/#{application}"
set :branch, 'development'

set :scm, :git
set :scm_verbose, true

set :deploy_via, :remote_cache
set :use_sudo, true
set :keep_releases, 3
set :user, 'deployer'

set :bundle_without, [:development, :test, :acceptance]

set :rake, "#{rake} --trace"

set :default_environment, {
  'PATH' => '/usr/local/rbenv/shims:/usr/local/rbenv/bin:$PATH'
}

after 'deploy:update_code', :upload_env_vars

after 'deploy:setup' do
  sudo "chown -R #{user} #{deploy_to} && chmod -R g+s #{deploy_to}"
end

namespace :deploy do
  desc <<-DESC
  Send a USR2 to the unicorn process to restart for zero downtime deploys.
  runit expects 2 to tell it to send the USR2 signal to the process.
  DESC
  task :restart, :roles => :app, :except => { :no_release => true } do
    run "sv 2 /home/#{user}/service/#{application}"
  end
end

task :upload_env_vars do
  upload(".env.#{rails_env}", "#{release_path}/.env.#{rails_env}", :via => :scp)
end
```

Now with two simple commands I can deploy with 0 downtime!

```
cap deploy:setup
cap deploy
```

I must thank czarneckid for sharing [his setup on Github] [6] from which I stole some useful portions and also [@bugant] [9] for his patience.

[1]: http://rapgenius.com/James-somers-herokus-ugly-secret-lyrics

[2]: https://github.com/blog/517-unicorn

[3]: http://www.opscode.com/chef/

[4]: https://github.com/applicationsonline/librarian-chef

[5]: http://matschaffer.github.com/knife-solo/

[6]: https://gist.github.com/czarneckid/4639793

[7]: https://www.digitalocean.com/

[8]: https://goresponsa.com

[9]: https://twitter.com/bugant