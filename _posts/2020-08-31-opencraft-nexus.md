---
layout: post
title:  "Setting up a Nexus Artifact Repository"
date:   2020-08-31
tags: software-engineering
---

After struggling with sharing Maven artifacts for longer than I want to admit, I finally set up a dedicated artifact repository for Opencraft.

For my PhD, I work on a research project called [Opencraft](https://atlarge-research.com/opencraft/). The goal of Opencraft is to discover and evaluate novel scalability techniques for large-scale online games. As part of this project, we design and develop our own large-scale Minecraft-like game, also called Opencraft.

Opencraft has tens of software dependencies, which are obtained from external sources by [Maven](https://maven.apache.org/). Although this works well for the majority of the dependencies, there a few dependencies that are more difficult to manage. Specifically, these dependencies introduce two challenges:

1. They exclusively use SNAPSHOT versions, no releases.
2. Only the _x_-most recent versions of these SNAPSHOTS are available from their artifact repository.

<span id="requirements">
Opencraft is a research project, which means many students and researchers use it to conduct their experiments. This introduces two important requirements:
</span>

1. We can compile previous versions of Opencraft years after they have been used for experiments.
2. We support reproducible builds. The Opencraft binary only changes if its own source code changes.

These requirements mean that dependencies must be permanently available, in case an older version of the code needs to be compiled (Requirement 1). It also means that, once a SNAPSHOT version of a dependency is used to compile Opencraft, that SNAPSHOT may no longer change (Requirement 2).

It turns out that we can meet both requirements by using our own artifact repository. The remainder of this post discusses how I set up such an artifact repository for Opencraft.

---

# Setup

My setup consists of a Docker Compose file that defines two services: a Sonatype Nexus 3, and an NGINX reverse proxy.
To set up these services correctly, we need to configure NGINX to use HTTPS and function as a reverse proxy for Nexus.

## NGINX

In hindsight, the configuration of the NGINX web server proved to be the majority of the work.
I was happy to find a tutorial for my use-case at <https://www.freecodecamp.org/news/docker-nginx-letsencrypt-easy-secure-reverse-proxy-40165ba3aee2/>. The steps shown in this section closely follow the ones from this tutorial.

The Docker NGINX image comes with a bunch of useful configuration files out of the box.
Unfortunately, these configuration files will no longer be accessible after mounting a host directory at the same location.
Therefore, we first copy the default NGINX configuration to the host by creating an NGINX container and copying its config files.

```
# Get nginx's default configuration to use as a template.
mkdir nginx
cd nginx
docker run -d --name nginx nginx
docker cp nginx:/etc/nginx/ .
docker container stop nginx
docker container rm nginx
```

Mounting a host directory in the Docker container allows me to keep the NGINX configuration on the host's file system.
This approach has two clear benefits. First, it enables updating NGINX without reconfiguration.
Second, it enables me to share the NGINX config files with team members using version control.

Next, we remove an unneeded configuration file, and create two directories to create and enable _sites_.
Each site is a service with a user-accessible web interface.
Currently, we only have one site: the Nexus.
This means that creating the two directories is not necessary for our current setup,
but it allows me to easily add more sites later.

```
cd nginx/conf.d
rm default.conf
mkdir sites-available sites-enabled
```

Creating these directories requires a slight change in `nginx.conf`: `include /etc/nginx/conf.d/*.conf;` becomes `include /etc/nginx/conf.d/sites-enabled/*.conf;`

Each site gets its own configuration file.
In this case, that means creating a configuration file at `site-available/nexus.conf` for the Nexus site:

```
upstream nexus {
  server        nexus:8081;
}

server {
  listen        443 ssl;
  server_name   opencraft-vm.labs.vu.nl;

  include	common.conf;
  include	/etc/nginx/ssl.conf;

  location / {
    proxy_pass  http://nexus;
    include	common_location.conf;
  }
}
```

The `upstream nexus` lets NGINX know that it can find a site called _nexus_ at `nexus:8081`.
The latter `nexus` is interpreted as a domain/host name.
Docker Compose makes sure NGINX will be able to correctly resolve this name to an address.

The `listen` and `server_name` match incoming requests that should be forwarded to the nexus site.

The two included config files are related to HTTPS. `common.conf` prevents users from connecting over plain HTTP:

```
# Only allow httpS connections.
add_header Strict-Transport-Security    "max-age=31536000; includeSubDomains" always;
add_header X-Frame-Options              SAMEORIGIN;
# Don't execute (potentially dangerous) files.
add_header X-Content-Type-Options       nosniff;
add_header X-XSS-Protection             "1; mode=block";
```

`ssl.conf` configures NGINX SSL options:

```
# These settings configure which ciphers and certificates to use.
ssl_protocols               TLSv1 TLSv1.1 TLSv1.2;
ssl_ecdh_curve              secp384r1;
ssl_ciphers                 "ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384 OLD_TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256 OLD_TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256";
ssl_prefer_server_ciphers   on;
ssl_session_timeout         10m;
ssl_session_cache           shared:SSL:10m;
ssl_session_tickets         off;

# Point NGINX to the location of certificates.
ssl_certificate /etc/ssl/private/cert-chain.pem;
ssl_certificate_key /etc/ssl/private/key.pem;
ssl_trusted_certificate /etc/ssl/private/chain-root.pem;
ssl_dhparam /etc/ssl/private/dh.pem;
ssl_stapling on;
ssl_stapling_verify on;

resolver 1.1.1.1 1.0.0.1;
```

The `location` option in `nexus.conf` matches the request path that should be forwarded to the nexus site.
In this case, we redirect requests on the root path (`/`).

Finally, we include a `common_path.conf` to forward information from the original requester when forwarding requests to the nexus site.

```
# Forward information from request to the service.
# Without these settings, the internal service sees all requests as coming from NGINX.
proxy_set_header    X-Real-IP           $remote_addr;
proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
proxy_set_header    X-Forwarded-Proto   $scheme;
proxy_set_header    Host                $host;
proxy_set_header    X-Forwarded-Host    $host;
proxy_set_header    X-Forwarded-Port    $server_port;
```

Because NGINX is acting as a reverse proxy, these headers are needed to give Nexus information about where the requests originate from.

Finally, we enable the site by creating a symbolic link.

```
cd sites-enabled
ln -s ../sites-available/nexus.conf .
```

## Docker Compose

Docker Compose allows us to define an application consisting of multiple services, where each service consists of one or more containers of a certain type.
In our case, it allows combining the Nexus and NGINX services in a single app.

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

This configuration file defines two services: _revproxy_ and _nexus_.
The revproxy service runs an NGINX image, exposes HTTP(S) ports, and mounts two host directories containing the configuration discussed in [the previous section](#NGINX).
The nexus service runs a Sonatype Nexus 3, and mounts a host directory containing its data and configuration files.

A few things to note:

1. Because Docker Compose connects all services to a bridge network by default, and the Nexus will only communicate with the NGINX service, the Nexus service does not need to bind ports to the host. This means the Nexus is only accessible via the NGINX reverse proxy, which is exactly what we want.
2. Despite not allowing plain HTTP requests, binding port 80 allows us to redirect users to HTTPS on port 443.
3. The `restart: always` rule makes sure the services are restarted after reboots and crashes.
4. All configuration is located in the mounted host directories (under _volumes_). This means that we can remove and recreate the containers without having to reconfigure them.

## Starting the Application

After performing the configuration in the previous two sections, we start the services by running `docker-compose up -d`.

## Configuring Nexus

We configure the Nexus via its web interface, which is now accessible via the browser.
Run `docker exec -it nexus cat /nexus-data/admin.password`, or look in the `admin.password` file in the mounted directory on the host, to obtain the default password.
Upon first login, the system will ask for a new password.

We can create new artifact repositories by first clicking the gear icon (![gear icon](/images/gear.png)) at the top left, followed by repositories (![repositories](/images/repositories.png)) in the navigation menu on the left-hand side.
Here we create several Maven repositories for Opencraft:

1. Several _proxy_ repositories, which point to other remote artifact repositories that contain Opencraft dependencies.
1. Two _hosted_ repositories, `opencraft-releases` and `opencraft-snapshots`, where we publish Opencraft artifacts.
2. One _group_ repository, `opencraft-group`, which combines all other repositories and makes them available through a single URL.

The proxy repositories, combined with their caching feature, are what allow us to meet the [requirements](#requirements) we formulated at the start of this post.
Once we obtain a dependency artifact, we need it to be permanently available (Requirement 1) and prevent it from changing (Requirement 2). However, some of these dependencies are only hosted for a limited amount of time, or are published as snapshots.

To solve this, we create a proxy repository for each repository that either hosts dependency artifacts for a limited time, or only makes artifacts available as snapshots.
This way, we only download the dependency artifact once, and keep it cached permanently in our own Nexus.
To prevent cached artifacts from getting overwritten or removed, we need to set the maximum age of cached artifacts to `-1` when creating the proxy repository.

![maximum artifact age](/images/maximum-artifact-age.png)

## Maven

Once the application is running and we have configured our artifact repositories,
we can modify our Maven projects to use our Nexus.
We do this by adding the following code to our `pom.xml`:

```
<repositories>
    <repository>
        <id>opencraft-group</id>
        <url>https://opencraft.labs.vu.nl/repository/opencraft-group</url>
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
    <snapshotRepository>
        <id>opencraft-snapshots</id>
        <url>https://opencraft-vm.labs.vu.nl/repository/opencraft-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```
The `<repository>` tag uses our group repository to let Maven retrieve dependencies through our Nexus.
Specifically, it makes Maven look for direct dependencies of our project in the provided repositories.
However, these dependencies can specify their own repositories in their own `pom.xml` files, which take precedence when downloading _their_ dependencies.
To make sure that we recursively cache all dependency artifacts, we need to make the following addition to our `~/.m2/settings.xml`:

```
<mirrors>
	<mirror>
		<id>opencraft-group</id>
		<url>https://opencraft.labs.vu.nl/repository/opencraft-group</url>
		<mirrorOf>*</mirrorOf>
	</mirror>
</mirrors>
```

This tells Maven that it should use a mirror for all (`*`) repositories, available at `https://opencraft.labs.vu.nl/repository/opencraft-group`. To make this work, we need to make sure that all dependencies can be retrieved via the group repository by adding all required proxy repositories.

An additional benefit of hosting our own artifact repository is the ability to host our own artifacts.
We configure this by adding the `<distributionManagement>` tag.
It instructs Maven to upload our own artifacts to our Nexus when running `mvn deploy`.

When deploying artifacts to our repository, we need to authenticate ourselves.
We do this by adding the following code to `~/.m2/settings.xml`:

```
<settings>
	<servers>
		<server>
			<id>opencraft-releases</id>
			<username>admin</username>
			<password>password</password>
		</server>
		<server>
			<id>opencraft-snapshots</id>
			<username>admin</username>
			<password>password</password>
		</server>
	</servers>
</settings>
```

to avoid using a plain-text password, we can use a [Maven master password](https://maven.apache.org/guides/mini/guide-encryption.html).

## Conclusion

We encountered software build problems because our projects depend on third-party software that exclusively use SNAPSHOT versions and are only temporarily available.
This was problematic for our research project, which requires the ability to compile and run code that can be multiple years old.
Setting up a Nexus and NGINX remote proxy solves these challenges.
Third-party software is now only retrieved once, and then remains permanently cached in our Nexus artifact repository.
This allows all Opencraft team members to build and compile code without the risk of dependencies no longer being available.
As a bonus, we can use the Nexus to publish our own artifacts, making it easier to share internal dependencies.

---

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
