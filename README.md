# Piwik QueuedTracking Plugin

[![Build Status](https://travis-ci.org/piwik/plugin-QueuedTracking.svg?branch=master)](https://travis-ci.org/piwik/plugin-QueuedTracking)

## Description

This plugin writes all tracking requests into a [Redis](http://redis.io/) instance instead of directly into the database. 
This is useful if you have too many requests per second and your server cannot handle all of them directly (eg too many connections in nginx or MySQL). 
It is also useful if you experience peaks sometimes. Those peaks can be handled much better by using this queue. 
Writing a tracking request into the queue is very fast (a tracking request takes in total a few milliseconds) compared to a regular tracking request (that 
takes multiple hundreds of milliseconds). The queue makes sure to process the tracking requests whenever possible even if it takes a while to process all requests after there was a peak.

*This plugin is currently BETA and there might be issues causing not tracked requests, wrongly tracked requests or duplicated tracked requests.*

Have a look at the FAQ for more information.

## FAQ

__What are the requirements for this plugin?__

* [Redis server 2.8+](http://redis.io/) - [Redis quickstart](http://redis.io/topics/quickstart)
* [phpredis PHP extension](https://github.com/nicolasff/phpredis) - [Install](https://github.com/nicolasff/phpredis#installingconfiguring)
* Transactions are used and must be supported by the SQL database.

__Where can I configure and enable the queue?__

In your Piwik instance go to "Settings => Plugin Settings". There is a config section for this plugin.

__When will a queued tracking request be processed?__

First you should know that multiple tracking requests will be inserted into the database at once using 
[bulk tracking](http://developer.piwik.org/api-reference/tracking-api#bulk-tracking) as soon as a configurable number 
of requests is queued. By default we will check whether enough requests are queued during a regular tracking request 
and start processing them right after sending a response to the browser to make sure a user won't have to wait until 
the queue has finished to process all requests. Have a look at this graph to see how it works: 

![How it works](screenshots/How_it_works.png)

__I do not want to process queued requests within a tracking request, what shall I do?__

Don't worry, if this solution doesn't work out for you for some reason you can disable it and process all queued 
requests using the [Piwik console](http://developer.piwik.org/guides/piwik-on-the-command-line). Just follow these steps:

* Disable the setting "Process during tracking request" in the Piwik UI under "Settings => Plugin Settings"
* Setup a cronjob that executes the command `./console queuedtracking:process` for instance every minute
* That's it

The `queuedtracking:process` command will make sure to process all queued tracking requests whenever possible and the 
command will exit as soon as there are not enough requests queued anymore. That's why you should setup a cronjob to start
the command every minute as it will just start processing again as soon as there are enough requests. Be aware that it won't 
speed up processing queued requests when starting this command multiple times. Only one process will actually replay 
queued requests at a time.

Example crontab entry that starts the processor every minute:

`* * * * * cd /piwik && ./console queuedtracking:process >/dev/null 2>&1`

__Can I keep track of the state of the queue?__

Yes, you can. Just execute the command `./console queuedtracking:monitor`. This will show the current state of the queue.

__How should the redis server be configured?__

Make sure to have enough memory to save all tracking requests in the queue. One tracking request in the queue takes about 2KB, 
20.000 tracking requests take about 50MB. All tracking requests of all websites are stored in the same queue.
There should be only one Redis server to make sure the data will be replayed in the same order as they were recorded. 
If you want to configure Redis HA (High Availability) it should be possible to use Redis Cluser, Redis Sentinel, ...
We currently write into the Redis default database by default but you can configure to use a different one.

__Why do some tests fail on my local Piwik instance?__

Make sure the requirements mentioned above are met and Redis needs to run on 127.0.0.1:6379 with no password for the
integration tests to work. It will use the database "15" and the tests may flush all data it contains. Make sure
it does not contain any important data.

__What if I want to disable the queue?__

You might want to disable the queue at some point but there are still some pending requests in the queue. We recommend to 
change the "Number of requests to process" in plugin settings to "1" and process all requests using the command 
`./console queuedtracking:process` shortly before disabling the queue and directly afterwards.

__How can I access Redis data?__

You can either acccess data on the command line via `redis-cli` or use a Redis monitor like [phpRedisAdmin](https://github.com/ErikDubbelboer/phpRedisAdmin).
In case you are using something like a Redis monitor make sure it is not accessible by everyone.

__The processor won't start processing again as it things another processor is processing the data already, what can I do?__

First make sure there is actually no processor processing any requests. For example by executing the command 
`./console queuedtracking:monitor`. In case you are using the command line to process tracking requests make sure there
is no processer running using the Linux command `ps`. If you are sure there is no process running you can release the lock
by executing the command `redis-cli del trackingProcessorLock`. Afterwards everything should work as normal again.
You should actually never have to do this as a lock automatically expires after a while. It just may take a while depending
on the amount of requests you are importing.

__Are there any known issues?__

* In case you are using bulk tracking the bulk tracking response varies compared to the regular one. We will always return 
 either an image or a 204 HTTP response code in case the parameter `send_image=0` is sent.
* Anything related with Cookies won't work
* By design this plugin can delay the insertion of tracking requests causing real time plugins to not show the actual data since
 under load tracking requests may take a while until they are replayed.

## Changelog

0.1.0 Initial Release

## Support

Please direct any feedback to [hello@piwik.org](mailto:hello@piwik.org)

## TODO

For usage with multiple redis servers we should lock differently: 
http://redis.io/topics/distlock eg using https://github.com/ronnylt/redlock-php 