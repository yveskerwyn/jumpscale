# Setup a Kubernetes cluster

Create a new VM.

Install JumpScale
```bash
curl https://raw.githubusercontent.com/Jumpscale/bash/development/install.sh?$RANDOM > /tmp/install.sh
bash /tmp/install.sh
source ~/.bashrc
ZInstall_host_js9_full
```

Or use:
- https://github.com/Jumpscale/0-robot/blob/master/utils/scripts/jumpscale_install.sh, which will also create a SSH key and     initialize JumpScale config manager on your host


Install Docker:
```python
j.tools.prefab.local.virtualization.docker.install()
```

Create a SSH key:
```bash
ssh-keygen -t rsa
```

Create private Git repositories, make sure to initialize them   :
- data repository
- config repository

Make sure `id_rsa.pub` created above is add to the Git organization where you created the data and config repositories.

Create a container:
```bash
config_repo="ssh://git@docs.greenitglobe.com:10022/yves/jsconfig.git"
data_repo="ssh://git@docs.greenitglobe.com:10022/yves/robotdata.git"
template_repo="ssh://git@github.com:openvcloud/0-templates.git, git@github.com:openvcloud/kubernetes.git"
internal_ip_address="$(ip route get 8.8.8.8 | awk '{print $NF; exit}')"
port_forwarding="$internal_ip_address:8000:6600"

#docker run --name zrobot -e data_repo=$data_repo -e config_repo=$config_repo -p $port_forwarding -v /root/.ssh:/root/.ssh -e auto_push=1 -e auto_push_interval=30 jumpscale/0-robot
docker run --name zrobot -e data_repo=$data_repo -e config_repo=$config_repo -p $port_forwarding -v /root/.ssh:/root/.ssh jumpscale/0-robot
```

In the above approach we mount `/root/.ssh`, making you SSH keys available in the Docker container. When the container starts it will check for `id_rsa.pub`.

In case you don't mount ... 

From the host, connect to the 0-robot:
```bash
zrobot robot connect main http://localhost:8000
```

Create following blueprint:
```yaml
services:
    - github.com/openvcloud/0-templates/openvcloud/0.0.1__be-gen-1:
        address: 'be-gen-1.demo.greenitglobe.com'
        location: 'be-gen-1'
        token: 'eyJhbGciOiJFUzM4NCIsInR5cCI6IkpXVCJ9.eyJhenAiOiJlMnpsTi03U0M2N3RhdjN0UlJuZG9VQUd4a1U1IiwiZXhwIjoxNTIxNDQ2OTQ5LCJpc3MiOiJpdHN5b3VvbmxpbmUiLCJzY29wZSI6WyJ1c2VyOmFkbWluIl0sInVzZXJuYW1lIjoieXZlcyJ9.qHv3EzSgXU7DBifuGRXCv5YWZb7B42dMFONJEvq4qGMVGmPLD138d6LYheeJaBk-Lplzhy_EGd1C4gDBeB3w9xuJBft_XSR_A3cEyGbO2qY4RptSJ88hkxllMRJE74G_'

    - github.com/openvcloud/0-templates/account/0.0.1__gigdevelopers:
        openvcloud: be-gen-1

    - github.com/openvcloud/0-templates/vdcuser/0.0.1__yves:
        openvcloud: be-gen-1
        provider: itsyouonline
        email: yves@gig.tech
        
    - github.com/openvcloud/0-templates/vdc/0.0.1__kuberyves:
        account: gigdevelopers
        location: be-gen-1
        users:
            - name: yves
              accesstype: CXDRAU
    
actions:
    actions: ['install']
```


Execute the blueprint:
```bash
zrobot blueprint execute bp.yaml
```



```yaml
services:
  # - github.com/openvcloud/0-templates/sshkey/0.0.1__mykey:
  #     passphrase: 'hello'

 
  - github.com/openvcloud/kubernetes/setup/0.0.1__k8s:
      vdc: kuberyves
      sshKey: mykey
      workers: 1

actions:
  # - template: github.com/openvcloud/0-templates/account/0.0.1
  #   actions: ['install']

  # - template: github.com/openvcloud/0-templates/vdcuser/0.0.1
  #   actions: ['install']

  - template: github.com/openvcloud/0-templates/vdc/0.0.1
    actions: ['install']

  # - template: github.com/openvcloud/0-templates/zrobot/0.0.1
  #   actions: ['install']
  - template: github.com/openvcloud/kubernetes/setup/0.0.1
    actions: ['install']
```