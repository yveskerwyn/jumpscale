# Creating Docker images

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
export docker_hub_username="yveskerwyn"
docker build --tag $docker_hub_username/ubuntu_python .
```

Create another image now adding JumpScale to the `jumpscale/ubuntu_python` image:
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

RUN cd /opt/code/github/jumpscale/core9; pip3 install -e .
RUN cd /opt/code/github/jumpscale/lib9; pip3 install -e .
RUN cd /opt/code/github/jumpscale/prefab9; pip3 install -e .

#ENTRYPOINT ["js9"]
```

Build the image:
```bash
docker build --build-arg docker_hub_username=$docker_hub_username --tag $docker_hub_username/js9_full:9.2.1 .
```

Test this image:
```bash
export iyo_organization="ays-organizations.docker-on-mac"
docker run -it --name js9-test -p "5000:5000" -e organization=$iyo_organization $docker_hub_username/js9_full:9.2.1 bash
```

Next create the AYS server image.

Let's first create an entrypoint script:
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

Build the image:
```bash
docker build --build-arg docker_hub_username=$docker_hub_username --tag $docker_hub_username/js9_ays:9.2.1 .
```

Start an AYS server container:
```bash
export external_ip_address="185.15.201.111"
docker run -d --name ays-server -p "5000:5000" -e organization=$iyo_organization $docker_hub_username -e external_ip_address=$external_ip_address jumpscale/js9_ays
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



```bash
mkdir -p /tmp/Dockerfiles/js9_portal
cd /tmp/Dockerfiles/js9_portal
vim docker-entrypoint.sh
```

on't forget:
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

Build the image:
```bash
docker build --build-arg docker_hub_username=$docker_hub_username --tag $docker_hub_username/js9_ays_portal:9.2.1 .
```

Error:
```python
Traceback (most recent call last):
  File "/usr/local/bin/js9", line 20, in <module>
    exec(toexec)
  File "<string>", line 1, in <module>
  File "/opt/code/github/jumpscale/prefab9/modules/js9/PrefabAtYourService.py", line 98, in load_ays_space
    self.prefab.web.portal.addSpace('{}apps/AYS'.format(self.repo_dir))
  File "/opt/code/github/jumpscale/prefab9/modules/web/PrefabPortal.py", line 212, in addSpace
    self.prefab.core.file_link(spacepath, dest_dir)
  File "/opt/code/github/jumpscale/prefab9/JumpScale9Prefab/PrefabCore.py", line 990, in file_link
    (self.shell_safe(source), self.shell_safe(destination)))
  File "/opt/code/github/jumpscale/prefab9/JumpScale9Prefab/PrefabCore.py", line 1323, in run
    cmd, checkok=checkok, die=die, showout=showout, env=env, timeout=timeout)
  File "/opt/code/github/jumpscale/core9/JumpScale9/tools/executor/ExecutorLocal.py", line 103, in execute
    outputStderr=outputStderr, timeout=timeout)
  File "/opt/code/github/jumpscale/core9/JumpScale9/sal/process/SystemProcess.py", line 196, in execute
    raise RuntimeError("Could not execute:%s\nstdout:%s\nerror:\n%s"%(rc,out,err))
RuntimeError: Could not execute:1
stdout:
error:
ln: failed to create symbolic link '/opt/jumpscale9/apps/portals/main/base/AYS': No such file or directory
Could not execute:1
stdout:
error:
ln: failed to create symbolic link '/opt/jumpscale9/apps/portals/main/base/AYS': No such file or directory
```