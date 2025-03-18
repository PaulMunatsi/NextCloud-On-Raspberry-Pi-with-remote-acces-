# Nextcloud and OnlyOffice Installation on Raspberry Pi with Tailscale and HTTPS (Dockerized Nextcloud)

This guide provides a step-by-step process for installing Nextcloud (Dockerized) and OnlyOffice on a Raspberry Pi, secured with Tailscale and HTTPS.

## Prerequisites

* **Raspberry Pi:**
    * Raspberry Pi 4 or 4B+ (recommended)
    * Reliable power supply
    * microSD card (at least 32GB, 64GB or larger recommended)
* **Raspberry Pi OS:**
    * Raspberry Pi OS (64-bit) Lite or Desktop (Lite recommended for server applications)
    * Ensure your OS is up-to-date:
        ```bash
        sudo apt update && sudo apt upgrade -y
        ```
* **Tailscale Account:**
    * Sign up for a free Tailscale account at [tailscale.com](https://tailscale.com).
* **Domain Name (Optional, but Recommended):**
    * A domain name can make accessing Nextcloud easier. You can use a free dynamic DNS service or a paid domain.
* **Basic Linux Command-Line Knowledge:**
    * Familiarity with navigating the terminal and executing commands.

## Step 1: Install Tailscale and Docker

1.  **Install Tailscale on your Raspberry Pi:**
    ```bash
    curl -fsSL [https://tailscale.com/install.sh](https://tailscale.com/install.sh) | sh
    ```
2.  **Authenticate Tailscale:**
    ```bash
    sudo tailscale up
    ```
    * Follow the instructions to link your Raspberry Pi to your Tailscale account.
3.  **Obtain your Tailscale IP:**
    ```bash
    tailscale ip
    ```
    * Note this IP address.
4.  **Enable Subnet routes (If needed):**
    * If you want to access other devices on your local network via tailscale, you must enable subnet routes in the admin panel of tailscale.
5.  **Install Docker:**
    ```bash
    curl -fsSL [https://get.docker.com](https://get.docker.com) -o get-docker.sh
    sudo sh get-docker.sh
    sudo usermod -aG docker $USER
    newgrp docker
    ```

## Step 2: Install and Configure Dockerized Nextcloud

1.  **Create Docker volumes:**
    ```bash
    sudo docker volume create nextcloud_data
    sudo docker volume create nextcloud_db
    ```
2.  **Run the Nextcloud Docker container:**
    ```bash
    sudo docker run -d -p 8080:80 -v nextcloud_data:/var/www/html -v nextcloud_db:/var/lib/mysql --name nextcloud nextcloud:latest
    ```
    * Nextcloud will be accessible on port 8080.
3.  **Initial Nextcloud Configuration:**
    * Open a web browser and navigate to `http://your_tailscale_ip:8080`.
    * Follow the Nextcloud setup wizard.
    * When asked about database configuration, use the following:
        * Database user: `root`
        * Database password: *leave blank*
        * Database name: `nextcloud`
        * Database host: `db`
    * Create an admin account.
4.  **Configure `trusted_domains`:**
    * Enter the nextcloud docker container.
        ```bash
        sudo docker exec -it nextcloud bash
        ```
    * Edit the config.php file.
        ```bash
        nano config/www/config.php
        ```
    * Add your tailscale IP to the trusted domains array.
        ```php
        'trusted_domains' =>
        array (
          0 => 'localhost',
          1 => 'your_tailscale_ip:8080',
        ),
        ```
    * Save and exit the file.
    * Exit the docker container.
        ```bash
        exit
        ```
5.  **Configure reverse proxy with Apache:**
    * Install apache.
        ```bash
        sudo apt install apache2 -y
        ```
    * create a nextcloud.conf file.
        ```bash
        sudo nano /etc/apache2/sites-available/nextcloud.conf
        ```
    * add the following configuration, replace your\_tailscale\_ip.
        ```apacheconf
        <VirtualHost *:80>
            ServerName your_tailscale_ip
            ProxyPreserveHost On
            ProxyPass / http://localhost:8080/
            ProxyPassReverse / http://localhost:8080/
            ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
            CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
        </VirtualHost>
        ```
    * Enable the site and modules.
        ```bash
        sudo a2ensite nextcloud.conf
        sudo a2enmod proxy proxy_http
        sudo systemctl restart apache2
        ```
    * Now nextcloud is accessible via http://your\_tailscale\_ip

## Step 3: Install OnlyOffice

1.  **Pull the OnlyOffice Document Server image:**
    ```bash
    sudo docker pull onlyoffice/documentserver
    ```
2.  **Run the OnlyOffice Document Server container:**
    ```bash
    sudo docker run -i -t -d -p 8081:80 onlyoffice/documentserver
    ```
    * OnlyOffice will be accessible on port 8081.
3.  **Configure Nextcloud to use OnlyOffice:**
    * In your Nextcloud web interface, go to "Apps" and enable the "OnlyOffice" app.
    * Go to Nextcloud "Settings" -> "OnlyOffice".
    * Enter your Raspberry Pi's Tailscale IP and port 8081 for the Document Server address (e.g., `http://your_tailscale_ip:8081`).
    * Save the settings.

## Step 4: Enable HTTPS with Tailscale

1.  **Enable HTTPS in Tailscale:**
    * In your Tailscale admin panel, enable HTTPS for your Raspberry Pi's Tailscale IP.
    * Tailscale will handle the certificate management for you.
2.  **Update Nextcloud configuration:**
    * Edit Nextcloud's `config.php` file inside the docker container.
        ```bash
        sudo docker exec -it nextcloud bash
        nano config/www/config.php
        ```
    * Add or modify the `trusted_domains` and `overwrite.cli.url` entries:
        ```php
        'trusted_domains' => array (
            0 => 'your_tailscale_ip',
        ),
        'overwrite.cli.url' => 'https://your_tailscale_ip',
        'overwriteprotocol' => 'https',
        ```
    * Replace `your_tailscale_ip` with your Raspberry Pi's Tailscale IP.
    * Save and exit the file, then exit the docker container.
3.  **Restart Apache:**
    ```bash
    sudo systemctl restart apache2
    ```
4.  **Access Nextcloud via HTTPS:**
    * Open a web browser and navigate to `https://your_tailscale_ip`.

## Important Considerations

* **Security:** Always use strong passwords and keep your software up to date.
* **Performance:** Raspberry Pi performance can be limited. Consider using an external SSD for Nextcloud data storage for better performance.
* **Backups:** Regularly back up your Nextcloud data and configuration.
* **Firewall:** if using a firewall, ensure that the correct ports are open.
* **Docker updates:** Regularly pull the latest nextcloud and onlyoffice docker images.
