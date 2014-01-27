### Default Groups

Hostgroups and their checks are set in [objects/hostgroups.cfg](objects/hostgroups.cfg) :

* `server` - only basic checks: ping, load
* `webserver` - `server` + httpd, mysql
* `decoded-web` - `webserver` + solr
* `madison-web` - Currently disabled


### Add a Host to Monitor

To setup a new host, copy the [hosts/hosts.cfg.sample](hosts/hosts.cfg.sample) file to `hosts/mysite.com.cfg` and enter your custom values.


### Debugging

To check your configuration:

    sudo /usr/sbin/nagios -v /etc/nagios/nagios.cfg

We do this because the normal `sudo service nagios checkconfig` is not very verbose.
