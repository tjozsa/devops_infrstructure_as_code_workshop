# LAMP stack example
## Verify Puppet Server and Agent
First, verify the Puppet service by running the following command on the Puppet master node. We need to make sure that the service is active and running:
```shell
systemctl status puppetserver
```
Next, list all of the signed and unsigned agent requests with the following command:
```shell
/opt/puppetlabs/bin/puppetserver ca list --all
```
You should see the following output:
```
Signed Certificates:
    puppet-client       (SHA256)  58:73:AE:62:04:9E:B8:0F:16:07:83:08:34:4A:00:D2:E6:82:9B:47:2A:00:9F:F4:08:AE:56:A8:E7:1B:6A:31
    puppet-master       (SHA256)  7F:23:98:18:0E:3F:0F:FD:3E:12:FD:43:A6:50:C2:4C:58:0F:C6:EB:B0:5A:2A:74:6F:D8:A0:95:BC:31:EA:47	alt names: ["DNS:puppet-master", "DNS:puppet-master"]	authorization extensions: [pp_cli_auth: true]
```
On the Puppet agent node, run the following command to test the connectivity between both nodes.
```shell
/opt/puppetlabs/bin/puppet agent --test
```
If everything is fine, you should get the following output:
```
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Retrieving locales
Info: Caching catalog for puppet-client
Info: Applying configuration version '1583136740'
Notice: Applied catalog in 0.05 seconds
```

## Create a LAMP Module
First, you will need to create a LAMP module on the Puppet master node. To create a module, you must create a directory whose name matches your module name in Puppet’s `modules` directory, and it must contain a directory called `manifests`, and that directory must contain an `init.pp` file.

To do so, change the directory to the Puppet module directory and create a directory named `lamp` and then a `manifests` directory inside it:
```
cd /etc/puppetlabs/code/modules/
mkdir -p lamp/manifests
```
Next, create a `init.pp` file that will contain a Puppet class that matches the module name. We’re going to use `nano` for this, but you can use any text editor you like.
```
nano lamp/manifests/init.pp
```
Add the following lines:
```ruby
class lamp {
  # execute 'apt-get update'
  exec { 'apt-update':                    # exec resource named 'apt-update'
    command => '/usr/bin/apt-get update'  # command this resource will run
  }

  # install apache2 package
  package { 'apache2':
    require => Exec['apt-update'],        # require 'apt-update' before installing
    ensure => installed,
  }

  # ensure apache2 service is running
  service { 'apache2':
    ensure => running,
  }

  # install mysql-server package
  package { 'mysql-server':
    require => Exec['apt-update'],        # require 'apt-update' before installing
    ensure => installed,
  }

  # ensure mysql service is running
  service { 'mysql':
    ensure => running,
  }

  # install php package
  package { 'php':
    require => Exec['apt-update'],        # require 'apt-update' before installing
    ensure => installed,
  }
  # ensure info.php file exists
  file { '/var/www/html/info.php':
    ensure => file,
    content => '',    # phpinfo code
    require => Package['apache2'],        # require 'apache2' package before creating
  }
}
```
Save and close the file when you are finished.

## Create the Main Manifest File
A manifest is a file that contains client configurations that can be used to install the LAMP stack on the agent node. The main puppet manifest file is located at the `/etc/puppetlabs/code/environments/production/manifests` directory.

You can create a new manifest file named site.pp with the following command:
```shell
nano /etc/puppetlabs/code/environments/production/manifests/site.pp
```
Add the following lines:
```ruby
node default { }

node 'puppet-agent' {
  include lamp
}
```
Save and close the file.

In the above file, a node block allows you to specify the Puppet code that will only apply to certain agent nodes. The default node applies to every agent node that does not have a node block specified.

Here, we have specified the `puppet-agent` node block that will apply only to our `puppet-agent` agent node. We have also added the `include lamp` snippet to get Puppet to use the lamp module on the agent node.

## Test the LAMP Module
Now, you will need to pull the LAMP configuration on the agent node from the master node.

On the agent node, run the following command to pull the LAMP configuration from the master node.
```shell
/opt/puppetlabs/bin/puppet agent --test
```
This will install and set up the LAMP stack on the agent node, as can be seen below:
```
Info: Using configured environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Retrieving locales
Info: Caching catalog for puppet-agent
Info: Applying configuration version '1594188800'
Notice: /Stage[main]/Lamp/Exec[apt-update]/returns: executed successfully
Notice: /Stage[main]/Lamp/Package[apache2]/ensure: created
Notice: /Stage[main]/Lamp/Package[mysql-server]/ensure: created
Notice: /Stage[main]/Lamp/Package[php]/ensure: created
Notice: /Stage[main]/Lamp/File[/var/www/html/info.php]/ensure: defined content as '{md5}d9c0c977ee96604e48b81d795236619a'
Notice: Applied catalog in 73.09 seconds
```
And with that, the LAMP stack has now been installed on your agent node. To check it, open your web browser and type the URL `http://agent-ip-address/info.php`. You should see your PHP page.

Source: https://www.linode.com/docs/applications/configuration-management/puppet/use-puppet-modules-to-create-a-lamp-stack/