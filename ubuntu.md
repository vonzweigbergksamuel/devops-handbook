# Ubuntu Server Setup

## Table of Contents

- [Ubuntu Server Setup](#ubuntu-server-setup)
  - [Table of Contents](#table-of-contents)
  - [Initial Server Setup (Not Required for Linneaus CSCloud Servers)](#initial-server-setup-not-required-for-linneaus-cscloud-servers)
    - [Create a New User](#create-a-new-user)
    - [Grant Administrative Privileges](#grant-administrative-privileges)
    - [Set Up Basic Firewall](#set-up-basic-firewall)
    - [Enable External Access for Your Regular User](#enable-external-access-for-your-regular-user)
  - [Install Software](#install-software)
    - [Update and Upgrade System Packages](#update-and-upgrade-system-packages)
  - [Configure Nginx for Next.js](#configure-nginx-for-nextjs)

<br>

## Initial Server Setup (Not Required for Linneaus CSCloud Servers)

### Create a New User

1. Log in to your server as the root user.

```sh
ssh root@server_ip_address
```

<br>

2. Use the `adduser` command to add a new user to your system. Replace `username` with the user you want to create:

```sh
adduser username
```

<br>

Follow the prompts to set a password and information for the new user. It is fine to press `ENTER` through all of the prompts.

<br>

### Grant Administrative Privileges

Use the `usermod` command to add the user to the `sudo` group. Replace `username` with the user you created in the previous step:

```sh
usermod -aG sudo username
```

<br>

### Set Up Basic Firewall

Ubuntu servers can use the `ufw` firewall to make sure only connections to certain services are allowed. We can set up a basic firewall very easily using this application.

First you can check the list of UFW application profiles by typing:

```sh
ufw app list
```

```sh
#Output
Available applications:
  OpenSSH
```

<br>

You can allow OpenSSH (SSH) connections through the firewall. This is important because if you deny SSH access, you will lock yourself out of your server. Enable SSH access by typing:

```sh
ufw allow OpenSSH
```

<br>

Now that you have the firewall configured to allow SSH connections, you can enable it.

```sh
ufw enable
```

<br>

You can check the status of the firewall by typing:

```sh
ufw status
```

```sh
#Output
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
```

<br>

### Enable External Access for Your Regular User

When you initially created your user, you were assigned a unique SSH key. You can copy this key to your new user so that you can log in as them using the same key.

To do this you can use the `rsync` command to copy the `.ssh` directory from the root user's home directory to your new user's home directory. This will copy all of your root user's keys to the new user.

```sh
rsync --archive --chown=username:username ~/.ssh /home/username
```

<br>

Now you should be able to SSH into your server as your new user:

```sh
ssh username@server_ip_address
```

<br>

## Install Software

### Update and Upgrade System Packages

Before you install any software, it's always recommended to update and upgrade system packages. SSH into your server as your regular user and run the following commands:

```sh
sudo apt update
sudo apt upgrade
```

<br>

> [!NOTE]
> If you choose to use a password when creating your regular user you will need to use `sudo` for some commands.

<br>

## Configure Nginx for Next.js

**Step 1 - Open the sites-available config file:**

```sh
sudo nano /etc/nginx/sites-available/your-app-name
```

<br>

**Step 2 - Add the following lines to the config file:**

```nginx
        # Enable Gzip compression for faster loading times
        gzip on;
        gzip_proxied any;
        gzip_types application/javascript application/x-javascript text/css text/javascript;
        gzip_comp_level 5;
        gzip_buffers 16 8k;
        gzip_min_length 256;

        # Serve static files from the .next/static directory
        location /_next/static/ {
                alias /var/www/name_of_app/.next/static/;
                expires 365d;
                access_log off;
        }
```

<br>

**Full config file example:**

```nginx
server {
        server_name domainname.com;

        gzip on;
        gzip_proxied any;
        gzip_types application/javascript application/x-javascript text/css text/javascript;
        gzip_comp_level 5;
        gzip_buffers 16 8k;
        gzip_min_length 256;

        location /_next/static/ {
                alias /var/www/name_of_app/.next/static/;
                expires 365d;
                access_log off;
        }

        location / {
                proxy_pass http://localhost:3000/;
                proxy_http_version 1.1;
                proxy_set_header Upgrade             $http_upgrade;
                proxy_set_header Connection          'upgrade';

                proxy_set_header Host                 $host;
                proxy_set_header X-Real-IP            $remote_addr;
                proxy_set_header X-Forwarded-For      $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto    $scheme;
                proxy_set_header X-Forwarded-Host     $host;
                proxy_set_header X-Forwarded-Port     $server_port;
        }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/domainname.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/domainname.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = domainname.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


        server_name domainname.com;
    listen 80;
    return 404; # managed by Certbot
}
```

<br>

**Save and exit the file.**

<br>

**Step 3 - Restart Nginx and run your application.**

```sh
sudo systemctl restart nginx
pm2 start ecosystem.config.cjs
```
