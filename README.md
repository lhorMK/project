# Dockerized WordPress with Nginx and Let's Encrypt

This project provides a simple setup for deploying WordPress with Nginx as a reverse proxy, using Docker. It also includes automatic SSL certificate generation and renewal via Let's Encrypt using Certbot.

## Project Structure

your_project_name/ 

                   ├── .env # Environment file with configuration variables 

                   ├── .gitignore # Ignores sensitive files like .env 
                   
                   ├── docker-compose.yml # Docker Compose configuration 
                   
                   ├── nginx.conf # Nginx configuration file 
                   
                   ├── webroot/ # Directory for your website files (used by Nginx) 
                   
                   ├── wordpress_data/ # WordPress data directory 
                   
                   ├── mariadb_data/ # MariaDB data directory 
                   
                   └── data/ # Data directory for storing SSL certificates

## Requirements

- Docker
- Docker Compose

Make sure Docker and Docker Compose are installed on your machine.

## Setup

### Step 1: Prepare Nginx and Run Without HTTPS (for certificate generation)

Before we can enable HTTPS on the server, we need to obtain SSL certificates from Let's Encrypt. To do that, we first need to run the Nginx server **without HTTPS** (HTTP only) so Certbot can validate the domain and generate the certificates.

1. Clone this repository:

    ```bash
    git clone https://github.com/lhorMK/project.git
    cd project
    ```

2. Create the `.env` file in the root directory and configure the following variables:

    ```ini
    DOMAIN_NAME=yourdomain.com
    SSL_CERT_PATH=/your/path/to/certificate
    ```

    - **`DOMAIN_NAME`**: The domain for your WordPress site.
    - **`SSL_CERT_PATH`**: Path to where your SSL certificates will be stored (e.g., `/your/path/to/certificate/yourdomain.com`).

    Make sure not to commit the `.env` file to version control as it may contain sensitive information.  
    Make sure to replace `yourdomain.com` with your actual domain name when you update the Nginx configuration.

4. **Modify Nginx Configuration**: 
    To allow the Nginx server to work without HTTPS initially, comment out the `listen 443` block and the `ssl_certificate` settings in the `nginx.conf` file. Your `nginx.conf` should look like this initially:

    ```nginx
    events {
        worker_connections 1024;
    }

    http {
      server {
        listen 80;
        server_name yourdomain.com www.yourdomain.com;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
            try_files $uri =404;
        }

        # Redirect HTTP to HTTPS (this will be re-enabled later)
        return 301 https://$host$request_uri;
      }

      server {
        listen 80;
        server_name yourdomain.com www.yourdomain.com;

        location / {
            proxy_pass http://wordpress:80;  # Proxy to the WordPress service
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        root /var/www/html;
        index index.php index.html index.htm;

        location ~ /\.ht {
            deny all;  # Deny access to .htaccess files
        }
      }
    }
    ```
    Make sure to replace `yourdomain.com` with your actual domain name when you update the Nginx configuration.

5. **Run Docker Compose without HTTPS**:

    Start the Docker containers using the following command:

    ```bash
    docker-compose up -d
    ```

    This will start the Nginx and WordPress containers with HTTP only. Nginx will listen on port 80, and Certbot will be able to request SSL certificates without the server requiring HTTPS first.

6. **Generate SSL Certificates**: 

    To request an SSL certificate from Let's Encrypt, you'll use the Certbot container. Run the following command to generate the certificates:

    ```bash
    docker-compose exec certbot certbot certonly --webroot -w /var/www/certbot -d yourdomain.com -d www.yourdomain.com
    ```

    - `certonly`: This tells Certbot to only generate the certificates without trying to automatically configure Nginx.
    - `--webroot -w /var/www/certbot`: Specifies the webroot directory that Certbot will use for domain validation.
    - `-d yourdomain.com -d www.yourdomain.com`: Your domain and its `www` subdomain.
    
    Certbot will contact Let's Encrypt, verify the domain, and generate SSL certificates. These certificates will be stored in `/your/path/to/certificate/yourdomain.com`.  
    Make sure to replace `yourdomain.com` with your actual domain name when you update the Nginx configuration.

7. **Re-enable HTTPS in Nginx Configuration**:

    After Certbot successfully generates the certificates, you can now modify the `nginx.conf` file to enable HTTPS.

    - Uncomment the `listen 443 ssl` block and specify the correct paths to your SSL certificates, like this:

    ```nginx
    server {
      listen 443 ssl;
      server_name yourdomain.com www.yourdomain.com;

      # Path to Let's Encrypt certificates
      ssl_certificate /your/path/to/certificate/yourdomain.com/fullchain.pem;
      ssl_certificate_key /your/path/to/certificate/yourdomain.com/privkey.pem;

      # Additional security settings
      ssl_protocols TLSv1.2 TLSv1.3;
      ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
      ssl_prefer_server_ciphers on;
      ssl_dhparam /etc/ssl/certs/dhparam.pem;

      location / {
          proxy_pass http://wordpress:80;  # Proxy to the WordPress service
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }

      root /var/www/html;
      index index.php index.html index.htm;

      location ~ /\.ht {
          deny all;  # Deny access to .htaccess files
      }
    }
    ```
    Make sure to replace `yourdomain.com` with your actual domain name when you update the Nginx configuration.

8. **Restart the Containers**:

    Now, restart your containers to apply the changes:

    ```bash
    docker-compose restart
    ```

    This will enable HTTPS on your website.

---

### Step 2: Generate Diffie-Hellman Parameters (Optional, but Recommended for Security)

To enhance the security of your SSL setup, it's recommended to generate **Diffie-Hellman parameters**. These parameters improve the strength of the SSL connection.

1. Generate the DH parameters with the following command:

    ```bash
    openssl dhparam -out ./dhparam.pem 2048
    ```

    This will generate a 2048-bit DH parameters file called `dhparam.pem`.

2. Copy the `dhparam.pem` file to your server or Docker container at `/etc/ssl/certs/dhparam.pem`.

3. The `ssl_dhparam` directive in your `nginx.conf` file should already point to this file:

    ```nginx
    ssl_dhparam /etc/ssl/certs/dhparam.pem;
    ```

    This will ensure stronger encryption for your SSL connections.

---

### Step 3: Automatic SSL Certificate Renewal

Certbot automatically installs a cron job to renew certificates, but you can manually renew the certificates with the following command:

```bash
docker-compose exec certbot certbot renew
```

## Accessing WordPress
WordPress will be available at http://localhost:8080.
Once the SSL certificate is issued by Certbot, it will be available over HTTPS at https://yourdomain.com.
Let's Encrypt SSL Certificates
Certbot will automatically create and renew SSL certificates for your domain. By default, the certificates will be stored in the /your/path/to/certificate directory (or the path specified in your .env file). Certbot checks for certificate renewal every 6 hours, and the certificates will be updated automatically.

## Nginx Configuration
The nginx.conf file is preconfigured to:

Handle HTTP to HTTPS redirects.
Serve your WordPress site over HTTPS using SSL.
Proxy requests to the WordPress container.
You can modify the nginx.conf file to suit your needs, for example, by adding custom server blocks or adjusting SSL settings.

## Database Configuration
MariaDB is used as the database backend for WordPress. The database will be automatically configured using the environment variables provided in the docker-compose.yml file.

MYSQL_ROOT_PASSWORD: The root password for MariaDB.

MYSQL_DATABASE: The name of the WordPress database.

MYSQL_USER: The username for the WordPress database.

MYSQL_PASSWORD: The password for the WordPress database user.

## Restarting Containers
If you need to restart the containers for any reason (e.g., to apply configuration changes or restart services), you can use:
```bash
docker-compose restart
```
## Debugging Docker Containers
If you encounter issues with the containers, you can use the following commands to debug:

Check the status of all containers:
```bash
docker-compose ps
```
View logs for all containers in real-time:
```bash
docker-compose logs -f
```
View logs for a specific container (e.g., Nginx):
```bash
docker-compose logs nginx
```
Check running containers on your system:
```bash
docker ps
```
