# Udacity Linux server configuration project
The goal of the project is to configure a linux server on Amazon Lightsail, update and secure it, setup a database and deploy one of the existing web applications.

## Project details
* **IP**: `52.59.125.2`
* **SSH PORT**: `2200`
* **URL**: [http://ec2-52-59-125-2.eu-central-1.compute.amazonaws.com/](http://ec2-52-59-125-2.eu-central-1.compute.amazonaws.com/)

## Project walkthrough

### Create an Amazon Lightsail instance

### Connecting to the server
* Obtain your keypair from Amazon Lightsail 
  * Go to https://lightsail.aws.amazon.com/ls/webapp/account/profile
  * Click SSH keys
  * Download your key
  * Place it in `~/.ssh` directory

* `ssh ubuntu@public_server_ip -i ~/.ssh/name_of_the_key.pem`
* In my case: `ssh ubuntu@52.59.125.2 -i ~/.ssh/LightsailDefaultPrivateKey-eu-central-1.pem`
### Securing the server
* Update all currently installed packages
  * Updating repository data with `sudo apt-get update`
  * Applying the update with `sudo apt-get upgrade`
* Change the SSH port from 22 to 2200
  * Create a backup of the sshd_config file: `sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config_backup`
  * Edit the port value with your favorite editor: `sudo nano /etc/ssh/sshd_config`
* Configure the UFW to only allow incoming connections for ssh, http and ntp
  * Make sure UFW is inactive while we configure it: `sudo ufw status`
  * Disable all incoming ports by default: `sudo ufw default deny incoming`
  * Enable all outgoing ports: `sudo ufw default allow outgoing`
  * Allow incoming SSH connections: `sudo ufw allow 2200/tcp`
  * Allow incoming HTTP connections: `sudo ufw allow www`
  * Allow incoming NTP connections: `sudo ufw allow ntp`
  * Enable UFW: `sudo ufw enable`

### Give *grader* access
* Create a new user name grader `sudo adduser grader`
* Give grader permission to sudo
  * Create sudoers file from grader: `sudo nano /etc/sudoers.d/grader`
  * Add the following: `grader ALL=(ALL:ALL) ALL`
* Create an SSH key pair for grader using the `ssh-keygen` tool
  * Create a keypair with `ssh-keygen`
  * Make sure you're logged in the server as grader: `su grader`
  * Create a `.ssh` directory: `mkdir .ssh`
  * Create an `authorized_keys` file: `nano ~/.ssh/authorized_keys`
  * Paste in your generated **public** key
### Prepare to deploy your project
* Configure the local timezone to UTC
  * Run `sudo dpkg-reconfigure tzdata`
  * Choose UTC (If it doesn't appear there, select __None of the above__ and you'll see UTC)
* Install and configure Apache to serve a Python mod_wsgi application
  * Install Apache: `sudo apt-get install apache2`
  * Install WSGI: `sudo apt-get install libapache2-mod-wsgi`
  * Make sure git is installed: `sudo apt-get install git`
  * Navigate to `cd /var/www`
  * Clone the catalog app: `sudo git clone https://github.com/raideer/udacity-project-catalog`
  * To install the project dependencies you'll need python pip: `sudo apt-get install python-pip`
  * `cd udacity-project-catalog`
  * Install dependencies: `pip install -r requirements.txt`
  * Create the wsgi app file: `touch app.wsgi`
  * Add the following to the `app.wsgi`:
  ```
  #!/usr/bin/python
  import sys
  sys.path.insert(0, "/var/www/udacity-project-catalog/")

  from project import app as application

  application.secret_key = "asecretkey"
  ```
  * Configure apache to use this wsgi file as our entrypoint: `sudo nano /etc/apache2/sites-enabled/000-default.conf`
  * Right before `</VirtualHost>` closing tag, add: `WSGIScriptAlias / /var/www/udacity-project-catalog/app.wsgi`
  * Restart apache `sudo service apache2 restart`
* Install and configure PostgreSQL
  * Install PostgreSQL `sudo apt-get install postgresql`
  * Move to `cd /etc/postgresql`
  * See what version you have installed by running `ls` (in my case: 9.5) and cd to that directory `cd 9.5`
  * Open up the pg_hba config file: `sudo nano main/pg_nba.conf`
  * Make sure all of these lines ar uncommented to disable remote connections
  ```
  local   all             postgres                                peer
  local   all             all                                     peer
  host    all             all             127.0.0.1/32            md5
  host    all             all             ::1/128                 md5
  ```
* Setting up the database
  * Since only the `postgres` user is allowed to run `psql` we'll have to switch: `sudo su - postgres`
  * Opent the postgresql interactive terminal: `psql`
  * Create db user called catalog: `CREATE USER catalog WITH PASSWORD 'password';`
  * Create a database for the catalog app: `CREATE DATABASE catalog WITH OWNER catalog;`
  * Connect to the created database: `\c catalog`
  * Grant access only to catalog user:
  * `REVOKE ALL ON SCHEMA public FROM public;`
  * `GRANT ALL ON SCHEMA public TO catalog;`;
  * Quit the psql terminal: CTRL-D or CMD-D
  * Jump back to the `ubuntu` user by pressing CTRL-D again
  * Since originally the catalog app ran on sqlite, we'll have to change the database url in the `project.py` file
  * `cd /var/www/udacity-project-catalog`
  * Open up the project.py file `sudo nano project.py`
  * Change this line `app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///db.sqlite'`    
    To `app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://catalog:password@localhost/catalog'`
  * Save and exit
  * Restart apache: `sudo service apache2 restart`
  * Visit the url to see the site in action!
  
  
  
