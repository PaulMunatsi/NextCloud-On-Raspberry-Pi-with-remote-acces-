# Nextcloud and OnlyOffice Installation on Raspberry Pi with Tailscale and HTTPS

This guide provides a step-by-step process for installing Nextcloud and OnlyOffice on a Raspberry Pi, secured with Tailscale and HTTPS.

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

## Step 1: Install Tailscale

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

## Step 2: Install and Configure Nextcloud

1.  **Install Apache, MariaDB, PHP, and other dependencies:**
    ```bash
    sudo apt install apache2 mariadb-server php php-gd php-mysql php-curl php-zip php-xml php-mbstring php-intl php-imagick php-redis redis-server unzip -y
    ```
2.  **Secure MariaDB:**
    ```bash
    sudo mysql_secure_installation
    ```
    * Follow the prompts to set a root password and secure your MariaDB installation.
3.  **Create a Nextcloud database and user:**
    ```bash
    sudo mysql -u root -p
    ```
    * Enter your MariaDB root password.
    * Execute the following SQL commands:
        ```sql
        CREATE DATABASE nextcloud;
        CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'your_secure_password';
        GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost';
        FLUSH PRIVILEGES;
        EXIT;
        ```
    * Replace `'your_secure_password'` with a strong password.
4.  **Download and extract Nextcloud:**
    ```bash
    wget [https://download.nextcloud.com/server/releases/latest.zip](https://download.nextcloud.com/server/releases/latest.zip)
    sudo unzip latest.zip -d /var/www/
    sudo mv /var/www/nextcloud /var/www/html/
    sudo chown -R www-data:www-data /var/www/html/
    ```
5.  **Configure Apache:**
    * Create a Nextcloud Apache configuration file:
        ```bash
        sudo nano /etc/apache2/sites-available/nextcloud.conf
        ```
    * Add the following content, replacing `your_tailscale_ip` with your Raspberry Pi's Tailscale IP:
        ```apacheconf
        <VirtualHost *:80>
            ServerName your_tailscale_ip
            DocumentRoot /var/www/html/
            <Directory /var/www/html/>
                Require all granted
                AllowOverride All
                Options FollowSymLinks MultiViews
                <IfModule mod_dav.c>
                    Dav off
                </IfModule>
            </Directory>
            ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
            CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
        </VirtualHost>
        ```
    * Enable the site and required Apache modules:
        ```bash
        sudo a2ensite nextcloud.conf
        sudo a2enmod rewrite headers env dir mime setenvif ssl
        sudo systemctl restart apache2
        ```
6.  **Complete Nextcloud installation:**
    * Open a web browser and navigate to `http://your_tailscale_ip`.
    * Follow the Nextcloud installation wizard, using the database credentials you created earlier.
    * It is recommeneded to setup redis as a memory caching system.

## Step 3: Install OnlyOffice

1.  **Install Docker:**
    ```bash
    curl -fsSL [https://get.docker.com](https://get.docker.com) -o get-docker.sh
    sudo sh get-docker.sh
    sudo usermod -aG docker $USER
    newgrp docker
    ```
2.  **Pull the OnlyOffice Document Server image:**
    ```bash
    sudo docker pull onlyoffice/documentserver
    ```
3.  **Run the OnlyOffice Document Server container:**
    ```bash
    sudo docker run -i -t -d -p 8081:80 onlyoffice/documentserver
    ```
    * OnlyOffice will be accessible on port 8081.
4.  **Configure Nextcloud to use OnlyOffice:**
    * In your Nextcloud web interface, go to "Apps" and enable the "OnlyOffice" app.
    * Go to Nextcloud "Settings" -> "OnlyOffice".
    * Enter your Raspberry Pi's Tailscale IP and port 8081 for the Document Server address (e.g., `http://your_tailscale_ip:8081`).
    * Save the settings.

## Step 4: Enable HTTPS with Tailscale

1.  **Enable HTTPS in Tailscale:**
    * In your Tailscale admin panel, enable HTTPS for your Raspberry Pi's Tailscale IP.
    * Tailscale will handle the certificate management for you.
2.  **Update Nextcloud configuration:**
    * Edit Nextcloud's `config.php` file:
        ```bash
        sudo nano /var/www/html/config/config.php
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
    * Restart Apache and PHP-FPM:
        ```bash
        sudo systemctl restart apache2 php7.4-fpm #adjust php version if needed
        ```
3.  **Access Nextcloud via HTTPS:**
    * Open a web browser and navigate to `https://your_tailscale_ip`.

## Important Considerations

* **Security:** Always use strong passwords and keep your software up to date.
* **Performance:** Raspberry Pi performance can be limited. Consider using an external SSD for Nextcloud data storage for better performance.
* **Backups:** Regularly back up your Nextcloud data and configuration.
* **Firewall:** if using a firewall, ensure that the correct ports are open.
