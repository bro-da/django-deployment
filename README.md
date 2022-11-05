# django-deployment
a simple step by step process for hosting django on any server with out hassle 

## introduction
I made this repo because some of my friends had difficulties deploying django web apps on aws because They had little to no experiance working on cli based system .Most get through (even me) by refering multiple pages and using various hacks but after at the end the deployment They get so overwhemed They forget the entire process so i am creating this repo so that those with difficulty can get the all the commands for deployment right here and as a source for further reference 

## steps

* create a server (ec2 instance works best and is pretty painless to create) 
    * make sure to open the http and https and ssh ports when creating the instance otherwise you have to go tinker on the security to get them to open
    * login to the ec2 instance through the .pem file you received during the instance creation process
* after login when we look to the prompt we could see we are logged in as ubuntu its always better to create a new user and unlock the ssh login through password     so we could login anywhere and login even if we lose our pem keys
* steps to create a user in ubuntu
```shell
sudo adduser USER
```
give the user any name you want

```shell
sudo usermod -aG sudo USER
```
this command helps to give the user USER sudo privilege 

* setup  ufw: This step is important not beacause its important (aws already has a firewall built-in ) but because i see many (even me ) people activate the wfw without allowing ssh and http port through i guess if we activate ufw without fully completing getting ssh connection to the instance requires extra tinkering 
 refer this if it happened to you [ssh not connecting](https://stackoverflow.com/questions/41929267/locked-myself-out-of-ssh-with-ufw-in-ec2-aws).

 ```shell
 sudo ufw allow OpenSSH
 sudo ufw enable 
 ufw status # continue only if you see openssh on the list
 ```

* steps to enable ssh password authentication on
    ```shell
    sudo nano /etc/ssh/sshd_config # find "PasswordAuthentication" enabling it  yes
    sudo service sshd restart
    ```
    only logout after doing this step and rechecking ufw allow list

* logout and reconnect through ssh using new user and password
    ```shell
    ssh USER@IP-ADDRESS
    ```

* update the ubuntu instance and install necessary software
```shell
sudo apt update
sudo apt install python3-venv python3-dev libpq-dev postgresql postgresql-contrib nginx curl # postgresql package and the next steps optional if youre running the postgres data on the same instance
```
* step to install postgresql on the instance
```shell 

sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo -u postgres psql 

```
necessary configureation for postgres

```shell
CREATE DATABASE myproject;
CREATE USER myprojectuser WITH PASSWORD 'password';
ALTER ROLE myprojectuser SET client_encoding TO 'utf8';
ALTER ROLE myprojectuser SET default_transaction_isolation TO 'read committed';
ALTER ROLE myprojectuser SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE myproject TO myprojectuser;
\q
```
pick and choose the commands and the values for user and myprojectuser

* get the django web app source code into the instance using scp or remote repositary (git)

```shell
git clone github-url.com
```

* youre django app has some dependencies to work properly (make sure to create a "requirements.txt" from inside youre env on local computer)
* create new python environment and activate it
```shell
sudo apt install python3-venv
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt # that is if youre requirements.txt is youre list of dependencies
```

* change the database configuration in youre youre-project/settings.py

```python3
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'myproject',
        'USER': 'myprojectuser',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '',
    }
}
```

* check youre django app is working or or has any error

```python3
python3 manage.py runserver
```
* if everything is jolly lets move on to setting nginx and gunicorn setup 


* Creating systemd Socket and Service Files for Gunicorn

    The Gunicorn socket will be created at boot and will listen for connections. When a connection occurs, systemd will automatically start the Gunicorn process to handle the connection.

    Start by creating and opening a systemd socket file for Gunicorn with sudo privileges:
    ```shell
    sudo nano /etc/systemd/system/gunicorn.socket
    ```
    * copy this into it 

    ```
    [Unit]
    Description=gunicorn socket

    [Socket]
    ListenStream=/run/gunicorn.sock

    [Install]
    WantedBy=sockets.target

```

press ctrl-x and y and enter to save and close

* create and open a systemd service file for Gunicorn with sudo privileges in your text editor. The service filename should match the socket filename with the exception of the extension
```shell
sudo nano /etc/systemd/system/gunicorn.service
```
* copy this into it and i have capitalised the parts which you need to change to suit youre needs
```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=USER
Group=www-data
WorkingDirectory=/home/USER/MYPROJECTDIR
ExecStart=/home/USER/MYPROJECTDIR/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          MYPROJECT.wsgi:application

[Install]
WantedBy=multi-user.target

```
* press ctrl-x and y and enter to save and close

run this command to add gunicorn to systemd and run it 
```shell
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```
run this command to troubleshoot 
```shell
sudo systemctl status gunicorn.socket
curl --unix-socket /run/gunicorn.sock localhost # run this to check if youre server is serving
```

### Configure Nginx to Proxy Pass to Gunicorn

Now that Gunicorn is set up, you need to configure Nginx to pass traffic to the process.

```shell
sudo nano /etc/nginx/sites-available/MYPROJECT
```
as always i will capitlaise the keywords you need to change
```
server {
    listen 80;
    server_name SERVER_DOMAIN_OR_IP;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/USER/PROJECTDIR;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```


```shell
sudo ln -s /etc/nginx/sites-available/MYPROJECT/etc/nginx/sites-enabled
```

```shell 
sudo nginx -t  #to check if there is any problem in nginx configuration 
sudo systemctl restart nginx # restart nginx
sudo ufw delete allow 8000
sudo ufw allow 'Nginx Full'
```






* nginx is most likely installed in earlier sudo apt install if not 

```shell
sudo apt install nginx
```

### troubleshooting
* Nginx Is Showing the Default Page Instead of the Django Application
this is most likely as said in the digital ocean 
> If Nginx displays the default page instead of proxying to your application, it usually means that you need to adjust the server_name within the /etc/nginx/sites-available/myproject file to point to your server’s IP address or domain name.Nginx uses the server_name to determine which server block to use to respond to requests. If you receive the default Nginx page, it is a sign that Nginx wasn’t able to match the request to a sever block explicitly, so it’s falling back on the default block defined in /etc/nginx/sites-available/default.The server_name in your project’s server block must be more specific than the one in the default server block to be selected.

what i do personally is mv the default nginx configuration to another place 
```shell
sudo mv /etc/nginx/sites-available/default ~/default
```

* Further Troubleshooting

For additional troubleshooting, the logs can help narrow down root causes. Check each of them in turn and look for messages indicating problem areas.

The following logs may be helpful:

    Check the Nginx process logs by typing: sudo journalctl -u nginx
    Check the Nginx access logs by typing: sudo less /var/log/nginx/access.log
    Check the Nginx error logs by typing: sudo less /var/log/nginx/error.log
    Check the Gunicorn application logs by typing: sudo journalctl -u gunicorn
    Check the Gunicorn socket logs by typing: sudo journalctl -u gunicorn.socket


* restarting the django app
```shell
git pull # after pushing from somewhere else and from the the projectdir location
sudo systemctl restart gunicorn
sudo nginx -t && sudo systemctl restart nginx
```


## references 
[where i copied most of this tutorial check this page for more additional information](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-22-04)
[initial server setup also from digital ocean](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04)