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
template_repo="https://github.com/openvcloud/0-templates.git,https://github.com/openvcloud/kubernetes.git"
internal_ip_address="$(ip route get 8.8.8.8 | awk '{print $NF; exit}')"
port_forwarding="$internal_ip_address:8000:6600"

#docker run --name zrobot -e data_repo=$data_repo -e config_repo=$config_repo -e template_repo=$template_repo -p $port_forwarding -v /root/.ssh:/root/.ssh -e auto_push=1 -e auto_push_interval=30 jumpscale/0-robot
docker run --name zrobot -e data_repo=$data_repo -e config_repo=$config_repo -e template_repo=$template_repo -p $port_forwarding -v /root/.ssh:/root/.ssh jumpscale/0-robot
```

In the above approach we mount `/root/.ssh`, making your SSH keys available in the Docker container. When the container starts it will check for `id_rsa.pub`. If it finds `id_rsa.pub` 0-robot will use this key to push to your data and configuration repositories, so make sure that this key is registered on your Git server.

In case you don't mount `/root/.ssh`, the container will generate a new `ìd_rsa` key and exit with the value of `id_rsa.pub`. Register this public key in your Git server, so 0-robot can push changes to your data and configuration repositories. Once done so, start the container:
```bash
docker start zrobot
```

From the host, connect to the 0-robot:
```bash
zrobot robot connect main http://$internal_ip_address:8000
```

Check:
```bash
zrobot robot list
zrobot robot current
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



Next let's deploy the Kubernetes cluster in the VDC, here's the blueprint:
```yaml
services:
  - github.com/openvcloud/0-templates/sshkey/0.0.1__mykey:
      passphrase: 'helloworld'

  - github.com/openvcloud/kubernetes/setup/0.0.1__k8s:
      vdc: kuberyves
      sshKey: mykey
      workers: 1

actions:
   - template: github.com/openvcloud/kubernetes/setup/0.0.1
     actions: ['install']
```


Execute the blueprint:
```bash
zrobot blueprint execute kuber.yaml
```

List the services:
```bash
zrobot service list
github.com/openvcloud/0-templates/account/0.0.1 - 4e16bfb7-cc05-4469-a2fe-ba9fd687d157 - gigdevelopers
github.com/openvcloud/0-templates/disk/0.0.1 - 1ed597bf-1fcd-49e5-89da-13649ad76db9 - Disk5186
github.com/openvcloud/0-templates/disk/0.0.1 - 39ccf0e1-c0fc-4743-9d9c-13dfd0624a9d - Disk5185
github.com/openvcloud/0-templates/node/0.0.1 - dc80ee7e-51ab-4131-88e0-ec11db6e05ec - k8s-little-helper
github.com/openvcloud/0-templates/openvcloud/0.0.1 - 30d2d0da-dfb0-4dca-b9aa-f4c4655fb1f4 - be-gen-1
github.com/openvcloud/0-templates/sshkey/0.0.1 - 3edf611d-f447-4045-88c3-ea62d0cfaf6c - mykey
github.com/openvcloud/0-templates/vdc/0.0.1 - aa7c9689-1d26-4277-8d30-9ec20f9d5c8a - kuberyves
github.com/openvcloud/0-templates/vdcuser/0.0.1 - 94135d4c-1310-4225-880c-81e0a9f85871 - yves
github.com/openvcloud/0-templates/zrobot/0.0.1 - 446feaf0-7423-41b3-aab8-c43eb4cc250b - k8s-little-bot
github.com/openvcloud/kubernetes/setup/0.0.1 - 15b8c2fa-06fb-49e8-8102-b75fc02db5cb - k8s
```

This will create three virtual machines:
```
worker-1
master
k8s-little-helper
```