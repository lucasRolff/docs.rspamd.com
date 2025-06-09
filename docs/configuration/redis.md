---
title: Redis configuration
---


# Redis configuration for Rspamd

This document provides instructions on setting up a Redis cache in Rspamd.

## Introduction

The [Redis](https://redis.io) cache server, is utilized as a highly efficient key-value storage by various Rspamd modules, including the following:

* [Ratelimit plugin](/modules/ratelimit) uses Redis to store limits buckets
* [Greylisting module](/modules/greylisting) stores data and meta hashes inside Redis
* [DMARC module](/modules/dmarc) can save DMARC reports inside Redis keys
* [Replies plugin](/modules/replies) requires Redis to save message ids hashes for outgoing messages
* [IP score plugin](/modules/ip_score) uses Redis to store data about AS, countries and networks reputation
* [Multimap module](/modules/multimap) can use Redis as readonly database for maps
* [MX Check module](/modules/mx_check) uses Redis for caching
* [Reputation module](/modules/reputation) uses Redis for caching
* [Neural network module](/modules/neural) uses Redis for data storage

Furthermore, Redis is used to store Bayes tokens in the [statistics](/configuration/statistic) module. Rspamd offers multiple configuration options for Redis storage. Moreover, Redis [replication](https://redis.io/docs/management/replication/)is supported, enabling Rspamd to **write** values to one set of Redis servers and **read** data from another set.

## Redis setup

There are couple of ways to configure Redis for a module. First of all, you can place all Redis options inside the relevant module's section:

~~~hcl
dmarc {
  servers = "127.0.0.1";
}
~~~

However, it is recommended to use the local and override directories for such configurations. For instance, you can create a file called `/etc/rspamd/local.d/dmarc.conf` for this specific case:
~~~hcl
# local.d/dmarc.conf
servers = "127.0.0.1";
~~~

You can specify multiple servers by separating them with commas. Each server can also have an optional port value:

~~~hcl
  servers = "serv1,serv2:6371";
~~~

By default, Rspamd uses port `6379` for Redis. Alternatively, you can define the full features of upstream options when specifying servers:

~~~hcl
  servers = "master-slave:127.0.0.1,10.0.1.1";
~~~

You can simplify the configuration of Redis options for each individual module by using a common `redis` section. This can be done by defining the section in a separate file, such as `/etc/rspamd/local.d/redis.conf`. This approach helps streamline the configuration process and make it more organized. 
For more detailed information, you can refer to the [upstreams documentation](/configuration/upstream).

~~~hcl
# /etc/rspamd/local.d/redis.conf
read_servers = "127.0.0.1,10.0.0.1";
write_servers = "127.0.0.1";
~~~

Please bear in mind that you should either use `servers` for both `read_servers` and `write_servers` or define `read_servers` and `write_servers` separately. So it is **either** `servers` or `read_servers`+`write_servers` **together**.

Additionally, you have the option to redefine Redis options inside the `redis` section specifically for the desired module or modules. This provides flexibility in configuring Redis options on a per-module basis:

~~~hcl
# /etc/rspamd/local.d/redis.conf
read_servers = "127.0.0.1,10.0.0.1";
write_servers = "127.0.0.1";

dmarc {
  servers = "10.0.1.1";
}
~~~

In this example, the `dmarc` module will use a different set of servers than other modules. If you want to exclude specific modules from using the common Redis options, you can add them to the `disabled_modules` list, like this:

~~~hcl
# /etc/rspamd/local.d/redis.conf
servers = "127.0.0.1";

disabled_modules = ["ratelimit"];
~~~

This configuration snippet prevents the `ratelimit` module from using the common Redis configuration. If Redis is not explicitly configured for this module (either in the `redis` -> `ratelimit` section or in the `ratelimit` section), the module will be disabled.

## Available Redis options

Rspamd supports the following Redis options (common for all modules):

* `servers`: [upstreams list](/configuration/upstream) for both read and write requests
* `read_servers`: [upstreams list](/configuration/upstream) for read only servers (usually replication slaves)
* `write_servers`: [upstreams list](/configuration/upstream) for write only servers (usually replication master)
* `timeout`: timeout in seconds to get reply from Redis (e.g. `0.5s` or `1min`)
* `db`: number of database to use (by default, Rspamd will use the default Redis database with number `0`)
* `password`: password to connect to Redis (no password by default)
* `username`: username to use when authenticating to Redis
* `prefix`: use the specified prefix for keys in Redis (if supported by module)
* `expand_keys` (1.7.0+): if set to `true` 'expand' key names used in queries (discussed further below)

## Key expansion

Starting from version 1.7.0, if you set the `redis.expand_keys` configuration parameter to `true`, special values can be replaced in key names before they are sent to Redis when performing queries via Lua (as in plugins like `multimap`, `ratelimit`, and `settings`). If you are using module-specific Redis configuration, you can specify this setting in the module configuration instead of `redis`.

Given this setting is enabled, where-ever names of keys could be specified in configuration special values could be used, for example `map = "redis://${ip}!foo` in multimap configuration would dynamically set key name to something like `1.2.3.4!foo`. Variable names which could be used are as follows:

* `ip`: sending IP of a message
* `principal_recipient`: the address of the [principal recipient](/lua/rspamd_task#mc5168) of a message (`Deliver-To` request header, first SMTP recipient or MIME recipient according to availability)
* `principal_recipient_domain`: the domain name of the principal recipient
* `esld_principal_recipient_domain`: the domain name of the principal recipient, normalised to eSLD
* `smtp_from`: SMTP sender address
* `smtp_from_domain`: SMTP sender address domain
* `esld_smtp_from_domain`: SMTP sender address domain, normalised to eSLD
* `smtp_from_domain_or_helo`: SMTP sender address domain or HELO if address is empty/absent
* `esld_smtp_from_domain_or_helo`: SMTP sender address domain or HELO if address is empty/absent, normalised to eSLD
* `mime_from`: MIME sender address
* `mime_from_domain`: MIME sender address domain
* `esld_mime_from_domain`: MIME sender address domain, normalised to eSLD

## Redis Sentinel

From the version 1.8.3, Rspamd supports [Redis Sentinel](https://redis.io/topics/sentinel). Sentinels could be defined as following:

~~~hcl
# local.d/redis.conf
sentinels = "127.0.0.1,10.0.0.1,10.0.0.2"; # Servers list (default port 5000)
sentinel_watch_time = 1min; # How often Rspam will query sentinels for masters and slaves
sentinel_masters_pattern = "^mymaster.*$"; # Defines masters pattern to match in Lua syntax (no pattern means all masters)
~~~

Rspamd will utilize sentinels to determine which servers should be designated as write_servers and which should be designated as read_servers. Although Redis sentinel does not fully support multi-master configurations, this feature can still be valuable for switching masters when accessing non-volatile data such as statistics, fuzzy storage, or neural network data. However, it is important to note that you must still configure an initial set of servers to be used in case no sentinel is accessible.
