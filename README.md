# Kong Gateway with Keycloak OIDC

## Installation

Create in the root directory of this project a file named `.env` containing the following:

```
DB_HOST="{AN IP address or domain}"
DB_PORT=25060
KEYCLOAK_DB_USER=keycloak
KEYCLOAK_DB_PASSWORD=********
KEYCLOAK_DATABASE=keycloak

KEYCLOAK_USER=admin
KEYCLOAK_PASSWORD="admin"
```

run the following
```
docker-compose run --rm kong kong migrations bootstrap
docker-compose run --rm kong kong migrations up
docker-compose --env-file .env up -d
```

### Issues
If you see on the keycloak login page `https required` and yu need to login, launch in the db
```
update REALM set ssl_required='NONE' where id = 'master';
```

## Configuration

### Keycloak Configuration
Once the containers are running, you need to start configuring your Keycloak server. The first step is to login
with the default credentials (admin/admin) to the administration console. Then you need to:
- Create Client:
    - Menu `Clients` > click `Create` button
    - Client ID: kong
    - Click `Save` button.
    - In **Settings** tab:
        - Access type: `Confidential`
        - Root URL: `http://localhost:8000`
        - Valid Redirect URL: `/mock/*`
        - Click `Save` button.
    - In **Credentials** tab:
        - Copy the Secret value (You will need it to configure the Kong plugin)

- Create User 
    - Menu `Users` > click `Add user` button
    - Username: user, enable the Email verified switch.
    - Click `Save` button.
    - In **Credentials** tab:
        - Set password, disable the Temporary credentials switch.
        - Click `Reset Password` button.
    
## Find your local IP
1. On linux type on a shell `hostname -I`
2. probably the first ip that shows up is the correct one.

3. To check if it is correct copy the ip, go into a browser and type
```
http://{The ip you copied}:8180
```

4. If the keycloak page shows up the IP is the correct one, in other case, just try with the others ip

## Konga Configuration
Open up your browser and go to http://localhost:1337
1. Create an account (It's just a local account)
2. Login
3. Add new connection:
- Name: `default`
- Kong Admin URL: `http://kong:8001/`
- Then save
4. Go To services and add a new service:
- Name: `Userinfo`
- Tags `keycloak` (Hit enter to save the tag)
- Url: `http://{your local ip}:8180/auth/realms/master/protocol/openid-connect/userinfo`
- Save
5. Go To services and add a new service:
- Name: `mock-request`
- Tags `testing` (Hit enter to save the tag)
- Url: `http://mockbin.org/request`
- Save
6. Go To services and click on `userinfo`:
- Click on Routes and add a new route
- Name: `Userinfo`
- Path: `/userinfo` (Hit enter to save the tag)
- Save
7. Go To services and click on `mock-request`:
- Click on Routes and add a new route
- Name: `Mock`
- Path: `/mock` (Hit enter to save the tag)
- Save

## Oidc Configuration
1. from a shell type 
- The client secret is the one you copied during the "Keycloak configuration"
- The local ip is the one you copied during  "Find your local IP"
```
curl -s -X POST http://localhost:8001/plugins \
  -d name=oidc 
  -d config.client_id=kong 
  -d config.client_secret=${CLIENT_SECRET} 
  -d config.discovery=http://${your local ip}:8180/auth/realms/master/.well-known/openid-configuration 
  -d config.session_name=MY_SESSION
```

## Redis cache configuration (Still Work In Progress)
```
curl -s -X PATCH http://localhost:8001/plugins/{kong_oidc_plugin_id}} 
          -d name=oidc 
          -d config.client_id=kong 
          -d config.client_secret=${CLIENT_SECRET} 
          -d config.discovery=http://${your local ip}:8180/auth/realms/master/.well-known/openid-configuration 
          -d config.introspection_endpoint=http://${your local ip}:8180/auth/realms/master/protocol/openid-connect/token/introspect 
          -d config.introspection_cache_ignore=false 
          -d config.session_name=MY_SESSION 
          -d config.session_storage=redis 
          -d config.session_redis_host=redis 
          -d config.session_redis_port=6379
```

#### For development purposes
```
docker-compose down
docker rmi kong-oidc_kong
docker-compose up -d
```

## Acknowledgment
This project is inspired by the articles Securing APIs with Kong and Keycloak. Find available the 
[part 1](https://www.jerney.io/secure-apis-kong-keycloak-1/) and 
[part 2](https://www.jerney.io/secure-apis-kong-keycloak-2/).








