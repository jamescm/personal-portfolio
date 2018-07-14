---
title: 'Making web apps with Docker Compose, part 2'
tags:
  - tutorial
  - docker
  - docker compose
  - javascript
  - node
  - development
  - software
  - security
date: 2018-07-05 04:53:30
---

In my [previous post](/personal-portfolio/2018/06/29/making-web-apps-with-compose) we discussed developing a web app with Docker Compose. We neglected to cover some important points about going from development to production, however.

I'm no security expert. I've deployed numerous production apps but that doesn't qualify me to speak as an expert. I'm going to share some things I've done to secure my apps but don't consider this guide comprehensive by any means!

# Principle of Least Privilege
## Adding a user to our container
A good starting point will be to employ the principle of least privilege. Let's add a user to our Dockerfile to run our app with.

```Dockerfile
FROM node:10.5.0-alpine
RUN adduser -D -s /bin/false app
ENV HOME=/home/app
COPY package*.json /$HOME/
WORKDIR /$HOME
USER app
RUN npm i --production --quiet
```

Rebuild the container and make sure the logged in user is `app`.

```sh
$ docker-compose exec web whoami
app
```
The `whoami` command responded with `app`; perfect! This is a pretty easy step forward to locking down our containers. 

## User namespaces

To quote [the relevant Docker documentation](https://docs.docker.com/engine/security/userns-remap) which I highly recommend reading:

> Linux namespaces provide isolation for running processes, limiting their access to system resources without the running process being aware of the limitations.
> The best way to prevent privilege-escalation attacks from within a container is to configure your container's applications to run as unprivileged users. For containers whose processes must run as the `root` user within the container, you can re-map this user to a less-privileged user on the Docker host.

Let's exploit a docker environment that isn't using user namespaces.

```sh
$ groups
user adm cdrom sudo docker
$ sudo touch /test.file # Let's make a fake "root" file that we want to access
$ ls -l /test.file
-rw-r--r-- 1 root root 0 Jun 30 02:32 /test.file
$ docker run -v /test.file:/test.file alpine ash -c "echo Uh oh > /test.file"
$ cat /test.file
Uh oh
```

It's possible to escalate user privileges to root via Docker with very little effort. Imagine how much worse this could be by mapping in the entire host filesystem. Let's make a quick change to enable default user namespacing with Docker.

```sh
# Change the daemon.json config to use default user namespace settings
$ sudo sh -c 'sudo echo "{ \"userns-remap\": \"default\" }" > /etc/docker/daemon.json'
$ sudo service docker restart
# Let's confirm that docker has created our remap user
# Unless defined in the userns-remap value, the default user is dockremap
$ id dockremap
uid=116(dockremap) gid=127(dockremap) groups=127(dockremap)
$ grep dockremap /etc/subuid
dockremap:362144:65536
$ grep dockremap /etc/subgid
dockremap:362144:65536
$ docker images # No more images from the previous namespace!
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```
Now everytime we create a new container, the container's root user will be mapped to the host user `dockremap`. Let's try our escalation attack again.

```sh
$ groups # The following user is in the docker group; not good
admin adm cdrom sudo docker
$ ls / # Let's modify /test.file again
bin  boot  cdrom  dev  etc  home lib media  mnt  opt  proc  root  run  sbin  srv  sys  test.file  tmp  usr  var  vmlinuz
$ docker run -v /test.file:/test.file alpine sh -c "echo Uh oh > /test.file"
ash: can't create test.file: Permission denied
```

**Note: This will disallow the anonymous voluming trick that allows us to build node_modules into our images!** We will address this in the next article.

## Removing the `docker` group

There is a common security flaw when setting up the Docker daemon that tends to be a newbie trap: the `docker` group.

```sh
$ groups
user adm cdrom sudo docker
```

If any user accounts belong to the `docker` group then they are highly vulnerable to exploitation. This group allows the user to execute Docker daemon commands without needing to escalate privileges to root. To demonstrate this vulnerability, let's try the previous attack once more with a simple change to disable namespacing.

```sh
$ docker run --userns=host -v /test.file:/test.file alpine ash -c "echo Not good > /test.file"
$ cat /test.file
Not good
```

Docker works under the assumption that only trusted users are given access to the daemon process. Adding a user to the `docker` group tells Docker that their account is to be trusted without requiring a password. If that account is ever compromised then all the privileges that Docker gives are available to the attacker.

The quickest and easiest way to eliminate this vulnerability is to *remove the `docker` group from any and all users*. [Read about it in Docker's security documentation](https://docs.docker.com/engine/security/security#docker-daemon-attack-surface).

> First of all, only trusted users should be allowed to control your Docker daemon. This is a direct consequence of some powerful Docker features. Specifically, Docker allows you to share a directory between the Docker host and guest container; and it allows you to do so without limiting the access rights of the container.
> This means you can start a container where the `/host` directory is the `/` directory on your host; and the container can alter your host filesystem without any restriction.

Enforce using `sudo` to escalate privileges to root in order to use Docker if you're in a production environment. [Don't use the `docker` group](https://fosterelli.co/privilege-escalation-via-docker.html).


In the next chapter, we'll rework our app to expect failure and improve resiliency.

## Read more

Part 1: [Docker Compose and Node: Getting started](/personal-portfolio/2018/06/29/making-web-apps-with-compose)
Part 2: Development to Production: Security
Part 3: Development to Production: Availability