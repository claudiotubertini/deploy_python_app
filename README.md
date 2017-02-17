# Linux-Server-Configuration-FSND-P5
This is the summary of the many steps to deploy and securing a python web applications on a Linux distribution (Ubuntu 14.04).
The application makes use of a PostgreSQL database and it is secured from a number of attack vectors.

## Project Access
#### IP address

**`52.27.162.111`**

#### Web Application URL
**`http://ec2-52-27-162-111.us-west-2.compute.amazonaws.com`**

## Project Overview

I configured a secured remote virtual machine to host a data driven web appllications and a database. This was accomplished through a provided Linux distribution by [Udacity](https://www.udacity.com), and an installed Apache2 server to serve Python Flask application that connects to a PostgresSQL database.

### Setup Virtual Machine and SSH into the server
Reference:
[SSH](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server)

[Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04)

The first time I logged into the server I used a private key supplied by udacity with the command:
```
ssh -i ~/.ssh/udacity_key.rsa  root@52.27.162.111
```
then I created a user called `grader` that was granted sudo privileges
```
adduser grader
```
given a strong password and associated to sudo group
```
gpasswd -a grader sudo
```
Then I edited the file `/etc/ssh/sshd_config` changing the access port from 22 to 2200, disabled root login but kept password login until the end of job (to avoid locking me out from the server). At the end of the file, added `UseDNS no` and `AllowUsers grader`. This will allow `grader` SSH login access.
then I restarted ssh service
```
sudo service ssh restart
```
I switched to the client machine and created a new user `grader`, then created an rsa key, accepting all default file names
```
sudo -u grader ssh-keygen -t rsa -b 2048 -f /home/grader/.ssh/id_rsa
```
Then I added grader public key to the server
```
ssh-copy-id grader@52.27.162.111 -p 2200
```
that allow me to connect without using a password. Because I frequently change the client (home, office, another office, ecc.) I added also the public key of the different machines. At the end of all the setup I will remove from `/etc/ssh/sshd_config` the password login with
`PasswordAuthentication no` but I will keep the chance to use my clients.

Now ssh again into the server as grader and update all available packages:
```
sudo apt-get update && sudo apt-get upgrade
```
There was also the need of a dist-upgrade
```
sudo apt-get dist-upgrade
```
and then
```
sudo apt autoremove --purge
```
Now is the time to configure the firewall `ufw` to only allow incoming connections for SSH(Port:2200), HTTP(Port:80) and NTP(Port:123).

* Check the status of UFW. Make sure it is `inactive`:

    ```
    sudo ufw status
    ```

* Deny all incoming connections as default so that we can allow the ones we need.

    ```
    sudo ufw default deny incoming
    ```

* Allow incoming TCP connection on SSH(Port:2200), HTTP(Port:80), NTP(Port:123)

    ```
    sudo ufw allow 2200/tcp
    sudo ufw allow 80/tcp
    sudo ufw allow 123/udp
    ```
* Enable the firewall:

    ```
    sudo ufw enable
    ```
* I've checked the time zone but it was already set to UTC:
```
sudo dpkg-reconfigure tzdata
```
and I installed ntp to sync server time
```
sudo apt-get install ntp
```
## Deploying the web application
Reference:
[Apache](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-apache-mysql-and-python-lamp-server-without-frameworks-on-ubuntu-14-04)
[Python and Flask](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)

* I've installed Apache web Server:

```
sudo apt-get install apache2
```

* Installed `mod_wsgi`, and `python-setuptools` helper package. This will serve Python apps from Apache:

    ```
    sudo apt-get install python-setuptools libapache2-mod-wsgi
    ```

* Restarted Apache server to load mod_wsgi.

    ```
    sudo a2enmod wsgi
    sudo service apache2 restart
    ```
* To remove the message: `Could not reliably determine the server's fully qualified domain name...`:

    ```
    echo "ServerName 52.27.162.111" | sudo tee /etc/apache2/conf-available/fqdn.conf
    sudo a2enconf fqdn
    ```
* Install git
```
sudo apt-get install git
```
* then I download in the home directory all the files of my application. Later I will put everything in place leaving behind git folder, empty directory, etc.
```
git clone https://github.com/claudiotubertini/P3-item-catalog.git
```

* In the`www` directory I setup the directory folder.

```
sudo mkdir Catalog
cd Catalog
sudo mkdir catalog
cd catalog
sudo mkdir static templates uploads
```
* then I copy the application:
```cp -a ~/P3-item-catalog/book_catalog/*  /var/www/Catalog/catalog
```
and I change the owner of the files
```
sudo chown -R www-data:www-data .
```
## The application.
For sake of clarity I change the name of the main file, that that contains `app.run()`, to `__init__.py`.
* In the `catalog` folder I put a wsgi file that will import `__init__.py` and it is that file that will be called by the apache wsgi server. This is the file `catalog.wsgi`:

```
#!/user/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/Catalog/catalog")

from __init__ import app as application

application.secret_key = 'xxxxxxxxxxxxxxxxxxxxxx'
```
* Now the virtual environment. In the catalog folder (lowercase)

    ```
sudo pip install virtualenv
sudo virtualenv venv
```

* I enable all permissions for the new virtual environment `venv`. By doing so, `sudo` would not be used inside the environment.

        ```
  sudo chmod -R 777 venv
        ```
  I've found the hard way that if you want to install packages only in the virtual environment use only `pip install`, with `sudo` you install packages globally. Note that with `apt-get install` you call ubuntu packages while `pip install` is for those contained in [PyPA repository](https://packaging.python.org/)

  ```
    (venv) $:/var/www/Catalog/catalog$ sudo apt-get install python-setuptools
    (venv) $:/var/www/Catalog/catalog$ pip install Flask
    (venv) $:/var/www/Catalog/catalog$ pip install httplib2
    (venv) $:/var/www/Catalog/catalog$ pip install requests
    (venv) $:/var/www/Catalog/catalog$ sudo apt-get install python-psycopg2
    (venv) $:/var/www/Catalog/catalog$ pip install oauth2client
    (venv) $:/var/www/Catalog/catalog$ pip install sqlalchemy
```
* I created a virtual host config file that calls `catalog.wsgi`:

    ```
    $: sudo vim /etc/apache2/sites-available/catalog.conf
    ```
* In the newly created `catalog.conf` file, I pasted in the following lines of code.
        ```
        <VirtualHost *:80>
            ServerName 52.27.162.111
            ServerAdmin admin@52.27.162.111
            ServerAlias ec2-52-27-162-111.us-west-2.compute.amazonaws.com
            WSGIScriptAlias / /var/www/Catalog/catalog.wsgi
            <Directory /var/www/Catalog/catalog/>
                Order allow,deny
                Allow from all
            </Directory>
            Alias /static /var/www/Catalog/catalog/static
            <Directory /var/www/Catalog/catalog/static/>
                Order allow,deny
                Allow from all
            </Directory>
            <Directory "/var/www/Catalog/"> # this directory allows to add .htaccess file
                AllowOverride All
            </Directory>
            ErrorLog ${APACHE_LOG_DIR}/error.log
            LogLevel warn
            CustomLog ${APACHE_LOG_DIR}/access.log combined
        </VirtualHost>

        ```
 * Enabled the Virtual Host.

        ```
        sudo a2ensite catalog.conf
        sudo service apache2 restart
        ```
* I changed in the file `__init__.py` all the relative path to absolute path. For example `static/1_image.img` should become `/var/www/Catalog/catalog/static/1_image.img`.


### Install and configure PostgreSQL
Reference: [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)

* Install the PostgreSQL database:

    ```
    sudo apt-get install postgresql postgresql-contrib
    ```

* Ensure that no remote connections are allowed. It should be default.

    ```
    sudo vim /etc/postgresql/9.3/main/pg_hba.conf
    ```


* Create a user `catalog` that it will be used for psql. Change to the default user `Postgres` and connect to the postgresql system:

    ```
    psql
    ```


* At this point I had a series of error messages about the `locale` that didn't allow me to launch `psql`. I updated the locale with:
```
$ sudo locale-gen "en_US.UTF-8"
$ sudo locale-gen "it_IT.UTF-8"
```
and I could login through psql to the database. Reference [stackoverflow 1](http://stackoverflow.com/questions/2724121/) but also [stackoverflow 2](http://dba.stackexchange.com/questions/135127/incomplete-installation-with-unsupported-localepostgresql-database-creation-problem-with-localization)

* Added the postgres user: `catalog` and setup users parameters.
Allowed the user `catalog` to be able to create database tables

    ```
    ALTER USER catalog CREATEDB;
    ```
    You can list the roles available in postgres, and their attribute:

    ```
    postgres=# \du
    ```

* Created a new database called `catalog` for the user: `catalog`:

    ```
    postgres=# CREATE DATABASE catalog WITH OWNER catalog;
    postgres=# \c catalog
    ```

* Revoke all rights on the database schema, and grant access to catalog only.

    ```
    catalog=# REVOKE ALL ON SCHEMA public FROM public;
    catalog=# GRANT ALL ON SCHEMA public TO catalog;

    ```

* Created postgresql database and amended the link to the database in the application, changing from sqlite to postgresql
Now the relevant command is the following:
    ```
    create_engine('postgresql://catalog:DB-PASSWORD@localhost/catalog') #DB-PASSWORD is the password of the user `catalog`


* To get Google+ authorization working on the Developer Console: http://console.developers.google.com, I added both `52.27.162.111` and `http://ec2-52-27-162-111.us-west-2.compute.amazonaws.com`

* To get Facebook authorization working,I added `http://ec2-52-27-162-111.us-west-2.compute.amazonaws.com` to app console.

