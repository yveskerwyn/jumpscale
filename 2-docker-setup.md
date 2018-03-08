# Docker setup

- [Create and publish Docker images](#create-images)
- [Create Docker containers](#create-containers)

<a id="create-images"></a>
## Create the Docker images

On a (virtual) machine with JumpScale installed execute the following.

Set variables:
```python

branch = "9.2.1"
tag="9.2.1"
ays_server_container_name = "ays-server"
ays_portal_container_name = "ays-portal"
ays_server_mounted_volume_path = "/opt/var/data/ays-server"
ays_portal_mounted_volume_path = "/opt/var/data/ays-portal"
```

Make sure you have installed ParallelSSH - still needed?:
```python
j.tools.prefab.local.runtimes.pip.install(package="parallel-ssh")
```

Check if Docker is installed
```python
j.tools.prefab.local.virtualization.docker.isInstalled()
```

If not yet installed first install Docker:
```python
#docker_branch = "17.12"
#j.tools.prefab.local.virtualization.docker.install(branch=docker_branch)
j.tools.prefab.local.virtualization.docker.install()
```

As a test, get the Ubuntu Docker image:
```python
import docker
docker_client = docker.from_env()
docker_client.images.pull(repository="ubuntu", tag="16.04")
```

In what follows we will create a Docker image for:
- [JumpScale 9.2.1](jumpscale-image)
- [AYS Server](#ays-server)
- [AYS Portal](#ays-portal)

<a id="jumpscale"></a>
### Create a JumpScale 9.2.1 image

Options:
- [ZUtils](#jumpscale-zutils)
- [Dockerfile](#jumpscale-dockerfile)
- [JumpScale](#jumpscale-jumpscale)


<a id="jumpscale-zutils"></a>
#### Create a JumpScale image using a the ZUtils

If not already installed, get the ZUtils:
```bash
export ZUTILSBRANCH=9.2.1
curl https://raw.githubusercontent.com/Jumpscale/bash/${ZUTILSBRANCH}/install.sh?$RANDOM > /tmp/install.sh
bash /tmp/install.sh
. ~/.bash_profile
```

Create the image:
```bash
export JS9BRANCH=9.2.1
ZInstall_js9_full
```

This will create following images:

```bash
jumpscale/js9_full
jumpscale/ubuntu_python
jumpscale/ubuntu
```

For more information about the ZUtils see https://github.com/Jumpscale/bash.


<a id="jumpscale-dockerfile"></a>
#### Create a JumpScale image using a Dockerfile

> Only for ays9 we run `install.sh` after running `pip3 install -e .` (which runs `setup.py`), it's interesting to check the `install.sh` in each of repositories in order to understand what you potentially miss when only running `pip3 install -e .`

First let's create an image which adds Python to a base Ubuntu 16.04 image, using following Dockerfile:
```bash
mkdir -p /tmp/Dockerfiles/ubuntu_python
cd /tmp/Dockerfiles/ubuntu_python
vim Dockerfile
```

Here's the Dockerfile:
```Dockerfile
FROM ubuntu:16.04
MAINTAINER Yves Kerwyn
RUN apt-get update
RUN apt-get install -y git \
                       curl \
                       mc \
                       openssh-server \
                       net-tools \
                       iproute2 \
                       tmux \
                       localehelper \ 
                       psmisc \
                       python3
                       
RUN curl -sk https://bootstrap.pypa.io/get-pip.py > /tmp/get-pip.py
RUN python3 /tmp/get-pip.py
RUN pip3 install --upgrade pip
RUN pip3 install tmuxp
RUN pip3 install gitpython
```

Build the image:
```bash
export docker_hub_username="yveskerwyn"
docker build --build-arg docker_hub_username=$docker_hub_username --tag $docker_hub_username/ubuntu_python .
```

Create another image now adding JumpScale core9, lib9 and prefab9 to the `jumpscale/ubuntu_python` image:
```bash
mkdir -p /tmp/Dockerfiles/js9_full
cd /tmp/Dockerfiles/js9_full
vim Dockerfile
```

Here's the Dockerfile:
```Dockerfile
ARG docker_hub_username="jumpscale"
FROM $docker_hub_username/ubuntu_python
MAINTAINER Yves Kerwyn

RUN apt-get install -y build-essential \
                       python3-dev \
                       libvirt-dev \
                       libssl-dev \
                       libffi-dev \
                       libssh-dev \
                       sqlite3 \
                       libsqlite3-dev 

RUN pip3 install Cython>=0.25.2 \
                 asyncssh>=1.9.0 \
                 numpy>=1.12.1 \
                 tarantool>=0.5.4 \
                 python-jose

#RUN ssh-keyscan -t rsa github.com >> /root/.ssh/known_hosts

RUN mkdir -p /opt/code/github/jumpscale 
WORKDIR /opt/code/github/jumpscale

RUN git clone -b 9.2.1 https://github.com/Jumpscale/core9.git
RUN git clone -b 9.2.1 https://github.com/Jumpscale/lib9.git
RUN git clone -b 9.2.1 https://github.com/Jumpscale/prefab9.git

RUN cp /opt/code/github/jumpscale/core9/mascot /root/.mascot.txt

RUN pip3 install -e /opt/code/github/jumpscale/core9
RUN pip3 install -e /opt/code/github/jumpscale/lib9
RUN pip3 install -e /opt/code/github/jumpscale/prefab9

#ENTRYPOINT ["js9"]
```

> Note that the `pip3 install` steps will only execute the `setup.py`, so the `install.sh` scripts are not run. 

Build the image:
```bash
docker build --build-arg docker_hub_username=$docker_hub_username --tag $docker_hub_username/js9_full:9.2.1 .
```

Test the image:
```bash
export iyo_organization="ays-organizations.docker-on-mac"
docker run -it --name js9-test -p "5000:5000" -e organization=$iyo_organization $docker_hub_username/js9_full:9.2.1 bash
```

<a id="jumpscale-jumpscale"></a>
#### Create JumpScale image using JumpScale

Create and start a container (keeping it alive for 1 hour):
```python
retval = j.sal.docker.client.api.create_container(image="ubuntu:16.04", name="jumpscale", command="sleep 3600")
js_container = j.sal.docker.client.containers.get(container_id="jumpscale")
js_container.start()
```

Get prefab access to the container:
```python
executor = j.tools.executor.getLocalDocker(container_id_or_name="jumpscale")
prefab = j.tools.prefab.get(executor=executor)
```

Update Ubuntu apt definition:
```python
prefab.system.package.mdupdate()
```

Install dependencies:
```python
prefab.system.package.install("python3-dev,git,curl,language-pack-en")
```

Install JumpScale:
```python
prefab.js9.js9core.install(branch=branch, full_installation=True)
```

Commit all changes:
```python
js_container.commit(repository="jumpscale/full", tag=tag)
```

Stop and remove the container:
```python
js_container.stop()
js_container.remove()
```

Makes sure you are owner of the JumpScale organization on Docker Hub and that you are logged in, if not the case login first:
```bash
ctrl+z
docker login
fg
```

Back in the interactive shell, publish the image - this is take a while:
```python
j.sal.docker.client.images.push("jumpscale/full", tag)
```

<a id="ays-server"></a>
### Create an AYS Server image

Options:
- [ZUtils](#ays-zutils)
- [Dockerfile](#ays-dockerfile)
- [JumpScale](#ays-jumpscale)

<a id="ays-zutils"></a>
#### Create a JumpScale image using a the ZUtils

If not already installed, get the ZUtils:
```bash
export ZUTILSBRANCH=9.2.1
curl https://raw.githubusercontent.com/Jumpscale/bash/${ZUTILSBRANCH}/install.sh?$RANDOM > /tmp/install.sh
bash /tmp/install.sh
. ~/.bash_profile
```

Create the image:
```bash
export JS9BRANCH=9.2.1
ZInstall_ays9
```

This will create following image:
```bash
jumpscale/ays9
```

And if not already created previously, it will first create the following images:
```bash
jumpscale/js9_full
jumpscale/ubuntu_python
jumpscale/ubuntu
```

For more information about the ZUtils see https://github.com/Jumpscale/bash.


<a id="ays-dockerfile"></a>
#### Create an AYS Server image using a Dockerfile

This Dockerfile will next to adding AYS to the `jumpscale/js9_full` image, also include an entrypoint to start the AYS server.

Let's first create an ENTRYPOINT script:
```bash
mkdir -p /tmp/Dockerfiles/js9_ays
cd /tmp/Dockerfiles/js9_ays
vim docker-entrypoint.sh
```

Here's the script: 
```bash
#!/bin/bash
set -e
js9 'j.clients.redis.start4core()'

if [ ! -z "$organization" ] ; then
    js9 'import os; org=os.environ["organization"]; j.tools.prefab.local.js9.atyourservice.configure(production=True, organization=org, restart=False, host="0.0.0.0", port=5000)'
fi
if [ ! -z "$external_ip_address" ] ; then
    js9 'import os; external_ip_address = os.environ["external_ip_address"]; public_ays_url = "http://{}:5000".format(external_ip_address); j.tools.prefab.local.js9.atyourservice.configure_api_console(url=public_ays_url)'
fi
js9 'j.tools.prefab.local.js9.atyourservice.start()'
tail -f /opt/var/log/jumpscale.log
```

Don't forget:
```bash
chmod +x ./docker-entrypoint.sh
```

Create the Dockerfile:
```bash
vim Dockerfile
```

Here's the Dockerfile:
```Dockerfile
ARG docker_hub_username="jumpscale"
FROM $docker_hub_username/js9_full:9.2.1
MAINTAINER Yves Kerwyn

RUN git clone -b 9.2.1 https://github.com/Jumpscale/ays9.git

RUN pip3 install -e /opt/code/github/jumpscale/ays9
RUN /opt/code/github/jumpscale/ays9/install.sh

EXPOSE 5000
VOLUME ["/root/js9host"]
COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
```

> Note we also run `install.sh` next to running `pip3 install -e .` (which runs `setup.py`), we only do this for the AYS repository; it's interesting to check the `install.sh` in each of repositories in order to understand what you potentially miss when only running `pip3 install -e .`, for AYS we need it in order to install the AYS server.

Build the image:
```bash
docker build --build-arg docker_hub_username=$docker_hub_username --tag $docker_hub_username/js9_ays:9.2.1 .
```

<a id="ays-jumpscale"></a>
#### Create an AYS Server image using JumpScale

Create and start a container (keeping it alive for 1 hour):
```python
host_config = j.sal.docker.client.api.create_host_config(port_bindings={5000: 5000})
retval = j.sal.docker.client.api.create_container(image="jumpscale/full:9.2.1", name=ays_server_container_name, command="sleep 3600", ports=[5000], host_config=host_config)
#retval = j.sal.docker.client.api.create_container(image="jumpscale/full:9.2.1", name=ays_server_container_name, command="sleep 3600", ports=[5000], host_config=host_config, entrypoint="")

#ays_container_id = retval.get('Id')
#ays_container = j.sal.docker.client.containers.get(container_id=ays_container_id)
ays_container = j.sal.docker.client.containers.get(container_id=ays_server_container_name)
ays_container.start()
```

Get prefab access to the container:
```python
executor = j.tools.executor.getLocalDocker(container_id_or_name=ays_server_container_name)
prefab = j.tools.prefab.get(executor=executor)
```

Install AYS:
```python
branch = "9.2.1"
prefab.js9.atyourservice.install(branch=branch)
```

Configure AYS:
```python
prefab.js9.atyourservice.configure(production=False, organization='', restart=False, host="0.0.0.0", port=5000)
```

Test the AYS server: 
```bash
ctrl+z
docker exec -it ays-server bash
eval $(ssh-agent)
js9
j.tools.prefab.local.js9.atyourservice.start()
exit
tmux at
ctrl+b d
exit
fg
```

Commit all changes:
```python
ays_container.commit(repository="jumpscale/ays-server", tag=tag)
```

Stop and remove the container:
```python
ays_container.stop()
ays_container.remove()
```

Makes sure you are owner of the JumpScale organization on Docker Hub and that you are logged in:
```bash
ctrl+z
docker login
fg
```

Back in the interactive shell:
```python
j.sal.docker.client.images.push("jumpscale/ays-server", tag)
```

<a id="ays-portal"></a>
### Create an AYS Portal image

Options to create an AYS Portal image:
- [ZUtils](#portal-zutils)
- [Dockerfile](#portal-dockerfile)
- [JumpScale](#portal-jumpscale)

<a id="portal-zutils"></a>
#### Create an AYS Portal image using the ZUtils

If not already installed, get the ZUtils:
```bash
export ZUTILSBRANCH=9.2.1
curl https://raw.githubusercontent.com/Jumpscale/bash/${ZUTILSBRANCH}/install.sh?$RANDOM > /tmp/install.sh
bash /tmp/install.sh
. ~/.bash_profile
```

Create the image:
```bash
export JS9BRANCH=9.2.1
ZInstall_portal9
```

This will create following image:
```bash
jumpscale/portal9
```

And if not already created previously, it will first create the following images:
```bash
jumpscale/js9_full
jumpscale/ubuntu_python
jumpscale/ubuntu
```

For more information about the ZUtils see https://github.com/Jumpscale/bash.

<a id="portal-dockerfile"></a>
#### Create an AYS Portal image using a Dockerfile

Let's first create an ENTRYPOINT script:
```bash
mkdir -p /tmp/Dockerfiles/js9_portal
cd /tmp/Dockerfiles/js9_portal
vim docker-entrypoint.sh
```

Here's the script: 
```bash
#!/bin/bash
set -e
js9 'j.clients.redis.start4core()'

#echo "export LC_ALL=C.UTF-8" >> /root/.profile || return 1
#echo "export LANG=C.UTF-8" >> /root/.profile || return 1
#js9 'j.tools.prefab.local.web.portal.install(start=False, branch="9.2.1")'

#Move to Dockerfile, so this is executed when building the image, not when creating the container
#js9 'j.tools.prefab.local.js9.atyourservice.load_ays_space()'

if [ ! -z "$organization" ] ; then
    js9 'import os; portal_client_id=os.environ["portal_client_id"];portal_secret=os.environ["portal_secret"];org=os.environ["organization"];redirect_url=os.environ["redirect_url"];j.tools.prefab.local.web.portal.configure(mongodbip="127.0.0.1", mongoport=27017, production=True, client_id=portal_client_id, client_secret=portal_secret, scope_organization=org, redirect_address=redirect_url, restart=False)'
fi

if [ ! -z "$ays_internal_ip_address" ] ; then
    js9 'import os; ays_internal_ip_address = os.environ["ays_internal_ip_address"]; external_ip_address = os.environ["external_ip_address"]; public_ays_url = "http://{}:5000".format(external_ip_address); private_ays_url = "http://{}:5000".format(ays_internal_ip_address); j.tools.prefab.local.js9.atyourservice.configure_portal(ays_url=private_ays_url, ays_console_url=public_ays_url, portal_name="main", restart=False)'
fi

js9 'j.tools.prefab.local.web.portal.start()'
tail -f /opt/var/log/jumpscale.log
```

Don't forget:
```bash
chmod +x ./docker-entrypoint.sh
```

Create the Dockerfile:
```bash
vim Dockerfile
```

Here's the Dockerfile:
```Dockerfile
ARG docker_hub_username="jumpscale"
FROM $docker_hub_username/js9_full:9.2.1
MAINTAINER Yves Kerwyn

#RUN ssh-keyscan -t rsa github.com >> /root/.ssh/known_hosts

RUN git clone -b 9.2.1 https://github.com/Jumpscale/portal9.git

#RUN pip3 install -e /opt/code/github/jumpscale/portal9
RUN cd /opt/code/github/jumpscale/portal9; ./install.sh 9.2.1
RUN js9 'j.tools.prefab.local.js9.atyourservice.load_ays_space(branch="9.2.1")'

EXPOSE 5000
VOLUME ["/root/js9host"]
COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
```

> Note that we run `install.sh` instead of `pip3 install`, which does the same (running `setup.py`) + installs the portal

Build the image:
```bash
docker build --build-arg docker_hub_username=$docker_hub_username --tag $docker_hub_username/js9_ays_portal:9.2.1 .
```

<a id="portal-jumpscale"></a>
#### Create an AYS Portal image using JumpScale

Create and start a container (keeping it alive for 1 hour):
```python
#host_config = {
#    'NetworkMode': 'default',
#    'PortBindings': {'8200/tcp': [{'HostIp': '', 'HostPort': '8300'}]}
#    }
# Same as: (note that the first port is the internal one, and second on the host)
host_config = j.sal.docker.client.api.create_host_config(port_bindings={8200: 8300})
#portal_container = j.sal.docker.client.containers.create(image="ubuntu:16.04", hostname=ays_portal_container_name, command="sleep 3600", ports=[8200], host_config=host_config)
retval = j.sal.docker.client.api.create_container(image="jumpscale/full:9.2.1", name=ays_portal_container_name, command="sleep 3600", ports=[8200], host_config=host_config)
portal_container_id = retval.get('Id')
portal_container = j.sal.docker.client.containers.get(container_id=portal_container_id)
portal_container.start()
```

> Check `j.sal.docker.client.api.create_container?` for guidance on the kwargs** you can pass to `j.sal.docker.client.containers.create()`, or see https://docker-py.readthedocs.io/en/1.7.0/port-bindings/.

Get prefab access to the container:
```python
#executor = j.tools.executor.getLocalDocker(container_id_or_name=portal_container.id)
executor = j.tools.executor.getLocalDocker(container_id_or_name=ays_portal_container_name)
prefab = j.tools.prefab.get(executor=executor)
```

Execute the following to install JumpScale portal:
```python
prefab.web.portal.install(start=False, branch=branch)
```

Add the AYS Portal app ("Cockpit"):
```python
prefab.js9.atyourservice.load_ays_space()
```

Configure the AYS Portal app - do this later when container runs:
```python
#internal_ays_url = "http://{}:5000".format(public_ip_address)
#prefab.js9.atyourservice.configure_portal(ays_url=internal_ays_url, ays_console_url=public_ays_url, portal_name="main", restart=False)
```

Commit all changes:
```python
portal_container.commit(repository="jumpscale/ays-portal", tag=tag)
```

Stop and remove the container:
```python
portal_container.stop()
portal_container.remove()
```

Makes sure you are owner of the JumpScale organization on Docker Hub and that you are logged in:
```bash
ctrl+z
docker login
fg
```

Back in the interactive shell:
```python
j.sal.docker.client.images.push("jumpscale/ays-portal", tag)
```


<a id="create-containers"></a>
## Create containers

In what follows we will create a Docker container for:
- [AYS Server](#ays-container)
- [AYS Portal](portal-container)

<a id="ays-container"></a>
### Create and start an AYS Server container

Options to create and start an AYS Server container:
- [ZUtils](#ays-container-zutils)
- [Dockerfile](#ays-container-dockerfile)
- [JumpScale](#ays-container-jumpscale)


<a id="ays-container-zutils"></a>
#### Create and start an AYS Server container using the ZUtils

Make sure you have created the AYS Server container using the ZUtils as documented in [Create a JumpScale image using the ZUtils](#ays-zutils).

```bash
ZDockerActive -b jumpscale/ays9 -i <name of your docker> -a "-p 5000:5000"
```

You can access it using `ZSSH`:
```bash
ZSSH
```

At this point the container is up but the AYS is not running yet. 


Before starting the AYS server, first execute the following in the interactive shell (`js9`) from within the container:
```python
j.clients.redis.start4core()
organization = "ays-organizations.docker-on-mac"
j.tools.prefab.local.js9.atyourservice.configure(production=False, organization=organization, restart=False, host="0.0.0.0", port=5000)
```

Then start the server:
```python
j.tools.prefab.local.js9.atyourservice.start()
```


<a id="ays-container-dockerfile"></a>
#### Create and start an AYS Server container using a Dockerfile

Run (= create + start) a Docker container with the AYS server image:
```bash
# DON'T USE THE VOLUME MAPPING: -v "/opt/var/data/ays-server:/root/js9host"
# docker run -it --name ays-server -p "5000:5000" -e organization="ays-organizations.docker-on-mac" -e external_ip_address="185.15.201.111" jumpscale/js9_ays bash

docker run -d --name ays-server -p "5000:5000" -e organization=$iyo_organization -e external_ip_address="192.168.16.184" $docker_hub_username/js9_ays:9.2.1

```

Check the container:
```bash
docker exec -it ays-server bash
```


In case you need to manually run the `docker-entrypoint.sh`:
```bash
apt-get update
apt-get install -y vim
vim /docker-entrypoint.sh
echo $external_ip_address
echo $organization
/docker-entrypoint.sh
```

Or manually from js9:
```python
import os
j.clients.redis.start4core()
j.tools.prefab.local.js9.atyourservice.install(branch='9.2.1')
org=os.environ["organization"]
external_ip_address=os.environ["external_ip_address"]
j.tools.prefab.local.js9.atyourservice.configure(production=True, organization=org, restart=False, host="0.0.0.0", port=5000)
public_ays_url = "http://{}:5000".format(external_ip_address)
j.tools.prefab.local.js9.atyourservice.configure_api_console(url=public_ays_url)
j.tools.prefab.local.js9.atyourservice.start()
```

Check:
- Updated `baseurl`: `vim /opt/code/github/jumpscale/ays9/JumpScale9AYS/ays/server/apidocs/api.raml`
- Updated AYS configuration: `vim ~/js9host/cfg/jumpscale9.toml`
- AYS is running: `tmux at`

<a id="ays-container-jumpscale"></a>
#### Create and start an AYS Server container using JumpScale



Or From JumpScale:
```python
retval = j.sal.docker.client.api.create_container(image="jumpscale/ays-server:9.2.1", name=ays_server_container_name, detach=True, stdin_open=False, ports=[5000], host_config=host_config, volumes=["/root/js9host"], entrypoint="")

add entrypoint script through prefab
...
commit
close
remove
```

### Locally - on the (virtual) machine used to create the images as documented above

Make sure the path to volume to be mounted into the container exists:
```python
j.sal.fs.createDir(newdir=ays_server_mounted_volume_path)
```

Create and start a container with AYS server:
```python
host_config = j.sal.docker.client.api.create_host_config(port_bindings={5000: 5000}, binds=["{}:/root/js9host".format(ays_server_mounted_volume_path)])
retval = j.sal.docker.client.api.create_container(image="jumpscale/ays-server:9.2.1", name=ays_server_container_name, detach=True, stdin_open=False, ports=[5000], host_config=host_config, volumes=["/root/js9host"], entrypoint="")
#retval = j.sal.docker.client.api.start(container=ays_server_container_name)
ays_container = j.sal.docker.client.containers.get(container_id=ays_server_container_name)
ays_container.start()
```

@TODO update `command` to use start+config server 
@TODO use the eqquivalent of `docker run -dit --restart unless-stopped ays-server`
```python

ays_executor = j.tools.executor.getLocalDocker(container_id_or_name=ays_server_container_name)
ays_prefab = j.tools.prefab.get(executor=ays_executor)
```

The default base request url used by the AYS API Console is `https://localhost:5000`; which will not work since https is not active yet. In order to have it work at this point we'll need to change it to `http://$external_IP_address:$external_ays_port`.



Configure AYS:
```python
ays_prefab.js9.atyourservice.configure(production=False, organization='', restart=False, host="0.0.0.0", port=5000)
```



First get the external IP address of the VDC:
```python
#external_ip_address = vdc.model["externalnetworkip"]
external_ip_address = "185.15.201.111"
external_ays_port = 5000
```

Configure the AYS API Console:
```python
public_ays_url = "http://{}:{}".format(external_ip_address, external_ays_port)
ays_prefab.js9.atyourservice.configure_api_console(url=public_ays_url)
```


<a id="portal-container"></a>
### Create an AYS Portal container

Options to create and start an AYS portal container:
- [ZUtils](#portal-container-zutils)
- [Dockerfile](#portal-container-dockerfile)
- [JumpScale](#portal-container-jumpscale)


<a id="portal-container-zutils"></a>
#### Create and start an AYS Server container using a the ZUtils

...


<a id="portal-container-dockerfile"></a>
#### Create and start an AYS Server container using a Dockerfile

First find out what the private IP address is of your AYS server container:
```bash
docker inspect -f '{{ .NetworkSettings.IPAddress }}' ays-server
```

This IP address needs to be passed as the value for the environment variable `ays_internal_ip_address` when creating the AYS portal container, here below we specified `172.17.0.2`.

Run (= create + start) a Docker container with the AYS portal image:
```bash
export ays_internal_ip_address="172.17.0.2"
export redirect_url="http://185.15.201.111:8200"
export iyo_organization="ays-organizations.docker-on-mac"
export external_ip_address="185.15.201.111"
export portal_client_id="ays-organizations.docker-on-mac"
export portal_secret="4t2EknNeZPqu8LdzaiVa7y8XLtH0K6bI2QUOS10yiLGwv5-KOLL5"
docker run -d --name ays-portal -p "8200:8200" -e ays_internal_ip_address=$ays_internal_ip_address -e external_ip_address=$external_ip_address -e portal_client_id=$portal_client_id-e portal_secret=$portal_secret -e redirect_url=$redirect_url -e organization=$iyo_organization jumpscale/js9_ays_portal
```

Check the container:
```bash
docker exec -it ays-portal bash
```

<a id="portal-container-jumpscale"></a>
#### Create and start an AYS Server container using JumpScale

Start a container with AYS portal:
```python
host_config = j.sal.docker.client.api.create_host_config(port_bindings={8200: 8200})
retval = j.sal.docker.client.api.create_container(image="jumpscale/ays-portal:9.2.1", name=ays_portal_container_name, command="bash", ports=[8200], host_config=host_config)
portal_container = j.sal.docker.client.containers.get(container_id=ays_portal_container_name)
portal_container.start()
```

```python
portal_executor = j.tools.executor.getLocalDocker(container_id_or_name=ays_portal_container_name)
portal_prefab = j.tools.prefab.get(executor=portal_executor)
```

### Remotely - on an OpenvCloud virtual machine 

Let's deploy the Docker containers on a OpenvCloud virtual machine, which is done is following steps:
- [Create or get an SSH key](#get-sshkey)
- [Create your JumpScale configuration repositoryCreate your JumpScale configuration repository](#create-jsconfig)
- [Create your JumpScale configuration instances](#jsconfig-instances)
- [Create a new virtual machine using the 9.3.0 JumpScale client for OpenvCloud](#create-vm)
- [Deploy the containers](#deploy-containers)

<a id="get-sshkey"></a>
### Create or get an SSH key

On your (virtual) machine with JumpScale 9.3.0 installed, you can either use an existing SSH key or create a new SSH key.

In order to list all SSH keys that are loaded by ssh agent:
```python
j.clients.sshkey.list()
```

The following will create a new key and 


Here's how to create a new one from JumpScale:
```python
import os
new_ssh_key_name = "new_key"
new_ssh_key_path = os.path.expanduser("~/.ssh/{}".format(new_ssh_key_name))
## still working on, not avaialble in dev branch
#j.clients.ssh.generate()
# Alternative:
sshkey = j.clients.sshkey.get(
```

In order to verify that you newly created key is loaded, execute:
```python
j.clients.ssh.ssh_keys_list_from_agent()
```

<a id="create-jsconfig"></a>
### Create your JumpScale configuration repository

Let's now assume you don't have any JumpScale config instances yet, and that you want to encrypt all JumpScale config instances with your newly created SSH key.

First you need to create a Git repository that will hold all your config instances, this is achieved by executing `js9_config init` in a new or existing Git repository:
```bash
mkdir $path_to_your_jsconfig_repo
cd $path_to_your_jsconfig_repo
git init
js9_config init
```

@todo # how to do this from JumpScale?


<a id="jsconfig-instances"></a>
### Create your JumpScale configuration instances

Now let's create a config instance for your ItsYou.online profile:
```python
import os
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]
iyo_config = {
    "baseurl": "https://itsyou.online/api",
    "application_id_": app_id,
    "secret_": secret
}
#j.tools.configmanager.configure(location="j.clients.itsyouonline", data=iyo_config, instance="main", sshkey_path=new_ssh_key_path)
j.clients.itsyouonline.get(data=iyo_config, instance="main", sshkey_path=new_ssh_key_path)
```



And now the config instance for OpenvCloud:
```python

```

Set the variables:
```python

account_name = "Account_of_Yves"
cloud_space_name = "docker-space"
location = "be-gen-1"
docker_host_name = "docker-host"
```



Get an OpenvCloud client, specifying the private key you used for encrypting the configuration instances:
```python
import os


ovc_client = j.clients.openvcloud.get(instance="main", sshkey_path=ssh_key_path)
```

Access the OpenvCloud account:
```python
account = ovc_client.account_get(name=account_name, create=False)
```

Get or create the VDC:
```python
cloud_space = account.space_get(name=cloud_space_name, location=location, create=True)
```

Create the Docker host, specifying the same ssh key:
```python
docker_host = cloud_space.machine_get(name=docker_host_name, create=True, sshkeyname=ssh_key_name)
```

Get the internal IP address of the host - will be used later:
```python
internal_ip_address = docker_host.model["nics"][0]["ipAddress"]
```

Get the public IP address and the cloud space - will be user later:
```python
public_ip_address = cloud_space.model["externalnetworkip"]
```

Install Docker:
```python
docker_host.prefab.virtualization.docker.install()
```

Make sure the Docker daemon port is forwarded - no, don't do this::
```python
#docker_host.portforward_create(publicport=2375, localport=2375)
#os.environ["DOCKER_HOST"] = "tcp://{}:{}".format(public_ip_address, 2375)
import os
os.environ["DOCKER_HOST"] = "tcp://localhost:{}".format(2375)
```

Deploy AYS server container:
```python

import docker

docker_client = docker.from_env()
```


...


Start the AYS server- locally from the Docker host - fails:
```python
ays_container = j.sal.docker.client.containers.get(container_id=ays_server_container_name)
executor = j.tools.executor.getLocalDocker(container_id_or_name=ays_server_container_name)
prefab = j.tools.prefab.get(executor=executor)
prefab.js9.atyourservice.start()
```

The above fails because no SSH agent is running yet in the container.

In the container you can manually fix this like this - doesn't work: 
```bash
eval $(ssh-agent)
```

Or like this in js on the container - doesn't work, see https://github.com/Jumpscale/lib9/issues/180:
```python
j.clients.ssh.start_ssh_agent()
```

