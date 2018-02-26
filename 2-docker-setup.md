# Docker setup

-[Create the Docker images](#create-images)
-[Publish to Docker Hub](#publish-images)
-[Deploy the Docker images](#deploy-images)

<a id="create-images"></a>
## Create the Docker images

On a (virtual) machine with JumpScale installed execute the following.

Set variables:
```python
import os
import yaml
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]
branch = "9.2.1"
tag="9.2.1"
```

Make sure you have installed ParallelSSH:
```python
j.tools.prefab.runtimes.pip.install(package="parallel-ssh")
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

Create and start a container (keeping it alive for 1 hour):
```python
container = j.sal.docker.client.containers.create(image="ubuntu:16.04", command="sleep 3600")
# container_id = "697d604ef29f"
# container = j.sal.docker.client.containers.get(container_id=container_id)
container.start()
```

Get prefab access to the container:
```python
executor = j.tools.executor.getLocalDocker(container_id_or_name=container.id)
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

Install AYS:
```python
prefab.js9.atyourservice.install(branch=branch)
```

Configure AYS:
```python
prefab.js9.atyourservice.configure(production=False, organization='', restart=True, host="0.0.0.0", port=5000)
```

Start the AYS server:
```python
prefab.js9.atyourservice.start()
```

First fix a missing dependency:
```python
prefab.local.core.run("pip install psutil==5.2.2")
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
