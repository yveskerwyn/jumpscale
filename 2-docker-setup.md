# Docker setup

-[Create the Docker images](#create-images)
-[Publish to Docker Hub](#publish-images)
-[Deploy the Docker images](#deploy-images)

<a id="create-images"></a>
## Create the Docker images

On a (virtual) machine with JumpScale installed execute the following.

Set variables:
```python
#import os
#import yaml
#app_id = os.environ["APP_ID"]
#secret = os.environ["SECRET"]
branch = "9.2.1"
tag="9.2.1"
```

Make sure you have installed ParallelSSH:
```python
j.tools.prefab.local.runtimes.pip.install(package="parallel-ssh")
```

Check if Docker is installed
```python
j.tools.prefab.local.virtualization.docker.isInstalled()
```

If not yet installed first install Docker:
```python
j.tools.prefab.local.virtualization.docker.install()
```

Get the Ubuntu image:
```python
import docker
docker_client = docker.from_env()
docker_client.images.pull(repository="ubuntu", tag="16.04")
```

In what follows we will create a Docker image for:
- [AYS Server](#ays-server)
- [AYS Portal](#ays-portal)

<a id="ays-server"></a>
### AYS Server

Create and start a container (keeping it alive for 1 hour):
```python
host_config = j.sal.docker.client.api.create_host_config(port_bindings={5000: 5000})
retval = j.sal.docker.client.api.create_container(image="ubuntu:16.04", name="ays-server", command="sleep 3600", ports=[5000], host_config=host_config)
ays_container_id = retval.get('Id')
ays_container = j.sal.docker.client.containers.get(container_id=ays_container_id)
ays_container.start()
```

Get prefab access to the container:
```python
executor = j.tools.executor.getLocalDocker(container_id_or_name=ays_container_id)
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

Install AYS:
```python
prefab.js9.atyourservice.install(branch=branch)
```

Configure AYS:
```python
prefab.js9.atyourservice.configure(production=False, organization='', restart=False, host="0.0.0.0", port=5000)
```

Start the AYS server:
```python
prefab.js9.atyourservice.start()
```

Commit all changes:
```python
container.commit(repository="jumpscale/ays-server", tag=tag)
```

Stop and remove the container:
```python
container.stop()
container.remove()
```

<a id="ays-portal"></a>
### AYS Portal

Create and start a container (keeping it alive for 1 hour):
```python
#host_config = {
#    'NetworkMode': 'default',
#    'PortBindings': {'8200/tcp': [{'HostIp': '', 'HostPort': '8300'}]}
#    }
# Same as: (note that the first port is the internal one, and second on the host)
host_config = j.sal.docker.client.api.create_host_config(port_bindings={8200: 8300})
#portal_container = j.sal.docker.client.containers.create(image="ubuntu:16.04", hostname="ays-portal", command="sleep 3600", ports=[8200], host_config=host_config)
retval = j.sal.docker.client.api.create_container(image="ubuntu:16.04", name="ays-portal", command="sleep 3600", ports=[8200], host_config=host_config)
portal_container_id = retval.get('Id')
portal_container = j.sal.docker.client.containers.get(container_id=portal_container_id)
portal_container.start()
```

> Check `j.sal.docker.client.api.create_container?` for guidance on the kwargs** you can pass to `j.sal.docker.client.containers.create()`, or see https://docker-py.readthedocs.io/en/1.7.0/port-bindings/.

Get prefab access to the container:
```python
#executor = j.tools.executor.getLocalDocker(container_id_or_name=portal_container.id)
executor = j.tools.executor.getLocalDocker(container_id_or_name=portal_container_id)
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
prefab.js9.js9core.install(branch=branch)
```

Execute the following to install JumpScale portal:
```python
prefab.web.portal.install(start=False, branch=branch)
```

This also starts MongoDB (in the same tmux session as AYS server) - @todo # we need to use a systemd unit file instead.

Add the AYS Portal app ("Cockpit"):
```python
prefab.js9.atyourservice.load_ays_space()
```

Start the portal:
```python
prefab.web.portal.start()
```

Configure the AYS Portal app:
```python
internal_ays_url = "http://{}:5000".format(public_ip_address)
prefab.js9.atyourservice.configure_portal(ays_url=internal_ays_url, ays_console_url=public_ays_url, portal_name="main", restart=True)
```

...

Commit all changes:
```python
portal_container.commit(repository="jumpscale/ays-portal", tag=tag)
```

Stop and remove the container:
```python
container.stop()
container.remove()
```



<a id="publish-images"></a>
## Publish images

```python
j.sal.docker.client.images.push("jumpscale/ays-server", tag)
```

<a id="deploy-images"></a>
## Deploy images

Set the variables:
```python
ssh_key_name = "bootstrap_vm2_key"
account_name = "Account_of_Yves"
cloud_space_name = "docker-space"
location = "be-gen-1"
docker_host_name = "docker-host"
```

Get an OpenvCloud client, specifying the private key you used for encrypting the configutation instances:
```python
import os

key_path = os.path.expanduser("~/.ssh/{}".format(ssh_key_name))
ovc_client = j.clients.openvcloud.get(instance="main", sshkey_path=key_path)
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
