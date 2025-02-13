# Debian Server
This is not "rootless" Docker. The default arrangement that Docker imposes on users is that the Docker daemon runs as the root user on the system, and so do all the containers within. Rootless Docker is where the actual Docker daemon has no root privileges. The setup I am arranging below is actually the Docker daemon running as root, but the containers are running rootless. In a very simple rundown; rootless Docker protects the system if the Docker daemon itself has flaws, whereas, what I am setting up below protects the system if the container has flaws, but not the daemon. You will have to decide what is best for your threat model.

## Installing Debian

## Installing Docker

## Setup Container User
We're prefixing `sudo` on the commands below since we're dealing with permissions. If you're currently `root` you can skip `sudo`.

### Create an Unprivileged User
Create the user that your docker containers will run as, enabled using either `user: ` in docker compose, or `--user` in docker run commands to spin up the containers. Note that not declaring a user in this way when starting containers will defaultly run them as the root user. You can pick a fun name for this user, I chose `containedude`.
```
sudo useradd -u 1001 -M -s /usr/sbin/nologin containedude
```

### Create a Shared Group
This group will become the shared space betweem you and the new container user, allowing containers to use their configs and directories while still maintaining you as the owner. Once again the actual name here doesn't matter, feel free to get creative and make it memorable. Just be careful the group doesn't already exist, an output of `sudo groups`.
```
sudo groupadd docker_opt
```

### Make Directories and set Permissions
Before we connect `containedude` to `docker_opt`, we can make some folders and setup their permissions. The folder I am creating below exists already in most linux setups, so then you'd just have to modifiy its permissions. You don't have to use the directory I've chose below. Replace `myuser` with your username, an output of command `whoami`.<br><br>
Creates the directory.
```
sudo mkdir /opt
```
Changes the group of the directory.
```
sudo chgrp -R docker_opt /opt
```
Takes ownership and sets group of the directory.
```
sudo chown -R myuser:docker_opt /opt
```
Permits equal rights for group and owner of the directory.
> The `770` represents the following permissions ..
> <br>7 = rwx = file or foler owner granted read write executable permissions.
> <br>7 = rwx = file or folder group granted read write executable permissions.
> <br>0 = all other users have no permissions.
```
sudo chmod -R 770 /opt
```
Sets up the directory so that any newly created items will match these permission settings.
```
sudo chmod +s /opt
```

### Add Users to Group
Once everything is ready we can add ourselves and the unprivileged container user to the group, effectively granting them access while maintaining ownership.<br><br>
Add yourself to the new user group.
> Logging out and back in or rebooting the computer is required before this takes affect.
> <br> Don't know who you are? run `whoami` after login.
```
sudo usermod -a -G docker_opt myuser
```
Add the unprivileged container user to the new group.
```
sudo usermod -a -G docker_opt containedude
```

### Applying to Containers
Make sure that in your `docker-compose.yaml` or Docker run commands that you include the user environment variable, using `user:` and `--user` respectively.
