---
layout: post
title:  "Basics of Securing SSH"
tags: ["ssh", "security", "basics"]
---

I am not a security expert, and this is not an authoritative list, but it's a good start. For more security measures, I suggest that you peruse [Server Fault](http://serverfault.com/) or read [this CentOS how-to entry](http://wiki.centos.org/HowTos/Network/SecuringSSH).

Note that I'm not describing here how to use public/private keys for authentication. Not only does this allow you to avoid entering a password when logging in, but it allows you to disable password authentication completely. This is because part of me wants to be able to log in from any client, so long as I can remember my password.

## Switch away from port 22

The majority of attacks attempt to log in on port 22, and if this fails, they simply move on to trying port 22 on another IP address. While the new port of `sshd` can be found with a simple [port scan](http://en.wikipedia.org/wiki/Port_scanner), most attacks are not that sophisticated.

Choose a new port less than 1024, also called a *well-known port*. A service listening on such a port is either running as `root` or was started as `root`. This makes it difficult for an attacker without root access to terminate `sshd` and then replace it with a SSH service under the control of the attacker. Consult the [IANA port number assignment table](http://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers) to avoid picking an important well-known port, such as 443 for [HTTPS](http://en.wikipedia.org/wiki/HTTP_Secure).

For this post, I'm going to choose port 789. And to be safe, we're going to change `sshd` to listen both on this port and on port 22 before cutting off access to the latter.

First, we must add a rule to the configuration of [iptables](http://en.wikipedia.org/wiki/Iptables) so that it accepts TCP connections on port 789. On many servers, these rules are stored in the file `/etc/sysconfig/iptables`. If this file does not exist, consult the documentation of your distribution to find or even create this file. Once you've opened it, the easiest way to add the rule is to copy and then modify the existing rule that accepts TCP connections on port 22. Although the definition of this rule can vary between distributions, it will contain both `--dport 22` and `-j ACCEPT`. On one of my servers, this rule is `-A INPUT -m state --state NEW -p tcp --dport 22 -j ACCEPT`. Add a rule below it that is similar but uses the new port. If switching to port 789, the file now contains:

```text
-A INPUT -m state --state NEW -p tcp --dport 22 -j ACCEPT
-A INPUT -m state --state NEW -p tcp --dport 789 -j ACCEPT
```

Next, reload iptables to use this new configuration. On some distributions, you may need to run `/etc/init.d/iptables reload`. On others, you may need to run `service iptables reload`.

Now that your server accepts TCP connections on port 789, we need `sshd` to listen on that port in addition to port 22. Open the file `/etc/sshd/sshd_config` and find the line `Port 22`. Add a line below it that uses the new port. If switching to port 789, the file now contains:

```text
Port 22
Port 789
```

Next, reload `sshd` to use this new configuration. On some distributions, you may need to run `/etc/init.d/sshd reload`. On others, you may need to run `service sshd reload`.

Now test logging into server on the new port. If switching to port 789, we log in like so:

```text
$ ssh -p 789 myserver
```

Once this succeeds, it's time to close port 22 and complete switching away.

First, open `/etc/sysconfig/iptables` again and delete the line containing `--dport 22` and `-j ACCEPT`. Then reload iptables like you did before. Second, open `/etc/sshd/sshd_config` again and delete the line `Port 22`. Then reload `sshd` like you did before.

Once this is done, logging into the server on the new port should still succeed, but logging in through port 22 should fail:

```text
$ ssh myserver
ssh: connect to host myserver port 22: Connection refused
```

## Disable root access

The majority of attacks attempt to log in as `root`. Otherwise, the attacker must find a username with root access before proceeding with the attack. By disabling SSH logins for the `root` user, we eliminate these unsophisticated attacks.

First, ensure that your user has permission to log in via ssh and run programs as `root` using `sudo`. If not, you must add your username to the file `/etc/sudoers`. You don't want to disable logging into the server as `root`, and then later find out that you have no way to run programs as `root`.

Next, open the file `/etc/sshd/sshd_config` again and search for a line containing `PermitRootLogin`. It may or may not be commented out. Regardless, change this line to:

```text
PermitRootLogin no
```

Next, reload `sshd` to use this new configuration, just as you did when switching away from port 22. Attempting to log in as `root` should prompt the user for a password like normal:

```text
$ ssh -p 789 root@myserver
root@myserver's password:
```

Even if you enter the correct password, the server will respond with `Permission denied, please try again.`, just as if you had entered an incorrect password.

## Install Fail2ban

From [its website](http://www.fail2ban.org/):

> "Fail2ban scans log files and bans IPs that show the malicious signs -- too many password failures, seeking for exploits, etc. Generally Fail2Ban then used to update firewall rules to reject the IP addresses for a specified amount of time, although any arbitrary other action could also be configured."

The `jail.conf` file, which may be in the directory `/etc/fail2ban`, declares available *jails*. Each jail combines a *filter*, which specifies how login failures are detected, with one or more *actions*, which specify how to enforce a ban from too many login failures.

Near the top of `jail.conf` you will find default settings for jails:

```text
ignoreip = 127.0.0.1/8
bantime  = 600
findtime = 600
maxretry = 3
```

These fit together like so: If a jail finds a host generating `maxretry` login failures in the last `findtime` seconds, and that host is not in `ignoreip`, then the jail bans that host for `bantime` seconds. Each jail declaration can override any of these settings. By changing these settings, you can make the banning criteria more or less restrictive.

Just below that you will find the declaration of the `ssh-iptables` jail, which uses iptables to block login attempts from a banned host:

```text
[ssh-iptables]

enabled = true
filter  = sshd
logpath = /var/log/sshd.log
action  = iptables[name=SSH, port=789, protocol=tcp]
```

The `logpath` parameter specifies what log file is monitored for login failures. If `/var/log/sshd.log` does not exist, you may need to find or change where `sshd` is logging to, such as `/var/log/secure`.

The `filter` line specifies the regular expressions that find login failures. The value `sshd` corresponds to the file `/etc/fail2ban/filter.d/sshd.conf`, which contains the lines:

```text
failregex = ^%(__prefix_line)s(?:error: PAM: )?Authentication failure for .* from <HOST>\s*$
            ^%(__prefix_line)sFailed (?:password|publickey) for .* from <HOST>(?: port \d*)?(?: ssh\d*)?\s*$
            ...
```

Any line in the file `logpath` matching one of these regular expressions is a login failure.

The `action` line specifies the one or more actions to take when a host should be banned or unbanned. The value `iptables` corresponds to the file `/etc/fail2ban/action.d/iptables.conf`. The `name`, `port`, and `protocol` parameters enclosed in square brackets override the default values defined in that file. It is important that the `port` value equals the new port number that `sshd` now listens on; above, we use 789.

The `sshd-iptables` jail adds its own chain to iptables. When a host fails logging in too many times and should be banned, this jail executes the `actionban` command defined in `iptables.conf`. This adds a `DROP` rule to its chain, thereby rejecting login attempts from the host:

```text
actionban = iptables -I fail2ban-<name> 1 -s <ip> -j DROP
```

After `bantime` seconds elapse, this jail executes the `actionunban` command defined in `iptables.conf`. This removes the rule:

```
actionunban = iptables -D fail2ban-<name> -s <ip> -j DROP
```

Other jail declarations follow `ssh-iptables` in `jail.conf`, each with a filter in `/etc/fail2ban/filter.d/` and actions in `/etc/fail2ban/action.d/`. Any number of jails can be enabled.

For more information on Fail2ban, consult [its manual](http://www.fail2ban.org/wiki/index.php/Manual).

