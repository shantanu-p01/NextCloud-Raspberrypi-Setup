# Setting Up Nextcloud with Docker, Portainer, Cloudflare Tunnel, and Uptime Kuma on a Raspberry Pi (No Port Forwarding Needed)

## Index
1. [Prerequisites](#1-prerequisites)
2. [Setting Up Docker](#2-setting-up-docker)
3. [Installing and Configuring Portainer](#3-installing-and-configuring-portainer)
4. [Creating Nextcloud Data Directories](#4-creating-nextcloud-data-directories)
5. [Setting Up Nextcloud and MariaDB with Docker Compose](#5-setting-up-nextcloud-and-mariadb-with-docker-compose)
6. [Configuring MariaDB](#6-configuring-mariadb)
7. [Setting Up Nextcloud](#7-setting-up-nextcloud)
8. [Configuring Cloudflare Tunnel for HTTPS](#8-configuring-cloudflare-tunnel-for-https)
9. [Update Nextcloud Configuration]()
10. [Fixing Nextcloud Errors](#10-fixing-nextcloud-errors)
11. [Enabling Large File Uploads](#11-enabling-large-file-uploads)
12. [Setting Up Cron Jobs with Uptime Kuma](#12-setting-up-cron-jobs-with-uptime-kuma)

---

### 1. Prerequisites
- Raspberry Pi
- A domain name added to Cloudflare
- Internet connection
- Access to Raspberry Pi via SSH

---

### 2. Setting Up Docker
#### Update and Upgrade System
```bash
sudo apt update && sudo apt upgrade -y
```

#### Install Docker
```bash
curl -sSL https://get.docker.com | sh
```

#### Add User to Docker Group
```bash
sudo usermod -aG docker $USER && logout
```

#### Log Back In
Log back into your Raspberry Pi and verify if the `docker` group is added:

```bash
groups
```

#### Verify Docker Installation
```bash
docker run hello-world
```

---

### 3. Installing and Configuring Portainer
#### Install Portainer
```bash
sudo docker pull portainer/portainer-ce:latest && sudo docker run -d -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
```

---

### 4. Creating Nextcloud Data Directories
#### Create Necessary Directories
```bash
sudo mkdir -p /srv/nextcloud_data/html /srv/nextcloud_data/apps /srv/nextcloud_data/config /srv/nextcloud_data/data /srv/nextcloud_data/themes/your_custom_theme /srv/nextcloud_data/db
```

---

### 5. Setting Up Nextcloud and MariaDB with Docker Compose
#### Access Portainer
Open Portainer in your browser by navigating to `http://[PI_IP_ADDRESS]:9000`. Create an account and select the local Docker environment.

#### Create a New Stack
In Portainer, navigate to "Stacks" and click "Add stack". Name your stack (e.g., `nextcloud`).

#### Docker Compose Configuration
Copy the following Docker Compose file into the editor:

```yaml
version: "2"
services:
  app:
    depends_on:
      - db
    environment:
      - MYSQL_PASSWORD=<Password Here>
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
    image: nextcloud
    links:
      - db
    ports:
      - "8080:80"
    restart: always
    volumes:
      - "/srv/nextcloud_data/html:/var/www/html"
      - "/srv/nextcloud_data/apps:/var/www/html/custom_apps"
      - "/srv/nextcloud_data/config:/var/www/html/config"
      - "/srv/nextcloud_data/data:/var/www/html/data"
  db:
    command: "--transaction-isolation=READ-COMMITTED --binlog-format=ROW"
    environment:
      - MYSQL_ROOT_PASSWORD=<Password Here>
      - MYSQL_PASSWORD=<Password Here>
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    image: mariadb:11.4.2
    restart: always
    ports:
      - "3306:3306"
    volumes:
      - "/srv/nextcloud_data/db:/var/lib/mysql"
```

Change the passwords and usernames for security purposes, then click "Deploy the stack".

---

### 6. Configuring MariaDB
#### Access the MariaDB Container
```bash
docker exec -it [db_container_id] /bin/bash
```

#### Update and Upgrade Packages
```bash
apt update && apt upgrade -y
```

#### Install MySQL Client
```bash
apt install mysql-client -y
```

#### Log into MySQL
```bash
mysql -u root
```
If that didn't work try and enter the password for root user

```bash
mysql -u root -p
```

#### Execute SQL Commands
```sql
CREATE USER 'nextcloud'@'%' IDENTIFIED BY '<Password Here>';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'%';
FLUSH PRIVILEGES;
SELECT User, Host FROM mysql.user;
EXIT;
```

#### Restart MariaDB Container
Restart the MariaDB Container from the Portainer

---

### 7. Setting Up Nextcloud
#### Access Nextcloud Setup Page
Navigate to `http://[PI_IP_ADDRESS]:8080` in your browser. Follow the setup instructions to create an admin account and log in.

---

### 8. Configuring Cloudflare Tunnel for HTTPS
#### Log into Cloudflare
Log in to Cloudflare and select your domain.

#### Clean Up DNS Records
Navigate to DNS settings and clean up non-useful DNS records.

#### Set SSL/TLS to "Full"
Go to the SSL/TLS section and set it to "Full".

#### Create a New Tunnel
Log in to the Zero Trust Dashboard, create a new tunnel, and select Docker as the method. Copy the provided command.

#### Run Cloudflare Command
```bash
docker run cloudflare/cloudflared:latest [command_provided_by_cloudflare]
```

**Go to Portainer** and navigate to `Containers`. Look for a container with an image named `cloudflare/cloudflared:latest`.

**Confirm the container name** (e.g., `upbeat_tesla`) and open it.

**Duplicate and Edit** the container:
  - **Go to Restart Policy** and select `always` if it is currently set to `never`.
  - **Click on Deploy the Container** and then **Restart**.

After deploying and restarting, you may notice that the process on your Raspberry Pi command line has stopped with the message "INF Metrics server stopped." This is expected and indicates that the container is properly set up.

#### Configure Tunnel
In the Cloudflare dashboard, configure the tunnel:

- Subdomain
- Domain
- Service Type: HTTP
- URL: [PI_IP_ADDRESS]:8080

Ensure the following settings are enabled:
- Disable Chunked Encoding
- No Happy Eyeballs

#### Verify the Tunnel
In your browser, navigate to `subdomain.domain`. If you see an error page, it indicates that the tunnel is successfully working.

---

### 9. **Update Nextcloud Configuration**
**Access the Nextcloud container** using the following command:
  ```bash
  docker exec -it <container_id> /bin/bash
  ```
  Replace `<container_id>` with your actual Nextcloud container ID.
Run the following commands inside the container:
  ```bash
  apt update
  apt install nano
  nano /var/www/html/config/config.php
  ```
Edit the `config.php` file as follows:
  Before:
  ```php
  array (
    0 => '[PI_IP_ADDRESS]:8080',
  ),
  ```
  After (using your subdomain.domain, e.g., `cloud.nemesis.in.net`):
  ```php
  array (
    0 => '[PI_IP_ADDRESS]:8080',
    1 => 'subdomain.domain',
  ),
  ```

Also add the following lines after `('installed' => true,)`:
  ```php
  'overwriteprotocol' => 'https',
  'default_phone_region' => 'IN',
  'enable_previews' => true,
  'skeletondirectory' => '',
  ```

Save the file in Nano by pressing `CTRL + O`, then `Enter`, and then `CTRL + X`.

Exit the container:
  ```bash
  exit
  ```

#### Restart Nextcloud Container
Restart the NextCloud Container from the Portainer

After the restart, navigate to `subdomain.domain` in your browser. You should see the Nextcloud login page, indicating a successful setup.

---

### 10. Fixing Nextcloud Errors
**Access the Nextcloud container** using the following command:
  ```bash
  docker exec -it <container_id> /bin/bash
  ```
  Navigate to root directory
  ```bash
    cd / 
  ```
#### Fix "Strict-Transport-Security" Error
Edit Apache configuration files to set the HSTS header:

```bash
nano /etc/apache2/sites-available/000-default.conf
```

Add the following within the `<VirtualHost>` block:

```apache
<IfModule mod_headers.c>
  Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
</IfModule>
```

Do the same for `default-ssl.conf` if it exists:

```bash
nano /etc/apache2/sites-available/default-ssl.conf
```

<details>
<summary><b>Additional Settings</b></summary>

**Fix CalDAV and CardDAV Warnings**

Edit Apache configuration files to set redirects:

```bash
nano /etc/apache2/sites-enabled/000-default.conf
```

Add the following lines:

```apache
Redirect 301 /.well-known/carddav https://cloud.nemesis.in.net/remote.php/dav
Redirect 301 /.well-known/caldav https://cloud.nemesis.in.net/remote.php/dav
Redirect 301 /.well-known/webdav https://cloud.nemesis.in.net/remote.php/dav
Redirect 301 /.well-known/webfinger https://cloud.nemesis.in.net/index.php
Redirect 301 /.well-known/nodeinfo https://cloud.nemesis.in.net/index.php
```
</details>

#### Restart Nextcloud Container
Restart the NextCloud Container from the Portainer

---

### 11. Enabling Large File Uploads
#### Edit .htaccess File
```bash
nano /var/www/html/.htaccess
```

Add the following lines at the top:

```apache
php_value upload_max_filesize 16G
php_value post_max_size 16G
php_value max_input_time 3600
php_value max_execution_time 3600
php_value memory_limit 2048M
```

#### Restart Nextcloud Container
Restart the NextCloud Container from the Portainer

---

### 12. Setting Up Cron Jobs with Uptime Kuma
#### Install Uptime Kuma
Create a new stack in Portainer named `uptime` and paste the following Docker Compose file:

```yaml
version: '3.3'

volumes:
  uptimekuma:

services:
  uptime-kuma:
    image: louislam/uptime-kuma
    container_name: uptime-kuma
    volumes:
      - uptimekuma:/app/data
    ports:
      - 3001:3001
```

Deploy the stack.

#### Access Uptime Kuma
Navigate to `http://[PI_IP_ADDRESS]:3001` and create an admin account.

#### Add a New Monitor
In Uptime Kuma, add a new monitor:

- Monitor Type: HTTP(s)
- Friendly Name: NextCloud
- URL: `https://subdomain.domain/cron.php`
- Heartbeat Interval: 60

Save and monitor the cron job.

We can also setup [Reverse proxy for Uptime-Kuma using Cloudflare Tunnel](https://github.com/louislam/uptime-kuma/wiki/Reverse-Proxy-with-Cloudflare-Tunnel)

---

By following these steps, you've successfully set up Nextcloud on your Raspberry Pi using Docker and Portainer, secured it with a Cloudflare tunnel, and ensured consistent performance monitoring with Uptime Kuma. This setup not only provides you with a robust, self-hosted cloud storage solution but also enhances security and reliability through the use of modern tools and practices. Enjoy your new, secure, and efficient Nextcloud instance!