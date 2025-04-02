# Setup CSCloud VPS

> [!NOTE]
> This section is for setting up a Node.js web server with Nginx as a reverse proxy, pm2 as a process manager, and SSL with Let's Encrypt. **This is the setup to follow for Linneaus CSCloud servers.**

<br>

- [Setup CSCloud VPS](#setup-cscloud-vps)
  - [Handle your private key](#handle-your-private-key)
    - [Add your key to the ssh-agent](#add-your-key-to-the-ssh-agent)
      - [Windows (Powershell)](#windows-powershell)
      - [MacOS (Bash/Zsh)](#macos-bashzsh)
  - [Login to the server](#login-to-the-server)
    - [Your first time logging in?](#your-first-time-logging-in)
    - [Already logged in once like above?](#already-logged-in-once-like-above)
  - [Update and Upgrade System Packages](#update-and-upgrade-system-packages)
  - [Install Node.js](#install-nodejs)
  - [Install PM2](#install-pm2)
  - [Install Nginx](#install-nginx)
  - [Configure Nginx](#configure-nginx)
  - [Setup SSL with Cerbot](#setup-ssl-with-cerbot)
  - [Update HTTP v1.1 to v2](#update-http-v11-to-v2)
  - [Connect to the Server via VsCode](#connect-to-the-server-via-vscode)
  - [Add and run an application](#add-and-run-an-application)

<br>

## Handle your private key

Download the private key from your "secret" repository on GitLab and add it to your local ssh directory (usually `~/.ssh` on MacOS and `C:\Users\username\.ssh` on Windows). You will need this key to log in to your server.

<br>

### Add your key to the ssh-agent

#### Windows (Powershell)

**Step 1** - Open Powershell as an administrator:

<br>

**Step 2** - Set the ssh-agent service to start automatically and start the service:

```sh
Set-Service ssh-agent -StartupType Automatic
Start-Service ssh-agent
```

<br>

**Step 3** - Verify that the service is running:

```sh
Get-Service ssh-agent
```

<br>

**Step 4** - Add your key to the ssh-agent:

Replace with the exact path to YOUR private key

```sh
ssh-add C:\Users\username\.ssh\your-key-name
```

<br>

#### MacOS (Bash/Zsh)

**Step 1** - Start the ssh-agent:

```sh
eval "$(ssh-agent -s)"
```

<br>

**Step 2** - Make sure the .ssh directory has the correct permissions:

```sh
chmod 700 ~/.ssh
```

<br>

**Step 3** - Create and edit the config file:

Open the config file

```sh
nano ~/.ssh/config
```

Add the following lines to the config file:

```sh
AddKeysToAgent yes
UseKeychain yes
```

<br>

**Step 4** - Add your key to the ssh-agent:

Replace `your-key-name` with the name of your key.

```sh
ssh-add --apple-use-keychain ~/.ssh/your-key-name
```

<br>

**Step 5** - Verify that the key was added:

```sh
ssh-add -l
```

<br>

**Step 6 (Optional)** - Configure host in the config file:

Open the config file like before and add the following lines:

```sh
Host ubuntu-name 
  HostName 12.345.678.901
  User ubuntu
  IdentityFile ~/.ssh/your-key-name
```

<br>

Replace `ubuntu-name` with the name you want to use to connect to the server.

Replace `12.345.678.901` with your server's IP address.

Replace `ubuntu` with your server's username.

Replace `~/.ssh/your-key-name` with the exact path to YOUR private key.

<br>

Easy access to the server in the future with the following command:

```sh
ssh ubuntu-name
```

<br>

## Login to the server

### Your first time logging in?

> [!TIP]
> You will find everything needed to login to your server in your "secret" repository on GitLab. You will need: **IP address**, **username**, and **private key**.

<br>

To make the first login easier reference your private key directly. If you have added a host to your ssh config file you can use the host name instead of the IP address and you can skip the `-i` flag and the path to your private key, as the config file will handle that for you.

<br>

`~/.ssh/your-key-name` - The path to your private key, replace `your-key-name` with the name of your key.

`user` - The name of your user on the server, should be `ubuntu` on Linneaus CSCloud servers.

`server_ip_address` - The IP address of your server.

<br>

```sh
ssh -i ~/.ssh/your-key-name user@server_ip_address
```

<br>

### Already logged in once like above?

Then you should not need to reference the key again as a fingerprint has been added to your known hosts file. Instead you can use the following command in the future:

```sh
ssh username@server_ip_address
```

<br>

> [!TIP]
> If your still having trouble logging in, paste the error codes you get into any AI/LLM model and explain what your trying to do.

<br>

## Update and Upgrade System Packages

> [!NOTE]
> When logged into the server the first step is to update and upgrade the system packages.

```sh
sudo apt update
```

```sh
sudo apt upgrade
```

<br>

## Install Node.js

To install Node.js, you will need to be in your home directory. You can get there by typing:

```sh
cd ~
```

<br>

Before you begin, ensure that curl is installed on your system. If curl is not installed, you can install it using the following command:

```sh
sudo apt-get install -y curl
```

<br>

1. **Download the setup script:**

```sh
curl -fsSL https://deb.nodesource.com/setup_22.x -o nodesource_setup.sh
```

2. **Run the setup script with sudo:**

```sh
sudo -E bash nodesource_setup.sh
```

3. **Install Node.js:**

```sh
sudo apt-get install -y nodejs
```

<br>

**Verify the installation:**

```sh
node -v
npm -v
```

```sh
#Example Output
v22.10.0
10.2.3
```

<br>

## Install PM2

PM2 is a process manager for Node.js applications. With PM2, you can keep applications alive forever, reload them without downtime, and easily manage application logs.

```sh
sudo npm install -g pm2
```

<br>

> [!NOTE]
> Later in this guide, you will use PM2 to start your Node.js application.

<br>

## Install Nginx

Nginx is a high-performance web server that is used to serve static content, reverse proxy, and load balance among other things.

**Step 1** - Install Nginx:

```sh
sudo apt install nginx
```

<br>

**Step 2** - Check the Web Server:

At the end of the installation process, Ubuntu starts Nginx. The web server should already be up and running.

You can check with the `systemctl` command:

```sh
sudo systemctl status nginx
```

<br>

You should see output similar to the following:

```sh
#Output
● nginx.service - A high-performance web server and a reverse proxy server
    Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
    Active: active (running) since Mon 2022-03-14 20:00:00 UTC; 24h ago
      Docs: man:nginx(8)
  Main PID: 2369 (nginx)
    Tasks: 2 (limit: 1153)
    Memory: 3.5M
    CGroup: /system.slice/nginx.service
            ├─2369 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
            └─2380 nginx: worker process
```

<br>

> [!TIP]
> You can access the default Nginx landing page to confirm that the software is running properly by navigating, in the browser, to your server’s IP address or domain name.

<br>

After you have your server’s IP address, enter it into your web browser’s address bar:

You will see the default Nginx landing page:

![Nginx Default Page](https://assets.digitalocean.com/articles/nginx_1604/default_page.png)

<br>

**Commands to Manage the Nginx Process**

Now that Nginx is installed and running, you can interact with it using the `systemctl` command.

<br>

> [!TIP]
> The following commands do not need to be used now but are useful for managing the Nginx process in the future.
>
> For example, if you make configuration changes to Nginx, you will need to restart the Nginx process to apply the changes.

<br>

To stop your web server, you can type:

```sh
sudo systemctl stop nginx
```

<br>

To start the web server when it is stopped, type:

```sh
sudo systemctl start nginx
```

<br>

To stop and then start the service again, type:

```sh
sudo systemctl restart nginx
```

<br>

If you are simply making configuration changes, Nginx can often reload without dropping connections. To do this, this command can be used:

```sh
sudo systemctl reload nginx
```

<br>

By default, Nginx is configured to start automatically when the server boots. If this is not what you want, you can disable this behavior by typing:

```sh
sudo systemctl disable nginx
```

<br>

To re-enable the service to start up at boot, you can type:

```sh
sudo systemctl enable nginx
```

<br>

## Configure Nginx

> [!CAUTION]
> When configuring NGINX for production, you should always turn of information that the server leaks. Do:
>
> Install nano for easier file editing (optional but recommended):
>
> ```sh
> sudo apt-get install nano
> ```
>
> Open the NGINX config file:
>
> ```sh
> sudo nano /etc/nginx/nginx.conf
> ```
>
> Remove the # in front of "server_tokens off;":
>
> ```sh
> server_tokens off;
> ```
>
> Save and exit the file. Restart NGINX:
>
> ```
> sudo systemctl restart nginx.service
> ```

<br>

**Useful paths**

- /etc/nginx/nginx.conf - Location of global config file
- /etc/nginx/sites-available - Directory for config files

<br>

**Step 1** - Create config file:

Go to the `/etc/nginx/sites-available` directory and create a new config file for your server:

Replace `cscloudX-XX.lnu.se` with your domain name

```sh
cd /etc/nginx/sites-available
sudo nano cscloudX-XX.lnu.se
```

<br>

**Step 2** - Create a symbolic link to the config file:

Create a symbolic link in the `/etc/nginx/sites-enabled` directory that points to the config file you just created in the `/etc/nginx/sites-available` directory:

Replace `cscloudX-XX.lnu.se` with your domain name

```sh
sudo ln -s /etc/nginx/sites-available/cscloudX-XX.lnu.se /etc/nginx/sites-enabled/
```

<br>

**Step 3** - Remove the default files:

Remove the default file in the `/etc/nginx/sites-available` directory:

```sh
cd /etc/nginx/sites-available
sudo rm default
```

<br>

Remove the symbolic link to the default file in the `/etc/nginx/sites-enabled` directory:

```sh
cd /etc/nginx/sites-enabled
sudo rm default
```

<br>

**Step 4** - Populate the config file you created in "sites-available":

Go into the config file you just created:

Replace `cscloudX-XX.lnu.se` with your domain name

```sh
sudo nano /etc/nginx/sites-available/cscloudX-XX.lnu.se
```

<br>

Add the following configuration to the file (replace `cscloudX-XX.lnu.se` with your domain name):

```nginx
server {
        server_name cscloudX-XX.lnu.se; # This is the server name, it should match your domain name
        index index.html index.htm index.nginx-debian.html;

        root /var/www/html;

        location / { # This is the location block, it tells Nginx how to handle requests to the server
                try_files $uri $uri/ =404;
        }
}
```

<br>

**Step 5** - Add other routes to config file:

If you have other routes in your application, you can add them to the config file like this:

```nginx
server {
        server_name cscloudX-XX.lnu.se; # This is the server name, it should match your domain name
        index index.html index.htm index.nginx-debian.html;

        root /var/www/html;

        location / { # This is the root location block
                try_files $uri $uri/ =404;
        }

        location /crud-app/ { # This is the location block for the /crud-app route
                proxy_pass http://localhost:3001/; # This will pass requests to the /crud-app route to port 3001
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
}
```

<br>

## Setup SSL with Cerbot

**Step 1** - Install Snapd:

```sh
sudo apt install snapd
```

<br>

**Step 2** - Install Certbot:

```sh
sudo snap install --classic certbot
```

<br>

**Step 3** - Prepare the Certbot command:

```sh
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

<br>

**Step 4** - Get and install the SSL certificate:

```sh
sudo certbot --nginx
```

<br>

> [!NOTE]
> Follow the prompts to install the SSL certificate.
>
> You will have to enter your email address and agree to the terms of service.

<br>

**Step 5** - Test the automatic renewal:

```sh
sudo certbot renew --dry-run
```

<br>
 
**Step 6** - Confirm the SSL certificate is working:

Navigate to your domain in a browser and confirm that the SSL certificate is working. You should see a padlock icon in the address bar and the connection should start with `https://`.

<br>

**This is how the file should look after installing the SSL certificate:**

Display it with the following command:

Replace `cscloudX-XX.lnu.se` with your domain name

```sh
cat /etc/nginx/sites-available/cscloudX-XX.lnu.se
```

<br>

```nginx
# Output
server {
        server_name cscloudX-XX.lnu.se; # This is the server name, it should match your domain name
        index index.html index.htm index.nginx-debian.html;

        root /var/www/html;

        location / { # This is the root location block
                try_files $uri $uri/ =404;
        }

        location /crud-app/ { # This is the location block for the /crud-app route
                proxy_pass http://localhost:3001/; # This will pass requests to the /crud-app route to port 3001
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
    ssl_certificate /etc/letsencrypt/live/cscloudX-XX.lnu.se/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/cscloudX-XX.lnu.se/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = cscloudX-XX.lnu.se) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


        server_name cscloudX-XX.lnu.se;
    listen 80;
    return 404; # managed by Certbot
}
```

<br>

> [!NOTE]
> All parts of the file that are managed by Certbot are marked with a comment. These parts should not be edited manually.
>
> The one exception is when adding **HTTP v2** support, which is covered in the next section.

<br>

## Update HTTP v1.1 to v2

**Step 1** - Open the sites-available config file:

Replace `cscloudX-XX.lnu.se` with your domain name

```sh
sudo nano /etc/nginx/sites-available/cscloudX-XX.lnu.se
```

<br>

**Step 2** - Add the following line to the config file:

> [!NOTE]
> Refer to the previous section for the full config file.

<br>

Add the line `http2` to the `listen 443 ssl;` block:

```nginx
  # Before
  listen 443 ssl; # managed by Certbot

  # After
  listen 443 ssl http2; # managed by Certbot
```

<br>

Example:

![Add http2](</img/Screenshot 2024-11-26 105725.png>)

<br>

**Save and exit the file.**

<br>

**Step 3** - Restart Nginx:

```sh
sudo systemctl restart nginx
```

<br>

## Connect to the Server via VsCode

> [!TIP]
> This section is for connecting to your server via VsCode. This is useful for adding and editing files on the server and running commands without needing to SSH into the server via the terminal.

<br>

**Step 1** - Install the Remote - SSH extension pack in VsCode.

<br>

**Step 2** - Click the "Remote Explorer" icon in the sidebar.

<br>

**Step 3** - Connect to your server:

If you have already added a host to your ssh config file your server should show up in the list of hosts. 

![Hosts](</img/Screenshot 2024-11-26 113017.png>)

<br>

If not, you can add a host by clicking the gear icon in the Remote Explorer sidebar and select your config file.

![Settings icon](</img/Screenshot 2024-11-26 111910.png>)

![Config file selection](</img/Screenshot 2024-11-26 112121.png>)

**Configure the file to connect to your server:**

```
Host cscloudX-XX (the name of your server, can be anything)
HostName 12.345.678.901 (ip address)
User ubuntu (the user to log in as)
IdentityFile C:\Users\username\.ssh\id_rsa (the path to your private key)
```

**Example:**

![Config file](/img/image.png)

<br>

**Step 4** - Click the **Connect to Host** button.

![Connect to server](</img/Screenshot 2024-11-26 113017.png>)

<br>

## Add and run an application

**Step 1** - Connect to the server via VsCode. (Refer to the previous section for instructions.)

<br>

**Step 2** - Navigate to the `/var/www` directory:

```sh
cd /var/www
```

<br>

**Step 3** - Create a new directory for your application:

```sh
sudo mkdir myapp
```

<br>

**Step 4** - Navigate into the new directory:

```sh
cd myapp
```

<br>

**Step 5** - Change the ownership of the directory to your regular user:

> [!IMPORTANT]
> If you do not change the ownership of the directory, you will not be able to add or edit files in the directory.

<br>

Replace `username` with the name of your user, usually `ubuntu` on Linneaus CSCloud servers.

```sh
sudo chown -R username:username /var/www/myapp
```

<br>

**Step 6** - Add your application files to the directory:

Do this by dragging and dropping the files into the directory in VsCode.

<br>

**Step 7** - Install the application dependencies:

```sh
npm install
```

<br>

**Step 8** - Create a ecosystem.config.cjs file in the root of your application:

> [!NOTE]
> This file is used by PM2 to start your application. You can configure the file to start your application with the correct environment variables.

```js
module.exports = {
  apps: [
    {
      name: "crud-app", // Name of your application, can be anything
      script: "./src/app.js", // Path to your main application file
      env: {
        NODE_ENV: "production", // Set the environment to production (should always be set to production when deploying publicly)
        PORT: 3000, // Set the port to 3000 (or whatever port your application uses)
        BASE_URL: "/", // Set the base URL

        // Add any other environment variables here
      },
    },
  ],
};
```

<br>

---

> [!IMPORTANT]
> The `PORT` and `BASE_URL` environment variables should be set to match the values in your Nginx config file.

<br>

Example:

```nginx
        location /crud-app/ { # This is the location block for the /crud-app route
                proxy_pass http://localhost:3001/; # This will pass requests to the /crud-app route to port 3001
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
```

<br>

```js
module.exports = {
  apps: [
    {
      name: "crud-app",
      script: "./src/app.js",
      env: {
        NODE_ENV: "production",
        PORT: 3001, // Set the port to 3001 to match the Nginx config file
        BASE_URL: "/api/", // Set the base URL to match the Nginx config file

        // Add any other environment variables here
      },
    },
  ],
};
```

<br>

**Step 9** - Start your application with PM2:

```sh
pm2 start ecosystem.config.cjs
```

<br>

**Step 10** - Check the status of your application:

```sh
pm2 status
```

<br>
You should see your application listed with a status of "online".

```sh
# Example Output
┌─────┬────────────┬────────┬───┬──────┬──────────┬──────────┬──────────────┐
│ id  │ name       │ mode   │ ↺ │status│ cpu      │ mem      │ user         │
├─────┼────────────┼────────┼───┼──────┼──────────┼──────────┼──────────────┤
│ 0   │ crud-app   │ fork   │ 0 │online│ 0.0%     │ 50.1 MB  │ ubuntu       │
└─────┴────────────┴────────┴───┴──────┴──────────┴──────────┴──────────────┘
```

<br>

**Step 11** - Save the PM2 process:

```sh
pm2 save
```
<br>

**Step 12** - Set PM2 to start on boot:

```sh
pm2 startup
```

<br>

> [!IMPORTANT]
> Follow the instructions that PM2 gives you to set it up. This usually involves running a command with `sudo` that PM2 provides.
> 
> When adding a new application, you will need to run the `pm2 save` command again to save the new application to the PM2 process list and have it start on boot.

<br>

