# Creating Docker images

- [Docker Setup](#docker)
- [Variables](#vars)
- [Base Image](#base-image)
- [JumpScale Image](#js9-image)
- [AYS Image](#ays-image)
- [Portal Image](#portal-image)

<a id="docker"></a>
## Docker Setup

In case not yet installed::
```bash
apt-get update
apt-get install -y vim curl
```

On a fresh machine, install Docker: https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce-1
```bash
apt-get update
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -


add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

apt-get update  

apt-get install -y docker-ce
```

<a id="vars"></a>
## Variables

```bash
export js_branch="9.2.1"
export docker_hub_username="yveskerwyn"
export iyo_organization="ays-organizations.docker-on-mac"
````

<a id="base-image"></a>
## Creating the base image

> Only for ays9 we run `install.sh` after running `pip3 install -e .` (which runs `setup.py`), it's interesting to check the `install.sh` in each of repositories in order to understand what you potentially miss when only running `pip3 install -e .`

Start with creating a base Ubuntu image with Python:
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
docker build --tag $docker_hub_username/ubuntu_python .
```


<a id="js9-image"></a>
## Creating the JumpScale Image


Create another image now adding JumpScale core9, lib9 and prefab9 to the `ubuntu_python` image:
```bash
mkdir -p /tmp/Dockerfiles/js9_full
cd /tmp/Dockerfiles/js9_full
vim Dockerfile
```

Here's the Dockerfile:
```Dockerfile
ARG docker_hub_username="jumpscale"
FROM $docker_hub_username/ubuntu_python
ARG js_branch="development"
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

RUN git clone -b $js_branch https://github.com/Jumpscale/core9.git
RUN git clone -b $js_branch https://github.com/Jumpscale/lib9.git
RUN git clone -b $js_branch https://github.com/Jumpscale/prefab9.git

RUN cp /opt/code/github/jumpscale/core9/mascot /root/.mascot.txt

RUN cd /opt/code/github/jumpscale/core9; pip3 install -e .
RUN cd /opt/code/github/jumpscale/lib9; pip3 install -e .
RUN cd /opt/code/github/jumpscale/prefab9; pip3 install -e .

ENTRYPOINT ["js9"]
```

> Note that the `pip3 install` steps will only execute the `setup.py` of each repository, so the `install.sh` scripts are not used here. 

Build the image, here using the 9.2.1 branch:
```bash
docker build --build-arg docker_hub_username=$docker_hub_username --build-arg js_branch=$js_branch --tag $docker_hub_username/js9_full:$js_branch .
```

Test the image:
```bash
docker run -it --rm --name js9-test -p "5000:5000" -e organization=$iyo_organization $docker_hub_username/js9_full:$js_branch
```

<a id="ays-image"></a>
## Creating the AYS server Docker image

This Dockerfile will next to adding AYS to the `js9_full` image, also include an ENTRYPOINT to configure and start the AYS server.

Let's first create the ENTRYPOINT script:
```bash
mkdir -p /tmp/Dockerfiles/ays_server
cd /tmp/Dockerfiles/ays_server
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
FROM $docker_hub_username/js9_full:$js_branch
ARG js_branch="development"
MAINTAINER Yves Kerwyn

RUN git clone -b $js_branch https://github.com/Jumpscale/ays9.git

RUN pip3 install -e /opt/code/github/jumpscale/ays9
RUN /opt/code/github/jumpscale/ays9/install.sh

EXPOSE 5000
VOLUME ["/root/js9host"]
COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
```

> Note we also run `install.sh` next to running `pip3 install -e .` (which runs `setup.py`), we only do this for the AYS repository; it's interesting to check the `install.sh` in each of repositories in order to understand what you potentially miss when only running `pip3 install -e .`, for AYS we need it in order to install the AYS server; which we could alternativelly also do by executing `j.tools.prefab.local.js9.atyourservice.install(branch="9.2.1")`

Build the image:
```bash
docker build --build-arg docker_hub_username=$docker_hub_username --tag $docker_hub_username/ays_server:$js_branch .
```

Test the AYS server image:
```bash
ifconfig | grep "inet " | grep -v 127.0.0.1
export external_ip_address="192.168.1.147"
docker run -d --name ays-server -p "5000:5000" -e organization=$iyo_organization -e external_ip_address=$external_ip_address $docker_hub_username/ays_server:$js_branch
```

Manually run the `docker-entrypoint.sh`:
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


<a id="portal-image"></a>
## Creating the AYS Portal Docker image

Let's first create an ENTRYPOINT script:
```bash
mkdir -p /tmp/Dockerfiles/ays_portal
cd /tmp/Dockerfiles/ays_portal
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
FROM $docker_hub_username/js9_full:$js_branch\
ARG js_branch="development"
MAINTAINER Yves Kerwyn

#RUN ssh-keyscan -t rsa github.com >> /root/.ssh/known_hosts

RUN git clone -b $js_branch https://github.com/Jumpscale/portal9.git

RUN pip3 install -e /opt/code/github/jumpscale/portal9
#RUN cd /opt/code/github/jumpscale/portal9; ./install.sh 9.2.1
RUN js9 'j.tools.prefab.local.web.portal.install(branch="9.2.1")'
RUN js9 'j.tools.prefab.local.js9.atyourservice.load_ays_space(branch="9.2.1")'

EXPOSE 5000
VOLUME ["/root/js9host"]
COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
```

> Note that we run `install.sh` instead of `pip3 install`, which does the same (running `setup.py`) + installs the portal

Build the image:
```bash
docker build --build-arg docker_hub_username=$docker_hub_username --tag $docker_hub_username/ays_portal:$js_branch .
```

This results (on Mac) in the issue as reported here: https://github.com/Jumpscale/prefab9/issues/179