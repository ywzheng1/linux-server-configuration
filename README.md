## Linux Server Configuration  
---
#### Server Details:
IP address: http://3.226.249.31/  
SSH port: 2200

- The Linux distribution is Ubuntu 16.04 LTS.  
- The virtual private server is Amazon Lighsail.  
- The web application is my [Item Catalog project](https://github.com/ywzheng1/item-catalog_dog-breed-group/tree/master).
- The database server is PostgreSQL.

You can visit http://3.226.249.31/ for website deployed.

![](images/application-screenshot.png)

# Configuration steps
---
#### 1.Create an instance in AWS Lightsail  
Go to [AWS Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home/resources) and create a new account / sign in with your account.

Click Create instance and choose Linux/Unix,OS only Ubuntu 16.04LTS

Choose a payment plan (the cheapest plan is enough for now and it's free for first month)

Click Create button to create an instance.

###### :page_with_curl: *Reference:*

ServerPilot, [How to Create a Server on Amazon Lightsail](https://serverpilot.io/community/articles/how-to-create-a-server-on-amazon-lightsail.html)

#### 2. Set up SSH key :key:
Go to account page from your AWS account. You will find your SSH key there.

Download your SSH key, the file name will be like `LightsailDefaultPrivateKey-*.pem`

Navigate to the directory where your file is stored in your terminal.

Run `chmod 600 LightsailDefaultPrivateKey-*.pem` to restrict the file permission.

Change name to `lightsail_key.rsa`.

Run a command `ssh -i lightsail_key.rsa ubuntu@3.226.249.31 in your terminal`, where 3.226.249.31 is the public IP address of the instance.


#### 3. Update and upgrade installed packages
```
sudo apt-get update
sudo apt-get upgrade
```

#### 4. Change the SSH port from 22 to 2200
Edit the /etc/ssh/sshd_config file: `sudo nano /etc/ssh/sshd_config`.  
Change the port number on line 5 from 22 to 2200.  
Save and exit using `CTRL+X` and confirm with Y.  
Restart SSH: `sudo service ssh restart`.  


#### 5. Set up Uncomplicated Fire Wall (UFW)
Configure UDW to allow only incoming request from port2200(SSH), port80 (HTTP) and port123 (NTP).

`sudo ufw status` -- utf should be inactive

`sudo ufw default deny incoming` -- deny all incoming requests

`sudo ufw default deny outgoing` -- allow all outgoing requests

`sudo ufw allow 2200/tcp` -- allow incoming ssh request

`sudo ufw allow 80/tcp` -- allow all http request

`sudo ufw allow 123/udp` -- allow ntp request

`sudo ufw deny 22` -- deny incoming request for port 22

`sudo ufw enable` -- enable ufw

`sudo ufw status` -- check current status of ufw

Go to AWS page and set up relevant ports from `networking` tab.

Click on the `Manage` option of the Amazon Lightsail Instance, then the Networking tab,
![](images/UFW-1.png)
Inside Manage, navigate to Networking tab, and then change the firewall configuration to match the internal firewall settings above.

Allow ports 80(TCP), 123(UDP), and 2200(TCP), and deny the default port 22.
![](images/UFW.png)


#### 6. Create a new user called grader and give an access
Run `udo adduser grader` to create a new user called grader

Create a new directory in sudoer directory with `sudo nano /etc/sudoers.d/grader`

Add `grader ALL=(ALL:ALL) ALL` in nano editor

Run `sudo nano /etc/hosts`

Set SSH keys for grader user with `ssh-keygen` in your local machine.

Copy the generated SSH to a virtual environment.

Run the following command in your virtual environment.

`su - grader` -- switch user

`mkdir .ssh`

`touch .ssh/authorized_keys`

`nano .ssh/authorized_keys` and copy your generated SSH key here.

Reload SSH with `service ssh restart`

Then now you can login grader user.

###### * Disable rootlogin

Open `/etc/ssh/sshd_config` and find PermitRootLogin and change it to no.


#### 7. Set up local time zone
Run `sudo dpkg-reconfigure tzdata` and choose UTC

#### 8. Install Apache application and wsgi module
Run `sudo apt-get install apache2` to install apache

Run `sudo apt-get install python-setuptools libapache2-mod-wsgi` to install mod-wsgi module

Start the server `sudo service apache2 start`

Enter public IP of the Amazon Lightsail instance into browser. If Apache is working, you should see:
![](images/apache2.png)

#### 9. Install git
Run `sudo apt-get install git`

Configure your username and email. git config --global user.name <username> and git config --global user.email <email>

#### 10. Clone your project
Run `cd /var/www and sudo mkdir catalog`

Change the owner to grader `sudo chown -R grader:grader catalog`

Run `sudo chmod catalog` to give a permission to clone the project.

Switch to the catalog directory and clone the Catalog project.

`cd catalog` and `git clone https://github.com/ywzheng1/item-catalog_dog-breed-group/tree/master`

Add catalog.wsgi file by running `sudo nano catalog.wsgi` and add the following code.

```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'secret'
Modify filenames to deploy on AWS.
```
Rename webserver.py to `__init__.py` by running command `mv webserver.py __init__.py`

In `__init__.py` file, replace app.run to following:
```
# app.run(host="0.0.0.0", port=8000, debug=True)
app.run()
```

#### 11. Install virtual environment and Flask framework
First install pip, `sudo apt-get install python-pip`

Run `sudo apt-get install python-virtualenv` to install virtual environment

Create a new virtuall environment with `sudo virtualenv venv` and activate it `source venv/bin/activate`

Change permissions to the viertual environment folder `sudo chmod -R 777 venv`

Install the following dependencies:
```
pip install httplib2
pip install requests
pip install --upgrade oauth2client
pip install sqlalchemy
pip install flask
sudo apt-get install libpq-dev
pip install psycopg2
```

#### 12. Configure Apache
Create a config file `sudo nano /etc/apache2/sites-available/catalog.conf`

Paste the following code
```
<VirtualHost *:80>
    ServerName Your-Public-IP-Address
    ServerAdmin Your-prefrred-email-address
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
Enable the new virtual host `sudo a2ensite catalog`

#### 13. Install and configure PostgressSQL
Run `sudo apt-get install PostgreSQL`

Login to postgress `sudo su - postgres`

After switch user to pstgres, run `psql`

Create a new user `CREATE USER catalog` WITH PASSWORD 'password' by running below command
```
postgres=# CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';
postgres=# ALTER ROLE catalog CREATEDB;
```

Create a DB called 'catalog' with `ALTER USER catalog CREATEDB` and `CREATE DATABASE catalog WITH OWNER catalog`

List the existing roles: `\du`. The output should be like this:
```
                  List of roles
Role name | Attributes                                                 | Member of
----------+------------------------------------------------------------+-----------
catalog   | Create DB                                                  | {}
postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```
Connect to the DB with `\c catalog`  

Change a grand from public to catalog `GRANT ALL ON SCHEMA public TO catalog`

Exit psql: \q.

Switch back to the grader user: `exit`.

Change the engine inside Flask application:
```
engine = create_engine('postgresql://catalog:password@localhost/catalog')
```

###### :page_with_curl: *Reference*

DigitalOcean: [How To Secure PostgreSQL on an Ubuntu VPS.](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)


#### 14. Restart Apache
Run sudo service apache2 restart and check http://3.226.249.31/


---

##### References
Flask document

Digital Ocean

iliketomatoes'repository
