
# Blazor Server & Application Configuration Docs

**1.Open Server Manager and Install `WebSocket`:**

* Select `Add Roles and Features` from Manage, From the Add Roles and Features Wizard go to Server Roles.
* Select `Web Server (IIS)` then Select (Web Server) then Select `Application Development`.
* Now, select WebSocket, proceed with the process, and complete the installation. By foll

![image](https://github.com/user-attachments/assets/c9d015e6-142a-4653-8287-6f5801b26368)
![image](https://github.com/user-attachments/assets/af63c614-1db2-4b12-a9ba-f2a4b497809d)

**2. Ensure the following code is added to your Blazor Server:**

* Add SignalR Service on `program.cs` 

```
builder.Services.AddSignalR();
```
* Check Razor Component configuration. 

* Configure `appsetting.json` with correct credentials

* Add `blazor.web.js` is added on the `App.razor`

```
<script src="_framework/blazor.web.js"></script>
```
  

# Blazor CarePro Load Balancer Docs

**1. Overview**

This document provides a step-by-step guide for setting up an NGINX load balancer with Blazor server support for CarePro. The setup includes sticky sessions, WebSocket configuration for Blazor, SSL integration, and access logging.

**2. Prerequisites**

  * Operating System: Ubuntu 20.04 or later
  * Software:
      * NGINX
      * Certbot (for SSL)
  * Permissions: Root or sudo access to the server
  * Domain: carepronginx.arcapps.org

**3. Installation**

**Install NGINX**

```
sudo apt update
sudo apt install nginx -y
```

**Install Certbot and NGINX Plugin**

```
sudo apt install certbot python3-certbot-nginx -y
```

**4. Configuration**

Define the Upstream in `nginx.conf`

Edit the NGINX configuration file:

```
sudo nano /etc/nginx/nginx.conf
```

**Add the following content:**

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    upstream backend_servers {
        # Backend server pool
        server 192.168.10.125:80 max_fails=3 fail_timeout=30s;
    }

    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format    main '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log    /var/log/nginx/access.log  main;
    error_log     /var/log/nginx/error.log;

    sendfile        on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}

```


**Enable Access Logging**

Create a new configuration file for the site:

```
sudo nano /etc/nginx/sites-available/carepronginx.arcapps.org
```

**Add the following:**

```
server {
    listen 80;
    server_name carepronginx.arcapps.org;

    location / {
        proxy_pass http://backend_servers;

        # Standard headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support for Blazor
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Timeout settings
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
        proxy_connect_timeout 600s;

        # Disable buffering for real-time apps
        proxy_buffering off;
    }

    # Optional: Custom access logs
    # access_log /var/log/nginx/carepronginx.access.log;
    # error_log /var/log/nginx/carepronginx.error.log;
}

```

**5. Testing and Reloading Configuration**


Enable the new site and disable the default configuration:

```
sudo ln -s /etc/nginx/sites-available/carepronginx.arcapps.org /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
```

**Test the NGINX configuration:**

```
sudo nginx -t
```

**Reload NGINX:**

```
sudo systemctl reload nginx
```

**6. SSL Integration**

Obtain a New SSL Certificate
Run Certbot to obtain and configure SSL:

```
sudo certbot --nginx -d carepronginx.arcapps.org
```

**Verify SSL Installation**
To confirm the SSL certificate installation:

```
sudo certbot certificates
```

**7. Validation**

Open https://carepronginx.arcapps.org in a browser to verify access.
Check WebSocket functionality in Blazor apps.
Confirm sticky sessions by observing consistent server responses during testing.

**8. Maintenance**

Renew SSL Certificates Automatically: Certbot automatically handles renewal. To manually test renewal:

```
sudo certbot renew --dry-run
```

**Monitor Logs:**

  * Access logs: `/var/log/nginx/access.log`
  * Error logs: `/var/log/nginx/error.log`


# Cookie Present Sticky CarePro Load Balancer Docs



**1. Overview**

This document outlines the process for configuring an NGINX load balancer with sticky sessions, SSL encryption, and API routing for CarePro domains. It ensures consistent session handling and secure connections for applications.

**2. Installation**

Install NGINX

```
sudo apt update
sudo apt install nginx -y
```

**Install Certbot and NGINX Plugin**

```
sudo apt install certbot python3-certbot-nginx -y
```

**3. Configuration**

Define Upstreams in `nginx.conf`

Edit the NGINX main configuration file:

```  
sudo nano /etc/nginx/nginx.conf
```

**Add the following content:**

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    # Map to handle session_id cookie
    map $cookie_session_id $session_id {
        "" $request_id;   # If no cookie, use a unique request ID
        default $cookie_session_id;
    }

    # Upstream servers
    upstream web_servers {
        hash $session_id consistent;
        server 192.168.10.125:80 max_fails=1 fail_timeout=30s;
        server 192.168.10.127:80 max_fails=1 fail_timeout=30s;
    }

    upstream api_servers {
        server 192.168.10.125:8081 max_fails=1 fail_timeout=30s;
        server 192.168.10.127:8081 max_fails=1 fail_timeout=30s;
    }

    include       mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log  /var/log/nginx/error.log;

    sendfile on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}

```

**Enable Access Logging**

Create a new site configuration file:

```
sudo nano /etc/nginx/sites-available/carepronginx.arcapps.org
```

**Add the following content:**

```
server {
    listen 443 ssl;
    server_name carepronginx.arcapps.org carepro-testing.ihmafrica.com;

    ssl_certificate /etc/letsencrypt/live/carepro-testing.ihmafrica.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/carepro-testing.ihmafrica.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    access_log /var/log/nginx/access.log;

    location / {
        # Set session cookie if it doesn’t exist
        if ($cookie_session_id = "") {
            add_header Set-Cookie "session_id=$request_id; Path=/; HttpOnly";
        }

        proxy_pass http://web_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /carepro-api/ {
        proxy_pass http://api_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }
}

```

**4. Testing and Reloading Configuration**

Link the new configuration file:

```
sudo ln -s /etc/nginx/sites-available/carepronginx.arcapps.org /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
```

**Test the configuration:**

```
sudo nginx -t
```

**Reload NGINX:**

```
sudo systemctl reload nginx
```

**5. SSL Configuration**

Obtain a New SSL Certificate
Run the following command to secure your site with SSL:

```
sudo certbot --nginx -d carepronginx.arcapps.org
```

**Verify SSL Installation**

Check the installed certificates:

```
sudo certbot certificates
```

**6. Validation**

Access the domain: https://carepronginx.arcapps.org.
Verify sticky session behavior by ensuring consistent server responses for repeated requests.
Confirm WebSocket support for Blazor apps.


**7. Maintenance**

Renew SSL Certificates Automatically
Certbot automatically renews SSL certificates. Test the renewal process:

```
sudo certbot renew --dry-run
```

**Monitor Logs**
   * Access logs: `/var/log/nginx/access.log`
   * Error logs: `/var/log/nginx/error.log`

