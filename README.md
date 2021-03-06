<!-- TOC -->

- [Name](#name)
- [Status](#status)
- [Synopsis](#synopsis)
- [Pre Requistes](#pre-requistes)
- [Description](#description)
- [Leaky Bucket and Sliding Window](#leaky-bucket-and-sliding-window)
- [Background Ratelimiter](#background-ratelimiter)
- [Foreground Ratelimiter](#foreground-ratelimiter)
- [Quick Example](#quick-example)
    - [Init and Access block](#init-and-access-block)
    - [Access Block](#access-block)
    - [API Specification](#api-specification)
        - [Require the library](#require-the-library)
        - [Create a rate limiting object](#create-a-rate-limiting-object)
        - [Check if we should rate limit](#check-if-we-should-rate-limit)
- [Redis Failure](#redis-failure)
- [Local Testing](#local-testing)
- [See Also](#see-also)

<!-- /TOC -->


# Name

Redis Backed Ratelimiter

----

# Status

In Development

----

# Synopsis

Implementation of global rate limiting for nginx using Redis as a back end.


----

# Pre Requistes

- Nginx that can use Lua.  For example [Openresty](https://github.com/openresty/)
-- Needs the nginx lua development kit: [lua-nginx-module](https://github.com/openresty/lua-nginx-module)

- The Lua Nginx cosocket library: [resty redis library](https://github.com/openresty/lua-resty-redis).

- A Redis Backend. This can be your own or an AWS elasticache redis master/replica setup (*Have not tried with AWS cluster mode*).

----

# Description

There are 2 variations of the rate limiting within the library, depending upon your needs for
accuracy and/or performance.  The recommended ratelimiter is the *background* ratelimiter (default).

The difference between `background` and `foreground` is that with the `foreground` mode, redis operations occur on the request thread.  This means latency of the redis calls impact the latency of a user's request; however it provides increase accuracy for ratelimiting requests.

The ratelimit is based on the cloudflare ratelimiter as described on this blog (I have no knowledge of the actual implementation details of the cloudflare rate limiter.  The implementation in here is soley based on the details described on the following blog):

- https://blog.cloudflare.com/counting-things-a-lot-of-different-things/

All of the rate limiting implementations share the same fundamental algorithm of the leaky bucket algorithm, and use the same approximation calculation as specified on the above blog:

```
rate = number_of_requests_in_previous_window * ((window_period - elasped_time_in_current_period) / window_period) + number_of_requests_in_current_window
```

# Leaky Bucket and Sliding Window

All the implementations provided use the same concept of:

- leaky bucket
- sliding window

This is implemented by storing in REDIS two counters per key (the key being the item you chose to rate limit on: ip address, client token).

When you check if a request is rate limited you pass in the [ngx.req.start_time()](https://github.com/openresty/lua-nginx-module#ngxreqstart_time).  If you are rate limiting using requests per second (r/s), the time is rounded down to the start of the second.  If you are rate limiting using requests per minute (r/m), the time is rounded down to the start of that second's minute.  

The counter that is incremented for each request is recorded against that rounded down time.

As a result we store 2 keys in Redis per rate limit counter.  The currently incrementing counter, and the previous period's counter.

For example for the following:
```
local key = ngx.var.remote_addr
local is_rate_limited = lim:is_rate_limited(key)
```

The keys in REDIS will look something like the following, for rate limit by seconds:

- PREVIOUS: <zone>:127.0.0.1_1525032761 == 2
- CURRENT:  <zone>:127.0.0.1_1525032762 == 2

The keys in REDIS will look something like the following, for rate limit by minute:

- PREVIOUS: <zone>:127.0.0.1_1525514700 == 0
- CURRENT:  <zone>:127.0.0.1_1525514760 == 2


When the rate limit is exceeded, a key entry for the current period is placed in within an [nginx shared dict](https://github.com/openresty/lua-nginx-module#ngxshareddict).  We can then check the shared dict to see if the rate has been exceeded for the current key without having to talk to REDIS.  The entry in the shared dict lasts for (expires after) the remaining time left in the current time window (i.e. if we are 15 seconds into the current minute, the entry will expire after 45 seconds).  The entry in the dict is based on the `<zone>:<key>`

The shared dict by default is named `ratelimit_circuit_breaker`, but can be changed per rate limiting object

```
lua_shared_dict authorisationapi_ratelimit_circuit_breaker 1m;
lua_shared_dict logic_ratelimit_circuit_breaker 1m;
```

```
location /login {
    access_by_lua_block {
        local config = { circuit_breaker_dict_name = "filter_ratelimit_circuit_breaker" }
        local zone = "filter"
        local lim, _ = ratelimit.new(zone, "2r/s", config)
    }
    ...
}
```

If you are using multiple shared dicts, you still need to made sure the "zone" is set differently, as the key will be stored in the same redis (unless you are configuring multiple redis endpoint, for rate limiting)

----


# Background Ratelimiter

The background rate limit performs redis operations in a the background using a light wieght thread. [ngx.timer.at](https://github.com/openresty/lua-nginx-module#ngxtimerat)
The reason for use of background thread is so that redis operations will not add to any client request's response time.  However, this does mean
it is possible for background tasks to accumulate in the server and exhaust system resources due to just too much client traffic.

To prevent extreme consequences like crashing the Nginx server, there are built-in limitations on both the number of "pending timers" and the number of "running timers" in an Nginx worker process. The "pending timers" here mean timers that have not yet been expired and "running timers" are those whose user callbacks are currently running.

The maximal number of pending timers allowed in an Nginx worker is controlled by the [lua_max_pending_timers](https://github.com/openresty/lua-nginx-module#lua_max_pending_timers) directive. The maximal number of running timers is controlled by the [lua_max_running_timers](https://github.com/openresty/lua-nginx-module#lua_max_running_timers) directive.

Make sure you check the error log for "`too many pending timers`" or "`lua_max_running_timers are not enough`"

By using background rate limiting, it also means it is possible to allow in a burst of traffic, before the current nginx's shared dict circuit breaker is flipped and traffic is rate limited.  As a result it is possible that more traffic that the allow rate limit is let through in a spike.  Depending upon the response time of redis to process the increment that takes the current requests over the limit, is the window of time that the spike of requests will be allowed through.

# Foreground Ratelimiter

The foreground ratelimit is exactly the same functionality of the `background` ratelimiter.  The only difference being the redis operations work on the request thread.
This means the latency of redis calls adds to the latency of the user's request.  The more latent that redis is, the more latent that the nginx request will be.

This increase in latency, and coupling, between nginx and redis; does have 1 benefit.  That of increased accuracy.  When the rate limit has been hit the request will be ratelimited.
With the background rate limiting that request are always let through, and it is only the shared dict that controls when ratelimiting occurs.  With the foreground limiting, the request is rate limited by both the values calculated from redis, and the shared dict


# Quick Example

There's a couple of ways to set up the rate limiting:

- A combination of `init_by_lua_block` and `access_by_lua_block`
- Entirely the `access_by_lua_block`

Which is entirely up to you.  For either, you need to set up the `lua_shared_dict` in the `http` regardless.


## Init and Access block

Inside the http block, set up the `init_by_lua_block` and the shared dict
```
http {
    ...
    lua_shared_dict ratelimit_circuit_breaker 10m;

    init_by_lua_block {
        local ratelimit = require "resty.greencheek.redis.ratelimiter.limiter"
        local red = { host = "127.0.0.1", port = 6379, timeout = 100}
        login, err = ratelimit.new("login", "100r/s", red)

        if not login then
            error("failed to instantiate a resty.greencheek.redis.ratelimiter.limiter object")
        end
    }

    include /etc/nginx/conf.d/*.conf;
}
```

Inside a `server` in one of the `/etc/nginx/conf.d/*.conf` includes, use the rate limit in a location or location blocks:

```
server {
    ....

    location /login {

        access_by_lua_block {
            if login:is_rate_limited(ngx.var.remote_addr) then
                return ngx.exit(429)
            end
        }

        #
        # return 200 "ok"; will not work, return in nginx does not run any of the access phases.  It just returns
        #
        content_by_lua_block {
             ngx.say('Hello,world!')
        }
    }
}
```

## Access Block


Inside the http block, set up thethe shared dict
```
http {
    ...
    lua_shared_dict ratelimit_circuit_breaker 10m;

    ...

    include /etc/nginx/conf.d/*.conf;

}
```


Inside a `server` in one of the `/etc/nginx/conf.d/*.conf` includes:
```
    location /login {
        access_by_lua_block {

            local ratelimit = require "resty.greencheek.redis.ratelimiter.limiter"
            local red = { host = "127.0.0.1", port = 6379, timeout = 100}
            local lim, err = ratelimit.new("login", "100r/s", red)

            if not lim then
                ngx.log(ngx.ERR,
                        "failed to instantiate a resty.greencheek.redis.ratelimiter.limiter object: ", err)
                return ngx.exit(500)
            end

            local is_rate_limited = lim:is_rate_limited(ngx.var.remote_addr)

            if is_rate_limited then
                return ngx.exit(429)
            end

        }

        content_by_lua_block {
             ngx.say('Hello,world!')
        }
    }
```

----

## API Specification

To use the rate `limiter` there's 3 steps:

- Import the module (`require`)
- Create a rate limiting object, by a zone
- Use the object to ratelimit based on a request parameter (remote addess, server name, etc)

### Require the library

To use any ratelimiter, you need the [resty redis library](https://github.com/openresty/lua-resty-redis).

```
local ratelimit = require "resty.greencheek.redis.ratelimiter.limiter"
```

### Create a rate limiting object

**syntax:**  local lim, err =  ratelimit.new(zone,ratelimit)

**example:** local lim, err =  ratelimit.new("login","1r/m")

Error will be returned if there's an issue constructing the object (memory issues, etc).  The library does not attempt to validate the connection to redis at construction time.  Redis is only contacted during the `is_rate_limited` call.

The rate limiting object is the object through which you set up the connection to the redis backend.
Each created object can have a different redis backend.  This way you can scale the rate limiting, having different sized redis servers for different
requirements.

To create a rate limiting `zone` you call `ratelimit.new(zone,ratelimit,[configuration])`.  The `zone` is a string, and the `ratelimit` is a string of the form: `"<num>(r/s|r/m)"`

- r/s (requests per second)
- r/m (requests per minute)


```
local login, err = ratelimit.new("login", "100r/s")
local rec, err = ratelimit.new("recommendations", "100r/m")
```

By default, the created object will connect to redis on `localhost` on port `6379`.   To config the redis settings you specify a configuration table:

```
{ host = <string|127.0.0.1>, port = <int|6379>, timeout = <millis|100>, connection_pool_size = <conns|100> , idle_keepalive_ms = <millis|10000>,  circuit_breaker_dict_name = <string|ratelimit_circuit_breaker>, expire_after_seconds = <int|window_size_in_seconds * 5>, background = <bool|true>  }
```

- host : the host name of the redis to connect to
- port : the port of the redis
- connection_pool_size : the size of the redis connection pool
- idle_keepalive_ms : the max time a connection in the redis pool can be keep open without activity
- circuit_breaker_dict_name : the shared dict circuit break that performs the rate limiting
- expire_after_seconds : the expiry time of the key in redis.  Do not change this.
- background: If the calls to redis occur on the request or not.  By default the redis operations are not performed on the request thread

The connection_pool_size should be configure in accordance with your nginx configuration

```
Basically if your NGINX handle n concurrent requests and your NGINX has m workers, then the connection pool size should be configured as n/m. For example, if your NGINX usually handles 1000 concurrent requests and you have 10 NGINX workers, then the connection pool size should be 100.
```

### Check if we should rate limit

**syntax:**  <object>.is_rate_limited(rate_limiter_key)

**example:** local rate_limited = login.is_rate_limited(ngx.var.server_name)

Rate limiting is done by calling the `is_rate_limited` function on the ratelimit object, within a `access_by_lua_block`

```
access_by_lua_block {
    if login:is_rate_limited(<value you are rate limiting on>) then
        return ngx.exit(429)
    end
}

proxy_pass xxxx
```

----

# Redis Failure

If redis is not working, then all requests will be allowed through (no rate limiting at all).  In the `error.log` you will see:

```
2018/05/06 10:54:46 [error] 16#16: *2 connect() failed (111: Connection refused), context: ngx.timer, client: 127.0.0.1, server: 0.0.0.0:9090
2018/05/06 10:54:46 [error] 16#16: *2 [lua] limiter.lua:229: |{"level" : "ERROR", "msg" : "failed_connecting_to_redis", "incremented_counter":"false", "key" : "login:127.0.0.1", "host":"127.0.0.1", "port":"6379" }|failed to create redis - connection refused, context: ngx.timer, client: 127.0.0.1, server: 0.0.0.0:9090
```

----

# Local Testing

There's a `Dockerfile` that can be used to build a local docker image for testing.  Build the image:

```
docker build --no-cache .
```

And then run from the root of the repo:


```
docker run --name ratelimiter --rm -it \
-v $(pwd)/conf.d:/etc/nginx/conf.d \
-v $(pwd)/nginx.conf:/usr/local/openresty/nginx/conf/nginx.conf \
-v $(pwd):/data \
-v $(pwd)/lib/resty/greencheek/redis/ratelimiter/limiter.lua:/usr/local/openresty/lualib/resty/greencheek/redis/ratelimiter/limiter.lua \
ratelimiter:latest /bin/bash
```

when in the contain run the `/data/init.sh` to start openresty and a local redis.  OpenResty will be running on port `9090`.
Gil Tene's fork of [wrk](https://github.com/giltene/wrk2) is also complied during the build of the docker image.

```
/data/init.sh
curl localhost:9090/login
wrk -t1 -c1 -d30s -R2 http://localhost:9090/login
```

There is a `nginx.config` and a `conf.d/default.config` example in the project for you to work with.  By default there are 3 locations:

/t
/login
/login_foreground

----

# See Also

* Rate Limiting with NGINX: https://www.nginx.com/blog/rate-limiting-nginx/
* the ngx_lua module: https://github.com/openresty/lua-nginx-module
* OpenResty: https://openresty.org/

[Back to TOC](#table-of-contents)
