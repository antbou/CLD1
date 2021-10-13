# CLD1

## Vm debian
```
apt-get install sudo
sudo apt install openssh-server
```
```
# Edit the /etc/ssh/sshd_config file

PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no

# Restart the ssh deamon
sudo systemctl restart sshd
```
### NGINX
```
sudo apt update

sudo apt install nginx

# Enable the service at startup
sudo systemctl enable nginx

# Remove the default Nginx website
sudo rm /etc/nginx/sites-enabled/default
```



## Client Windows
```
# Genérer les clefs
ssh-keygen -t rsa -b 4096 -C "Comment"
```
```
# Transferer la clef publique sur le serveur
cat ~/.ssh/id_rsa.pub | ssh abo@192.168.127.130 "cat >> ~/.ssh/authorized_keys"
```

```
server {
        listen 80;
        listen [::]:80;

        server_name client1.ch www.client1.ch;

        root /home/client1/www;
        index index.html index.php;

        location / {
                try_files $uri $uri/ /index.php$is_args$args  =404;
        }

        location ~ \.php$ {
                fastcgi_param SCRIPT_FILENAME $fastcgi_script_name;
                fastcgi_pass unix:/var/run/php/php7.4fpm-client1.sock;
                fastcgi_index index.php;
                include /etc/nginx/fastcgi_params;
        }
}
```