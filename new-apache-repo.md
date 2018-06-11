# Create local RPM repo on Apache HTTP Server 2.4

Get packages with ease in closed environments.

## Prerequisites

A lot of directives changed with Apache HTTP Server version 2.4. This guide is to provide a set of instructions to easily deploy HTTP repository. You will need:

1. Red Hat Enterprise Linux/CentOS Full DVD ISO
2. Latest httpd package installed on the repository server
3. Latest createrepo package installed one the repository server
4. At least one open port from target servers to repository server

This example is based on OS packages, however you can use the same way to create repository of any packages type, for example when deploying **Airgap repository**.

## Prepare repository directory
All commands have to be executed as root user/with sudo command.

- `yum -y install createrepo` Install necessary packages
- `mkdir /data/repo /mnt/iso` Create repository directory
- `mount rhel-server-7.3-x86_64-dvd.iso /mnt/iso` Mount OS ISO on /mnt/iso
- `cp /mnt/iso/Packages/* /data/repo/.` Copy all packages to the repository data
- `cd /data/repo && createrepo .` Change direcotry and create repository metadata

## Configure Apache HTTP Server 2.4
Install Apache HTTP Server `yum -y install httpd`. It's default configuration direcotry is /etc/httpd/conf. However, it's possible to add multiple configuration files to /etc/httpd/conf.d to extend Apache's basic functionallity.

Install the server itself:
- `yum -y install httpd`

Create the configuration file in `/etc/httpd/conf.d/` directory:
- `vim repo.conf` with content:

```ssh
Listen PORT

<Directory "/data/repo">
  Options MultiViews Indexes SymLinksIfOwnerMatch IncludesNoExec
  DirectorySlash Off
  Require all granted
  AllowOverride None
</Directory>

<VirtualHost *:PORT>
  DocumentRoot "/data/repo/"
  ServerName YOURIPADDRESS
  ErrorLog "/data/repo/error.log"
  CustomLog "/data/repo/access.log" combined
</VirtualHost>
```

Restart the Apache HTTP Server:
- `service httpd restart`

## Configure repository on the target systems:
Create a file in `/etc/yum.repos.d/` named `httpiso.repo` with content:
```
[YOUROSNAME-ISO]
gpgcheck=1
name=YOUROSNAME - ISO
baseurl=http://YOURIPADDRESS:PORT/
enabled=1
proxy=_none_
```
Update yum metadata:
- `yum clean all && yum makecache fast`

Install or update desired packages from your new repository!
