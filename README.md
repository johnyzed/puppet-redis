#redis

####Requisits
In order to compute dynamically the memory usage of the redis server, this module uses a custom fact called "memorysize_mb".
Such a file can be create using a custun fact script like [this one](https://gist.github.com/johnyzed/8bed34f99667ace51016) .

####Table of Contents

1. [Overview - What is the graphite module?](#overview)
2. [Setup - The basics of getting started with redis](#setup)
    * [Beginning with redis - Installation](#beginning-with-redis)
    * [Run multiple instances on same host](#run-multiple-instances-on-same-host)
    * [Setting up a Matser Slave architecture](#setting-up-a-matser-slave-architecture)
    * [Setting up sentinel with two monitors](#setting-up-sentinel-with-two-monitors)
3. [Usage - The class and defined types available for configuration](#usage)
    * [Classes and Defined Types](#classes-and-defined-types)
        * [Class: redis::install](#class-redisinstall)
        * [Defined Type: redis::server](#defined-type-redisserver)
        * [Defined Type: redis::sentinel](#defined-type-redissentinel)
4. [Limitations - OS compatibility, etc.](#limitations)
5. [Contributing to the graphite module](#contributing)

##Overview

This module installs and makes basic configs for multiple redis instances on
the same node. It installs redis from source. (http://redis.io/)
It also can configure the monitoring server Sentinel or create a master/slave architecture.

##Setup

**What redis affects:**

* packages/configuration to compile and install redis from source
* services/configuration files to run multiple redis and sentinels

###Beginning with redis

To just compile and install redis binaries. As default the
latest stable release will be used.

```puppet
  class { 'redis::install': }
```

To install a specific redis version use the following parameters.
Most of the time you will only need `redis_version`.

```puppet
  class { 'redis::install':
  	redis_version     => '2.8.8',
	redis_build_dir   => '/opt',
	redis_install_dir => '/usr/bin'
  }
```

###Run multiple instances on same host

As example run two redis instances on port 6379 and 6380.

```puppet
node 'redis.my.domain' {

  # install latest stable build.
  class { 'redis::install': }

  redis::server {
    'instance1':
      redis_memory    => '1g',
      redis_ip        => '0.0.0.0',
      redis_port      => 6379,
      redis_mempolicy => 'allkeys-lru',
      redis_timeout   => 0,
      redis_nr_dbs    => 16,
      redis_loglevel  => 'notice',
      running         => true,
      enabled         => true
  }

  redis::server {
    'secondRedis':
      redis_memory    => '112m',
      redis_ip        => '0.0.0.0',
      redis_port      => 6380,
      redis_mempolicy => 'allkeys-lru',
      redis_timeout   => 0,
      redis_nr_dbs    => 2,
      redis_loglevel  => 'info',
      running         => true,
      enabled         => true
  }
}
```
###Setting up a Matser Slave architecture

You can create a Redis Master Slave architecture.

```puppet
node 'redis.my.domain' {

  # install latest stable build.
  class { 'redis::install': }

  redis::server {
    'master':
      redis_memory    => '1g',
      redis_ip        => '0.0.0.1',
      redis_port      => 6379,
      redis_mempolicy => 'allkeys-lru',
      redis_timeout   => 0,
      redis_nr_dbs    => 16,
      redis_loglevel  => 'notice',
      running         => true,
      enabled         => true
  }

  redis::server {
    'slave':
      redis_memory    => '112m',
      redis_ip        => '0.0.0.2',
      redis_port      => 6379,
      redis_mempolicy => 'allkeys-lru',
      redis_timeout   => 0,
      redis_nr_dbs    => 2,
      redis_loglevel  => 'info',
      redis_is_slave  => true,
      redis_master_IP_for_slave => '0.0.0.1',
      running         => true,
      enabled         => true
  }
}
```
###Setting up sentinel with two monitors

You can create multiple sentinels on one node. But most of the time you will
want to create a sentinel with one or more monitors configured.

```puppet
node 'sentinel.my.domain' {

  # install latest stable build.
  class { 'redis::install': redis_version => '2.8.8' }

  redis::sentinel {'clusters':
    monitors => {
      'mymaster' => {
        master_host             => '127.0.0.1',
        master_port             => 6378,
        quorum                  => 2,
        down_after_milliseconds => 30000,
        parallel-syncs          => 1,
        failover_timeout        => 180000
      },
      'securetRedisCluster' => {
        master_host             => '10.20.30.1',
        master_port             => 6379,
        quorum                  => 2,
        down_after_milliseconds => 30000,
        parallel-syncs          => 5,
        failover_timeout        => 180000,
        auth-pass => 'secret_Password',
        notification-script => '/tmp/notify.sh',
        client-reconfig-script => '/tmp/reconfig.sh'
      }
    }
  }
```

##Usage

###Classes and Defined Types

This module compiles and installs redis with the class `redis::install`. 
The redis service(s) are configured with the defined type `redis::server`.

####Class: `redis::install`

This class downloads, compiles and installs redis. It does not configure any
redis services. This is done by defimed type redis::server.

**Parameters within `redis::install`:**

#####`redis_version`

The redis version to be installed. 
By default, the latest stable build will be installed.

#####`redis_build_dir`

Default is '/opt/' (string)
The dir to store redis source code. This will result in a
directoy like '/opt/redis-2.8.8/'

#####`redis_install_dir`

Default is '/usr/bin' (string).
The dir to which the newly built redis binaries are copied. 

####Defined Type: `redis::server`

Used to configure redis instances. You can setup multiple redis servers on the
same node. See the setup examples.

**Parameters within `redis::server`

#####`redis_name`

Name of Redis instance. Default: call name of the function.

#####`redis_memory`

Default is '100mb' (string).
Sets amount of memory used. eg. 100mb or 4g.

#####`redis_ip`

Default is '127.0.0.1' (string). Listen IP of redis.

#####`redis_port`

Listen port of Redis. Default: 6379

#####`redis_mempolicy`

Algorithm used to manage keys. See Redis docs for possible values. Default: allkeys-lru

#####`redis_timeout`

Default: 0

#####`redis_nr_dbs`

Number of databases provided by redis. Default: 1

#####`redis_dbfilename`

Name of database dump file. Default: dump.rdb

#####`redis_dir`

Default is '/var/lib' (string)
Path for persistent data. Path is <redis_dir>/redis_<redis_name>/.

#####`redis_log_dir`

Default is '/var/log' (string).
Path for log. Full log path is <redis_log_dir>/redis_<redis_name>.log. 

#####`redis_loglevel`

Loglevel of Redis. Default: 'notice' (string)

#####`running`

Configure if Redis should be running or not. Default: true (boolean)

#####`enabled`

Configure if Redis is started at boot. Default: true (boolean)

####Defined Type: `redis::sentinel`

Used to configure sentinel instances. You can setup multiple sentinel servers
on the same node. And you can configure multiple monitors within a sentinel.
See the setup examples.

**Parameters within `redis::sentinel`

#####`sentinel_name`

Name of Redis instance. Default: call name of the function.

#####`sentinel_port`

Listen port of Redis. Default: 6379

#####`sentinel_log_dir`

Default is '/var/log' (string).
Path for log. Full log path is `sentinel_log_dir`/sentinel_`sentinel_name`.log.

#####`monitors`

Default is
```
{
  'mymaster' => {
    master_host             => '127.0.0.1',
    master_port             => 6379,
    quorum                  => 2,
    down_after_milliseconds => 30000,
    parallel-syncs          => 1,
    failover_timeout        => 180000,
    ### optional
    auth-pass => 'secret_Password',
    notification-script => '/var/redis/notify.sh',
    client-reconfig-script => '/var/redis/reconfig.sh'
  },
}
```
Hashmap of monitors.

#####`running`

Configure if Redis should be running or not. Default: true (boolean)

#####`enabled`

Configure if Redis is started at boot. Default: true (boolean)


##Limitations

This module is tested on CentOS 6.5 and Debian 7 (Wheezy) and should also run without problems on

* RHEL/CentOS/Scientific 6+
* Debian 6+
* Ubunutu 10.04 and newer

##Contributing

Echocat modules are open projects. So if you want to make this module even better, you can contribute to this module on [Github](https://github.com/echocat/puppet-redis).
