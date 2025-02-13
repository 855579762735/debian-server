# Debian Server
This is not "rootless" Docker. The default arrangement that Docker imposes on users is that the Docker daemon runs as the root user on the system, and so do all the containers within. Rootless Docker is where the actual Docker daemon has no root privileges. The setup I am arranging below is actually the Docker daemon running as root, but the containers are running rootless. In a very simple rundown; rootless Docker protects the system if the Docker daemon itself has flaws, whereas, what I am setting up below protects the system if the container has flaws, but not the daemon. You will have to decide what is best for your threat model.
## 📜 Prerequesites
1. Installing Debian - https://www.debian.org/releases/bookworm/debian-installer/
2. Installing Docker - https://docs.docker.com/engine/install/debian/#install-using-the-repository

## 👤 Setup Container User
We're prefixing `sudo` on the commands below since we're dealing with permissions.
> If you're currently `root` you can skip `sudo`.

Things you should determine yourself .. ( hopefully you just know what I'm doing. )
> containedude = the username I chose for my unprivileged container user, this can be whatever you like.<br>
> myuser = this is your username or it's ID, be sure to use your own username and ID in its place.<br>
> docker_opt = the group name I chose for the shared group between myself and the unprivileged container user.<br>
> /opt = the directory I chose to setup all this stuff.<br>

### Create an Unprivileged User
Create the user that your docker containers will run as, enabled using either `user: ` in docker compose, or `--user` in docker run.
```
sudo useradd -u 1001 -M -s /usr/sbin/nologin containedude
```

### Create a Shared Group
Becomes the shared space betweem you and the new container user, containers can read and write but you're the owner.
```
sudo groupadd docker_opt
```

### Make Directories and set Permissions
Before we connect `containedude` to `docker_opt`, we can make some folders and setup their permissions.<br><br>
Create the directory. `/opt` may already exist.
```
sudo mkdir /opt
```
Change the group of the desired directory.
```
sudo chgrp -R docker_opt /opt
```
Take ownership and set the group of the desired directory.
```
sudo chown -R myuser:docker_opt /opt
```
Permit equal rights for the group and owner of the desired directory.
> The `770` represents the following permissions ..
> <br>7 = rwx = file or foler owner granted read write executable permissions.
> <br>7 = rwx = file or folder group granted read write executable permissions.
> <br>0 = all other users have no permissions.
```
sudo chmod -R 770 /opt
```
Set up the directory so that any newly created items will match these permission settings aforementioned.
```
sudo chmod +s /opt
```
You can examine the directory's permissions once created by ..
```
sudo ls -lR /opt
```

### Add Users to Group
Add ourselves and the unprivileged container user to the group, granting them access while maintaining ownership.<br><br>
Add yourself to the new group.
> Logging out and back in or rebooting the computer is required before this takes affect.
> <br> Don't know who you are? run `whoami` after login.
```
sudo usermod -a -G docker_opt myuser
```
Add the unprivileged container user to the new group.
```
sudo usermod -a -G docker_opt containedude
```

### Applying to Containers and Limitations
Make sure that in your `docker-compose.yaml` or Docker run commands that you include the user environment variable, using `user:` and `--user` respectively. Currently containers that require access to the docker socket will fail running as an unprivileged user. There are also times where the container demands specific permissions on certain files. There are also times where containers will attempt to write to privileged directories and fail if there's no volume setup for those directories, for example `/data`, even if you don't actually need the data stored in those directories on the host you may need to add them to the volumes section.
> **Note about Traefik Reverse Proxy**<br>
> Traefik will refuse to start if it's `acme.json` file has the permissions of 700, to remedy this run `sudo chmod 600 acme.json`. All other files and folders in Traefik do not seem to require this adjustment.

## 📃 Create Docker Compose Files
We can now get started with the fun part of actually creating a docker compose file and adding services! To create the files run the following ..
```
cd /opt && sudo touch docker-compose.yaml && sudo touch .env
```
If you plan on keeping your `.env` and `docker-compose.yaml` files or other sensitive data within this folder, in my example `/opt`, be sure to run the following commands on those files so that the unprivileged user can't access them.
```
sudo chown myuser:myuser docker-compose.yaml && sudo chmod 600 docker-compose.yaml
```
```
sudo chown myuser:myuser .env && sudo chmod 600 .env
```
