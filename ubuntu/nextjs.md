# Next.js on Ubuntu

> [!NOTE]
> This guide will help you configure an Ubuntu server to host a Next.js application.
> It will not walk you through setting up the Ubuntu server from scratch.
> If you need help setting up an Ubuntu server, refer to the [setup](setup.md).

<br>

- [Next.js on Ubuntu](#nextjs-on-ubuntu)
  - [With Nginx and PM2](#with-nginx-and-pm2)
    - [Change the Nginx config](#change-the-nginx-config)
    - [Env Variables and ecosystem.config.cjs](#env-variables-and-ecosystemconfigcjs)

<br>

## With Nginx and PM2

> [!IMPORTANT]
> Refer to the [setup](setup.md) guide to set up the server with Node, Nginx, PM2, and Let's Encrypt.
> That guide also includes instructions on how to set up the nginx config file. This guide will focus on changes specific to Next.js.

<br>

### Change the Nginx config

1. Open the sites-available config file:

```sh
sudo nano /etc/nginx/sites-available/your-app-name
```

<br>

2. Add the following lines to the config file:

Replace `name_of_app` with the name of your application.

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
                alias /var/www/name_of_app/.next/static/; # Replace name_of_app with your app name
                expires 365d;
                access_log off;
        }
```

<br>

**Full config file example:**

> [!CAUTION]
> If you plan to run your next.js app on a subdirectory and have added a `basePath` in your `next.config.js`, you will need to match the location block with the `basePath` value. 
> 
> Normally the location block route would end with a `/` (e.g. `/some_name/`), but because the `basePath` value does not accept a trailing slash, the location block should not have a trailing slash either. 
> 
> Otherwise you will get in a loop of infinite redirect because nginx will keep adding the trailing slash to the URL.

<br>

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

        location /some_name {
                proxy_pass http://localhost:3001/;
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

3. Restart Nginx and run your application.

```sh
sudo systemctl restart nginx
pm2 start ecosystem.config.cjs
```

<br>

### Env Variables and ecosystem.config.cjs

Normally you put your env variables under `env` in your `ecosystem.config.cjs` file but when deploying a Next.js app the env variables are used when the app is built. Because of this, you can use the `.env` file you used during development.

Just update the `ecosystem.config.cjs` file to run `npm start`:

```js
module.exports = {
  apps: [
    {
      name: "name_of_app",
      script: "npm",
      cwd: "/var/www/name_of_app",
      args: "start"
    },
  ],
};
```

<br>

Remember to run `npm run build` before starting the app with the ecosystem file.

```sh
npm run build
pm2 start ecosystem.config.cjs
```

<br>
