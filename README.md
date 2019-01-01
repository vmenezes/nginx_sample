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


