# Linux Server Configuration

by Leigh Michael Forrest

IP Address: 35.163.207.244

Accessible SSH port: 2200

Application URL: http://35.163.207.244

---

## Configuration walkthrough

### 1- create the user *grader* and give user sudo permissions

1. Log in to VM as `root` in SSH: `ssh -i ~/.ssh/udacity_key.rsa root@35.163.207.244
` Note: as of this writing, `root` remote access has been disabled.

2. Create a new user into the command line: `$ sudo adduser grader`

3. Create a `/etc/sudoers.d/grader` file and add the following in the text editor:
  "grader ALL=(ALL:ALL) ALL". After the text is entered, save it.

### 2- Update all currently installed packages

  1. While logged in as a sudo user, run the following commands: `sudo apt-get update`
    and `sudo apt-get upgrade`

### 3- Configure server's timezone to UTC

  1. While logged in as a sudo user, run the following: `sudo dpkg-reconfigure tzdata`

### 4-Configure and enforce key-based authentication

  1. On your local machine, run *ssh-keygen* : `ssh-keygen` and follow the
  commands on the screen. note: for this project, I created key-pair
  named `udacity_grader`

  2. While logged in as a *grader*, create the following file:
    `.ssh/authorized_keys` None: you may have disabled password login
    at this point. You may have to login as *root*.

  3. Paste the public key you created into the file `.ssh/udacity_grader`

  4. While still logged in as *grader*, change the file permissions:
    `sudo chmod 700 .ssh` and `sudo chmod 644 .ssh.authorized_keys`

  5. You are able to log into the server: `$ ssh -i ~/.ssh/udacity_grader grader@35.163.207.244`

  6. using `sudo nano /etc/ssh/sshd_config`, find the PasswordAuthentication line and replace `yes` with `no`.

  7. While here, go to `PermitRootLogin` and set it to `no`

  8. You now have to reset the ssh service: `sudo service ssh restart`


### 5-Change SSH port from 22 to 2200

  1. using `sudo nano /etc/ssh/sshd_config`, find `port` line and set it to `2200`

  2. You can now login with this line: `$ ssh -i ~/.ssh/udacity_grader grader@35.163.207.244 -p 2200`

### 6- Set up UFW firewall

  1. run the following comands:
  `sudo ufw deny incoming`

  `sudo ufw allow outgoing`

  `sudo ufw allow ssh`

  `sudo ufw allow 2200/tcp`

  `sudo ufw allow 80/tcp`

  `sudo ufw allow 123/udp`

  `sudo ufw deny 22`

  2. Before enabling firewall, check the status: `sudo ufw status` Make sure
  that 2200, 80

  3. Enable the firewall: `sudo ufw enable`

### 7-Install necessary packages

  1. install apache: `suo apt-get install apache2`

  2. install mod_wsgi: `sudo apt-get install libapache2-mod-wsgi python-dev`

  3. Enable mod_wsgi and start the server:

    `sudo a2enmod wsgi`

    `sudo service apache2 start`

  4. install git: `sudo apt-get install git`

    `git config --global user.name <name>`

    `git config --global user.email <email_address>`

  5. install postgresql: `sudo apt-get install postgresql postresql-contrib`

  6. install pip `sudo apt-get install python-pip`

  7. To properly start running the application, you will need to install Flask
    globally: `sudo apt-get install python-flask`

  8. In addition, you will need to install psycopg2 globally: `sudo apt-get install python-psycopg2
`

  9. install virtualenv: `sudo pip install virtualenv`

### 8- Clone the application and setup virtual environment

  1. Logged in as *grader*, run `cd /var/www` and `mkdir catalog`.

  2. cd into the new `catalog` directory, and clone the git repository:

  `git clone https://github.com/leighmforrest/neighborhood_bazaar.git catalog`

  3. Rename the main file: `mv application.py catalog_application.py`

  4. in `/var/www/catalog`, create the *catalog.wsgi* file:

  `cd /var/wwq/catalog`

  `sudo nano catalog.wsgi`

  The file should contain the following:

  `import sys
   import logging
   logging.basicConfig(stream=sys.stderr)
   sys.path.insert(0, "/var/www/catalog/")`

   `from catalog.catalog_application  import app as application`

  5. Create the virtual environment: `sudo virtualenv venv`

  6. Activate virtual environment: `source venv/bin/activate`

  7. Install all of the application's python dependencies:
    `cd /var/www/catalog/catalog`

    `pip install -r requirements.txt`

# 9- Setup and enable virtual host

  1. `sudo nano /etc/apache2/sites-available/catalog.conf`

  2. Paste in the following code:

  ```<VirtualHost *:80>
    ServerName 35-163-207-244.us-west-2.compute.amazonaws.com
    ServerAlias 35-163-207-244.us-west-2.compute.amazonaws.com
    ServerAdmin admin@35.163.207.244
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

  3. Enable the virtual host: `sudo a2ensite catalog`

# 10- Set up the database

  1. Use the user *postgres* and enter the posgtgres shell:
    `sudo su - postgres`

    `psql`

  2. Create a new user *catalog*: `CREATE USER catalog WITH PASSWORD 'thepassword'`

  3. Connect to database: `\c catalog`

  4. Create the catalog database with *catalog* as the owner:
     `CREATE DATABASE catalog WITH OWNER catalog;`

  5. Revoke all rights: `REVOKE ALL ON SCHEMA public FROM public;`

  6. Lock down permissions to only let catalog create tables:
     `GRANT ALL ON SCHEMA public TO catalog;`

  7. Log out of PostgreSQL: '\q'

  8. Double check that remote connections cannot be made to the database:

    `sudo nano /etc/postgresql/<db_version>/main/pg_hba.conf`

# 11- Set up application

  1. In the newly-renamed *catalog_application* find the line for `SQLALCHEMY_DATABASE_URI`. Replace the value with:
    `postgresql://catalog:thepassword@localhost/catalog`

  2. replace `SECRET_KEY`, `FACEBOOK_ID`, `FACEBOOK_SECRET` environment variables
     with the appropriate values.

     *NOTE: Hard-coded values are not a good practice, but environment variables
      are difficult to set up in mod_wsgi.*

  3. Get out of the virtual environment and run `python init.py` This should set up
     the database tables. If you are in a virtual environment, this may not work.

  4. Set the url correctly at `https://developers.facebook.com/` You will need
     the url. `http://35.163.207.244` is acceptable.
