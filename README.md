# Linux Server Configuration Project

## Project Overview
We  take a baseline installation of a Linux distribution on a virtual machine and prepare it to host web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.


## Access Specifications
- IP address: 52.40.193.59
- Accessible SSH Port: 2200
- URL: http://ec2-52-40-193-59.us-west-2.compute.amazonaws.com/
- SSH: ssh -i ~/.ssh/udacity_key.rsa grader@52.40.193.59 -p 2200
- Private key: only the project reviewer has access to it other than myself. 
- Ubuntu 14.04 (provided)


### Summary of software installed
- NTP for Time Synchronization
- fail2ban
- unattended-upgrades
- Apache
- Apache Mod WSGI
- Python
- PostgreSQL
- Git
- Flask
- httplib2
- SQLAlquemy
- requests
- oauth2client
- psycopg2
- werkzeug


### Summary of configurations made
1. Launched the virtual machine. Downloaded the Private Key. Moved it to the .ssh directory of the local machine. Applied `chmod 600` to the key. Logged in to the server as root via ssh.


2. Created a new user named `grader` via `sudo adduser grader`.
Once added, it asks for a password. In this case we keep it simple: `grader` and leave all the other prompts empty (This password will be unaccesible from outside the server as we'll configure later on). We confirm the information and move on. 

Added the new user `grader` to the sudoers directory. 
Thus, we `sudo nano /etc/sudoers.d/grader` and add the following text inside the file: `grader ALL=(ALL:ALL) ALL`. After, we make sure to save it.
To check if we did this right, we run the following command to see if `grader` is there: `sudo ls /etc/sudoers.d`. 

To give the new user sudo permission, we need to copy the authorized keys to this one and set priviledges. We do this running the following commands in order:

- `cd /home/grader`
- `mkdir .ssh`
- `cp /root/.ssh/authorized_keys /home/grader/.ssh/`
- `chmod 700 .ssh`
- `chmod 644 .ssh/authorized_keys`
- `chown -R grader .ssh`
- `chgrp -R grader .ssh`

We can now login with the new user: 
`ssh -i ~/.ssh/udacity_key.rsa grader@52.40.193.59`

Eliminated the ability to login with Root: 
`sudo nano /etc/ssh/sshd_config` and we set PermitRootLogin to `no`.
To enforce key-based authentication, we also set PasswordAuthentication as `no`and save it. Last, we run `sudo service ssh restart` so that these changes are taken into effect.


3. Updated all currently installed packages
First, we check what updates are available by running `sudo apt-get update`. After we make sure that the changes won't break our installation, we run `sudo apt-get upgrade` to install all the updates. It will prompt us to confirm that we want to go ahead with the changes and we say yes. During this part of the process, some packages might ask for additional confirmations, which after thoroughly reading, we can accept or decline.


4. Configured the local timezone to UTC
We run `sudo dpkg-reconfigure tzdata` to open time configuration. We select `None of the above` in the list and find UTC there and confirm. It is also good practice to enhance time synchronization using NTP. We do this by running the command: `sudo apt-get install ntp`. We confirm yes. (as explained in Ubuntu Time reference in Third Party Resources at the end of this document).


5. Changed the SSH port from 22 to 2200.
We `sudo nano /etc/ssh/sshd_config`, find the Port line and edit it to 2200. For this to take effect, we need to restart the service with `sudo service ssh restart`.


6. Configured the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
First, we `sudo ufw status` to make sure that our firewall is inactive. 
Before turning it on, we make sure that all incoming traffic is denied via `sudo ufw default deny incoming` and we allow all outgoing with `sudo ufw default allow outgoing`.

To allow the connections for this project, we run:
- `sudo ufw allow 2200/tcp`
- `sudo ufw allow 80/tcp`
- `sudo ufw allow 123/udp`

And last, we enable the Firewall:
`sudo ufw enable`
 
 All set!

Additionally, to ban attackers and look out for repeated unsuccessful login attempts, we can install `fail2ban` to protect SSH. We do so by running the following commands in order:

Install and make it accessible:
- `sudo apt-get install fail2ban`
- `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`

Edit it:
- `sudo nano /etc/fail2ban/jail.local`

In here, we update the destemail to the admin's email. We are installing this as a preventive measure, as having key-based authentication enforced should be enough. This is still useful for other things such as smtp/imap-logins. 

Last, we restart fail2ban by running `sudo service fail2ban restart`.


7. Installed and configured Apache to serve a Python mod_wsgi application

Install Apache:
- `sudo apt-get install apache2`

Install Mod_wsgi (which is an Apache HTTP server mod that enables us to serve Flask apps later on).
- `sudo apt-get install libapache2-mod-wsgi python-dev`

We need to enable it with: `sudo a2enmod wsgi` and after, we start the Apache service with: `sudo service apache2 start`.


8. Installed and configured PostgreSQL (Do not allow remote connections). Created a new user named catalog that has limited permissions to your catalog application database.

Install PostgreSQL
`sudo apt-get install postgresql`

Note: Remote connections are disabled by default.


We change into user postgres (created during PostgreSQL installation)
- `sudo su - postgres`
We connect to the system with:
- `psql`
We create the database user and include its password
- `CREATE USER catalog WITH PASSWORD 'verystrongpassword';`
We give user catalog the ability to create databases
- `ALTER USER catalog CREATEDB;`
We create a database named catalog owned by our catalog user
- `CREATE DATABASE catalog WITH OWNER catalog;`
We connect to the database catalog
- `\c catalog`
We revoke all rights from it
- `REVOKE ALL ON SCHEMA public FROM public;`
We now let only the catalog role create tables in the catalog database
- `GRANT ALL ON SCHEMA public TO catalog;`
We can check that our user catalog is created and has its role assigned to create databases with the command `\du`, and our that our catalog database is present with the command `\dt`.
We can now log out from PostgreSQL:`\q`. Then we can return to the grader user by typing: `exit`.


9. Installed git, cloned and setup the Catalog App project (from my GitHub repository from earlier in the Nanodegree program) so that it functions correctly when visiting your server’s IP address in a browser. Remember to set this up appropriately so that your .git directory is not publicly accessible via a browser!

Install git:
- `sudo apt-get install git`

Go to the directory where you want the project to be located and clone it:
- `cd /var/www`
- `sudo git clone https://github.com/romanrodriguez/fullstack-nanodegree-item-catalog.git`
- `mv fullstack-nanodegree-item-catalog catalog`


Note: At this point, we need to update some .py files cloned with the new database connection created before and one of them with a new name. We will edit the name first and then the three files that need changing the sqlite connection to the postgreSQL one. 

According to Digital Ocean's Documentation mentioned at the end of this document, it is good practice renaming our `catalog.py` to `__init__.py`. We change directory to be inside /catalog/catalog and run `mv catalog.py __init__.py` to do this.

We copy the following and replace it in the files afterwards:
`engine = create_engine('postgresql://catalog:verystrongpassword@localhost/catalog')`

`sudo nano /var/www/catalog/catalog/__init__.py`
`sudo nano /var/www/catalog/catalog/database_setup.py`
`sudo nano /var/www/catalog/catalog/populate_catalog_database.py`

We will also add the absolute path of our `client_secrets.json` to our `__init__.py` so that it can find it.
`sudo nano /var/www/catalog/catalog/__init__.py`

Add absolute path to CLIENT_ID = json.loads and to oauth_flow in gconnect:     Absolute path to add: `/var/www/catalog/catalog/client_secrets.json`

We continue with our installation:
Install PIP and all required Python Packages to run the project
- `sudo apt-get install python-pip`
- `sudo apt-get install python-psycopg2`
- `sudo apt-get install python-sqlalchemy`
- `sudo pip install werkzeug==0.8.3`
- `sudo pip install flask==0.9`
- `sudo pip install Flask-Login==0.1.3`
- `sudo pip install oauth2client`
- `sudo pip install requests`
- `sudo pip install httplib2`

After having all the packages installed, we can go ahead and onfigure the app:
- `cd /var/www`
- `sudo chown -R grader catalog`
- `sudo chgrp -R grader catalog`
- `cd /var/www/catalog`

We create a catalog.wsgi file in the `/var/www/catalog/` directory to serve the application over the mod_wsgi. We create the file: `sudo echo >catalog.wsgi`. And we edit it: `sudo nano catalog.wsgi`.  The file should look like this:

``` 
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```

Then, we need to configure and enable a new virtual host by creating a virtual host conifg file: `sudo nano /etc/apache2/sites-available/catalog.conf`.
Paste in the following lines of code:

```
<VirtualHost *:80>
    ServerName 52.40.193.59
    ServerAlias ec2-52-40-193-59.us-west-2.compute.amazonaws.com
    ServerAdmin grader@52.40.193.59
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Now we can start serving the application. Enable the new virtual host first.
`sudo a2ensite catalog`
Restart Apache so this takes effect.
`sudo service apache2 reload`


Create the database and populate it.
1. `python /var/www/catalog/catalog/database_setup.py`
2. `python /var/www/catalog/catalog/populate_catalog_database.py`

If we run `python __init__.py`, we can see that the `client_secrets.json` will either not be there or will need to be modified to cater for this server.
In this case, we need the to adjust our project credentials in the Google Developers Console: `https://console.developers.google.com/apis/credentials?`
We go to Edit Settings for this particular app and add the hostname and public IP address to the Authorized JavaScript origins as such:
Add `http://52.40.193.59`.
Add `http://ec2-52-40-193-59.us-west-2.compute.amazonaws.com`.
 
And add the host name + 'oauth2callback' to Authorized redirect URIs, as such: `http://ec2-52-40-193-59.us-west-2.compute.amazonaws.com/oauth2callback`

We re-download `client_secrets.json` and include it in this server with the following steps.

Open the downloaded JSON file with a text editor and copy its contents to your clipboard. Back in our terminal we `cd catalog` and create the file with
`echo >client_secrets.json`. Then we edit it with `sudo nano /catalog/client_secrets.json` and paste the content from our clipboard in there and save.

We restart Apache to relaunch the app
`sudo service apache2 restart`


To make sure we restrict access to the .git folder we can:
`cd /var/www/catalog` and edit the .htaccess with `sudo nano .htaccess`.
Add this in the file and save: 

`RedirectMatch 404 /\.git`


To make sure our server is always up-to-date with important updates we can set it up to automatically do so without our supervision. We do this by setting automatic Ubuntu server package updates:
`sudo apt-get install unattended-upgrades`
`sudo dpkg-reconfigure --priority=low unattended-upgrades`


To read the error log and debug issues I used: 
`sudo cat /var/log/apache2/error.log`

### Third-party resources used
- Stack Overflow
- [Ubuntu Time](https://help.ubuntu.com/community/UbuntuTime#Using_the_Command_Line_.28terminal.29)
- [Udacity Forums Shared Resources](https://discussions.udacity.com/t/markedly-underwhelming-and-potentially-wrong-resource-list-for-p5/8587)
- [Additional Udacity Forums Shared Resources](https://discussions.udacity.com/t/project-5-resources/28343)
- [Digital Ocean Documentation](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
