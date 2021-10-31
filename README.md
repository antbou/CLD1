# Shared hosting

## To do First
```
apt update && apt upgrade
apt-get install sudo
```
## Installation
### Ssh
```
# Install ssh service
apt install ssh

# Create user group
sudo addgroup ssh_users

# Add user into group 
sudo usermod -aG ssh_users <USERNAME>

# Edit the /etc/ssh/sshd_config file
AllowGroups ssh_users

# Restart the ssh deamon
systemctl restart sshd
```
### Nginx
```
# Install nginx
apt install nginx

# Enable the service at startup
systemctl enable nginx
```

### Php
```
# Install php-fpm
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

## Security
```
nano /etc/php/7.4/fpm/php.ini
Trouvez alors la ligne correspondant à cgi.fix_pathinfo et mettez

cgi.fix_pathinfo=0
Ceci permet d'éviter que PHP FPM corrige les chemin qui lui sont envoyé ce qui peut engendrer l'éxécution de scripts non désirés. (en effet par défaut si PHP ne trouve pas un fichier il essaiera de corriger le chemin et on peut alors lui faire éxécuter du php sur d'autre type de fichier en tapant par exemple img.jpg/fake.php)
```

## Create a website for a client
### Create user, homedirectory and ssh access
```
sudo useradd client1 -m -d /home/client1

sudo sudo passwd client1

sudo usermod -aG ssh_users client1

mkdir /home/client1/www

touch /home/client1/www/index.php

chown client1:www-data -R /home/client1

chmod 750 -R /home/client1
```

### Create user website
```
# cp /etc/php/7.4/fpm/pool.d/www.conf /etc/php/7.4/fpm/pool.d/client1.conf
# nano /etc/php/7.4/fpm/pool.d/client1.conf

[client1]

;chroot = /home/client1/www

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
listen = /var/run/php/php7.4fpm-client1.sock

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

...

; Pass environment variables like LD_LIBRARY_PATH. All $VARIABLEs are taken from
; the current environment.
; Default Value: clean env
env[HOSTNAME] = $HOSTNAME
env[PATH] = /usr/local/bin:/usr/bin:/bin
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp


# sudo systemctl start php7.4-fpm

# nano /etc/nginx/sites-available/client1
server {
        listen 80;
        listen [::]:80;

        server_name client1.ch www.client1.ch;
        root /home/client1/www;
        index index.php index.html index.html;

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_pass unix:/var/run/php/php7.4fpm-client1.sock;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
        }
}


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

CREATE DATABASE client1;

GRANT ALL PRIVILEGES ON client1.* TO 'client1'@localhost IDENTIFIED BY 'password';

FLUSH PRIVILEGES;
```

### SCP
scp -pr ./<FOLDER>/* <USERNAME>@<HOSTNAME>:~/www