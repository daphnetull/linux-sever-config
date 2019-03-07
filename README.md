This is the final project of [Udacity's Fullstack Nanodegree Course](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004).

Successful completion of this project demonstrates the ability to deploy the Item Catalog project onto an Ubuntu Linux web server instance using [AWS Lightsail](https://aws.amazon.com/lightsail/) with public key encryption.  Tools used include the [Apache HTTP Server](http://httpd.apache.org/) to respond to HTTP requests and serve a webpage, [Flask](http://flask.pocoo.org/) to serve the database contents to the web app, [SQLAlchemy](https://www.sqlalchemy.org/) as an Object-Relational Mapping tool to convert database contents into Python objects, and [PostgreSQL](https://www.postgresql.org/) to serve as the database schema itself.

Below are the steps taken to complete this project:

## Beginning steps

1. Logged into lightsail instance with ssh
2. As root, executed the commands`sudo apt-get update` to update the package list, and then `sudo apt-get upgrade` to upgrade the packages themselves.

Sources:
* https://medium.com/@GalarnykMichael/aws-ec2-part-2-ssh-into-ec2-instance-c7879d47b6b
* https://classroom.udacity.com/nanodegrees/nd004/parts/b2de4bd4-ef07-45b1-9f49-0e51e8f1336e/modules/56cf3482-b006-455c-8acd-26b37b6458d2/lessons/4331066009/concepts/48010894520923

## User creation

1. Created `grader` user and gave this user sudo access with `sudo adduser grader`and `sudo usermod -aG sudo grader

### Sources:

* https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/managing-users.html
* https://stackoverflow.com/questions/47803532/cant-login-as-a-newly-added-user-in-aws-lightsail-instance

## Public Key Encryption

1. On my local machine, I used the `ssh-keygen` tool to generate a public and private key.
2. Copied the public key, created .ssh folder in the `grader` account, created `authorized_keys` files and copied the public key by typing `nano authorized_keys`
3. I changed the `.ssh` directory file permissions by typing `chmod 700 .ssh`
4. I changed the `authorized_keys` file permissions by typing `chmod 600 .ssh/authorized_keys`

### Sources:
* https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/managing-users.html


## Installed all necessary packages: Python, Flask, SQLAlchemy, PostgreSQL, etc. 
* Python - `sudo apt-get install python-pip`
* Flask - `sudo pip install Flask`
* SQLAlchemy - `sudo pip install Flask-SQLAlchemy`
* Psycopg2 - `sudo pip install psycopg2`
* OAuth2 Client - `sudo pip install oauth2client`
* Requests - `sudo pip install requests`
* httplib2 - `sudo pip install httplib2`
* PostreSQL -`sudo apt-get install postgresql`

## sshd configuration

1. Edited the sshd configuration file by typing `sudo nano /etc/ssh/sshd_config`
2. Changed `PermitRootLogin prohibit-password` to `PermitRootLogin no`
3. Changed `PasswordAuthentication yes` to `PasswordAuthentication no`
4. Changed `Port 22` to `Port 2200`
5. Restarted ssh with `sudo service ssh restart`
6. Added a custom TCP Port of 2200 to my AWS Lightsail instance network settings.
7. At this point I began logging into the instance on my local terminal with the command `ssh -i /home/daphne/Downloads/daphne.pem ubuntu@54.159.128.204.xip -p 2200`

### Sources:
* https://classroom.udacity.com/nanodegrees/nd004/parts/b2de4bd4-ef07-45b1-9f49-0e51e8f1336e/modules/56cf3482-b006-455c-8acd-26b37b6458d2/lessons/4331066009/concepts/48010894820923
* http://manpages.ubuntu.com/manpages/xenial/man5/sshd_config.5.html

## UFW (uncomplicated firewall) configuration

1. `sudo ufw default deny incoming` to deny all default incoming requests
2. `sudo ufw default allow outgoing` to allow all outgoing default requests
3. `sudo ufw allow ssh` to allow ssh
4. `sudo ufw allow 2200/tcp` to open port 2200
5. `sudo ufo allow www` to open port 80
6. `sudo ufw enable` to activate firewall

### Sources:
* https://knowledge.udacity.com/questions/19521

## PostreSQL Setup

1. Logged into `postgres` user with `sudo su - postgres`
2. `psql` to open the database
3. `CREATE USER catalog;` to create the catalog database user
4. `ALTER USER catalog WITH PASSWORD catalog;` to create password
5. `CREATE DATABASE catalog WITH OWNER catalog;` to create the database
6. `REVOKE ALL ON schema public FROM public;` to revoke public access
7. `GRANT ALL ON schema public TO catalog;` to give catalog user complete control
8. `\q` to quit the schema

### Sources:
* https://help.ubuntu.com/community/PostgreSQL
* https://dba.stackexchange.com/questions/35316/why-is-a-new-user-allowed-to-create-a-table
* http://killtheyak.com/use-postgresql-with-django-flask/

## Cloning and re-configuring the Item Catalog Project

1. Imported the Item Catalog project by navigating to `/var/www/` and typing `sudo git clone https://github.com/daphnetull/item-catalog.git catalog` to clone it and rename it as `catalog`
2. Renamed `project.py` as `__init__.py` which wsgi looks for to run the Flask application logic
3. In `__init__.py`, `database_setup.py`, and `catalogitems.py` changed the `sqlite:///itemcatalog.db`to `postgresql://catalog:catalog@localhost/catalog`
4. Changed path to `client_secrets.json` in `__init__.py` to absolute path of `/var/www/catalog/client_secrets.json`
5. In `__init__.py`, removed the parameters of the `run()` method.

### Sources:
* https://devops.ionos.com/tutorials/deploy-a-flask-application-on-ubuntu-1404/
* https://github.com/AbigailMathews/FSND-P5

## Apache and wsgi configuration

1. Installed Apache with `sudo apt-get install apache2`
2. Installed mod-wsgi with 	`sudo apt-get install libapache2-mod-wsgi python-dev`
3. `sudo a2enmod wsgi` to enable the module
4. `sudo service apache2 restart` to restart Apache
5. Created `catalog.wsgi` with `sudo nano catalog.wsgi`
`import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/var/www/')

from catalog import app as application
application.secret_key = 'super_secret_key'`
6. Navigated to `/etc/apache2/sites-available`
7. Created Apache configuration file with `sudo nano catalog.conf`
`<VirtualHost *:80>
      ServerName 54.159.128.204.xip.io
      ServerAdmin daphne@daphneedle.com
      WSGIScriptAlias / /var/www/catalog/catalog.wsgi
      <Directory /var/www/catalog/>
          Order allow,deny
          Allow from all
      </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>`
8. Disabled default Apache site with `sudo a2dissite 000-default`
9. Enabled catalog app site with `sudo a2ensite catalog`
10. Reloaded and restarted Apache with `sudo service apache2 reload` and ` sudo service apache2 restart`

### Sources:
* https://knowledge.udacity.com/questions/31871
* https://askubuntu.com/questions/25374/how-do-you-install-mod-wsgi
* https://devops.ionos.com/tutorials/deploy-a-flask-application-on-ubuntu-1404/
* https://github.com/SteveWooding/fullstack-nanodegree-linux-server-config/blob/master/README.md
* http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/
* https://mastizada.com/blog/flask-on-apache-with-mod_wsgi/
* https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps

## Reconfigured OAuth 

1. In Google Developers Console, changed the JavaScript origins URL to `http://54.159.128.204.xip.io/` and the redirect URI’s accordingly
2. Re-downloaded `client_secrets.json` and changed the contents of the same file on my server to match.
3. Navigated to http://54.159.128.204.xip.io/ in my browser to test the app.
