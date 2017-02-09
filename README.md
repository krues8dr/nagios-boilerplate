# Nagios Configuration

## Overview

This configuration uses templates to make configuration quick and relatively
easy.

The main stumbling block in creating templates for Nagios is that instead of the
usual object-inheritance that most programmers are familiar with, it instead
reverses the relationships.  As a result, a "parent" template must list the
objects that inherit from it – e.g., `hostgroups` have `hostgroup_members`.

This does make it easier to have multiple inheritance, where a parent `hostgroup`
may monitor a web service, and another parent will monitor a database service,
and then a `host` child can belong to both of these `hostgroup`s.  However, it
is generally a good idea to combine `hostgroups` into logical combinations of
services at a high level, so that `hosts` do not have to have many parents; this
will make maintenance much easier.

### Contacts

New users to contact should be added to the [objects/contacts.cfg](objects/contacts.cfg)
file.  By default, these should be added to the `admins` group, for 24-7
monitoring.  You can create new groups if you need more granular controls.

### Services

There is a default service named `generic-service` which all services extend.
This sets a default of 24-7 monitoring, hourly notifications, 15 minute check
intervals, etc. All of the other services extend this service. This is defined
in [objects/services.cfg](objects/services.cfg).

The services that we can monitor by default include if the host is alive, the
load, the webserver is responding, and so on. These are defined in
[objects/hostgroups.cfg](objects/hostgroups.cfg) since they must come after the
`hostgroups` in the load order.

### Default Host Groups

Hostgroups and their checks are set in [objects/hostgroups.cfg](objects/hostgroups.cfg):

* `server`: only basic checks: ping, load
* `webserver`: `server` + httpd, mysql

## Add a Host to Monitor

To setup a new host, copy the [hosts/hosts.cfg.sample](hosts/hosts.cfg.sample)
file to `hosts/mysite.com.cfg` and enter your custom values. All hosts need a
default `check_command` to determine if the host is `up`, we use `check-host-alive`
which uses `ping` by default.

You'll need to reload the nagios config afterwards. If your installation
includes the standard service manager (upstart, etc), you can run `reload`,
otherwise you'll need to send a HUP to the process:

```
kill -HUP <nagios_pid>
```

You'll need to install the NRPE client on the new host you'll be monitoring.
There are detailed instructions below on how to do this.

We can test that the service is running using a local check on the host server.
Note that the location of the nagios plugins might be different on your distro:

```
/usr/lib/nagios/plugins/check_load -w 15,10,5 -c 30,25,20
```

We can then verify that it's making the SSL handshake from the nagios server:
```
/usr/lib64/nagios/plugins/check_nrpe -H <your new host's address> -c check_load -a '-w 15,10,5 -c 30,20,10'
```

See `Debugging` below if this isn't working.

Note that if you're monitoring specific services that require connection
credentials, such  as MySQL or PostgreSQL databases, you'll need to create a
user for those on the new hosts, and set the correct credentials in the host's file.


### NRPE on Debian/Ubuntu

The NRPE package included in APT doesn't have the `--enable-command-args` flag
set in the build, so we cannot use external parameters when running checks.  To
fix this, we need to recompile.

First, we'll install everything:

```
apt-get install nagios-nrpe-server nagios-plugins
```

Then we'll need to make sure the `main` src distro is enabled.  Open up
`/etc/apt/sources.list` and look for a line like:

`deb-src http://archive.ubuntu.com/ubuntu xenial main restricted`

You might see a vendor's mirror replacement instead, for instance on Digital
Ocean they use their own mirror:

`deb-src http://mirrors.digitalocean.com/ubuntu xenial main restricted`

If this line is commented out, enable it and save the file.  Then update apt to
let it know about the new repo.

```
sudo apt-get update
```

Now install our dev tools.

```
apt-get install libssl-dev dpatch debhelper libwrap0-dev autotools-dev build-essential fakeroot hardening-includes
```

We'll be using `libssl` but it's not in an obvious place, so we'll want to link it.

```
ln -s /usr/lib/x86_64-linux-gnu/libssl.so /usr/lib/libssl.so
```


Download the source for nrpe from APT.

```
apt-get source nagios-nrpe-server
```

You may receive some warnings when downloading (statements prefaced with `W:` at
the beginning of the line).  These are normal and shouldn't cause any problems.

Go into the directory this created (named something like `nagios-nrpe-2.15`).
Edit the `debian/rules` file inside. Look for `override_dh_auto_configure`, and
add `--enable-command-args` to the end.  Don't forget to add the `\` at the end
of the line before this line to tell it that it's a multi-line argument.  It
should look like this when you're done:

```
override_dh_auto_configure:
        ./configure \
                --prefix=/usr \
                --enable-ssl \
                --with-ssl-lib=/usr/lib/$(DEB_HOST_MULTIARCH) \
                --sysconfdir=/etc \
                --localstatedir=/var \
                --libexecdir=/usr/lib/nagios/plugins \
                --libdir=/usr/lib/nagios \
                --enable-command-args
```

Edit the change log for the package with `dch -i` to document the change you
made.

After that, we can build our package – this is very similar to standard software
building on most linux distros.

```
./configure
make all
dpkg-buildpackage
```

The previous step will leave us several packages we can install in the parent
directory.  We'll want to install the server one package, the name might be
slightly different on your system:

```
dpkg -i nagios-nrpe-server_2.15-1ubuntu1_amd64.deb
```

Now that NRPE is installed, we want to make sure it's not overwritten by a later
update, so we need to mark them as such.

```
apt-mark hold nagios-nrpe-server
```

Once that's done, edit `/etc/nagios/nrpe.cfg` to add your Nagios server's
IP address, so we can connect to NRPE from another box.

```
allowed_hosts=127.0.0.1,<your nagios server IP>
```

And enable remote arguments sent from the nagios server (via `check_nrpe`):

```
dont_blame_nrpe=1
```

Then we can start the nrpe service.

```
/etc/init.d/nagios-nrpe-server start
```

### NRPE on RHEL/Fedora/CentOS

Since the NRPE package from the repo has `--enable-command-args` enabled in the
build, we can use the default package.  First we'll want to update the epel RPM.

```
rpm -Uvh http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
```

Install nrpe and all of our plugins to do different types of checks.

```
sudo yum install nrpe nagios-plugins nagios-plugins-all
sudo chkconfig --level 2345 nrpe on
```

Edit `/etc/nagios/nrpe.cfg` to add your Nagios server's IP address, so we
can connect to NRPE from another box.

```
allowed_hosts=127.0.0.1,<your nagios server IP>
```

And enable remote arguments sent from the nagios server (via `check_nrpe`):

```
dont_blame_nrpe=1
```

Then we can start the nrpe service.

```
sudo service nrpe start
```

## Debugging

The normal `sudo service nagios checkconfig` is not very verbose. To check your
configuration run:

```
sudo /usr/sbin/nagios -v /etc/nagios/nagios.cfg
```

If you're having trouble with the NRPE hosts on the monitored servers, you'll
need to check their logs.  In most cases, NRPE problems are logged to the
default logs, either `/var/log/syslog` (Debian) or `/var/log/messages` (RHEL).

If you receive an error like `Warning: Daemon is configured to accept command arguments from clients!`
check that `dont_blame_nrpe` is enabled and that the server was compiled with
the `--enable-command-args` flag, per above.

If you receive an error about not being able to make the SSL handshake, check
that the nagios server is in the `allowed_hosts` list, per above.

If you can't login to the web admin, make sure that the `passwd` file exists in
`/etc/nagios` and is readable by the webserver.  It must have the `nagiosadmin`
user, which we use to login.  You can change the password for this user:

```
htpasswd /etc/nagios/passwd nagiosadmin
```
