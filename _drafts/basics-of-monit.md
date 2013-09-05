---
layout: post
title:  "Basics of Monit"
tags: ["monit", "monitoring", "basics"]
---

From [its website](http://mmonit.com/monit/):

> "Monit is a free open source utility for managing and monitoring processes, programs, files, directories and filesystems on a UNIX system. Monit conducts automatic maintenance and repair and can execute meaningful causal actions in error situations."

When run as a daemon, Monit runs continuously, waking up after 
Every cycle Monit wakes up and evaluates each of its *service tests*. As we will see, these are clauses that follow in the configuration file and specify how to test that processes, files, directories, network connections, and system resources work as expected or are within expected tolerances. A failing service test generates an error, also called an *alert*.

The configuration file is typically found at `/etc/monitrc` or `/etc/monit.conf`. You can assert that it contains no errors by running `monit -t`:

```text
$ monit -t 
Control file syntax OK
```

Installing Monit on a Red Hat or CentOS machine adds a script to `/etc/init.d/monit`. Ensure that Monit starts when the system starts by using `chkconfig`:

```text
$ sudo /sbin/chkconfig monit on
$ sudo /sbin/chkconfig
...
monit          	0:off	1:off	2:on	3:on	4:on	5:on	6:off
...
```

I love Monit for the completeness of its feature set. But just as much, I love Monit for the conciseness and the readability of its [configuration language](http://mmonit.com/monit/documentation/monit.html). As this post demonstrates, with very little effort you can understand other configuration files, and borrow from them to write your own.

(On the readability side, especially interesting are the *noise keywords* like `and`, `with`, `has`, `using`, and `program`. Monit ignores these keywords in a configuration file, increasing its resemblance to English and improving its readability. For example, the alert criteria `protocol http timeout 15 seconds` can be written as `protocol http and with timeout 15 seconds`.)

## Preamble

Here's the preamble I write for most of my Monit configuration files:

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

The first line configures Monit to continuously as a daemon with a cycle length of 120 seconds, or two minutes:

```text
set daemon 120
```

If you create a dedicated Gmail account for Monit, the end configures Monit to send you these alerts via email through that account:

```text
set mailserver smtp.gmail.com port 587
    using tlsv1 with timeout 30 seconds
    username "username" password "password"
set alert email@address.com with reminder on 15 cycles
```

On the first line, replace `"username"` with the Gmail username. If you are using [Google Apps](http://www.google.com/intx/en/enterprise/apps/business/) for your domain, use the full e-mail address. Likewise, replace `"password"` with the password for that account. On the second line, replace `email@example.com` with the e-mail address to which Monit should send the alerts. The last part, `on 15 cycles`, means that if an alert starts TODO. Because each cycle is two minutes, this means Monit sends an email for a continuously alerting condition once every 30 minutes instead of once every 2 minutes.

The following two lines start the embedded web server of Monit:

```text
set httpd port 2812 address localhost
    allow localhost
```

Through this web interface, you can start, stop, and restart processes, determine which service tests are currently alerting, and disable and enable monitoring for 

## Service tests

Following the preamble should be one or more service tests. As mentioned earlier, Monit uses these on every cycle to ensure that processes, files, directories, network connections, and system resources work as expected. 

### Processes

The majority of service tests may ensure that processes are running as expected. Monit tests whether a process is running by checking for the existence of the PID file specified by `with pidfile`. This file simply contains the [identifier of the running process](http://en.wikipedia.org/wiki/Process_identifier); for more details consult [this answer on Stack Overflow](http://stackoverflow.com/a/8296204/400717) and [this answer on the Unix & Linux Stack Exchange](http://unix.stackexchange.com/a/12818/19089). If a process is not running, then Monit executes the command specified by `start program`.

The following `check process` test monitors [uWSGI](http://projects.unbit.it/uwsgi/):

```text
check process uwsgi with pidfile /usr/local/var/run/uwsgi/uwsgi.pid
  start program = "/etc/init.d/uwsgi start"
  stop program = "/etc/init.d/uwsgi stop"
```

This service test is very simple: If Monit cannot find the PID file `/usr/local/var/run/uwsgi/uwsgi.pid`, then it assumes that uWSGI is not running and executes `/etc/init.d/uwsgi start`.

The following `check process` test, which monitors [Redis](http://redis.io/), is only slightly more complicated:

```text
check process redis with pidfile /usr/local/var/run/redis/redis.pid
  start program = "/etc/init.d/redis start"
  stop program = "/etc/init.d/redis stop"
  if failed port 6379 with timeout 3 seconds then alert
```

If Monit cannot find the PID file `/usr/local/var/run/redis/redis.pid`, then it assumes that Redis is not running and executes `/etc/init.d/redis start`. If Redis is running, then Monit asserts that it is listening for new connections on port 6379. This generates an alert if Monit cannot establish a connection on this port after 3 seconds.

The following `check process` test monitors [CouchDB](http://couchdb.apache.org/). Like the preceding service test for Redis, :

```text
check process couchdb with pidfile /usr/local/var/run/couchdb/couchdb.pid
  start program = "/etc/init.d/couchdb start"
  stop program = "/etc/init.d/couchdb stop"
  if failed port 5984
      with protocol http request "/some_db"
      with timeout 5 seconds
  then alert
```

If Monit cannot find the PID file `/usr/local/var/run/couchdb/couchdb.pid`, then it assumes that CouchDB is not running and executes `/etc/init.d/couchdb start`. If CouchDB is running, then Monit asserts that an HTTP GET request for URL `/some_db` succeeds on port 80. This generates an alert if Monit does not receive a reply to this request after 5 seconds.

This last `check process` test monitors [Nginx](http://nginx.org/) with multiple actions:

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

This Nginx deployment is configured to delegate requests to the uWSGI process monitored by the service entry `uwsgi`. Nginx cannot run correctly if uWSGI is not running correctly. The last line, `depends on uwsgi`, captures this dependency, and affects how the service entry `uwsgi` is started and monitored:

* If `uwsgi` is stopped, then `nginx` is stopped.
* Likewise, if `uwsgi` is unmonitored, then `nginx` is unmonitored.
* If `uwsgi` is started, it first stops `nginx` if it is running. Then once `uwsgi` is running, `nginx` is started again.
* Likewise, if `uwsgi` is monitored, it first unmonitors `nginx` if it is monitored. Then once `uwsgi` is monitored, `nginx` is monitored again.

The service entry `nginx`, however, can be started and monitored independently.

### Connectivity

The `check host` test monitors the connectivity of another host:

```
check host myhost with address myhostname
  if failed icmp type echo
      count 5 with timeout 5 seconds
      2 times within 3 cycles
  then alert
```

You can replace `myhost` with a name of your choosing. Each cycle, this tests whether the host with address `myhostname` can be reached via ping, or [ICMP](http://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) echo requests. The `count 5 with timeout 5 seconds` specifies that 5 consecutive echo requests will be sent to `myhostname` per cycle. If no response for any of these requests is received within 5 seconds of sending them, then Monit assumes that `myhostname` is down for the cycle. This generates an alert if `myhostname` is down for at least two of three consecutive cycles.

### Space usage

The `check filesystem` test monitors the space usage of a filesystem:

```
check filesystem rootfs with path /dev/xvda1
  if space usage > 85% for 3 cycles then alert
```

You can replace `rootfs` with a name of your choosing. This generates an alert if the filesystem `/dev/xvda1` is more than 85% full for consecutive three cycles, or six minutes.

### Memory or CPU usage

The `check system` test monitors memory or CPU usage:

```
check system myserver
  if memory > 85% 2 times within 3 cycles then alert
  if cpu(user) > 75% for 2 cycles then alert
  if cpu(system) > 50% for 2 cycles then alert
```

You can replace `myserver` with a name of your choosing. This generates an alert when any of the following criteria are met:

* If the memory usage of the system is greater than 85% for at least two of three consecutive cycles.
* If the CPU spends more than 75% of two consecutive cycles in user space.
* If the CPU spends more than 50% of two consecutive cycles in system or kernel space.

For more configuration examples, consult [this wiki page](http://mmonit.com/wiki/Monit/ConfigurationExamples). For details about the configuration language, consult [its manual](http://mmonit.com/monit/documentation/monit.html).

