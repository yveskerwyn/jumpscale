# Docker setup


- [ItsYou.online](#iyo)
- [Set variables](#vars)
- [Create Docker containers](#create-containers)
- [Add the AYS templates](#ays-templates)


<a id="iyo"></a>
## ItsYou.online

Create an ItsYou.online organization, and create an API access key:
- Callback URL http://<public-ip-address>:8200
- Check the option to enable client credentials grant type usage


<a id="vars"></a>
## Set variables

```bash
export js_branch="9.2.1"
export docker_hub_username="yveskerwyn"
export iyo_organization="<check ItsYou.online>"
export external_ip_address="195.134.212.26"
export redirect_url="http://$external_ip_address:8200"
export portal_client_id=iyo_organization
export portal_secret="<check ItsYou.online>"
```

<a id="create-containers"></a>
## Create containers

Install Docker: https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce-1

In what follows we will create a Docker container for:
- [AYS Server](#ays-container)
- [AYS Portal](portal-container)
- [Reverse-proxy Server](#caddy)


<a id="ays-container"></a>
### Create and start an AYS Server container

Run (= create + start) a Docker container with the AYS server image:
```bash
docker run -d --name ays-server -p "5000:5000" -e organization=$iyo_organization -e external_ip_address=$external_ip_address $docker_hub_username/ays_server:$js_branch
```

Check the container:
```bash
docker exec -it ays-server bash
```

Check that AYS running in a TMUX session:
```bash
tmux at
```

(`CTRL+B D` to exit the TMUX session)


<a id="portal-container"></a>
### Create an AYS Portal container

First find out what the private IP address is of your AYS server container:
```bash
docker inspect -f '{{ .NetworkSettings.IPAddress }}' ays-server
```

This IP address needs to be passed as the value for the environment variable `ays_internal_ip_address` when creating the AYS portal container, here below we specified `172.17.0.2`.

Run (= create + start) a Docker container with the AYS portal image:
```bash
export ays_internal_ip_address="172.17.0.2"
docker run -d --name ays-portal -p "8200:8200" -e ays_internal_ip_address=$ays_internal_ip_address -e external_ip_address=$external_ip_address -e portal_client_id=$portal_client_id -e portal_secret=$portal_secret -e redirect_url=$redirect_url -e organization=$iyo_organization $docker_hub_username/ays_portal:$js_branch
```

> Make sure to update the callback URL in ItsYou.online.

Check the container:
```bash
docker exec -it ays-portal bash
```

<a id="caddy"></a>
### Create and start Docker container for Caddy

@TODO


<a id="ays-templates"></a>
## Add the AYS templates

In order for AYS to process OpenvCloud blueprints, you first need to upload the OpenvCloud templates.

Adding the AYS templates for OVS, using curl:
```bash
curl --request POST \
  --url http://185.15.201.111:5000/ays/template_repo \
  --header 'authorization: Bearer eyJhbGciOiJFUzM4NCIsInR5cCI6IkpXVCJ9.eyJhenAiOiJlMnpsTi03U0M2N3RhdjN0UlJuZG9VQUd4a1U1IiwiZXhwIjoxNTIwNjA4MzIyLCJpc3MiOiJpdHN5b3VvbmxpbmUiLCJzY29wZSI6WyJ1c2VyOm1lbWJlcm9mOmF5cy1vcmdhbml6YXRpb25zLmRvY2tlci1vbi1tYWMiXSwidXNlcm5hbWUiOiJ5dmVzIn0.PgeqU9Ndm3_MAXM1WiMVd6WVtxSbdVCxUW--MA4FPmvYOMFg5d2cWPF_bSO1F48aRvS5rG6o9ayLmpM44thIeKlYhvaFARv5khjijt_2hdSll7ciY2LY75NOp26syZ3q' \
  --header 'content-type: application/json' \
  --data '{"url":"https://github.com/openvcloud/ays_templates","branch": "master"}'
```

Or for the partnerportal templates: `{"url": "https://github.com/yveskerwyn/ays_partnerportal_templates", "branch": "9.2.1"}`