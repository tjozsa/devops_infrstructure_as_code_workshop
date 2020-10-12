# Deploying Nginx with Hiera

The following is an example deployment of Nginx (and PHP-FPM) on Ubuntu 14.04 or later, using Puppet3, Hiera, along with the role and profiles pattern we discussed earlier on. We will be using the popular [jfryman/puppet-nginx](https://github.com/jfryman/puppet-nginx) and [mayflower/puppet-php](https://github.com/mayflower/puppet-php) modules for that.

First step is to configure Hiera by configuring `/etc/puppet/hiera.yaml`:
```yaml
---
:backends:
  - yaml
 
:hierarchy:
  - "nodes/%{::fqdn}"
  - "roles/%{::role}"
  - common
 
:yaml:
  :datadir: /etc/puppet/hiera/

:logging:
  - console
```

Hiera does a lookup of your YAML configuration under the `datadir`. It first searches for a file with the hostname (fqdn), then one with the assigned role, and then a common file for all your servers. Keep in mind that every time you edit this file (shouldn’t be too often) you have to restart your Puppet server (if running inside Passenger, just restart Apache: `sudo service apache2 restart`).

The hostname YAML file (in my case `/etc/puppet/hiera/nodes/nodefqdn.yaml`) is where the specifics of this host are stored:
```yaml
---
roles:
   - roles::www

vdomain: example.com

nginx::config::vhost_purge: true
nginx::config::confd_purge: true

nginx::nginx_vhosts:
  "%{hiera('vdomain')}":
    ensure: present
    rewrite_www_to_non_www: true
    www_root: "/srv/www/%{hiera('vdomain')}/"
    try_files:
      - '$uri'
      - '$uri/'
      - '/index.php$is_args$args'

nginx::nginx_locations:
  'php':
    ensure: present
    vhost: example.com
    location: '~ .php$'
    www_root: "/srv/www/%{hiera('vdomain')}/"
    try_files:
      - '$uri'
      - '/index.php =404'
    location_cfg_append:
      fastcgi_split_path_info: '^(.+\.php)(.*)$'
      fastcgi_pass: 'php'
      fastcgi_index: 'index.php'
      fastcgi_param SCRIPT_FILENAME: "/srv/www/%{hiera('vdomain')}$fastcgi_script_name"
      include: 'fastcgi_params'
      fastcgi_param QUERY_STRING: '$query_string'
      fastcgi_param REQUEST_METHOD: '$request_method'
      fastcgi_param CONTENT_TYPE: '$content_type'
      fastcgi_param CONTENT_LENGTH: '$content_length'
      fastcgi_intercept_errors: 'on'
      fastcgi_ignore_client_abort: 'off'
      fastcgi_connect_timeout: '60'
      fastcgi_send_timeout: '180'
      fastcgi_read_timeout: '180'
      fastcgi_buffer_size: '128k'
      fastcgi_buffers: '4 256k'
      fastcgi_busy_buffers_size: '256k'
      fastcgi_temp_file_write_size: '256k'
    
  'server-status':
    ensure: present
    vhost: "%{hiera('vdomain')}"
    location: /server-status
    stub_status: true
    location_cfg_append:
      access_log: off
      allow: 127.0.0.1
      deny: all

serverdensity_agent::plugin::nginx::nginx_status_url: "http://%{hiera('vdomain')}/server-status"

nginx::nginx_upstreams:
  'php':
    ensure: present
    members:
      - unix:/var/run/php5-fpm.sock

php::fpm: true

php::fpm::settings:
  PHP/short_open_tag: 'On'

php::extensions:
    json: {}
    curl: {}
    mcrypt: {}

php::fpm::pools:
  'www':
    listen: unix:/var/run/php5-fpm.sock
    pm_status_path: /php-status
```

And to finish the Hiera part, we can add some configuration common to all servers, like the [Server Density](https://www.serverdensity.com/) monitoring credentials, in the `/etc/puppet/hiera/common.yaml` file:
```yaml
serverdensity_agent::sd_account: bencer
serverdensity_agent::api_token: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```
With the Hiera side of things complete, we can now configure the main Puppet manifest. This is how `/etc/puppet/manifests/site.pp` should look like:
```ruby
node default {
  hiera_include('roles')
}
```
This instructs Puppet to include the roles classes defined in the role section we added in the YAML file. We therefore don’t need all the node definitions in our `.pp` files code anymore (so things stay clean and awesome).

To make all this possible, we need to create the necessary classes for roles and profiles. The roles are created as an additional module under `/etc/puppet/modules/roles`. Within that folder we need to create another manifest folder with the role definition. In our case, it would be `/etc/puppet/modules/roles/manifests/www.pp`:
```ruby
class roles::www {
  include profiles::nginx
  include profiles::php
  include profiles::serverdensity
  class { 'serverdensity_agent::plugin::nginx': }
}
```
This class includes the different profiles. Here it’s the Nginx profile, the PHP profile and the Server Density agent profile. In addition we call the class that installs the Server Density Nginx plugin.

![Nginx monitoring](https://blog.serverdensity.com/wp-content/uploads/2016/02/nginx_monitoring.png)

This is Server Density monitoring Nginx in a LEMP stack setup. By the way, if you are looking for a hosted monitoring tool that integrates with Puppet, you should sign up for a 2-week trial of Server Density.

Then the profiles module needs to define these 3 previous profiles. What these profile classes do is call the 3rd party modules we are using, getting the configuration specifics from Hiera.

The Nginx profile class is defined on `/etc/puppet/modules/profiles/manifests/nginx.pp`:
```ruby
class profiles::nginx {
  class{ '::nginx': }
}
```
The PHP profile class is defined on `/etc/puppet/modules/profiles/manifests/php.pp`:
```ruby
class profiles::php {
  class{ '::php': }
}
```
And finally the Server Density agent profile class is defined on `/etc/puppet/modules/profiles/manifests/serverdensity.pp`:
```ruby
class profiles::serverdensity {
  class{ '::serverdensity_agent': }
}
```
To finish, we need to install the modules we are using from the Puppet forge:
```shell
sudo puppet module install jfryman-nginx --modulepath /etc/puppet/modules
sudo puppet module install mayflower-php --modulepath /etc/puppet/modules
sudo puppet module install serverdensity-serverdensity_agent --modulepath /etc/puppet/modules
```
And we are done. Wait for the next Puppet run to apply changes or do it now in your target server with:
```shell
sudo puppet agent --test --debug
```

Source: https://blog.serverdensity.com/deploying-nginx-with-puppet/