# Odoo Instalation on Debian 12 in a Proxmox CT

I followed the guide here: https://www.rosehosting.com/blog/how-to-install-odoo-18-on-debian-12/

Just in case to avoid loosing it I will copy the content below.

If you run into an error in Step 6 on line:

su - odoo18

Do this:

adduser --system --home=/opt/odoo18 --group odoo18
usermod -s /bin/bash odoo18

Then continue.

Reverse proxy is no need for me.





# Original post

How to Install Odoo 18 on Debian 12
November 18, 2024 by Jeff Wilson
How to Install Odoo 18 on Debian 12

Odoo, formerly OpenERP, is a suite of open-source business applications. It is widely regarded as a leading open-source enterprise resource planning (ERP) software. Odoo offers a diverse range of modules that can be incorporated into a single application, significantly contributing to its widespread adoption. The latest version of Odoo, Odoo 18, brings additional features that improve its usability. The updated interface includes keyboard shortcuts, facilitating the selection of records and enabling multiple selections with ease. This article will guide you on how to install Odoo 18 on Debian 12.

Table of Contents

    Prerequisites
    Step 1. Login to VPS and Update the System
    Step 2. Install Dependencies
    Step 3. Add System User
    Step 4. Install PostgreSQL
    Step 5. Install wkhtmltopdf
    Step 6. Install Odoo
    Step 7. Create Odoo Systemd Unit file
    Step 8. Install and Configure Reverse Proxy

Prerequisites

    A Debian 12 VPS with at least 2GB of RAM.
    SSH root access, or user with sudo privileges.

Conventions

# – given commands should be executed with root privileges either directly as a root user or by use of sudo command
$ – given commands should be executed as a regular user

Step 1. Login to VPS and Update the System

First of all, we need to log in to our Debian 12 VPS through SSH using your favorite terminal:

ssh root@IP_Address -p Port_number

Replace “root” with a user with sudo privileges. Replace “IP_Address” and “Port_Number” with your server’s IP address and SSH port number. Next, let’s make sure that we’re on Debian 12. You can do that like this:

# lsb_release -a

The command should return an output similar to this:

No LSB modules are available.
Distributor ID: Debian
Description: Debian GNU/Linux 12 (bookworm)
Release: 12
Codename: bookworm

Before starting, you have to make sure that all Debian packages installed on the server are up to date. You can do this by running the following commands:

# apt update -y

Step 2. Install Dependencies

At the time of this writing, Debian 12 ships with Python 3.11. Since Odoo 18 requires at least Python 3.11, we do not need to install a new Python. Let’s install its dependencies first before proceeding to the next step.

apt install build-essential wget git python3.11-dev python3.11-venv \
    libfreetype-dev libxml2-dev libzip-dev libsasl2-dev \
    node-less libjpeg-dev zlib1g-dev libpq-dev \
    libxslt1-dev libldap2-dev libtiff5-dev libopenjp2-7-dev libcap-dev

Step 3. Add System User

There are some methods to install Odoo. The simplest and easiest way is to install it from the repository. But we are going to install Odoo 18 using a Python virtual environment. Since it will be running under a regular system user, we will create a new system user. Let’s execute this command below to add a new system user.

# useradd -m -d /opt/odoo18 -Urs /bin/bash odoo18

Alright! A new system user named ‘odoo18’ has been added. Its home directory is /opt/odoo18, which is the directory we are going to use for the installation.
Step 4. Install PostgreSQL

Please be informed that Odoo only supports PostgreSQL. In this step, we will install a PostgreSQL server from the default Debian 12 repository.

# apt install postgresql

After installing PostgreSQL, the service will automatically run, and now it is time to create a PostgreSQL user with the same name as the new system user. Run the following command to create a PostgreSQL user:

# su - postgres -c "createuser -s odoo18"

Our Odoo 18 will run the service using the new PostgreSQL user. Let’s proceed to the next step.
Step 5. Install wkhtmltopdf

Wkhtmltopdf, a command line tool, is an open-source solution for converting HTML data into PDF format using a Qt Webkit. Debian 12 also provides this package in their repository, but we will use the patched with QT version. However, since the .DEB package for Debian 12 is still not available on their official download page.

# wget http://archive.ubuntu.com/ubuntu/pool/main/o/openssl/libssl1.1_1.1.1f-1ubuntu2_amd64.deb
# apt install ./libssl1.1_1.1.1f-1ubuntu2_amd64.deb

# wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-2/wkhtmltox_0.12.6.1-2.bullseye_amd64.deb
# apt install ./wkhtmltox_0.12.6.1-2.bullseye_amd64.deb

If you see an error message, you can execute this command:

# apt install -f

Step 6. Install Odoo

In this step, we must switch to the system user we created earlier. Let’s switch to the system user ‘odoo18’ to download Odoo files from GitHub and create a new Python environment.

# su - odoo18

Next, let’s download Odoo from GitHub.

$ git clone https://www.github.com/odoo/odoo --depth 1 --branch 18.0 odoo18

Create a Python virtual environment.

This Odoo installation method enables you to install multiple Odoo versions on your server by using a Python virtual environment. Let’s create a new Python virtual environment for our Odoo 18.

$ python3.11 -m venv odoo18-venv

At this point, we have a new Python virtual environment under the directory /opt/odoo18/odoo18-venv. We need to activate it before installing Odoo.

$ source odoo18-venv/bin/activate

Once executed, your shell prompt would look like this:

(odoo18-venv) odoo18@debian12:~$

Next, let’s install Odoo 18.

(odoo18-venv) odoo18@debian12:~$ pip install wheel setuptools pip --upgrade
(odoo18-venv) odoo18@debian12:~$ pip install -r odoo18/requirements.txt

Wait until the installation finishes; it will be under the directory /opt/odoo18/odoo18. We can create a new directory to store our custom Odoo add-ons now.

$ mkdir /opt/odoo18/odoo18/custom-addons

Done! Let’s exit from user ‘odoo18’ and create an Odoo configuration file.

$ exit

The command above should bring you back to the previous user, in this case, root.

nano /etc/odoo18.conf

Paste the following content into the file.

[options]
admin_passwd = m0d1fyth15
db_host = False
db_port = False
db_user = odoo18
db_password = False
addons_path = /opt/odoo18/odoo18/addons,/opt/odoo18/odoo18/custom-addons

Replace m0d1fyth15 with a stronger password. This will be your Odoo’s master password. Save the file, then exit from nano editor.
Step 7. Create Odoo Systemd Unit file

To manage the newly installed Odoo 18, we need to create a systemd service file. Let’s run the command below to complete this step.

# systemctl edit odoo18 --force --full

The command above will bring you to a text editor. Insert the following content into the systemd unit file.

[Unit]
Description=odoo18
Requires=postgresql.service
After=network.target postgresql.service

[Service]
Type=simple
SyslogIdentifier=odoo18
PermissionsStartOnly=true
User=odoo18
Group=odoo18
ExecStart=/opt/odoo18/odoo18-venv/bin/python3 /opt/odoo18/odoo18/odoo-bin -c /etc/odoo18.conf
StandardOutput=journal+console

[Install]
WantedBy=multi-user.target

Save the file, then exit. And do not forget to reload the systemd service and then run Odoo.

# systemctl daemon-reload
# systemctl enable --now odoo18

Check if Odoo is starting by running this command:

# systemctl status odoo18

Now, you can navigate to http://YOUR_SERVER_IP_ADDRESS:8069, and you will see the default Odoo page
How to Install Odoo 18 on Debian 12

When working with an Odoo database, you will be prompted for the master password. The master password is the one in your Odoo configuration file; it is the value of admin_passwd. Make sure to use a strong password for your Odoo master password.
Step 8. Install and Configure Reverse Proxy

To access your Odoo website at http://yourdomain.com instead of http://YOUR_SERVER_IP_ADDRESS:8069, we must install a web server and configure it as a reverse proxy. A reverse proxy offers many benefits, such as load balancing, caching, compression, and serving static content. Let’s install the web server.

# apt install nginx

On the Debian 12 server, Nginx should be up and running upon installation. Let’s create a new Nginx server block now.

# nano /etc/nginx/conf.d/odoo.conf

Insert the following into that file.

upstream odoo18 {
   server 127.0.0.1:8069;
   }

upstream odoochat {
   server 127.0.0.1:8072;
   }

server {
   listen 80;
   server_name yourdomain.com;

   access_log /var/log/nginx/odoo18.access.log;
   error_log /var/log/nginx/odoo18.error.log;

   proxy_buffers 16 64k;
   proxy_buffer_size 128k;

location / {
   proxy_pass http://odoo18;
   proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
   proxy_redirect off;

   proxy_set_header Host $host;
   proxy_set_header X-Real-IP $remote_addr;
   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   proxy_set_header X-Forwarded-Proto https;
   }

location /longpolling {
   proxy_pass http://odoochat;
   }


location ~* /web/static/ {
   proxy_cache_valid 200 60m;
   proxy_buffering on;
   expires 864000;
   proxy_pass http://odoo18;
   }
}

Replace yourdomain.com with your domain name or subdomain name pointing to your server IP address. Then, save the file and exit from the editor.

To apply the changes, we can restart the Nginx.

# systemctl restart nginx

That’s it. You should now be able to access Odoo 18 at http://yourdomain.com.

If you prefer Apache to Nginx, you can check and follow our post on how to install Odoo 17 on Ubuntu 24.04.

Congratulations! You have followed this article and successfully installed Odoo 18 on your Debian 12 server.

Of course, you don’t have to install Odoo 18 on Debian 12 if you have an active Debian VPS Hosting service with us. In that case, you can ask our expert Linux admins to install Odoo 18 on Debian 12 for you. Our admins will install and set up Odoo 18 immediately without any additional fee, along with many helpful configurations and optimizations we can do for you. In fact, we’ll gladly install Odoo and any other services you may require and make sure they all play nice with one another.

If you liked this post about installing Odoo 18 on Debian 12, please share it with your friends or leave a comment below.

Tags debian 12, how to install, odoo 18
How To Configure Nginx as a Reverse Proxy on Ubuntu 24.04
Nameservers: What Are They And How Do They Work?



