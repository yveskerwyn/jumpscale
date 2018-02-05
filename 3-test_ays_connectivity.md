# Test AYS Connectivity

```python
import os
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]

public_ays_url = "https://{}/api".format(caddy_domain)

ays_clients_org_name = "js9portal-mr4"
api_key_label = "install"

# Connect with a JWT using the API key set for the IYO organization:
iyo_user = j.clients.itsyouonline.get_user(application_id=app_id, secret=secret)
ays_clients_org = iyo_user.organizations.get(global_id=ays_clients_org_name)
ays_clients_org_api_key = ays_clients_org.api_keys.get(label=api_key_label)
ays_clients_org_secret = ays_clients_org_api_key.model["secret"]
ays = j.clients.ays.get(url=public_ays_url, client_id=ays_clients_org_name, client_secret=ays_clients_org_secret)

# Or connect using a JWT for a user that is member of the IYO organization instead:
scope = 'user:memberof:{}'.format(ays_clients_org_name)
jwt = j.clients.itsyouonline.get_jwt(client_id=app_id, secret=secret, scope=scope)
ays = j.clients.ays.get_with_jwt(url=public_ays_url, jwt=jwt)
ays.repositories.list()
```