## How to crate bosh-release

Let's create bosh release used in [BOSH Tutorial](http://mariash.github.io/learn-bosh/)!

### init

```
bosh init-release --dir=simple-server-boshrelease --git
cd simple-server-boshrelease
```

### ruby package

#### add blob

```
bosh generate-package ruby

curl -L -O -J https://cache.ruby-lang.org/pub/ruby/2.1/ruby-2.1.4.tar.gz
curl -L -O -J https://rubygems.org/downloads/bundler-1.6.3.gem

bosh add-blob ruby-2.1.4.tar.gz ruby/ruby-2.1.4.tar.gz
bosh add-blob bundler-1.6.3.gem ruby/bundler-1.6.3.gem

rm -f ruby-2.1.4.tar.gz
rm -f bundler-1.6.3.gem
```

#### spec

```
cat <<"EOF" > packages/ruby/spec
---
name: ruby

dependencies: []

files:
- ruby/ruby-2.1.4.tar.gz
- ruby/bundler-1.6.3.gem
EOF
```

#### packaging

```
cat <<"EOF" > packages/ruby/packaging
set -e

tar xzf ruby/ruby-*.tar.gz
(
  set -e
  cd ruby-2.1.4
  LDFLAGS="-Wl,-rpath -Wl,${BOSH_INSTALL_TARGET}" CFLAGS='-fPIC' ./configure --prefix=${BOSH_INSTALL_TARGET} --disable-install-doc --with-opt-dir=${BOSH_INSTALL_TARGET} --without-gmp
  make
  make install
)

${BOSH_INSTALL_TARGET}/bin/gem install ruby/bundler-1.6.3.gem --local --no-ri --no-rdoc
EOF
```

### simple-server package


```
bosh generate-package simple-server

git submodule add https://github.com/making/simple-server.git src/simple-server
```

#### spec

```
cat <<"EOF" > packages/simple-server/spec
---
name: simple-server

dependencies:
- ruby

files:
- simple-server/**/*
EOF
```

#### packaging

```
cat <<"EOF" > packages/simple-server/packaging
set -e

cp -r simple-server/* ${BOSH_INSTALL_TARGET}
cd ${BOSH_INSTALL_TARGET}
mkdir -p ${BOSH_INSTALL_TARGET}/gem_home
/var/vcap/packages/ruby/bin/bundle install --local --no-prune --path ${BOSH_INSTALL_TARGET}/gem_home
EOF
```



### app job

```
bosh generate-job app
```

#### ctl

```
cat <<"EOF" > jobs/app/templates/ctl
#!/bin/bash

RUN_DIR=/var/vcap/sys/run/app
LOG_DIR=/var/vcap/sys/log/app

PIDFILE=$RUN_DIR/app.pid
RUNAS=vcap

export PATH=/var/vcap/packages/ruby/bin:$PATH
export BUNDLE_GEMFILE=/var/vcap/packages/simple-server/Gemfile
export GEM_HOME=/var/vcap/packages/simple-server/gem_home/ruby/2.1.0

function pid_exists() {
  ps -p $1 &> /dev/null
}

case $1 in

  start)
    mkdir -p $RUN_DIR $LOG_DIR
    chown -R $RUNAS:$RUNAS $RUN_DIR $LOG_DIR

    echo $$ > $PIDFILE

    exec chpst -u $RUNAS:$RUNAS \
      bundle exec ruby /var/vcap/packages/simple-server/app.rb \
      -p <%= p("port") %> \
      -o 0.0.0.0 \
      >>$LOG_DIR/server.stdout.log 2>>$LOG_DIR/server.stderr.log
    ;;

  stop)
    PID=$(head -1 $PIDFILE)
    if [ ! -z $PID ] && pid_exists $PID; then
      kill $PID
    fi
    while [ -e /proc/$PID ]; do sleep 0.1; done
    rm -f $PIDFILE
    ;;

  *)
  echo "Usage: ctl {start|stop|console}" ;;
esac
exit 0
EOF

chmod +x jobs/app/templates/ctl
```

#### monit

```
cat <<"EOF" > jobs/app/monit
check process app
  with pidfile /var/vcap/sys/run/app/app.pid
  start program "/var/vcap/jobs/app/bin/ctl start"
  stop program "/var/vcap/jobs/app/bin/ctl stop"
  group vcap
EOF
```

#### spec

```
cat <<"EOF" > jobs/app/spec
---
name: app
templates:
  ctl: bin/ctl

packages:
- simple-server
- ruby

properties:
  port:
    description: "Port on which server is listening"
    default: 8080
EOF
```


### router job

```
bosh generate-job router
```

#### ctl

```
cat <<"EOF" > jobs/router/templates/ctl
#!/bin/bash

RUN_DIR=/var/vcap/sys/run/router
LOG_DIR=/var/vcap/sys/log/router

PIDFILE=$RUN_DIR/router.pid
RUNAS=vcap

export PATH=/var/vcap/packages/ruby/bin:$PATH
export BUNDLE_GEMFILE=/var/vcap/packages/simple-server/Gemfile
export GEM_HOME=/var/vcap/packages/simple-server/gem_home/ruby/2.1.0

function pid_exists() {
  ps -p $1 &> /dev/null
}

case $1 in

  start)
    mkdir -p $RUN_DIR $LOG_DIR
    chown -R $RUNAS:$RUNAS $RUN_DIR $LOG_DIR

    echo $$ > $PIDFILE

    export CONFIG_FILE=/var/vcap/jobs/router/config/config.json

    exec chpst -u $RUNAS:$RUNAS \
      bundle exec ruby /var/vcap/packages/simple-server/router.rb \
      -p <%= p("port") %> \
      -o 0.0.0.0 \
      >>$LOG_DIR/server.stdout.log 2>>$LOG_DIR/server.stderr.log
    ;;

  stop)
    PID=$(head -1 $PIDFILE)
    if [ ! -z $PID ] && pid_exists $PID; then
      kill $PID
    fi
    while [ -e /proc/$PID ]; do sleep 0.1; done
    rm -f $PIDFILE
    ;;

  *)
  echo "Usage: ctl {start|stop|console}" ;;
esac
exit 0
EOF

chmod +x jobs/router/templates/ctl
```

#### config.json.erb

```
cat <<"EOF" > jobs/router/templates/config.json.erb
{
  "servers": <%= p("servers") %>
}
EOF
```

#### monit

```
cat <<"EOF" > jobs/router/monit
check process router
  with pidfile /var/vcap/sys/run/router/router.pid
  start program "/var/vcap/jobs/router/bin/ctl start"
  stop program "/var/vcap/jobs/router/bin/ctl stop"
  group vcap
EOF
```

#### spec

```
cat <<"EOF" > jobs/router/spec
---
name: router
templates:
  ctl: bin/ctl
  config.json.erb: config/config.json

packages:
- simple-server
- ruby

properties:
  port:
    description: "Port on which server is listening"
    default: 8080
  servers:
    description: "List of servers to redirect requests"
    default: []
EOF
```

### config/final.yml

```
cat <<"EOF" > config/final.yml
---
blobstore:
  provider: local
  options:
    blobstore_path: /tmp/simple-server
final_name: simple-server
EOF
```


### create release

```
bosh create-release --name=simple-server --force --timestamp-version --tarball=/tmp/simple-server-boshrelease.tgz
bosh upload-release /tmp/simple-server-boshrelease.tgz
```


### manifest


```
cat <<"EOF" > manifest.yml
---
name: simple-server

releases:
- name: simple-server
  version: latest

stemcells:
- os: ubuntu-trusty
  alias: trusty
  version: latest

instance_groups:
- name: app
  jobs:
  - name: app
    release: simple-server
    properties:
      port: 8080
  instances: 2
  stemcell: trusty
  azs: [z1]
  vm_type: default
  networks:
  - name: default
    static_ips: ((app-ips))

- name: router
  jobs:
  - name: router
    release: simple-server
    properties:
      port: 8080
      servers: ((app-ips))
  instances: 1
  stemcell: trusty
  azs: [z1]
  vm_type: default
  networks:
  - name: default
    static_ips: ((router-ip))

update:
  canaries: 1
  max_in_flight: 3
  canary_watch_time: 30000-600000
  update_watch_time: 5000-600000
EOF
```

### deploy

```
bosh deploy -d simple-server manifest.yml -v app-ips="[10.0.16.30,10.0.16.31]" -v router-ip=10.0.16.40
```


### link

```
cat <<"EOF" > jobs/app/spec
---
name: app
templates:
  ctl: bin/ctl

provides:
- name: app
  type: conn
  properties:
  - port

packages:
- simple-server
- ruby

properties:
  port:
    description: "Port on which server is listening"
    default: 8080
EOF
```



```
cat <<"EOF" > jobs/router/spec
---
name: router
templates:
  ctl: bin/ctl
  config.json.erb: config/config.json

consumes:
- name: app
  type: conn

packages:
- simple-server
- ruby

properties:
  port:
    description: "Port on which server is listening"
    default: 8080
EOF
```


```
cat <<"EOF" > jobs/router/templates/config.json.erb
{
  "servers": <%= JSON.dump(link('app').instances.map { |x| x.address }) %>
}
EOF
```


```
cat <<"EOF" > manifest.yml
---
name: simple-server

releases:
- name: simple-server
  version: latest

stemcells:
- os: ubuntu-trusty
  alias: trusty
  version: latest

instance_groups:
- name: app
  jobs:
  - name: app
    release: simple-server
    properties:
      port: 8080
  instances: 2
  stemcell: trusty
  azs: [z1]
  vm_type: default
  networks:
  - name: default

- name: router
  jobs:
  - name: router
    release: simple-server
    properties:
      port: 8080
  instances: 1
  stemcell: trusty
  azs: [z1]
  vm_type: default
  networks:
  - name: default
    static_ips: ((router-ip))

update:
  canaries: 1
  max_in_flight: 3
  canary_watch_time: 30000-600000
  update_watch_time: 5000-600000
EOF
```

```
bosh deploy -d simple-server manifest.yml -v router-ip=10.0.16.40
```