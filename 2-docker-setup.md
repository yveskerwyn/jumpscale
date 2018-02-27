# Docker setup

- [Create the Docker images](#create-images)
- [Publish to Docker Hub](#publish-images)
- [Deploy the Docker images](#deploy-images)

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
#docker_branch = "17.12"
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
#j.tools.prefab.local.virtualization.docker.install(branch=docker_branch)
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
#ays_container_id = retval.get('Id')
#ays_container = j.sal.docker.client.containers.get(container_id=ays_container_id)
ays_container = j.sal.docker.client.containers.get(container_id="ays-server")
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

Start the AYS server - doesn't work from development branch:
```python
# prefab.js9.atyourservice.start()
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

<a id="deploy-images"></a>
## Deploy images

Let's deploy the Docker containers on a OpenvCloud virtual machine, which is done is following steps:
- [Create or get an SSH key](#get-sshkey)
- [Create your JumpScale configuration repositoryCreate your JumpScale configuration repository](#create-jsconfig)
- [Create your JumpScale configuration instances](#jsconfig-instances)
- [Create a new virtual machine using the 9.3.0 JumpScale client for OpenvCloud](#create-vm)
- [Deploy the containers](#deploy-containers)

<a id="get-sshkey"></a>
### Create or get an SSH key

On your (virtual) machine with JumpScale 9.3.0 installed, you can either use an existing SSH key or create a new SSH key.

Here's how to create a new one from JumpScale:
```python
import os
new_ssh_key_name = "new_key"
new_ssh_key_path = os.path.expanduser("~/.ssh/{}".format(ssh_key_name))
j.clients.ssh.ssh_keys_load(path=new_ssh_key_path)
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
    "application_id_": app_id,
    "secret_": secret
}
j.tools.configmanager.configure(location="j.clients.itsyouonline", data=iyo_config, instance="main", sshkey_path=new_ssh_key_path)
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
