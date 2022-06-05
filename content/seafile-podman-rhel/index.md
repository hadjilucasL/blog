+++
title = "Deploy Seafile with Podman (in RHEL) behind SWAG & Authelia"
date = 2022-06-05
[taxonomies]
tags = ["linux", "self-hosted", "storage", "podman", "docker"]
+++

> **⚠ WARNING: This post is still under construction.**  
> Hopefully will have time to finish it off soon.
> 
> 
Jubilee weekend here in the UK (yayy 🥳) so I had time to experiment with a couple of file-hosting systems.

I was looking into Nextcloud but browsing through Reddit I saw [Seafile](https://www.seafile.com)
being recommended. So decided to try it out on a test RHEL 8 vm instance.  This post covers both the professional 
and community editions of Seafile.

<!-- more -->

As I recently found out, Docker has been replaced by [Podman](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/building_running_and_managing_containers/index) 
and docker-compose by [podman pods](https://developers.redhat.com/blog/2019/01/15/podman-managing-containers-pods) 
in the Redhat ecosystem.

There are probably several good posts on the advantages of switching from docker to podman but from a quick read 
the pros focus around security, as it allows you to run/debug containers rootless and deamonless. It also plays well with
systemd as we will see later.

The Seafile developers have [provided](https://manual.seafile.com/docker/pro-edition/deploy_seafile_pro_with_docker/) 
several docker compose yaml files so it took some effort and help from [this post](https://forum.seafile.com/t/seafile-with-podman/12815) 
to translate them to podman commands.

The credentials for the Seafile private docker registry can be found [here](https://customer.seafile.com/downloads/) 
at the time of writing. 

Anyway, to business! Here are the commands to pull the docker images and setup a seafile pod with all the necessary containers.
The professional version includes both Elasticsearch, for full text search in files and the [office preview](https://manual.seafile.com/deploy_pro/office_documents_preview/)
component.

Caveat: I didn't have time to properly investigate the most optimal network setup for the containers, so you will see 
a lot of references to localhost (127.0.0.1) instead of referencing containers by name or using unix sockets.

### Pull the docker images
First login to the Seafile private registry and all necessary docker images.
You can try using the latest elasticsearch container but this was the version used in the docker compose yaml 
provided by Seafile.
```bash
podman login -u seafile -p <see-link-above-for-password> docker.seadrive.org

podman pull docker.io/library/mariadb
podman pull docker.io/library/memcached
podman pull docker.io/library/elasticsearch:7.16.2
podman pull docker.io/seafileltd/office-preview
podman pull docker.seadrive.org/seafileltd/seafile-pro-mc
podman pull docker.io/seafileltd/seafile-mc:latest   # community edition image
```

### Create the pod hosting the containers
Then create a pod, named 'seafile', and expose the ports for Seahub (port 8000, the web gui) 
and Seafile file server api (port 8082, used to download/upload files in the gui and desktop clients).
I am exposing these this since I am bypassing the build in nginx server later on as I use my own reverse proxy (SWAG).
```bash
podman pod create -p 65080:8000 -p 65082:8082 --name seafile
```

If you would like to keep their own nginx then you can expose only port 80 (http) or 443 (https) by altering 
the command as follows: 
```bash
podman pod create -p 65080:80 -p 65443:443 --name seafile 
```

### Create mount points for the containers

The last directory will contain also all your files, so you can choose to place this 
somewhere where you have plenty of disk space.

```bash
mkdir -p /opt/seafile-db/mysql
mkdir -p /opt/seafile-elasticsearch/data
mkdir -p /opt/seafile-office-preview
mkdir -p /opt/seafile-data
```


### MariaDB (MySQL)
Now lets create MariaDB container in our 'seafile' pod. 
The volume option 'Z' informs Podman to label the content with a private unshared label. According to the podman docs, 
labeling systems like SELinux  require that proper labels are placed on volume content mounted into a container. 
Without a label, SELinux might prevent the processes running inside the container from using the content.
Lastly the 'U' option tells Podman to use the correct host UID and GID based on the UID and GID within the container, 
to change recursively the owner and group of the source volume.
Without these I would get permission issues within the container but your mileage may vary.
More information on the volume options can be found in the
[podman documentation](https://docs.podman.io/en/latest/markdown/podman-run.1.html#volume-v-source-volume-host-dir-container-dir-options).

```bash
podman create \
  --pod seafile \
  --name seafile-db \
  -e MYSQL_ROOT_PASSWORD='a-secure-password' \
  -e MYSQL_LOG_CONSOLE=true \
  -v /opt/seafile-db/mysql:/var/lib/mysql:Z,U \
  mariadb
```

### Memcached
Create memcached container in our 'seafile' pod and set the server to use 256 megabytes of RAM for item storage.  
```bash
podman create \
  --pod seafile \
  --name seafile-memcached \
  --entrypoint '["memcached", "-m 256"]' \
  memcached
```

### Elasticsearch
Create elasticsearch container and allow it to use up to 2GB of memory. 
The ulimit changes are important as this container can use a lot of memory,
and have open many files at a time.

This is only used in the Professional version.

```bash
podman create \
  --pod seafile \
  --name seafile-elasticsearch \
  --memory 2g \
   --ulimit nofile=65535:65535,memlock=-1:-1 \
  -e discovery.type=single-node \
  -e bootstrap.memory_lock=true \
  -e "ES_JAVA_OPTS=-Xms1g -Xmx1g" \
  -v /opt/seafile-elasticsearch/data:/usr/share/elasticsearch/data \
  elasticsearch:7.16.2
```

To get this container working I also had to change the owner of the source volume directory to
the user with id 1000. I think this has something to do with the way the container sets the UID/GID.. but
you can try to mount with the 'U' option as used in the MariaDB container. I didn't have time to try that.
```bash
chown -R 1000.1000 /opt/seafile-elasticsearch
```

#### Note
At one point I also tweaked the system resource limits for different users, 
but I do not think this made a difference - especially after it is deployed as a service. 
It was just me randomly copy-pasting from the web.
Nevertheless, it's here for completeness.

Open `/etc/security/limits.conf` and append at the end
```text
*                soft    nofile         1024000
*                hard    nofile         1024000
*                soft    memlock        unlimited
*                hard    memlock        unlimited
elastic          soft    nofile         1024000
elastic          hard    nofile         1024000
elastic          soft    memlock        unlimited
elastic          hard    memlock        unlimited
root             soft    nofile         1024000
root             hard    nofile         1024000
root             soft    memlock        unlimited
```

### Office Preview 
Seafile Professional supports previewing office documents, such as .doc and .xlx, in the webgui by converting them to PDF.
To support this you need to setup the office preview container. The container needs to expose it's service, on port 8089,
so that it can be reached by the main Seafile container.
```bash
podman create \
  --pod seafile \
  --name seafile-office-preview \
  -v /opt/seafile-office-preview:/shared:U \
  --entrypoint '["bash", "start.sh"]' \
  --publish 8089:8089 \
  seafileltd/office-preview:latest
```

### Seafile Container
Now for the final step, let's create our Seafile container in seafile pod.
For a list of tz database time zones see [this helpful wikipedia page](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

Make sure that the source volume, in this case `/opt/seafile-data`, is on a location that has enough space for your files.

```bash
podman create \
  --pod seafile \
  --name seafile-seafile \
  -v /opt/seafile-data:/shared:U \
  -e DB_HOST=127.0.0.1 \
  -e DB_ROOT_PASSWD='a-secure-password' \
  -e TIME_ZONE='Europe/London' \
  -e SEAFILE_ADMIN_EMAIL='your-email@your-domain.com' \
  -e SEAFILE_ADMIN_PASSWORD='b-secure-password' \
  -e SEAFILE_SERVER_LETSENCRYPT=false \
  -e SEAFILE_SERVER_HOSTNAME='my-seafile.my-domain.com' \
  docker.seadrive.org/seafileltd/seafile-pro-mc
```

For the community edition you can simply replace the last line with `seafileltd/seafile-mc`.

### Start the pod
Phew! Hopefully all commands run without a hiccup, and we are ready to launch.

First start the containers supporting the main seafile service (seafile-seafile container). To be honest I also launched
them all at once and didn't really have an issue - but this way you can see if any of the supporting ones have any issues.

```bash
podman start seafile-memcached seafile-db seafile-elasticsearch
```

If no issues then start the main seafile service 
```bash
podman start seafile-seafile
```

### Deploy with systemd

```bash
mkdir ~/seafile && cd ~/seafile
podman generate systemd --new --files --name seafile
mv *.service /etc/systemd/system/
```

```
container-seafile-db.service
container-seafile-elasticsearch.service
container-seafile-memcached.service
container-seafile-office-preview.service
container-seafile-seafile.service
pod-seafile.service
```

### Allow binding to Seahub/Seafile file server from outside the container

`/opt/seafile-data/seafile/conf/gunicorn.conf.py`

Change the line:
```text
bind = "127.0.0.1:8000"  (could also be localhost:8000)
```

to

```text
bind = "0.0.0.0:8000"
```

### Elasticsearch configuration

`/opt/seafile-data/seafile/conf/seafevents.conf`

```text
[INDEX FILES]
external_es_server = true
es_host = 127.0.0.1
es_port = 9200
enabled = true
interval = 10m
```


### LDAP Configuration

In this case I am using FreeIPA but you can change the BASE/USER_DN per your configuration.

`/opt/seafile-data/seafile/conf/ccnet.conf`

```text
[LDAP]
HOST = ldap://ldap.my-domain.com:389
BASE = cn=users,cn=accounts,dc=my-domain,dc=com
USER_DN = uid=admin,cn=users,cn=accounts,dc=my-domain,dc=com
PASSWORD = c-secure-password
LOGIN_ATTR = mail
```

### Seahub configuration

`/opt/seafile-data/seafile/conf/seahub_settings.py`

```text
SERVICE_URL = "https://my-seafile.my-domain.com/"  <-- change this to match your domain

DATABASES = {
 < Do not change anything here >
}


CACHES = {
    'default': {
        'BACKEND': 'django_pylibmc.memcached.PyLibMCCache',
        'LOCATION': '127.0.0.1:11211', <-- change this to point to 127.0.0.1
    },
    'locmem': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
    },
}
COMPRESS_CACHE_BACKEND = 'locmem' 
TIME_ZONE = 'Europe/London'
FILE_SERVER_ROOT = "https://my-seafile.my-domain.com/seafhttp"  <-- change this to match your domain
OFFICE_CONVERTOR_ROOT = 'http://127.0.0.1:8089' <-- change this to point to 127.0.0.1

# Needed for authentication with Authelia to work
ENABLE_REMOTE_USER_AUTHENTICATION = True
REMOTE_USER_HEADER = 'HTTP_REMOTE_USER'
REMOTE_USER_CREATE_UNKNOWN_USER = True
REMOTE_USER_DOMAIN = 'my-domain.com'
REMOTE_USER_ACTIVATE_USER_AFTER_CREATION = True
```

### Authelia Rule Configuration

> **⚠ WARNING: Not fully resolved but working.**  
> 
> After you authenticate with Authelia, you still get the Seafile login page.
> 
> Just click the 'Single Sign-On' link on the login page and you will be logged in automatically. 

Bypass the file server api for the desktop client to work properly.

`/config/configuration.yml`

```yaml
  rules:
    - domain: my-seafile.my-domain.com
      resources:
        - "^/seafhttp/.*$"
        - "^/api.*$"
      policy: bypass
    - domain: my-seafile.my-domain.com
      policy: one_factor
```

### SWAG Nginx configuration

To route https://my-seafile.my-domain.com to the correct destination.

`/config/nginx/proxy-confs/seafile.subdomain.conf`

```text
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name my-seafile.*;

    include /config/nginx/ssl.conf;

    client_max_body_size 0;

    # enable for Authelia
    include /config/nginx/authelia-server.conf;

    location / {
        # enable for Authelia
        include /config/nginx/authelia-location.conf;

        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        
        set $upstream_app your-server-ip-address;               <--- need to set this
        set $upstream_port 65080;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;

    }

    location /seafhttp {        
        rewrite ^/seafhttp(.*)$ $1 break;
        
        include /config/nginx/authelia-location.conf;
        
        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        
        proxy_pass http://your-server-ip-address:65082;            <--- need to set this
   }

}
```

### Setup the DNS record for the subdomain

CNAME record to point `my-seafile.my-domain.com` subdomain to `my-domain.com`, if `my-domain.com` is where your reverse proxy (SWAG) is hosted on. 

## General commands

#### Open a shell in a running container by name
```bash
podman exec -it seafile-office-preview /bin/bash
```
