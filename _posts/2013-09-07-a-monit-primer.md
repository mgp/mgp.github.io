---
layout: post
title:  "A Monit Primer"
tags: ["monit", "monitoring", "basics"]
---

From [its website](http://mmonit.com/monit/):

> "Monit is a free open source utility for managing and monitoring processes, programs, files, directories and filesystems on a UNIX system. Monit conducts automatic maintenance and repair and can execute meaningful causal actions in error situations."

I love Monit for the completeness of its feature set. But just as much, I love Monit for the conciseness and the readability of its [configuration language](http://mmonit.com/monit/documentation/monit.html). As we will see, with very little effort you can understand other configuration files, and borrow from them to write your own.

## Configuration concepts

At the heart of a Monit configuration file are its service entries and their service tests:

* A *service entry* specifies a resource that Monit should monitor, such as a daemon process, file, directory, remote host, and so on.
* A *service test* is defined inside a service entry and defines an error condition for the resource. This condition should not be satisfied if the resource is performing normally. Examples include a process not accepting connections on a specified port, a file having an unexpected checksum, CPU usage exceeding a given threshold, and so on.

Monit typically runs continuously as a daemon, during which it periodically wakes up and evaluates each service test defined in its configuration file. If the error condition of a service test is satisfied, then Monit generates an error, also called an *alert*. Monit sends these alerts via email to one or more specified addresses specified in the configuration file. The time between Monit waking up to evaluate these tests is called a *cycle* and is also specified in the configuration file.

Additionally, you can toggle *monitoring* for each service entry. When monitoring is disabled for an entry, Monit will not evaluate its service tests and consequently will not generate alerts for it.

Upon startup, Monit looks for this configuration file at `/etc/monitrc` or `/etc/monit.conf`. To assert that Monit can find and parse this file, run `monit -t`:

```text
$ monit -t 
Control file syntax OK
```

Now let's look at the contents of such a file.

## Preamble

Here's a basic preamble for a Monit configuration file:

```text
set daemon 120
set logfile /var/log/monit/monit.log
set eventqueue
    basedir /var/monit
    slots 5000
set httpd port 2812 address localhost
    allow localhost
set mailserver smtp.gmail.com port 587
    using tlsv1 with timeout 30 seconds
    username "username" password "password"
set alert email@example.com with reminder on 15 cycles
```

The first line configures Monit to run continuously as a daemon with a cycle length of 120 seconds, or two minutes:

```text
set daemon 120
```

The following starts the embedded web server of Monit:

```text
set httpd port 2812 address localhost
    allow localhost
```

Through its interface you can start, stop, and restart processes, toggle monitoring for service entries, and determine which ones have failing tests that are generating alerting. Moreover, starting the web server enables use of the Monit client, which uses the web server to communicate with the daemon process. The `port 2812 address localhost` binds the web server to port 2812 on the loopback device, while `allow localhost` restricts the client to localhost. Therefore, to access the web interface from another computer, you must first tunnel in. Consult the manual for more security options, like support for SSL and [HTTP Basic Authentication](http://www.ietf.org/rfc/rfc2617.txt).

If you create a dedicated Gmail account for Monit, the end configures Monit to send alerts via email through that account:

```text
set mailserver smtp.gmail.com port 587
    using tlsv1 with timeout 30 seconds
    username "username" password "password"
set alert email@address.com with reminder on 15 cycles
```

Replace `"username"` with the Gmail username. If you are using [Google Apps](http://www.google.com/intx/en/enterprise/apps/business/) for your domain, use the full email address. Likewise, replace `"password"` with the password for that account. Then replace `email@example.com` with the email address to which Monit should send the alerts. The part `with reminder on 15 cycles` means if a service test is generating an alert on every cycle, then Monit will only send an email on the first cycle and every 15 cycles thereafter. Because each cycle is two minutes, Monit will only send an email once every 30 minutes.

These emails have a very simple format. For example, Monit could deliver the following message if it restarts [Redis](http://redis.io/):

```text
Does not exist Service redis
    Date:   Wed, 21 Aug 2013 14:24:08
    Action: restart
    Host:   frontend08
Your faithful employee,
monit
```

## Service entries and tests

Following the preamble should be one or more service entries, each containing its respective service tests.

Each service entry starts with `check [type] [identifier]`, where:

* `type` specifies the type of resource that Monit should monitor, such as `process` for a process, `host` for the connection to a remote host, and so on.
* `identifier` is a unique identifier for this service entry, used in the web interface and included in any alert messages delivered via email.

An entry restricts the service tests defined inside of it to those that are meaningful for its resource. Each service test has the form `if [body] then [action]`, where:

* `body` specifies an error condition for the resource. Complicated ones can be split across multiple lines.
* `action` specifies what Monit should do if the error condition is satisfied. There are many, but below we will only look at `alert` and `restart`.

Now let's look at some service entry types, and some of the service tests they can define.

### Processes

A service entry starting with `check process` contains service tests to ensure that a daemon process is running as expected. Monit always tests whether such a process is running by checking for the existence of its PID file, which simply contains the [process identifier](http://en.wikipedia.org/wiki/Process_identifier). (For more details about PID files, consult [this answer on Stack Overflow](http://stackoverflow.com/a/8296204/400717) and [this answer on the Unix & Linux Stack Exchange](http://unix.stackexchange.com/a/12818/19089).) The expected location of the PID file is specified in the service entry by `with pidfile`. If a process is not running, then Monit executes the command specified by `start program`.

For example, the following `check process` entry monitors [uWSGI](http://projects.unbit.it/uwsgi/):

```text
check process uwsgi with pidfile /usr/local/var/run/uwsgi/uwsgi.pid
  start program = "/etc/init.d/uwsgi start"
  stop program = "/etc/init.d/uwsgi stop"
```

This service entry is very simple: If Monit cannot find the PID file `/usr/local/var/run/uwsgi/uwsgi.pid`, then it assumes that uWSGI is not running and executes `/etc/init.d/uwsgi start`.

The following `check process` entry monitors Redis and introduces a service test:

```text
check process redis with pidfile /usr/local/var/run/redis/redis.pid
  start program = "/etc/init.d/redis start"
  stop program = "/etc/init.d/redis stop"
  if failed port 6379 with timeout 3 seconds then alert
```

If Monit cannot find the PID file `/usr/local/var/run/redis/redis.pid`, then it assumes that Redis is not running and executes `/etc/init.d/redis start`. If Redis is running, then Monit evaluates the service test on the last line, which attempts to connect to Redis on port 6379. If Monit cannot connect after 3 seconds, then it generates an alert.

The following `check process` entry monitors [CouchDB](http://couchdb.apache.org/) and contains a more complicated service test:

```text
check process couchdb with pidfile /usr/local/var/run/couchdb/couchdb.pid
  start program = "/etc/init.d/couchdb start"
  stop program = "/etc/init.d/couchdb stop"
  if failed port 5984
      with protocol http request "/some_db"
      with timeout 5 seconds
  then alert
```

If Monit cannot find the PID file `/usr/local/var/run/couchdb/couchdb.pid`, then it assumes that CouchDB is not running and executes `/etc/init.d/couchdb start`. If CouchDB is running, then Monit evaluates the service test at the end, which attempts to request from CouchDB the URL `/some_db` over HTTP on port 5984. If Monit does not receive a response after 5 seconds, then it generates an alert.

(To quickly switch to the topic of readability, `and`, `with`, `has`, `using`, and `program` are examples of *noise keywords*. Monit ignores these keywords in a configuration file, increasing its resemblance to English and improving its readability. For example, `protocol http timeout 5 seconds` can be written as `protocol http and with timeout 5 seconds`.)

This last `check process` entry monitors [Nginx](http://nginx.org/) and contains two service tests:

```text
check process nginx with pidfile /var/run/nginx.pid
  start program = "/etc/init.d/nginx start"
  stop program = "/etc/init.d/nginx stop"
  if failed port 443 type tcpssl protocol http
      request "/some/path" hostheader "domain.com"
      with timeout 5 seconds
  then alert
  if failed port 443 type tcpssl protocol http
      request "/some/path" hostheader "domain.com"
      with timeout 10 seconds
      3 times within 4 cycles
  then restart
  depends on uwsgi
```

If Monit cannot find the PID file `/var/run/nginx.pid`, then it assumes that Nginx is not running and executes `/etc/init.d/nginx start`.

But if Nginx is running, then Monit evaluates the two service tests that follow. In the first test, Monit attempts to request from Nginx the URL `/some/path` over HTTPS on port 443. The `Host` header of this request is `domain.com`, which is required by HTTP/1.1 to work with [virtual hosting](http://en.wikipedia.org/wiki/Virtual_hosting). If Monit does not receive a response after 5 seconds, then it generates an alert. In the second test, Monit requests the same URL over HTTPS on the same port and using the same `Host` header. But instead of Monit generating an alert if it does not receive a response after 10 seconds on any cycle, it waits until it fails to receive such a response for at least three of four consecutive cycles. When this happens, Monit generates an alert and restarts Nginx in an attempt to fix the problem. To restart Nginx, Monit executes `stop program` and then `start program`.

This Nginx deployment is configured to delegate requests to the uWSGI process monitored by the service entry `uwsgi`. Nginx cannot run correctly if uWSGI is not running correctly. The last line, `depends on uwsgi`, captures this dependency and affects the entry `uwsgi` in the following ways:

* If `uwsgi` is stopped, then `nginx` is stopped.
* Likewise, if `uwsgi` is unmonitored, then `nginx` is unmonitored.
* If `uwsgi` is started, it first stops `nginx` if it is running. Then once `uwsgi` is running, `nginx` is started again.
* Likewise, if `uwsgi` is monitored, it first unmonitors `nginx` if it is monitored. Then once `uwsgi` is monitored, `nginx` is monitored again.

The service entry `nginx`, however, can be started, stopped, monitored, and unmonitored independently.

### Connectivity

The following `check host` entry monitors the connectivity of another host:

```
check host myhost with address myhostname
  if failed icmp type echo
      count 5 with timeout 5 seconds
      2 times within 3 cycles
  then alert
```

Upon evaluating its service test, Monit attempts to reach the address `myhostname` via ping, or [ICMP](http://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) echo requests. The `count 5 with timeout 5 seconds` specifies that 5 consecutive echo requests will be sent to `myhostname` per cycle. If no response for any of these requests is received within 5 seconds of sending them, then Monit assumes that `myhostname` is down for the cycle. If `myhostname` is assumed down for at least two of three consecutive cycles, then Monit generates an alert.

### Space usage

The following `check filesystem` entry monitors disk usage:

```
check filesystem rootfs with path /dev/xvda1
  if space usage > 85% for 3 cycles then alert
```

Upon evaluating its service test, Monit generates an alert if the filesystem at path `/dev/xvda1` is more than 85% full for three consecutive cycles, or six minutes.

### Memory or CPU usage

The following `check system` entry monitors memory and CPU usage:

```
check system myserver
  if memory > 85% 2 times within 3 cycles then alert
  if cpu(user) > 75% for 2 cycles then alert
  if cpu(system) > 65% for 2 cycles then alert
```

Upon evaluating its three service tests, Monit generates an alert when any of the following error conditions are met:

* If the memory usage of the system is greater than 85% for at least two of three consecutive cycles.
* If the CPU spends more than 75% of two consecutive cycles in user space.
* If the CPU spends more than 50% of two consecutive cycles in system or kernel space.

Monit can also monitor the memory or CPU usage of a daemon process given similar service tests in its `check process` entry. When doing this, `memory` monitors the memory usage of the process itself, and `totalmemory` monitors the total memory usage of the process and its children. Likewise, `cpu` monitors the CPU usage of the process itself, and `totalcpu` monitors the total CPU usage of the process and its children. Both `totalmemory` and `totalcpu` are useful in the case of uWSGI, where each worker process is a fork of a master process:

```text
check process uwsgi with pidfile /usr/local/var/run/uwsgi/uwsgi.pid
  ...
  if totalmemory > 75% for 2 cycles then alert
  if totalcpu > 50% for 2 cycles then alert
```

## Knowing more

I hope I've shared with you practical excerpts from a Monit configuration file. What I've shown here is only a small fraction of what Monit can do; to know its full power, consult [its manual](http://mmonit.com/monit/documentation/monit.html). And for more configuration examples, consult [this wiki page](http://mmonit.com/wiki/Monit/ConfigurationExamples).

