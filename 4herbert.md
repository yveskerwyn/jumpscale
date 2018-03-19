Example Zero-Robot Blueprints

- [Virtual Datacenter](#vdc)
- [Kubernetes Cluster](#kubernetes)
- [Wordpress Site](#wordpress)

<a id="vdc"></a>
## Virtual Datacenter

Blueprint for creating your virtual datacenter:
```yaml
services:
    - github.com/openvcloud/0-templates/openvcloud/0.0.1__my-ovc:
        address: be-gen-1.demo.greenitglobe.com
        location: be-gen-1
        token: '***'

    - github.com/openvcloud/0-templates/account/0.0.1__my-account:
        openvcloud: my-ovc

    - github.com/openvcloud/0-templates/vdcuser/0.0.1__user-yves:
        openvcloud: my-ovc
        provider: itsyouonline
        email: yves@gig.tech
        
    - github.com/openvcloud/0-templates/vdc/0.0.1__my-vdc:
        account: my-account
        location: be-gen-1
        users:
            - name: user-yves
              accesstype: CXDRAU
    
actions:
    actions: ['install']
```

What you see:
- the value of `token` is an encrypted JSON Web token (JWT) representing an ItsYou.online user with access rights the OpenvCloud account that is labeled as `my-account`; the JWT is encrypted with the private key of the blueprint author
- with this `token` you get a client connection to the OpenvCloud system that is accessible on `https://be-gen-1.demo.greenitglobe.com`, labeled as `my-ovc`
- the third section of the blueprint defines an ItsYou.online user with username `yves`, labeled as `user-yves`; if the user doesn't exist yet it will be created
- the actual VDC is defined in the last section, providing admin access to the user `yves`, the VDC is labeled as `my-vdc` and can be referenced in all next blueprints


<a id="kubernetes"></a>
## Kubernetes Cluster

Creating a Kubernetes cluster hosted in the VDC `my-vdc` is done using the following blueprint:
```yaml
services:
  - github.com/openvcloud/0-templates/sshkey/0.0.1__my-key:
      passphrase: 'helloworld'

  - github.com/openvcloud/kubernetes/setup/0.0.1__my-kubenetes:
      vdc: my-vdc
      sshKey: my-key
      workers: 1

actions:
   - template: github.com/openvcloud/kubernetes/setup/0.0.1
     actions: ['install']
```

What you see:
- in the first section a SSH key is defined and label as `my-key`, this SSH key needed for accessing the virtual machines of the Kubernetes node
- the Kubernetes cluster itself is defined to be hosted in  `my-vdc` and only have one worker node

<a id="wordpress"></a>
# Wordpress Site


Creating a Kubernetes server in the VDC `my-vdc` is done using the following blueprint:
```yaml
services:
    vdc: my-vdc
    wordpress_title: "Herbert Blog"
    admin_user_: '***'
    admin_password_: '***'
    admin_email: "info@threefoldtoken.com"
    db_name: 'wordpress'
    db_user: 'admin'
    db_password_: "***"
    #wordpress_path: "/opt/var/data/www"
    plugins: 
        - 'rest-api'
        - 'rest-api-meta-endpoints'
        - "https://github.com/WP-API/Basic-Auth/archive/master.zip"

    s3_instance_name: 'my-s3'
    s3_object_name: 'Divi.zip'
    s3_bucket_name: 'itenv'
    
    #theme_path: '/tmp/themes/divi'
    thema_name: 'Divi'
    server_donain: 'herbert.b.grid.tf'

actions:
   - template: github.com/openvcloud/wordpress/setup/0.0.1
     actions: ['install']
```

What you see:
- the themes are fetched from the S3 deployment `my-s3`
- the attributes `wordpress_path` and `theme_path` are commented out to indicated that they are optional; when not specifying the defaults will be used