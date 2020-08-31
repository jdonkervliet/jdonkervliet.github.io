---
layout: post
title:  "Setting up a Nexus Artifact Repository"
date:   2020-08-25
tags: software-engineering
---

For my PhD, I work on a project called Opencraft. Opencraft is a research project from the Massivizing Computer Systems group at the VU Amsterdam. As part of this project, we design and develop a large-scale Minecraft-like game, which is also called Opencraft.

Opencraft has tens of software dependencies, which are obtained from external sources by Maven. Although this works well for the majority of the dependencies, there a few dependencies that are more difficult to manage. Specifically, these dependencies introduce two challenges:

1. They exclusively use SNAPSHOT versions, no releases.
2. Only the _x_-most recent versions of these SNAPSHOTS are available from their artifact repository.

Opencraft is a research project, which means multiple people use it to conduct their experiments. This introduces two important requirements:

1. We can compile previous versions of Opencraft. For example, those used in earlier experiments.
2. We support reproducable builds. The Opencraft binary only changes if its own source code changes.

These requirements mean that, once a SNAPSHOT version of a dependency is used to compile Opencraft, that SNAPSHOT may no longer change (Requirement 1). It also means that dependencies must be permanently available, in case an older version of the code needs to be compiled (Requirement 2).

It turns out that we can meet both requirements by using our own artifact repository. The remainder of this post documents how we set up such an artifact repository for the Opencraft research project.

# Setup

I set up the artifact repository on a server running a clean install of Ubuntu Linux. The first task is creating a new user.

```
# adduser jdonkervliet
# usermod -aG sudo jdonkervliet
# su - jdonkervliet
$
```

Having a non-root account is good practice for day-to-day operations, but cumbersome during setup. For the remainder of this post, I assume that commands are run by the root account.

Typically now is a good moment to disable password-based login by setting the following option in `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
```

Don't forget to configure your firewall. In this case, the firewall was already configured and only needed to be turned on.

```
systemctl start iptables
systemctl enable iptables
```

## Docker

We use Docker to reduce the effort needed to maintain and configure the required software.

```
apt update
apt upgrade
apt install docker.io
apt install docker-compose
systemctl start docker
systemctl enable docker

# Allow me to run docker without using 'sudo'.
sudo usermod -aG docker jdonkervliet

# Get nginx's default configuration to use as template.
mkdir nginx
cd nginx
docker run -d --name nginx nginx
docker cp nginx:/etc/nginx/ .
docker container stop nginx
docker container rm nginx
cd ..
```

```docker-compose up -d```
TODO: How to make sure it always restarts?


```
version: '3'

services:
  revproxy:
    container_name: revproxy
    hostname: revproxy
    image: nginx
    restart: always
    ports:
      - 80:80
      - 443:443
    volumes:
      - /path/to/config:/etc/nginx
      - /path/to/keys:/etc/ssl/private
    depends_on:
      - nexus
  nexus:
    container_name: nexus
    hostname: nexus
    image: sonatype/nexus3
    restart: always
    volumes:
      - /path/to/nexus/data:/nexus-data
```

A few things to note:

1. Because Docker Compose by default creates a bridge network over which services can communicate, the nexus service does not need to bind ports to the host.
2. The `restart: always` rule makes sure the services are restarted after reboots and crashes.

## NGINX

```
cd nginx/conf.d
rm default.conf
mkdir sites-available sites-enabled
vim site-available/nexus.conf
```

```
upstream nexus {
  server        nexus:8081;
}

server {
  listen        80;
  server_name   <server domain>;

  location / {
    proxy_pass  http://nexus;
  }
}
```

```
cd sites-enabled
ln -s ../sites-available/nexus.conf .
```

```
cd ../..
vim nginx.conf
```

`include /etc/nginx/conf.d/*.conf;` to `include /etc/nginx/conf.d/sites-enabled/*.conf;`

## Running

After correctly setting the NGINX configuration, the services can be brought to live by running `docker-compose up -d`.

## Maven

```
<repositories>
    <repository>
        <id>opencraft-group</id>
        <url>https://opencraft-vm.labs.vu.nl/repository/opencraft-group/</url>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </repository>
</repositories>

<distributionManagement>
    <repository>
        <id>opencraft-releases</id>
        <url>https://opencraft-vm.labs.vu.nl/repository/opencraft-releases/</url>
    </repository>
</distributionManagement>
```

```
<settings>
	<servers>
		<server>
			<id>opencraft-releases</id>
			<username>admin</username>
			<password>...</password>
		</server>
	</servers>
</settings>
```

You can use a [Maven master password](https://maven.apache.org/guides/mini/guide-encryption.html) to avoid using a plain-text password in your configuration file.

## Result

**Sources**

1. <https://blog.sonatype.com/maxences-technical-corner>
2. <https://hub.docker.com/r/sonatype/nexus3/>
3. <https://stackoverflow.com/questions/36879595/cant-use-nexus-repository-manager-3-0-default-admin-user>
2. <https://www.freecodecamp.org/news/docker-nginx-letsencrypt-easy-secure-reverse-proxy-40165ba3aee2/>
3. <https://blog.sonatype.com/running-the-nexus-platform-behind-nginx-using-docker>
3. <https://stackoverflow.com/questions/36879595/cant-use-nexus-repository-manager-3-0-default-admin-user>
4. <https://github.com/030/n3dr>
5. <https://www.mojohaus.org/versions-maven-plugin/examples/lock-snapshots.html>
6. <https://docs.github.com/en/actions/language-and-framework-guides/publishing-java-packages-with-maven>
