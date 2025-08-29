# Deploying to Digital Ocean with Automated Deployments

This guide provides step-by-step instructions for deploying this static website to a Digital Ocean Droplet and setting up a GitHub webhook for automatic deployments whenever new commits are pushed to the repository.

**Hostname:** `zapp.sytes.net`

## Table of Contents
1.  [Part 1: Manual Deployment to Digital Ocean](#part-1-manual-deployment-to-digital-ocean)
2.  [Part 2: Automated Deployment with GitHub Webhooks](#part-2-automated-deployment-with-github-webhooks)

---

## Part 1: Manual Deployment to Digital Ocean

This section guides you through setting up a server on Digital Ocean and deploying your site manually.

### 1. Prerequisites

*   A Digital Ocean account.
*   Your domain `zapp.sytes.net` ready to be pointed to a new IP address.
*   An SSH key added to your Digital Ocean account for secure login.

### 2. Create a Digital Ocean Droplet

1.  Log in to your Digital Ocean account.
2.  Click **Create > Droplets**.
3.  Choose an image: **Ubuntu 22.04 (LTS) x64** is a good choice.
4.  Choose a plan: The basic, shared CPU plan is sufficient for a simple static site.
5.  Choose a datacenter region closest to your users.
6.  Authentication: Select **SSH Keys** and choose the key you've added to your account.
7.  Choose a hostname for your Droplet (e.g., `zapp-server`).
8.  Click **Create Droplet**.

Once the Droplet is created, copy its public IP address.

### 3. Connect to Your Droplet

Open your terminal and connect to the Droplet as the `root` user using its IP address:

```bash
ssh root@YOUR_DROPLET_IP
```

### 4. Install and Configure Apache

1.  **Update your package list and install Apache:**

    ```bash
    apt update
    apt install apache2 -y
    ```

2.  **Create a directory for your website:**

    ```bash
    mkdir -p /var/www/zapp.sytes.net
    ```

3.  **Clone your repository into this new directory:**
    *(Replace `YOUR_GITHUB_USERNAME/YOUR_REPOSITORY` with your actual GitHub repository path)*

    ```bash
    git clone https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git /var/www/zapp.sytes.net
    ```

4.  **Create an Apache Virtual Host configuration file:**

    ```bash
    nano /etc/apache2/sites-available/zapp.sytes.net.conf
    ```

    Paste the following configuration into the file. This tells Apache where to find your website files and how to serve them.

    ```apache
    <VirtualHost *:80>
        ServerName zapp.sytes.net
        DocumentRoot /var/www/zapp.sytes.net

        <Directory /var/www/zapp.sytes.net>
            AllowOverride All
            Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```

5.  **Enable the new site and disable the default site:**

    ```bash
    a2ensite zapp.sytes.net.conf
    a2dissite 000-default.conf
    ```

6.  **Test the Apache configuration and restart Apache:**

    ```bash
    apache2ctl configtest
    systemctl restart apache2
    ```

### 5. Configure the Firewall

Allow HTTP and HTTPS traffic through the firewall:

```bash
ufw allow 'Apache Full'
```

### 6. Configure Your DNS

1.  Go to your DNS provider for `sytes.net` (this might be No-IP, Dynu, or another dynamic DNS provider).
2.  Update the A record for `zapp.sytes.net` to point to your Droplet's public IP address.
3.  Wait for the DNS changes to propagate. You can check the status using a tool like `dnschecker.org`.

Once propagated, you should be able to visit `http://zapp.sytes.net` in your browser and see your website.

---

## Part 2: Automated Deployment with GitHub Webhooks

This section explains how to set up a webhook that listens for `push` events from your GitHub repository and automatically pulls the latest changes onto your server.

### 1. Create a Deployment Script

On your Droplet, create a shell script that will pull the latest changes from your repository.

1.  **Create the script file:**

    ```bash
    nano /root/deploy.sh
    ```

2.  **Add the following content:** This script navigates to your website directory, pulls the latest changes from the `main` branch, and logs the date of the update.

    ```bash
    #!/bin/bash
    cd /var/www/zapp.sytes.net
    git pull origin main
    echo "Last deployment: $(date)" >> /root/deployment.log
    ```

3.  **Make the script executable:**

    ```bash
    chmod +x /root/deploy.sh
    ```

### 2. Create a Webhook Listener

We'll use a simple Node.js server to listen for webhook notifications from GitHub.

1.  **Install Node.js and npm:**

    ```bash
    apt install nodejs npm -y
    ```

2.  **Create a new directory for your listener and navigate into it:**

    ```bash
    mkdir /root/webhook-listener
    cd /root/webhook-listener
    ```

3.  **Initialize a new Node.js project and install `express`:**

    ```bash
    npm init -y
    npm install express
    ```

4.  **Create the listener file:**

    ```bash
    nano index.js
    ```

5.  **Add the following Node.js code:** This server listens on port `8080`. When it receives a POST request on the `/webhook` path, it runs the `deploy.sh` script.

    ```javascript
    const express = require('express');
    const { exec } = require('child_process');
    const app = express();
    const PORT = 8080;

    app.use(express.json());

    app.post('/webhook', (req, res) => {
        // We're looking for a push to the 'main' branch
        if (req.body.ref === 'refs/heads/main') {
            console.log('Webhook received. Running deployment script...');
            exec('/root/deploy.sh', (error, stdout, stderr) => {
                if (error) {
                    console.error(`exec error: ${error}`);
                    return res.status(500).send('Deployment script failed.');
                }
                console.log(`stdout: ${stdout}`);
                console.error(`stderr: ${stderr}`);
                res.status(200).send('Deployment successful.');
            });
        } else {
            res.status(200).send('Ignoring webhook event.');
        }
    });

    app.listen(PORT, () => console.log(`Server listening on port ${PORT}`));
    ```

### 3. Set Up the GitHub Webhook

1.  In your GitHub repository, go to **Settings > Webhooks**.
2.  Click **Add webhook**.
3.  **Payload URL:** Enter `http://zapp.sytes.net:8080/webhook`.
4.  **Content type:** Select `application/json`.
5.  **Secret:** For better security, you should generate and add a secret token. The Node.js script can be updated to validate this secret. For this basic setup, we'll leave it blank.
6.  **Which events would you like to trigger this webhook?** Select **Just the `push` event.**
7.  Click **Add webhook**.

### 4. Run the Listener as a Service

To ensure the webhook listener runs continuously, we'll set it up as a `systemd` service.

1.  **Create a service file:**

    ```bash
    nano /etc/systemd/system/webhook.service
    ```

2.  **Add the following service configuration:**

    ```ini
    [Unit]
    Description=Webhook listener for auto-deployment
    After=network.target

    [Service]
    ExecStart=/usr/bin/node /root/webhook-listener/index.js
    Restart=always
    User=root
    Environment=PATH=/usr/bin:/usr/local/bin
    Environment=NODE_ENV=production
    WorkingDirectory=/root/webhook-listener

    [Install]
    WantedBy=multi-user.target
    ```

3.  **Enable and start the service:**

    ```bash
    systemctl enable webhook.service
    systemctl start webhook.service
    ```

4.  **Check the status of the service:**

    ```bash
    systemctl status webhook.service
    ```

Your setup is now complete! When you push a new commit to the `main` branch of your GitHub repository, the webhook will trigger the listener on your server, which will then run the deployment script to pull the latest changes.
