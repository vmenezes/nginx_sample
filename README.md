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

