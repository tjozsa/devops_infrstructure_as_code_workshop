# NTP installion example
First, make sure the NTP package is not already present on the server, the following command will return nothing if the telnet is not present on the server:
```
rpm -qa | grep -i ntp		
```
As we can see, the NTP package is already present on the server. Let's remove the existing NTP package:
```
yum remove ntp
```
After removing the package, ensure that ntp.conf file is not existing:
```
ls -lrt /etc/ntp.conf		
```
Verify the ntp service does not exist by running the following command:
```
systemctl status ntp		
```
Create a new .pp file to save the code. From the command line:
```
vi demontp.pp
```	
Change to insert mode by pressing `i` from the keyboard.

Type the following code to create a new file:
```ruby
# Class Definition 
class ntpconfig {
    # Installing NTP Package 
  package {"ntp": 
    ensure=> "present",
    }
    # Configuring NTP configuration file 
  file {"/etc/ntp.conf": 
    ensure=> "present", 
    content=> "server 0.centos.pool.ntp.org iburst\n",
    }
    # Starting NTP services 
  service {"ntpd": 
    ensure=> "running",
    }
}
```
After done with editing : press `esc`

To save the file, press `:wq!`

Next step is to **check** whether the code has any syntax errors. Execute the following command:

puppet parser validate demontp.pp		
Make sure that you switched to the `root` to be able to complete the test without any error, by executing the command :
```
su root
```
Test is the next step in the code creation process. Execute the following command to perform a smoke test:
```
puppet applies demontp.pp --noop
```	
Last step is to **run** the puppet in real mode and verify the output.
```
puppet apply demontp.pp
```	
Puppet didn't perform anything because the demo class was just defined but not declared.

So, until you declare the puppet class, the code will not get applied.

Let's declare the demo class inside the same code using include class name at the end of the code:
```ruby
# Class Definition 
class ntpconfig {
    # Installing NTP Package 
  package {"ntp": 
    ensure=> "present",
    }
    # Configuring NTP configuration file 
  file {"/etc/ntp.conf": 
    ensure=> "present", 
    content=> "server 0.centos.pool.ntp.org iburst\n",
    }
    # Starting NTP services 
  service {"ntpd": 
    ensure=> "running",
    }
}

# Class Declaration 
include ntpconfig
```
Again **check** whether the code has any syntax errors. Execute the following command:

puppet parser validate demontp.pp		
Make sure that you switched to the `root` to be able to complete the test without any error, by executing the command :
```
su root
```
Testing is the next step in the code creation process. Execute the following command to perform a smoke test:
```
puppet apply demontp.pp --noop
```
Last step is to run the puppet in real mode and verify the output.
```
puppet apply demontp.pp
```	
This time the code gets applied because the class was defined and then declared.

Ensure that ntp.conf is now existing:
```
ls -lrt /etc/ntp.conf
```
Verify the ntp service has been started by running the following command:
```
systemctl status ntpd
```

Source: https://www.guru99.com/puppet-tutorial.html#12