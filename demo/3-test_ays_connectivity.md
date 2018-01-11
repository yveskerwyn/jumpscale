# Test AYS Connectivity

```python
import os
app_id = os.environ["APP_ID"]
secret = os.environ["SECRET"]

public_ays_url = "http://185.193.143.74:5000"

ays_clients_org_name = "js9portal-mr4"
api_key_label = "install"

# Connect with a JWT using the API key set for the IYO organization:
iyo_user = j.clients.itsyouonline.get_user(app_id, secret)
ays_clients_org = iyo_user.organizations.get(ays_clients_org_name)
ays_clients_org_api_key = ays_clients_org.api_keys.get(api_key_label)
ays_clients_org_secret = ays_clients_org_api_key.model["secret"]
ays = j.clients.ays.get(public_ays_url, ays_clients_org_name, ays_clients_org_secret)

# Or connect using a JWT for a user that is member of the IYO organization instead:
scope = 'user:memberof:{}'.format(ays_clients_org_name)
jwt = j.clients.itsyouonline.get_jwt(app_id, secret, scope=scope)
ays = j.clients.ays.get_with_jwt(public_ays_url, jwt)

ays.repositories.list()
```