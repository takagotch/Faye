### Faye
---
https://faye.jcoglan.com/ruby.html

https://github.com/faye/faye

```ruby
# config/recipes/faye.rb
set_default(:faye_user) { user }
set_default(:faye_pid) { "#{current_path}/tmp/pids/faye.pid" }
namespace :faye do
  desc "Setup Faye initializer"
  task :setup, roles: :app do
    template "faye_init", "/tmp/faye_init"
    run "chmod +x /tmp/faye_init"
    run "#{sudo} mv /tmp/faye_init /etc/init.d/faye_#{application}"
    run "#{sudo} update-rc.d -f faye_#{application} defaults"
  end
  after "deploy:setup", "faye::setup"
  %w[start stop restart].each do |command|
    desc "#{command} faye"
    task command, roles: :app do
      run "service faye_#{application} #{command}"
    end
    after "deploy:#{command}", "faye:#{command}"
  end
end

# config/deploy.rb
set :faye_pid, "#{deploy_to}/shared/pids/faye.pid"
set :faye_config, "#{deploy_to}/current/faye.ru"
namespace :faye do
  desc "Start Faye"
  task :start do
    run "cd #{deploy_to}/current && bundle exec rackup #{faye_config} -s thin -E production -D --pid #{faye_pid}"
  end
  desc "Stop Faye"
  task :stop do
    run "kill `cat #{faye_pid}` || true"
  end
end
before 'deploy:update_code', 'faye:stop'
after 'deploy:finalize_update', 'faye:start'

# config/deploy.rb
set :user, "deployer"
set :application, "my_application"
set :deploy_to, "/home/#{user}/apps/#{application}"
set :deploy_via, :remote_cache
set :use_sudo, false
load "config/recipes/faye"

# config/recipes/templates/faye_init
#
#!/bin/sh
# Provides: faye
# Required-Start: $remote_fs $syslog
# Required-Stop: $remote_fs $syslog
# Default-Start: 2 3 4 5
# Default-Stop: 0 1 6
# Short-Description: Manage faye server
# Description: Start, stop, restart faye server for a specific application.
### END INIT INFO
set -e
AS_USER=<%= faye_user %>
RAILS_ENV=production
APP_ROOT=<%= current_path %>
FAYE_CONF="faye.ru"
PID=<%= faye_pid %>
CMD="cd <%= current_path %>; RAILS_ENV=$RAILS_ENV bundle exec rackup $FAYE_CONF -s thin -E $RAILS_ENV -D --pid $PID"
set -u
OLD_PIN="$PID.oldbin"
sig () {
  test -s "$PID" && kill -$1 `cat $PID`
}
oldsig(){
  test -s $OLD_PIN && kill -$1 `cat $OLD_PIN`
}
run(){
  if [ "$(id -un)" = "$AS_USER" ]; then
    eval $1
  else
    su -c "$1" - $AS_USER
  fi
}
case "$1" in
start)
  sig 0 && echo >&2 "Already running" && exit 0
  run "$CMD"
  ;;
stop)
  sig QUIT && exit 0
  echo >&2 "Not running"
  ;;
force-stop)
  sig TERM && exit 9
  echo >&2 "Couldn't reload, starting '$CMD' instead"
  ;;
restart|reload)
  sig HUP && echo reloaded OK && exit 0
  echo >&2 "Couldn't reload, starting '$CMD' instead"
  run "$CMD"
  ;;
*)
  echo >&2 "Usage: $0 <start|stop|restart|force-stop>"
  exit 1
  ;;
esac
```

```sh
#!/bin/sh
exec 2>&1
export USER=deploy
export RAILS_ENV=production
APP_ROOT=/home/deploy/my_project
FAYE_CONF="$APP_ROOT/current/faye.ru"
FAYE="/usr/local/rvm/bin/rvm ree@my_project exec bundle rackup $FAYE_CONF -s thin -E $RAILS_ENV --pid $APP_ROOT/shared/pids/faye.pid"
echo $FAYE
cd $APP_ROOT/current
exec $FAYE


```

```
```

