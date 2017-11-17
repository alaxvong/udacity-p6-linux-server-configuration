Linux Server Configuration
========================

This README outlines the steps I used to run the previously completed [project 4](https://github.com/alaxvong/udacity-p4-item-catalog-application) on an Amazon Lighsail Ubunutu server. 

### Server Info
IP Address: 52.34.171.107

SSH Port: 2200

#### 1. SSH into server
- Download ubuntu user private key from Amazon Lightsail account menu item.
- Save private key to __~/.ssh directory__
- Connect to the server: `$ ssh -i ~/.ssh/PRIVATEKEYFILE.pem ubuntu@52.34.171.107`

---

#### 2. Update packages
- `$ sudo apt-get update`
- `$ sudo apt-get upgrade`

You may be prompted a couple of times in the middle of the upgrade about packages having been locally modified. I chose to "Keep the local version currently installed".

---
#### 3. Add new user
- Create a user named, __grader__ `$ sudo adduser grader`. I set the UNIX password to "grader" and left all other user information blank.
- Add grader to sudo user group for root privileges `$ sudo usermod -aG sudo grader`

---

#### 4. Generate SSH Key
- In a new shell window, generate a SSH key on your local machine: `$ ssh-keygen -t rsa`. I named it grader and left password blank.
- Copy the contents of the public key `cat ~/.ssh/grader.pub | pbcopy` 

---

#### 5. Setup SSH for Grader on Server
As the Ubuntu user, create __.ssh__ directory in grader's root directory
- `$ cd /home/grader`
- `$ sudo mkdir .ssh`

Create __authorized_keys__ file in __.ssh__
- `$ cd .ssh`
- `$ sudo touch authorized_keys`

Paste contents of __grader.pub__ from your local machine into __authorized_keys__
- `$ sudo nano authorized_keys`

Change Permissions for __.ssh__ and __authorized_keys__
- `$ sudo chmod 700 /home/grader/.ssh`
- `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`

Change owner of .ssh directory from Ubuntu to grader
- `$ sudo chown -R grader:grader /home/grader/.ssh`

From this point you should be able to SSH into the server with the following command:
- `$ ssh -i ~/.ssh/grader grader@52.34.171.107`

---

#### 6. Change SSH Port to 2200
- `$ sudo nano /etc/ssh/sshd_config` Find the __Port__ line and edit the value to __2200__.
- Refresh service `$ sudo service ssh restart.`

---

#### 7. Disable default login for root
- `$ sudo nano /etc/ssh/sshd_config` Change the value after 'PermitRootLogin' to 'no'
- Refresh service `$ sudo service ssh restart`

---

#### 8. Configure UFW (Uncomplicated Firewall)
```
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 123/tcp
$ sudo ufw enable
```
- The following command will now be necessary to SSH into the server:
`$ ssh -i ~/.ssh/grader grader@52.34.171.107 -p 2200`
- You may need to manually add TCP 2200 to the Firewall settings under the Networking tab of your Amazon Lightsail Ubunut Instance to SSH back into the server.

---

#### 9. Configure the local timezone to UTC

- `$ sudo dpkg-reconfigure tzdata`
- Select __None of the above__ then __UTC__

---

#### 10. Install Apache and WSGI
- `$ sudo apt-get install apache2`
- `$ sudo apt-get install libapache2-mod-wsgi python-dev`
- `$ sudo a2enmod wsgi`
- `$ sudo service apache2 start`
- You should see the Apache2 Ubunutu Default page when you visit the Public IP of your instance at this point.

---

#### 11. Set up project directory
- Install Git: `$ sudo apt-get install git` (Should already be installed using Amazon Lighsail)
- `$ cd /var/www`
- `$ sudo mkdir catalog`
- `$ sudo chown -R grader:grader catalog`
- `$ cd catalog`
- `$ git clone https://github.com/alaxvong/udacity-p4-item-catalog-application`

---

#### 12. Configure WSGI
- `$ touch catalog.wsgi`
- `$ nano catalog.wsgi`
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from project import app as application
```
- Install __PIP package manager__: `$ sudo apt-get install python-pip`
- Install virtualenv `$ sudo pip install virtualenv`
    - Create virtualenv named __venv__ `$ sudo virtualenv venv`
    - Activate __venv__ `$ source venv/bin/activate`
    - Change virtualenv permission `$ sudo chmod -R 777 venv`
- Configure Apache2 (Be sure to swap out your ServerName) `$ sudo nano /etc/apache2/sites-available/catalog.conf`
```
<VirtualHost *:80>
    ServerName 52.34.171.107
    ServerAlias
    ServerAdmin admin@52.34.171.107
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/project/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/project/static
    <Directory /var/www/catalog/project/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
- `$ sudo a2ensite catalog`
- `$ service apache2 reload` Authenticate file under grader

---

#### 13. Configure PostgreSQL
Install PostgreSQL dependencies
- `$ sudo apt-get install libpq-dev python-dev`
- `$ sudo apt-get install postgresql postgresql-contrib`
- `$ pip install psycopg2`

Switch to __postgres__ user `$ sudo su - postgres`

Configure database with limited permission user __catalog__ 
- `$ psql`
- `# CREATE USER catalog WITH PASSWORD 'catalogpw';`
- `# ALTER USER catalog CREATEDB;`
- `# CREATE DATABASE catalog WITH OWNER catalog;`
- `# \c catalog`
- `# REVOKE ALL ON SCHEMA public FROM public;`
- `# GRANT ALL ON SCHEMA public TO catalog;`
- `# \q`
- `$ exit`

---

#### 14. Setup Project
- Rename cloned __git directory__ to __project__ `$ mv GitRepoName project`
- `$ cd project`
- Install python dependancies to run the application `$ pip install -r requirements.txt`
- rename __your_main_project_file.py__ to __\_\_init\_\_.py__ `$ mv project.py __init__.py`
- Replaced the database engine related lines in the python files with `engine = create_engine('postgresql://catalog:catalogpw@localhost/catalog')` 
    - `$ nano database_setup.py` and `$ nano database_populate.py`
- `$ service apache2 reload`

---
#### References
---

- [DigitalOcean - Create a sudo user on ubuntu ](https://www.digitalocean.com/community/tutorials/how-to-create-a-sudo-user-on-ubuntu-quickstart)
- [DigitalOcean - How to set up SSH Keys](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2)
- [DigitalOcean - Deploy Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
- [PostgreSQL Docs](https://www.postgresql.org/docs/9.5/static/index.html)
