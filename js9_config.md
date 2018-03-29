# Using js9_config

js9_config --help

js9_config sandbox
js9_config sandbox --help


js9_config init --help
js9_config init

js9_config init --silent
js9_config init -s

js9_config init -s --key
js9_config init -s -k


js9_config init -s -k --path
js9_config init -s -k -p


js9_config configure --help
js9_config configure --location j.clients.openvcloud --instance test /root/.ssh/id_rsa
js9_config configure --location j.clients.openvcloud --instance test --silent /root/.ssh/id_rsa
js9_config configure -l j.clients.openvcloud -i test -s /root/.ssh/id_rsa



js9_config get --location j.clients.sshkey
js9_config get -l j.clients.sshkey
js9_config get -l sshkey

(defaults to -i main)

js9_config get -l sshkey --instance main
js9_config get -l sshkey -i main

js9_config delete --help



js9_config reset
js9_config reset --help
js9_config reset --location j.clients.openvcloud --instance test
js9_config reset -l j.clients.openvcloud -i test
js9_config reset -l openvcloud -i test
js9_config reset --force
js9_config reset -f

js9_config test



##

In case you didn't yet load any SSH key:

```bash
js9_config init
* found 0 keys from ssh-agent
* JS9 init: ssh key found in agent is:'/root/.ssh/id_rsa'
* JS9 init: Is it ok to use this one:'/root/.ssh/id_rsa'?
 [y/n]: y
* JS9 init: Do you want to use a git based CONFIG dir, y/n?
 [y/n]: n
* JS9 init: will create config dir in '/opt/cfg/myconfig/', your config will not be centralised! Is this ok?
 [y/n]: y
* JS9 init: ssh key found in agent is:'/root/.ssh/id_rsa'
* JS9 init: Is it ok to use this one:'/root/.ssh/id_rsa'?
 [y/n]:
```


What happens:
- JumpScale first sees that no SSH keys is loaded
- It then sees that there is one SSH key in `~/.ssh`
- You are then asked whether it is OK to load that SSH key
- The SSH key gets loaded; at this point you might need to enter the passphrase
- Next JumpScale look for an existing configuration repository
    - First it will check the value of `cfg["myconfig"]["path"]`
    - Because no value was set for the `path` key JumpScale will scan all your Git repositories under $CODEDIR (`j.core.state.config_js["dirs"]["CODEDIR"]`, typically `/opt/code`) 
- In this case no existing configuration repository was found, so JumpScale will want to create one for you
- You offered to create a Git repository 
- Here we choose to create the configuration repository in a regular directory
- JumpScale asks for confirmation, reminding you that it will not be "centralized", meaning that configuration data can not be pushed to a Git server
- The configuration repository gets created in the default `/opt/cfg/myconfig/`
    - JumpScale will now update the value of the `path` key in the JumpScale configuration file 
- JumpScale then checks the loaded SSH keys, it finds the the one which just got loaded, it will need that
    - JumpScale will now update the value of the `sshkeyname` key in the JumpScale configuration file 
- Next JumpScale will create two configuration instances:
    - `/opt/cfg/myconfig/j.clients.sshkey/id_rsa.toml`
    - `/opt/cfg/myconfig/j.tools.myconfig/main.toml`, for which you will need to provide values for `email`, `fullname` and `login_name`


Alternatively you can choose for a Git repository instead of a regular repository:
```bash
js9_config init
* found 0 keys from ssh-agent
* JS9 init: ssh key found in agent is:'/root/.ssh/id_rsa'
* JS9 init: Is it ok to use this one:'/root/.ssh/id_rsa'?
 [y/n]: y
* JS9 init: Do you want to use a git based CONFIG dir, y/n?
 [y/n]: y
* JS9 init: Specify a url like: 'ssh://git@docs.grid.tf:7022/despiegk/config_despiegk.git'
url: ssh://git@docs.greenitglobe.com:10022/yves/jsconfig.git
* None:pull:ssh://git@docs.greenitglobe.com:10022/yves/jsconfig.git ->/opt/code/docs/yves/jsconfig
* git clone ssh://git@docs.greenitglobe.com:10022/yves/jsconfig.git -> /opt/code/docs/yves/jsconfig
* mkdir -p /opt/code/docs/yves;cd /opt/code/docs/yves;git -c http.sslVerify=false clone   ssh://git@docs.greenitglobe.com:10022/yves/jsconfig.git /opt/code/docs/yves/jsconfig
* JS9 init: ssh key found in agent is:'/root/.ssh/id_rsa'
* JS9 init: Is it ok to use this one:'/root/.ssh/id_rsa'?
 [y/n]: y
```

The above will only work if the SSH key that is loaded was associated with your Git server account.

> As it turns out, after a successful initialization no SSH key is loaded. That is because if you didn't export the `SSH_AUTH_SOCK` variable then JumpScale will create a socket file under `/tmp/sshagent_socket`. if you `export SSH_AUTH_SOCK=/tmp/sshagent_socket` and then try to do `ssh-add -l` then it will work:

