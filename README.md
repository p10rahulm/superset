# Running Superset with Postgresql DB, Redis Caching, Nginx Server and SSL certification
---

## Get a server up on Google Cloud

- Go to [ Google cloud console ](https://console.cloud.google.com/)
- Set it up fully with credentials billing etc. (Currently already setup)
- Click on the top left and click on compute engine
- Click on create instance.
- Choose standard options.
- 1 CPU Ubuntu 16.04 instance should work. Allow HTTP and HTTPS
- The instance should be created.
---

## Basic Machine Setup

- Install git bash on your local system [Git bash download](https://git-scm.com/downloads)
- Setup rsa on system [SSH Key Setup](https://help.github.com/enterprise/2.10/user/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/ )
- On the google cloud compute system, add the public ssh key that you obtained to the system. You can do this by clicking on edit for the instance
- Login for the first time from the browser window
- If you are logged in as Admin, then fine, else create a new user 

```
#Add User
sudo useradd Admin
# Create password etc

# Add sudo privileges to user
sudo usermod -a -G sudo Admin

#change user to Admin
su Admin

#cd into their ssh directory
cd ~/.ssh

#change the authorized keys to add your local key
nano authorized_keys
```

- Make a list of the ssh keys of all users you want to login to the remote system

```
# Add the keys to the authorized_keys file
# A typical key looks as follows
ssh-rsa AAAAB3N.....abcdef My Name <my.name@your_domain.com>
```

- Now log out of the webbrowser terminal
- Login to terminal from git bash. Type 
```
ssh Admin@ip-address-of-remote

#if that doesn't work
ssh -i ~/.ssh/id_rsa Admin@ip-address-of-remote

# Now you should be logged in
```
---

## Getting the first working Superset version up

- Setup the user who will be using the superset
```
#Add User on who's account the superset will run
sudo useradd flaskuser
# Create password etc

# Add sudo privileges to user
sudo usermod -a -G sudo flaskuser

# Change to that user and go to base directory
su flaskuser
cd ~
```
- Add the basic dependencies

```
sudo apt-get install build-essential libssl-dev libffi-dev python-dev python-pip libsasl2-dev libldap2-dev
```

- Install a virtualenvironment

```
# Install virtualenv module
pip install virtualenv

# Create venv a virtual environment named venv
virtualenv venv


# Activate venv
$ . /home/flaskuser/venv/bin/activate

```

- Upgrade pip etc

```
pip install --upgrade setuptools pip

```

- Now do the actual setup and see if it works
- [Follow instructions from here](https://superset.incubator.apache.org/installation.html)

```
# Important note, don't use sudo on any of the below commands

# Install superset
pip install superset

# Create an admin user (you will be prompted to set username, first and last name before setting a password)
fabmanager create-admin --app superset
# I created my_first_name.last_name user here. note the password that you input here

# Initialize the database
superset db upgrade

# Load some data to play with
superset load_examples

# Create default roles and permissions
superset init

# Start the web server on port 80
superset runserver -p 80
```

- Go the browser and type the ip of the instance. This connects by default to the http port. This should give you superset. Login and see that it works

- Now that you have it setup, don't expect to log back into it until some other steps are done
---

## Getting it up on a domain name registrar (I have used godaddy)

- login to [godaddy.com](http://godaddy.in/)
- Setup Account.
- Buy a domain that you like
- Then click on your name on the top right and click on manage my domains
- If using an existing domain, then add an A Host to link to your particular server IP
- If using a new domain, then add @ forwarding to your IP. and then also add www domain forwarding to @
- Once all this is done and takes effect, you should see http://xyz.domain-name.com reflects the front page you saw earlier
---

## Setup some superset essentials

- Take server down by hitting ctrl+C
- In the venv environment check your PYTHONPATH
- [Find your PYTHONPATH](http://www.dummies.com/programming/python/how-to-find-path-information-in-python/)
```
# Open the Python Shell.
$ python

>>> #import sys library
>>> import sys
>>> for p in sys.path:
>>>		print(p)	
>>> # for python 2.7 use 'print p'
>>> exit()
```
- choose one of the paths printed there for saving superset_config.py
- I chose /home/flaskuser/venv/local/lib/python2.7/site-packages/superset_config.py
```
# Create a new config file
nano /home/flaskuser/venv/local/lib/python2.7/site-packages/superset_config.py
```
- The base settings of the file are as follows:

```
#---------------------------------------------------------
# Superset specific config
#---------------------------------------------------------
ROW_LIMIT = 5000
SUPERSET_WORKERS = 4
# You will be serving site on port 8000 from gunicorn which sits in front of flask, and then send to nginx
SUPERSET_WEBSERVER_PORT = 8000
#---------------------------------------------------------

#---------------------------------------------------------
# Flask App Builder configuration
#---------------------------------------------------------
# Your App secret key
SECRET_KEY = '\2\mthisismyscretkey\1\2\a\b\y\h'

# The SQLAlchemy connection string to your database backend
# This connection defines the path to the database that stores your
# superset metadata (slices, connections, tables, dashboards, ...).
# Note that the connection information to connect to the datasources
# you want to explore are managed directly in the web UI
SQLALCHEMY_DATABASE_URI = 'sqlite:////home/flaskuser/.superset/superset.db'

# Flask-WTF flag for CSRF
CSRF_ENABLED = True

# Set this API key to enable Mapbox visualizations
MAPBOX_API_KEY = ''
```

- We will be making several changes to this file as we go about modifying some things
---

## Setting up postgresql

### References before starting:

- [Linode Guide](https://www.linode.com/docs/databases/postgresql/how-to-install-postgresql-on-ubuntu-16-04)
- [Digital Ocean Guide](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-14-04)

### Install basic stuff:

- ensure in venv environment
- pip install psycopg2
```
pip install psycopg2
```

### Get the postgres system up:

- Download postgres
```
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install postgresql postgresql-contrib
```
- Create a postgres user

```
# By default postgres creates a user called postgres
sudo -i -u postgres
# Check postgres working
psql
>>> # Quit
>>> \q
# Get back into user with sudo access
exit
``` 
- Create password for user postgres
```
sudo passwd postgres
# Store somewhere safe
```
- Login to postgres user again

```
su - postgres
# If above doesn't work, try sudo -i -u postgres

# Create a user who will access the database. I am calling this user flaskuser
createuser --pwprompt --interactive
# When it asks whether user is superuser say yes
# Save password securely

# Create a database named superset
createdb superset

# Test the db.
psql superset
superset=# 
#This is working 

# Grant user access
superset=# GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA superset TO flaskuser;

# exit
superset=# \q
```
- Ensure you are still user postgres
- Edit /etc/postgresql/9.5/main/pg_hba.conf
```
nano /etc/postgresql/9.5/main/pg_hba.conf
```
- Change the file as below
```
# "local" is for Unix domain socket connections only
# local    all        all             peer
# change above to 
local    all        all             md5

```
- Return to normal unix shell as flaskuser
```
# exit postgres user
exit

# Restart postgresql service
sudo service postgresql restart
su - postgres

# test the service
psql -U flaskuser -W superset
```

- Check that postgres is working at any time (say after restart)
```
htop
# if command doesn't work, then 'sudo apt-get install htop'
```


### Changing superset_config.py

- Change the superset_config.py to reflect the following db changes:
```
# Change the line
# SQLALCHEMY_DATABASE_URI = 'sqlite:////home/flaskuser/.superset/superset.db'
# to
SQLALCHEMY_DATABASE_URI = 'postgresql+psycopg2://flaskuser:your_password@localhost/superset'
```

### Check whether postgres working

- Now do the following steps to ensure postgres working

```
# Login as flaskuser
sudo su flaskuser

# activate virtualenvironment
cd ~
. ./venv/bin/activate

# Install superset again. This step can be skipped also
pip install superset
# Almost everything shoudl say already installed. Make sure no errors. 

# Initialize the database
superset db upgrade
# This is the important part. Ensure the shell says it's pulling the superset_config file and also running on postgresql. If not go back to previous steps and ensure everything ok

# Load the examples into new database
superset load_examples

# Create default roles and permissions in the new database
superset init

# Start the web server on port 80
superset runserver -p 80
```
- Ensure working from a browser then close/interrupt the server by hitting ctrl+C
---

## Running Redis Caching

### References

- This reference - [Redis Install and configure](https://www.rosehosting.com/blog/how-to-install-configure-and-use-redis-on-ubuntu-16-04/) is comprehensive
- Upgrade apt-get
```
sudo apt-get update
sudo apt-get upgrade
```

- Install the redis server
```
sudo apt-get install redis-server
```

- Change the redis configuration file
```
sudo nano /etc/redis/redis.conf
```
- Add the following lines to redis.conf
```
# 128 MB max memory
maxmemory 128mb
# When mem overflow remove according to LRU algorithm
maxmemory-policy allkeys-lru
```

- Restart and enable redis on reboot
```
# restart redis service
sudo systemctl restart redis-server.service

# enable on reboot
sudo systemctl enable redis-server.service

# ensure redis shows up in htop
htop
#F10 to exit htop
```

### Monitoring & Maintaining Redis

- Monitoring live
```
redis-cli monitor
```
- Flush Cache
```
# Enter the command prompt
redis-cli 

> # In the command prompt flushall to purge cache
> flushall
```

### Install Redis for python

- Reference here: [Redis for Python ](https://pypi.python.org/pypi/redis)
- Ensure you are logged in as flaskuser
```
sudo su flaskuser
```
- Activate environment
```
# goto home
cd ~

# activate venv
. ./venv/bin/activate

```

- Install flask

```
pip install redis
```

### Changing superset_config.py

- Change superset and flask to accept connections
- This reference: [Superset Redis Issue ](https://github.com/ApacheInfra/superset/issues/390)

- Modify the superset_config.py as per follows:
```
#---------------------------------------------------------
# Superset specific config
#---------------------------------------------------------
ROW_LIMIT = 5000
SUPERSET_WORKERS = 4

SUPERSET_WEBSERVER_PORT = 8000
#---------------------------------------------------------

#---------------------------------------------------------
# Flask App Builder configuration
#---------------------------------------------------------
# Your App secret key
SECRET_KEY = '\2\mthisismyscretkey\1\2\a\b\y\h'

# The SQLAlchemy connection string to your database backend
# This connection defines the path to the database that stores your
# superset metadata (slices, connections, tables, dashboards, ...).
# Note that the connection information to connect to the datasources
# you want to explore are managed directly in the web UI
SQLALCHEMY_DATABASE_URI = 'postgresql+psycopg2://flaskuser:your_password@localhost/superset'

# Flask-WTF flag for CSRF
CSRF_ENABLED = True

# Set this API key to enable Mapbox visualizations
MAPBOX_API_KEY = ''

CACHE_DEFAULT_TIMEOUT = 86400
CACHE_CONFIG = {
'CACHE_TYPE': 'redis',
'CACHE_DEFAULT_TIMEOUT': 86400,
'CACHE_KEY_PREFIX': 'superset_',
'CACHE_REDIS_HOST': 'localhost',
'CACHE_REDIS_PORT': 6379,
'CACHE_REDIS_DB': 1,
'CACHE_REDIS_URL': 'redis://localhost:6379/1'
}

```
---

## Getting Mapbox

- This is a good time to get mapbox api key which is the default configured map on superset
- Login to mapbox.com
- Create an account
- Once logged in go to top right and click on account
- Click on API access tokens and get one
- Add this to your superset_config.py
- The should have this line added:

```
# Set this API key to enable Mapbox visualizations
MAPBOX_API_KEY = 'pk.eyJ1IjoicmFodWwtbWFkZHkiLCJhIjoiY2o0YTF2OTBrMHRxcTJxbzZxcm5zbHV5aiJ9.jVLtVWHVGnRJcqsMa8XiMA'

```
---

## Setting up Nginx

- What we are looking for on nginx is to run a reverse proxy. The actual server is gunicorn sitting in front of flask, which is the framework hosting superset
- Some references for this section: [Nginx with Reverse Proxy ](https://www.nginx.com/resources/admin-guide/reverse-proxy/), [nginx.conf for optimized performance](https://www.linode.com/docs/web-servers/nginx/configure-nginx-for-optimized-performance), [ Setting up postgres nginx and gunicorn ](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-debian-8 ),  [How to install nginx ](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-16-04)

 
### Install Nginx

- Install commands
```
# Install
sudo apt-get update
sudo apt-get install nginx

# Start and Reload
sudo nginx -s start
sudo nginx -s reload


# Check status
systemctl status nginx
# It should be running


```
- At this point if you browse to your site, it should show nginx bad gateway.


### nginx conf

- There are two files to change, nginx.conf and the file that it includes in sites-enabled
- Feel free to change the configuration as per requirements of your server
- Open nginx.conf as follows:
```
sudo nano /etc/nginx/nginx.conf
```

- See file below
- If you want to understand what each of these mean go to [Reference ](https://www.linode.com/docs/web-servers/nginx/configure-nginx-for-optimized-performance)

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;

#from linode nginx.config optimizer

events {
    worker_connections 650;
    use epoll;
    multi_accept on;
}

http {
		#from linode nginx.config optimizer
		keepalive_requests 100000;

		sendfile on;
		tcp_nopush on;
		tcp_nodelay on;
		keepalive_timeout 65;
		types_hash_max_size 2048;

		client_header_timeout  3m;
		client_body_timeout    3m;
		send_timeout           3m;

		open_file_cache max=1000 inactive=20s;
		open_file_cache_valid 30s;
		open_file_cache_min_uses 5;
		open_file_cache_errors off;

		gzip on;
		gzip_min_length  1000;
		gzip_buffers     4 4k;
		gzip_types       text/html application/x-javascript text/css application/javascript text/javascript text/plain text/xml applica$
		gzip_disable "MSIE [1-6]\.";

		# [ debug | info | notice | warn | error | crit | alert | emerg ]
		error_log  /var/log/nginx.error_log  warn;

		log_format main      '$remote_addr - $remote_user [$time_local]  '
		  '"$request" $status $bytes_sent '
		  '"$http_referer" "$http_user_agent" '
					'"$gzip_ratio"';

		log_format download  '$remote_addr - $remote_user [$time_local]  '
		  '"$request" $status $bytes_sent '
		  '"$http_referer" "$http_user_agent" '
					'"$http_range" "$sent_http_content_range"';

		map $status $loggable {
			~^[23]  0;
			default 1;
		}

		#-------------from basic conf file-----------

		include /etc/nginx/mime.types;
		default_type application/octet-stream;


		access_log /var/log/nginx/access.log;

		include /etc/nginx/conf.d/*.conf;
		include /etc/nginx/sites-enabled/*;

}


```

- If you notice this includes the files in sites-enabled. For now we should have no sites enabled.
- You can see these files in the sites-enabled folder
```
cd /etc/nginx/sites-enabled/
ls

# If there is anything you can remove as follows
rm file-to-be-removed.conf
```

- Now create a config file in sites-available and hard link it to sites-enabled

```
cd /etc/nginx/sites-available
sudo nano superset.conf
```

- create a really simple config file to start off. We will change this later.
```
server {
        listen   80; 
        server_name your.domain.com; 

        location / {
			proxy_buffers 16 4k;
            proxy_buffer_size 2k;
            proxy_pass http://127.0.0.1:8000;

        }
}
```

- Now hardlink this superset.conf into the sites-enabled folder
```
# Hardlinking files
sudo ln -s /etc/nginx/sites-available/superset.conf /etc/nginx/sites-enabled

# Test for syntax
sudo nginx -t
sudo nginx -s reload
```

- The output should look as follows:

```
#Output
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
``` 

### Test Server

- This is a good time to test whether server works (without the SSL bit)

```
# Login as flaskuser
sudo su flaskuser

# activate virtualenvironment
cd ~
. ./venv/bin/activate

# Start the web server on port 8000 (Note that we have changed the server port)
superset runserver -p 8000
```
- Ensure working from a browser then close/interrupt the server by hitting ctrl+C
- If above is not working, go back and ensure all steps followed
---

## Getting SSL Up

- We will be using Let's encrypt for SSL.
- The references for this section: [Digital Ocean: Getting letsencrypt](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04), [Server Fault answer on how to setup letsencrypt with reverse proxy ](https://serverfault.com/questions/768509/lets-encrypt-with-an-nginx-reverse-proxy/784940#784940?newreg=eb9dc440179b4c97a8eb9c642c377eae), [ Raymii.org strong security](https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html)

### Install certbot

```
# Add the repo
sudo add-apt-repository ppa:certbot/certbot

# Update apt-get
sudo apt-get update

# Install certbot
sudo apt-get install certbot
```

### Obtain certificate

- First some prerequisites:
  * We will be using the 'well-known' application to help us with the 
  * We are using letsencrypt which gives us the certificate for free
  * But it needs to verify our site first. It does this by doing some checks (which it calls challenges) within http://sub.domain.com/.well-known

- Create some folders for nginx to redirect /.well-known requests
```
# Go to below folder. If html directory doesn't exist within www, you can create the same with mkdir command as below
cd /var/www/html/
#make directory .well-known. we will be redirecting nginx requests here.
mkdir .well-known

# Go back to nginx directory
cd /etc/nginx/sites-available
```
- Go to the nginx config file
```
sudo nano /etc/nginx/sites-available/superset.conf
```

- Change it slightly to allow letsencrypt to certify your site
```
server {
    listen 80;
    server_name sub.domain.com www.sub.domain.com;
    […]
	
	
	# Add the below
    location /.well-known {
            alias /var/www/html/.well-known;
    }

    location / {
        # proxy commands go here
        […]
    }
}
```

- Now, you can use the certbot client to request a certificate from Let's Encrypt using the webroot plugin (as root):

```
certbot certonly --webroot -w /var/www/sub.domain.com/ -d sub.domain.com -d www.sub.domain.com
```

- The certificate is now installed in /etc/letsencrypt/live/sub.domain.com/
```
ls /etc/letsencrypt/live/sub.domain.com/
```

- Now establish some strong security
```
#Establish a 2048 bit diffie Helman group Params
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

- sudo nano /etc/nginx/sites-available/superset.conf
```
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_name analysis.atidiv.com;
	return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;

	ssl_certificate /etc/letsencrypt/live/your.domain.com/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/your.domain.com/privkey.pem;

	# from https://cipherli.st/
	# and https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	ssl_prefer_server_ciphers on;
	ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
	ssl_ecdh_curve secp384r1;
	ssl_session_cache shared:SSL:10m;
	ssl_session_tickets off;
	ssl_stapling on;
	ssl_stapling_verify on;
	resolver 8.8.8.8 8.8.4.4 valid=300s;
	resolver_timeout 5s;
	# Disable preloading HSTS for now.  You can use the commented out header line that includes
	# the "preload" directive if you understand the implications.
	#add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
	add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
	add_header X-Frame-Options DENY;
	add_header X-Content-Type-Options nosniff;

	ssl_dhparam /etc/ssl/certs/dhparam.pem;


    # Uncomment this line only after testing in browsers,
    # as it commits you to continuing to serve your site over HTTPS
    # in future
    add_header Strict-Transport-Security "max-age=31536000";

    access_log /var/log/nginx/sub.log combined;

    server_name your.domain.com;

	location /.well-known {
		alias /var/www/html/.well-known;
	}

    location / {
			proxy_buffers 16 4k;
			proxy_buffer_size 2k;
			proxy_pass http://127.0.0.1:8000;
			#from linode nginx optimizer
			proxy_set_header   Host             $host;
			proxy_set_header   X-Real-IP        $remote_addr;
			proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
			proxy_connect_timeout      90;
			proxy_send_timeout         90;
			proxy_read_timeout         90;
			proxy_busy_buffers_size    4k;
			proxy_temp_file_write_size 4k;
			proxy_temp_path            /etc/nginx/proxy_temp;
    }

}

```
- Reload nginx after change
```
sudo nginx -s reload
```

### Renewing

- run the certbot renew command
```
certbot renew --renew-hook "service nginx reload"
```
- Add to crontab
```
# at 4:47am/pm, renew all Let's Encrypt certificates over 60 days old
47 4,16   * * *   root   certbot renew --quiet --renew-hook "service nginx reload"
```
- Dry run he renewal
```
certbot --dry-run renew
```
- Force early renewal to test
```
certbot renew --force-renew --renew-hook "service nginx reload"
```
---

## Run Production Server

- Now we are good to start the server proper
- For this we use the application screen. This helps in persistent sessions. ie those that don't go off when you log off from the bash running the server
- Reference [Persistent SSH Sessions ](https://unix.stackexchange.com/questions/479/keep-ssh-sessions-running-after-disconnection)
- Run screen
```
# Run Screen
screen
>Press enter
> New terminal session

# Change user to flaskuser
sudo su flaskuser

# Go to home directory
cd ~

# Activate virtual environment
. ./venv/bin/activate

# Initialize the database
superset db upgrade
# Ensure everything goes right

# Load some data to play with
superset load_examples
# Ensure everything goes right

# Create default roles and permissions
superset init
# Ensure everything goes right

# Start the web server on port 8000
superset runserver - p 8000

# ctrl a+d to log off screen
# you can type screen -r to log back into screen running server
```

- Now your server is running with redis, SSL, Postgresql and Nginx!
---

## Colours:

- grep search for any the colours on this page [Colours used by superset ](https://github.com/ApacheInfra/superset/blob/master/superset/assets/javascripts/modules/colors.js )
```
sudo grep -rnwl '#7b0051' /home/flaskuser/venv/local/lib/python2.7/site-packages/superset
# Results as follows
/home/flaskuser/venv/local/lib/python2.7/site-packages/superset/data/__pycache__/__init__.cpython-35.pyc
/home/flaskuser/venv/local/lib/python2.7/site-packages/superset/data/__init__.py
/home/flaskuser/venv/local/lib/python2.7/site-packages/superset/data/__init__.pyc
/home/flaskuser/venv/local/lib/python2.7/site-packages/superset/static/assets/dist/explore.f6a614ef75857893cf0e.entry.js
/home/flaskuser/venv/local/lib/python2.7/site-packages/superset/static/assets/dist/dashboard.e444b819184014cf5be3.entry.js
/home/flaskuser/venv/local/lib/python2.7/site-packages/superset/static/assets/javascripts/modules/colors.copy
/home/flaskuser/venv/local/lib/python2.7/site-packages/superset/static/assets/coverage/lcov-report/javascripts/modules/colors.js.html
```
- You can do a sed replace after a grep search for each of the colours in the colour scheme 'bnbcolors' with your own 
- reference: [How to grep and replace ](https://stackoverflow.com/questions/15402770/how-to-grep-and-replace)
```
grep -rl matchstring somedir/ | xargs sed -i 's/string1/string2/g'
```


