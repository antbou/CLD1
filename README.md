# CLD1 SHAREHOSTING

Realized by `Anthony` and `Sou`.

## Software used

- VMware Workstation 16 Pro
- Git Bash

## Virtual machine

### Choice of Operating System

- Debian Is Stable and Dependable.
- Debian has a large community.
  - It is easy to find solutions to the problem
- debian is small
  - This means that it doesn't need a lot of resources (RAM, Core, storage space) to run.

### VM specs

| Name                 | Value                                            |
| -------------------- | ------------------------------------------------ |
| Linux distribution   | [Debian 11](https://www.debian.org/CD/http-ftp/) |
| Language             | EN-US                                            |
| Keyboard layout      | fr-CH                                            |
| Open ports           | 22 (ssh), 80 (nginx)                             |
| PHP version          | 7.4                                              |
| MariaDB version      | 10.6                                             |
| Processors           | 1                                                |
| Cores per processors | 1                                                |
| RAM                  | 2048 MB                                          |
| Network Adapter      | NAT                                              |
| Virtual disk type    | SCSI                                             |
| Disk size            | 20 GB                                             |

#### Installation of the operating system

1. When you start the installation, select the option `Install`.
1. Select the language of your country, then the language of the keyboard you want to use.
1. You can change the name. You can leave the domain name blank, unless you need to use a specific domain.
1. Now choose a password for the administrator `root`.
1. Then select the `name` and `password` of the new user.
1. Select `Guided - use an entire disk`, then skip the steps until you get to the question `Write the changes to disks` where you should put `yes`.
1. For package management tool step, you should put `no`.
1. For the mirror section, select your country and follow the steps.
1. When you select `Software Selection`, deselect everything except `Common System Utilities`.
1. When the installation wizard asks if you want to install `GRUB`, select `Yes`. Then choose `/dev/sda`.

## Installation

### Prerequisites

update the system

```
apt update && apt upgrade
```

Install sudo

```
apt-get install sudo
```

### SSH

```
# Install ssh service
apt install ssh

# Create ssh_users group. Only users that belong to this group will be able to connect to this machine using ssh.
sudo addgroup ssh_users

# Do not forget to add your current user to the ssh_users group, otherwise you won't be able to connect through ssh
sudo usermod -aG ssh_users <USERNAME>

# Update sshd_config file
nano /etc/ssh/sshd_config
```

```
AllowGroups ssh_users
```

```
# Restart SSH
systemctl restart sshd
```

You can now use ssh to connect from another machine.

### Nginx

```
# Install nginx
apt install nginx

# Enable the service at startup
systemctl enable nginx
```

### PHP

```
# Install php-fpm (latest version which is 7.4)
apt install php-fpm

# Enable the service at startup
systemctl enable php7.4-fpm
```

### MariaDB

```
# Install MariaDB
apt install mariadb-server

# Enable the service at startup
systemctl enable mariadb
```

### Security

This allows to avoid that PHP FPM corrects the path which are sent to him what can generate the execution of unwanted scripts. (indeed by default if PHP does not find a file it will try to correct the path and one can then make it execute php on other type of file by typing for example img.jpg/fake.php)

```
# Update php.ini
nano /etc/php/7.4/fpm/php.ini
```

```
# Then find the line corresponding to cgi.fix_pathinfo
cgi.fix_pathinfo=0
```

## User isolation

In this section we’ll show you how we proceeded to isolate the users. This will be done by creating different php-fpm pools for each nginx server block (site or virtual host).

Each site has its own PHP-FPM Pool. Scripts are run by the www-data user, the home folder of the user belongs to that same user. The files are therefore in a user’s personal directory and cannot be accessed by other users.

## Create a website for a new client

If you have covered the steps above, you should already have one functional website on your machine. Unless you have specified a custom fqdn for it, you should be able to access it under the IP of the your machine remotely.

Now we’ll create a new site (client1.ch) with its own php-fpm pool and Linux user.

Note that `client1` can be replaced by any name.

### Create user, homedirectory and ssh access

```
# Create user
sudo useradd client1 -m -d /home/client1

# Set password for the user
sudo sudo passwd client1

# Add the user create in the ssh group (ssh_users)
sudo usermod -aG ssh_users client1

# Create the web root directory
mkdir /home/client1/www

touch /home/client1/www/index.php

chown client1:www-data -R /home/client1

chmod 750 -R /home/client1
```

The Unix permissions are 750. Each user has full permissions (7) and the user's associated group has permission to read and execute (5) the directory, but the world has no (0) permission.

### Configuring php-fpm

```
# copy the file ww.conf under the name client1.conf
cp /etc/php/7.4/fpm/pool.d/www.conf /etc/php/7.4/fpm/pool.d/client1.conf

# update client1.conf
nano /etc/php/7.4/fpm/pool.d/client1.conf
```

```
[client1]

; Per pool prefix
; It only applies on the following directives:
; - 'access.log'
; - 'slowlog'
; - 'listen' (unixsocket)
; - 'chroot'
; - 'chdir'
; - 'php_values'
; - 'php_admin_values'
; When not set, the global prefix (or /usr) applies instead.
; Note: This directive can also be relative to the global prefix.
; Default Value: none
;prefix = /path/to/pools/$pool

; Unix user/group of processes
; Note: The user is mandatory. If the group is not set, the default user's group
;       will be used.
user = client1
group = client1

; The address on which to accept FastCGI requests.
; Valid syntaxes are:
;   'ip.add.re.ss:port'    - to listen on a TCP socket to a specific IPv4 address on
;                            a specific port;
;   '[ip:6:addr:ess]:port' - to listen on a TCP socket to a specific IPv6 address on
;                            a specific port;
;   'port'                 - to listen on a TCP socket to all addresses
;                            (IPv6 and IPv4-mapped) on a specific port;
;   '/path/to/unix/socket' - to listen on a unix socket.
; Note: This value is mandatory.
listen = /run/php/php7.4-fpm-client1.sock

; Set listen(2) backlog.
; Default Value: 511 (-1 on FreeBSD and OpenBSD)
;listen.backlog = 511

; Set permissions for unix socket, if one is used. In Linux, read/write
; permissions must be set in order to allow connections from a web server. Many
; BSD-derived systems allow connections regardless of permissions. The owner
; and group can be specified either by name or by their numeric IDs.
; Default Values: user and group are set as the running user
;                 mode is set to 0660
listen.owner = www-data
listen.group = www-data

; Pass environment variables like LD_LIBRARY_PATH. All $VARIABLEs are taken from
; the current environment.
; Default Value: clean env
env[HOSTNAME] = $HOSTNAME
env[PATH] = /usr/local/bin:/usr/bin:/bin
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp
```

In the above configuration note these specific options:

- `[client1]` is the name of the pool. For each pool you have to specify a unique name.
- `user` and `group` stand for the Linux user and the group under which the new pool will be running.
- `listen` should point to a unique location for each pool.
- `listen.owner` and `listen.group` define the ownership of the listener, i.e. the socket of the new php-fpm pool. Nginx must be able to read this socket. That’s why the socket is created with the user and group under which nginx runs - www-data.
- The option `chroot` is not included in the above configuration on purpose. It would allow you to run a pool in a jailed environment, i.e. locked inside a directory. This is great for security because you can lock the pool inside the web root of the site. We found that when we specify the `chroot`, the php scripts cannot access the shell command ( system(), shell_exec() etc.. ).
- `env` set some environment variables such as PATH and HOSTNAME so that program's full path does not have to be specified.

Once you have finished with the above configuration restart php-fpm for the new settings to take effect with the command

```
# restart php7.4
sudo systemctl restart php7.4-fpm
```

### Configuring nginx

Once we have configured the php-fpm pool for our site we’ll configure the server block in nginx.

```
# Create the Virtual Host
nano /etc/nginx/sites-available/client1
```

Copy the following :

```
server {
        listen 80;
        listen [::]:80;

        server_name client1.ch www.client1.ch;
        root /home/client1/www;
        index index.php index.html index.html;

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_pass unix:/var/run/php/php7.4-fpm-client1.sock;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
        }
}
```

The above code shows a common configuration for a server block in nginx.

- Web root is /home/client1/www
- The server name uses the domaine client1.ch
- fastcgi_pass specifies the handler for the php files. For every site you should use a different unix socket such as /var/run/php/php7.4-fpm-client1.sock

To enable the above site you have to create a symlink to it in the directory /etc/nginx/sites-enabled/

```
# Enable the website
sudo ln -s /etc/nginx/sites-available/client1 /etc/nginx/sites-enabled/

# Test the configuration
sudo nginx -t

# Restart nginx
sudo systemctl restart nginx

# Restart php
sudo systemctl restart php7.4-fpm
```

### Database

```
# Connect to mariadb
sudo mariadb

# Add a new database
CREATE DATABASE client1;

# Add a user
CREATE USER 'client1'@localhost IDENTIFIED BY 'password';

# Give the user the rights to his database
GRANT ALL PRIVILEGES ON client1.* TO 'client1'@localhost IDENTIFIED BY 'password';

# Refresh rights
FLUSH PRIVILEGES;
```

## Deploy website (Scp)

It is possible to transfer files to the website with SCP as follows .

```
# from client side
scp -pr ./<FOLDER>/* <USERNAME>@<HOSTNAME>:~/www
```

## Hosts file (client side)

You have to change the host file because there is no DNS.

### For windows

Open the hosts file with a text editor (with administrator rights) which is located here `C:\Windows\System32\drivers\etc`
Add the last line as below

```
ip_machine client1.ch www.client1.ch
```

### For MAC OS

1. Open the hosts file with a text editor located here `/etc/hosts`
1. Add, as in case 1, a line with the ip and domain names

You can now access the site from your browser by connecting to client1.ch

# Sources

- https://wiki.debian.org/fr/SSH
- https://grafikart.fr/tutoriels/nginx-692
- https://ostechnix.com/allow-deny-ssh-access-particular-user-group-linux/
- https://www.digitalocean.com/community/tutorials/how-to-host-multiple-websites-securely-with-nginx-and-php-fpm-on-ubuntu-14-04
- https://www.ionos.fr/digitalguide/serveur/configuration/fichier-host/
