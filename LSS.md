# Linux Server Setup
In this walkthrough we will be using an ODROID with Ubuntu Minimal as our server, and using the terminal along with Webmin to install the different free software necessary to run our server.

ODROID [Ubuntu, Fedora] login credentials  
This list may not be 100% accurate depending on your OS image

| UserName | Password |
| -------- | -------- |
| root | odroid |
| odroid | odroid |

Update software and install OpenSSH server
```
apt update
apt upgrade
apt dist-upgrade
apt install openssh-server
reboot
```

You can now use a terminal on another system to connect to the server via SSH  
Use the command `hostname -I` to get your servers IP address
```
ssh [username]@[ip-address]
```

Install speedtest and test server connection speed
```
apt install speedtest-cli
speedtest-cli --secure
```

Install cURL and Webmin
```
apt install curl
curl -o setup-repos.sh https://raw.githubusercontent.com/webmin/webmin/master/setup-repos.sh
sh setup-repos.sh
apt-get install webmin
rm setup-repos.sh
```

You can now access Webmin at `https://ip-address:10000/`  
Default login:
| UserName | Password |
| -------- | -------- |
| root | same as your root user |

You can change the password for a user by going to `System->Change Passwords` in the left panel

To set a desired Hostname, the option can be found by going to `Networking->Network Configuration` then select `Hostname and DNS Client`

You can set a static IP and edit the network by going to `Networking->Network Configuration` then select `Network Interfaces`

If you get locked out of accessing Webmin because of a misconfigured ip address, you can use the following steps to configure your network interface to use DHCP using the `ip` command:
```
ip link set eth0 down
dhclient eth0
ip link set eth0 up
ip addr show eth0
```

In the left Webmin panel, under `Un-used Modules`, you can install the desired modules  
For this tutorial series we are installing the following:
- Apache Webserver
- Dovecot IMAP/POP3 Server
- MySQL Database Server
- Postfix Mail Server
- ProFTPD

After installing your modules you should click on `Refresh Modules` in the left panel, this will move the modules you installed into the `Servers` category in the left panel. Then finally reboot the system. You should do this whenever you install a new module.

# PHP Installation
PHP is available in Ubuntu Linux. Unlike Python, which is installed in the base system, PHP must be added.

To install PHP and the Apache PHP module you can enter the following command at a terminal prompt:
```
apt install php libapache2-mod-php
```

You can run PHP scripts at a terminal prompt. To run PHP scripts at a terminal prompt you should install the php-cli package. To install php-cli you can enter the following command:
```
apt install php-cli
```

You can also execute PHP scripts without installing the Apache PHP module. To accomplish this, you should install the php-cgi package via this command:
```
apt install php-cgi
```

To use MySQL with PHP you should install the php-mysql package, like so:
```
apt install php-mysql
```

Similarly, to use PostgreSQL with PHP you should install the php-pgsql package:
```
apt install php-pgsql
```

Install cURL for PHP, applications like stripe need this:
```
apt-get install php-curl
```

Recommemded to refresh Webmin modules and reboot after PHP Installation.

# Next
- [Apache Setup](Apache-Setup.md)
