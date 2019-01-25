# NGINX Sample

## Summary

The objective of this repo is to have bare minimum sample/explanation of
NGINX web server.
It also comes with a `Vagrantfile` that allow anyone to test it on
Virtual Machine.

The VM used on this repo was tested on `Ubuntu 18.04 64bits` with
`Oracle VirtualBox 5.2.10` and `Vagrant 2.0.2`.

Processes running on port `8000` inside the VM can be accessed on host computer
at `http://localhost:8000` and processes on port `80` can be accessed at
`http://localhost:5000`.


## Installing NGINX

```
git clone git@github.com:vmenezes/nginx_sample.git
cd nginx_sample
vagrant up
```

This created the Ubuntu18.04 VM with port forwarding and run a basic Ansible
playbook that sets the VM timezone to New York and run an `apt-get update`.
Now lets shell on the VM, install and take a look on default settings of
NGINX.

```
vagrant ssh
sudo apt install nginx -y
```

Or you can use the Ansible playbook instead of `apt` to install it as
`ansible-playbook -i "localhost," -c local /vagrant/playbook_install_nginx.yml`.


## Inspecting NGINX default config

Lets start by checking is the service is running with
`sudo systemctl status nginx` and comfirm it is listenning on port `80`
using `sudo netstat -ltnp`.

Ok, now that we know NGINX Systemd service is running on port 80, confirm
everything is fine by opening your brownser at `http://localhost:5000`
(remember we forwarded port `80` to `5000` on the `Vagrantfile`).
A quick `whereis nginx` will give us some hints of where it is installed
and where is the config folder.

```
cd /etc/nginx
ll
```

This is NGINX config. For basic usage we just need to know 3 pieces in here:

- `nginx.conf`= is the global settingsof NGINX(we can leave as is for now)
- `sites-available/`= folder containing one file for each site available
- `sites-enabled/`= folder containing links for sites-available we want enabled

`ll sites-enabled/` show us that only `/etc/nginx/sites-available/default`
is enabled. Scamming thru that site config with `vim sites-available/default`
we can see that most of it is conf but that are few very self-explained
lines that we can take notes as `listen 80 default_server;`,
`root /var/www/html;`, `index index.html index.htm index.nginx-debian.html;`
and `location / {`.

If you check `cat /var/www/html/index.nginx-debian.html` we may realize
it matches the welcome website displayed on port `5000` of `localhost`
of your browser.

As we used `systemctl` to check if NGINX was running earlier, it is clear
the service is being managed by Systemd so lets take a look at it as well.

```
cd /etc/systemd
find . -iname *nginx*
```

So seems like NGINX is wanted by `multi-user` "stage". Digging a bit deeper
with `ll ./system/multi-user.target.wants/nginx.service` we see that it is
actually using a sample Systemd service provided by NGINX installation(we
can use this as a sample if we need to create a custom Systemd NGINX service
as well).

Breakdown of default config:

- Systemd service is a link at
`/etc/systemd/system/multi-user.target.wants/nginx.service` that points to
`/lib/systemd/system/nginx.service`
- General NGINX confit is found at `/etc/nginx/nginx/conf`
- The site running is a link at `/etc/nginx/sites-enabled/default` that
points to `/etc/nginx/sites-available/default`
- `/etc/nginx/sites-available/default` is serving the webpage
`/var/www/html/index.nginx-debian.html`


## Disable default site

```
cd /etc/nginx
sudo rm sites-enabled/default
ll sites-*
```

Even thou we don't have any site enabled now, if we refresh the browser at
`http://localhost:5000` it still serving the page because we didn't reload
NGINX service. Do it with `sudo systemctl restart nginx` and now confirm
the NGINX is not listening on port 80 by running `sudo netstat -ltnp`
and trying to refresh the browser.

This could also be accomplished using the provided Ansible playbook as
`ansible-playbook -c local -i 'localhost,' /vagrant/playbook_disable_default.yml`


## Re-enable default site

To enable the website again we just need to do the reverse. Create the link
on `sites-enabled` to `sites-available/default` and restart the service.

```
cd /etc/nginx
ll sites-*
sudo ln -s /etc/nginx/sites-available/default sites-enabled/default
ll sites-*
sudo systemctl restart nginx
```

This could also be accomplished using the provided Ansible playbook as
`ansible-playbook -c local -i 'localhost,' /vagrant/playbook_enable_default.yml`


## Enable our static site on NGINX

Lets start by deleting the default site and creating a sample HTML page

```
sudo rm /etc/nginx/sites-enabled/default
cd /var/www/
sudo mkdir mysite
cd mysite
sudo touch index.html
echo "<p>Hi, go to <a href=new.html>new site</a></p>" | sudo tee index.html
sudo touch new.html
echo "<p>This is <strong>MY NEW SITE</strong></p>" | sudo tee new.html
```

Now lets create a new site on NGINX pointing to this HTML page at
`http://localhost:5000/`

```
cd /etc/nginx
sudo touch sites-available/mysite
sudo sh -c "cat > sites-available/mysite" << EOL
server {
    listen 80;
    listen [::]:80;
    root /var/www/mysite;
    server_name mysite.com;
}
EOL
```

We can enable this site on NGINX with
`sudo ln -s /etc/nginx/sites-available/mysite sites-enabled/mysite`, check
if NGINX config are still valid using running `sudo nginx -t` and restart
NGINX service with `sudo systemctl restart nginx`.

Notes:

- As no `index` line present on `site-enabled/mysite`, it serves `index.html`
by default
- As no `location` line present on `site-enabled/mysite, it assumes `/`
by default
- If MY_NEW_FILE.html is created on `/var/www/mysite` it can be accessed
thru browser without restarting NGINX at
`http://localhot:5000/MY_NEW_FILE.html`
- Id new folder NEW_DIR created on `/var/www/mysite` with `index.html` within,
it can be accessed without restarting NGINX at
`http://localhot:5000/NEW_DIR`

This could also be accomplished using the provided Ansible playbook as
`ansible-playbook -c local -i 'localhost,' /vagrant/playbook_custom_static.yml`


# Add extra routes to our static site

Sometimes not all files that need to be served by NGINX will be inside
the same folder or even if they are, we may want to give it a custom route.

For example, if we add the file `/var/www/mysite/auth/users/index.html`
it can be seem at `http://localhost:5000/auth/users/` or 
`http://localhost:5000/auth/users/index.html`. If we want this folder to
be routed at `http://localhost:5000/members/` instead, we can just add
a new `location /members/ {}` to our `sites-available/mysite` as below:

```
cd /etc/nginx
sudo sh -c "cat > sites-available/mysite" << EOL
server {
    listen 80;
    listen [::]:80;
    root /var/www/mysite;
    server_name mysite.com;
    location /members/ {
        alias /var/www/mysite/auth/users/;
    }
}
EOL
sudo systemctl restart nginx
```

This could also be accomplished using the provided Ansible playbook as
`ansible-playbook -c local -i 'localhost,' /vagrant/playbook_alias_on_static.yml`


# Flask app served by NGINX

Flask is a very popular Python web framework for development of
web projects and micro-services. Lets see how to create a Flask
Hello Word app, run it local for to ensure it is working and then
serve it thru NGINX. Start by installing some requirements.

```
sudo apt install virtualenv -y
cd /var/www/
sudo mkdir myflasksite
sudo chown vagrant myflasksite
cd myflasksite
virtualenv -p python3 venv
source venv/bin/activate
pip freeze
pip install flask==1.0.2
pip freeze > requirements.txt
```

Now that we have a folder for the project, proper Python virtual environment
with Flask installed and the project `requirements.txt` its time to create
sample app and run it.

```
cat > hello.py << EOL
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
  return 'Hello World!'
EOL
FLASK_APP=hello.py flask run -h 0.0.0.0
```

This leaves the Flask development server running on port `5000` and you can
confirm by opening your browser at `http://localhost:5000` or running on a
local terminal `curl -i -X GET http://localhost:5000`. As the name suggests,
this server isnt intended for production but it is an easy way to comfirm
our Python code is running.

You can press `CTRL c` to stop the development server and lets now setup it
on Systemd with Gunicorn and NGINX.

```
pip install gunicorn==19.9.0
pip freeze > requirements.txt
cat > wsgi.py << EOL
from hello import app

if __name__ == '__main__':
    app.run()
EOL
venv/bin/gunicorn --workers 2 --bind 0.0.0.0:5000 -m 007 wsgi:app
```

This again leaves our small Flask app running on port `5000` but at this time
served by Gunicorn instead of Flask development server. We can again test
with `curl -i -X GET http://localhost:5000` and stop with `CTRL c`.

Now lets create setup Gunicorn as a service on Systemd so that it is always
running on background of our server and make NGINX proxy HTTP requests to it.

```
cd /etc/systemd/system
cat > myapp-gunicorn.service << EOL
[Unit]
Description=MyApp Gunicorn
After=multi-user.target

[Service]
WorkingDirectory=/var/www/myflasksite 
ExecStart=/var/www/myflasksite/venv/bin/gunicorn --workers 2 --bind 0.0.0.0:5000 -m 007 wsgi:app
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOL
sudo systemctl daemon-reload
sudo systemctl status myapp*
sudo systemctl start myapp*
```

Now Gunicorn will always serve our app on port 5000 as soon server boots.
We can see some logs with `sudo journalctl -e` or `sudo tail /var/log/syslog`.
The service can be controlled/monitored thru Systemd with
`sudo systemctl status|start|stop|restart myapp*`.

To have NGINX proxing HTTP requests to Gunicorn just setup NGINX as follows:

```
cd /etc/nginx/
cat > sites-available/myapp-site << EOL
server {
    listen 80;
    location / {
        include proxy_params;
        proxy_pass http://localhost:5000;
    }
}
EOL
sudo rm sites-enabled/*
sudo ln -s /etc/nginx/sites-available/myapp-site sites-enabled/myapp-site
sudo systemctl restart nginx
```

Congrats, now you have Gunicorn serving your app at 0.0.0.0:5000 and
NGINX forwarding HTTP requests from port 80 to your Gunicorn.
(Vagrant then forward port 80 from VM to 8080 of your computer)

Just one thing to get it a bit cleaner, you want HTTP requests to come
thru NGINX but currently Gunicorn is serving it to the public(this was nice
for testing but we should fix it now). Just change `0.0.0.0:5000` to
`127.0.0.1:5000` on `/etc/systemd/myapp-gunicorn.service`. Then
`sudo systemctl daemon-reload` and `sudo systemctl restart myapp*`

