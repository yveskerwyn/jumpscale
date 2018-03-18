# Getting started with Zero-Robot

See: https://github.com/Jumpscale/0-robot/blob/master/docs/getting_started.md


- [Using a Docker container](#docker)
- [On your host](#host)


<a id="docker"></a>
## Using a Docker container

Check script to create a Docker image for 0-robot:
https://github.com/Jumpscale/0-robot/blob/master/utils/scripts/packages/dockerbuild.py

Create a container:
```bash
docker run --name 0-robot -d -p 192.168.103.254:6600:6600 -v /root/.ssh:/root/.ssh -e data-repo=ssh://git@docs.greenitglobe.com:10022/yves/zrobot3.git -e config-repo=ssh://git@docs.greenitglobe.com:10022/yves/myjsconfig.git -e template-repo=https://github.com/zero-os/0-templates jumpscale/0-robot:latest
```

In the Docker container [`dockerentrypoint.py`](https://github.com/Jumpscale/0-robot/blob/master/utils/scripts/packages/dockerentrypoint.py) is used as an ENTRYPOINT that will start 0-robot (`zrobot server start`) using the environment variables you passed when starting the container, following environment variables can be passed:
```bash
-e listen 
-e data-repo=repo=ssh://git@docs.greenitglobe.com:10022/yves/zrobot3.git
-e template-repo=https://github.com/zero-os/0-templates
-e config-repo=ssh://git@docs.greenitglobe.com:10022/yves/myjsconfig.git
-e debug
-e telegram-bot-token
-e telegram-chat-id
-e auto-push
-e auto-push-interval
```

Normally the above should start the zrobot... 

If not:
```bash
docker exec -it 0-robot bash
zrobot server start --listen :6600 --data-repo ssh://git@docs.greenitglobe.com:10022/yves/zrobot3.git --config-repo ssh://git@docs.greenitglobe.com:10022/yves/myjsconfig.git --template-repo https://github.com/zero-os/0-templates
```

In to interact, you either use the zrobot client, or the JumpScale client. The zrobot client installs as part of the zrobot installation; so there's no separate client (yet).

Using the JumpScale:

```python

j.clients.zrobot.get

```



<a id="host"></a>
## On your host


Make sure you have a repository for your JumpScale configuration, in an empty repository do:
```bash
js9_config init
```

```bash
zrobot server start --listen :6600 --template-repo https://github.com/zero-os/0-templates --data-repo git@github.com:yveskerwyn/zrobot1.git
```

Using the helloworld templates:
```bash
zrobot server start --listen :6600 --template-repo https://github.com/chrisvdg/zrobot-helloworld --data-repo git@github.com:yveskerwyn/zrobot1.git
```

Or leave out the repos:
```bash
zrobot server start --listen :6600 --data-repo git@github.com:yveskerwyn/zrobot1.git
```

Connect:
```bash
zrobot robot connect main http://localhost:6600
```

This will create a `main.toml` in the sub directory `j.clients.zrobot` of your JumpScale configuration repository.

Execute a blueprint:
```bash
zrobot blueprint execute blueprint.yaml
```

Check the result, listing the services that got created:

```bash
zrobot service list
```

Example blueprint:
```
services:
  - github.com/chrisvdg/zrobot-helloworld/helloworld/0.0.1__service1:
      msg: "Hello there!"

actions:
  - template: github.com/chrisvdg/zrobot-helloworld/helloworld/0.0.1
    service: service1
    actions: ['write_msg_tmp']
```

Implementation:
```python
from js9 import j
from zerorobot.template.base import TemplateBase

class Helloworld(TemplateBase):
    version = '0.0.1'
    template_name = "helloworld"

    def __init__(self, name=None, guid=None, data=None):
        super().__init__(name=name, guid=guid, data=data)

    def write_msg_tmp(self, msg=None):
        if msg is not None:
            self.data['msg'] = msg

        if not self.data['msg']:
                self.data['msg'] = "No message was provided to service or function"
        
        j.sal.fs.writeFile("/tmp/msg.txt", self.data['msg'], append=False)
```

Schema:
```python
@0xcd6cbbc16ecee888;

struct Schema {
    msg @0: Text;
}
```

## Testing OVC templates

Also see input from e-katia/azmy: https://gist.github.com/katia-e/b9d317b19c329fe3e26be72b941fdd86

Use bootstrap-vm2:
```bash
ssh -A root@bootstrap.evolutief.be -p2200
```

Start a new tmux session:
```
tmux new -s 0-robot
#tmux a -t 0-robot
```

Create a new Git repository: https://docs.grid.tf/yves/zrobot2

Start a new Zero Robot:
```bash
#zrobot server start --listen :6600 --template-repo https://github.com/openvcloud/0-templates --data-repo ssh://git@docs.greenitglobe.com:10022/yves/zrobot3.git
zrobot server start -L :6000 -T git@github.com:openvcloud/0-templates.git -D git@github.com:yveskerwyn/zrobot.git
```

The above will:
- Use the current branch on openvcloud/0-templates.git
- Clone ssh://git@docs.grid.tf:7022/yves/zrobot2.git -> /opt/code/docs/yves/zrobot2
- Look for the JumpScale configuration repository (so you to create one first)

Split the tmux session window in two pane: `CTRL+B " ENTER`

Connect to the 0-robot:
```bash
zrobot robot connect main http://localhost:6000
```

Now let's create a blueprint: https://gist.github.com/katia-e/d52666a8c0b0ee8b56c7eee477c8bf83

```bash
mkdir /opt/var/blueprints
cd /opt/var/blueprints
vim bp.yaml
```

Here's the blueprint for creating a VDC:
```yaml
services:
    - github.com/openvcloud/0-templates/sshkey/0.0.1__bootstrap_vm2_key:
        path: "/root/.ssh/bootstrap_vm2_key"

    - github.com/openvcloud/0-templates/openvcloud/0.0.1__switzerland:
    
    - github.com/openvcloud/0-templates/vdc/0.0.1__myvdc:
        location: ch-g8-4

actions:
    - template: github.com/openvcloud/0-templates/vdc/0.0.1
        actions: ['install']
```

The above blueprint requires that there is already a `switzerland` config instance for OpenvCloud on the 0-robot host. In case you want to create a new config instance, you'll have to specify the config data for the new config instance:

```yaml
services:
    - github.com/openvcloud/0-templates/sshkey/0.0.1__bootstrap_vm2_key:
        path: "/root/.ssh/bootstrap_vm2_key"
        
    - github.com/openvcloud/0-templates/openvcloud/0.0.1__switzerland:
        address: "ch-gen-1.demo.greenitglobe.com"
    
    - github.com/openvcloud/0-templates/vdc/0.0.1__myvdc:
        location: ch-g8-4

actions:
    - template: github.com/openvcloud/0-templates/vdc/0.0.1
        actions: ['install']
```

Execute a blueprint:
```bash
zrobot blueprint execute bp.yaml
```

Check the result, listing the services that got created:

```bash
zrobot service list
```


````
ssh-add <key>
````

The above is required in order for 0-robot to work

How does the 0-robot know which the loaded keys to use?

```
zrobot robot list
```

in case you didn't yet load a ssh key you'll get an error

in case there was not js config repo configured you will be requested to configure/initialize it

> specify the key for encrypting the secret info, i.e. you take id_rsa > this will get copied into 
~/js9host/cfg/jumpscale9.cfg as a value for the sshkeyname in the section [myconfig]
> specify full name, email and login name > /opt/cfg/myconfig/j.tools.config/main.toml

```
zrobot robot connect main http://localhost:6000
```
=> a new config instance will get created for 0-robot: /opt/cfg/myconfig/j.clients.zrobot/main.toml

> use `js9_config configure -i elgouna -l j.clients.zrobot` to add as many 0-robot config instances

> you can also create a `./secureconfig` and `./key directort` in the current dir, in that case it will use that as a "sandbox" config manageger


```
js9_config configure -i elgouna -l j.clients.zrobot
```

```
zrobot robot blueprint execute
```

Tested blueprints:

Creating a user:
```yaml
services:
    - github.com/openvcloud/0-templates/openvcloud/0.0.1__myovc:
        location: 'be-g8-3'
        address: 'be-g8-3.demo.greenitglobe.com'
        token: '*************************************************************'
    - github.com/openvcloud/0-templates/vdcuser/0.0.1__adminss:
        openvcloud: myovc
        provider: itsyouonline
        email: test@g.com

actions:
      actions: ['install']
```

Creating an account (1):
```yaml
services:
    - github.com/openvcloud/0-templates/openvcloud/0.0.1__ovc:
        location: 'be-g8-3'
        address: 'be-g8-3.demo.greenitglobe.com'
        token: 'eyJhbGciOiJFUzM4NCIsInR5cCI6IkpXVCJ9.eyJhenAiOiJ3cnpjSXVyZzR6WEVqbEpPQTF6MENYODA2akt4IiwiZXhwIjoxNTIxMzAyNzUxLCJpc3MiOiJpdHN5b3VvbmxpbmUiLCJzY29wZSI6WyJ1c2VyOmFkbWluIl0sInVzZXJuYW1lIjoieHRyZW14In0._ix7xg5LzP0QoL4L89208OuZnBYIMSftFUigWZTwSiqdYSbT3P8kXunD_2ZqXj4OLDN_r59H6wSrTAutkmybQCaxI1drgP9vRv86PioZwEK4EuuUBUrSuIb_b8VbNrTO'
    - github.com/openvcloud/0-templates/account/0.0.1__adminDatasAccount:
        openvcloud: ovc

actions:
      actions: ['install']
```


Creating an account (2):
```yaml
services:
    - github.com/openvcloud/0-templates/openvcloud/0.0.1__ovc:
        location: 'be-g8-3'
        address: 'be-g8-3.demo.greenitglobe.com'
        token: 'eyJhbGciOiJFUzM4NCIsInR5cCI6IkpXVCJ9.eyJhenAiOiJ3cnpjSXVyZzR6WEVqbEpPQTF6MENYODA2akt4IiwiZXhwIjoxNTIxMzAyNzUxLCJpc3MiOiJpdHN5b3VvbmxpbmUiLCJzY29wZSI6WyJ1c2VyOmFkbWluIl0sInVzZXJuYW1lIjoieHRyZW14In0._ix7xg5LzP0QoL4L89208OuZnBYIMSftFUigWZTwSiqdYSbT3P8kXunD_2ZqXj4OLDN_r59H6wSrTAutkmybQCaxI1drgP9vRv86PioZwEK4EuuUBUrSuIb_b8VbNrTO'
    - github.com/openvcloud/0-templates/vdcuser/0.0.1__user1:
        provider: itsyouonline
        email: dslkadjaskl@g.com
        openvcloud: ovc
    - github.com/openvcloud/0-templates/account/0.0.1__newacc1:
        users:
            - name: adminRobot
              accesstype: CXDRAU
        openvcloud: ovc

actions:
      actions: ['install']
```

Creating a user (2):
```yaml
services:
    - github.com/openvcloud/0-templates/openvcloud/0.0.1__ovc:
        location: 'be-g8-3'
        address: 'be-g8-3.demo.greenitglobe.com'
        token: 'eyJhbGciOiJFUzM4NCIsInR5cCI6IkpXVCJ9.eyJhenAiOiJ3cnpjSXVyZzR6WEVqbEpPQTF6MENYODA2akt4IiwiZXhwIjoxNTIxMzAyNzUxLCJpc3MiOiJpdHN5b3VvbmxpbmUiLCJzY29wZSI6WyJ1c2VyOmFkbWluIl0sInVzZXJuYW1lIjoieHRyZW14In0._ix7xg5LzP0QoL4L89208OuZnBYIMSftFUigWZTwSiqdYSbT3P8kXunD_2ZqXj4OLDN_r59H6wSrTAutkmybQCaxI1drgP9vRv86PioZwEK4EuuUBUrSuIb_b8VbNrTO'
    - github.com/openvcloud/0-templates/vdcuser/0.0.1__newuser1:

actions:
      actions: ['install']
```

Then create new account, and add newly created user:
```yaml
services:
    - github.com/openvcloud/0-templates/openvcloud/0.0.1__ovc:
        location: 'be-g8-3'
        address: 'be-g8-3.demo.greenitglobe.com'
        token: 'eyJhbGciOiJFUzM4NCIsInR5cCI6IkpXVCJ9.eyJhenAiOiJ3cnpjSXVyZzR6WEVqbEpPQTF6MENYODA2akt4IiwiZXhwIjoxNTIxMzAyNzUxLCJpc3MiOiJpdHN5b3VvbmxpbmUiLCJzY29wZSI6WyJ1c2VyOmFkbWluIl0sInVzZXJuYW1lIjoieHRyZW14In0._ix7xg5LzP0QoL4L89208OuZnBYIMSftFUigWZTwSiqdYSbT3P8kXunD_2ZqXj4OLDN_r59H6wSrTAutkmybQCaxI1drgP9vRv86PioZwEK4EuuUBUrSuIb_b8VbNrTO'
    - github.com/openvcloud/0-templates/vdcuser/0.0.1__newuser1:
    - github.com/openvcloud/0-templates/account/0.0.1__newacc3:
        users:
            - name: newuser1
              accesstype: CXDRAU
        openvcloud: ovc

actions:
      actions: ['install']
```