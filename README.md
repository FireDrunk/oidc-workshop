> Requirements:
> 
> `docker` & `docker-compose` & `Postman` have to be installed.

## Assignment 1 - Create a Client:
### Important!: Shutdown all other instances of keycloak.

Start Keycloak & Grafana using Docker Compose.

Execute `docker-compose up -d` in this directory.
This will start up a Keycloak and Grafana instance on `localhost`.

- Go to the Keycloak interface at http://localhost:8080
- Login using `admin/admin`
- Go to `Clients` in the Configure menu on the left.
- Create a new Client, Choose `openid-connect` (the default) as the Client Protocol, and set the root URL to `http://localhost:3000`
- Scroll down to the `Access Type` field, and set that to `confidential`
- Press Save
- Go to the Credentials Tab, take not of the secret.

Go back to `Realm Settings` using the left Configure menu.  
Click on the `OpenID Endpoint Configuration` button at the bottom `Endpoints` section on the page.

The `.well-known/.openid-configuration` page of the correct Realm should open.  
This page is formatted in JSON and contains the configuration parameters of the Workshop realm.

Try to find the URLs for
- `authorization_endpoint` (ends with `/auth`)
- `token_endpoint` (ends with `/token`)
- `userinfo_endpoint` (ends with `/userinfo`)

## Assignment 2 - Postman
Open Postman and add a new Request.
- Enter `http://localhost:3000` as the address, set the method to POST.
- Select the `Authorization` tab and select `OAuth 2.0` as the Type.
- Set the following options:
  - Grant Type: `Authorization Code`
  - Callback URL: `http://localhost:3000`
  - Deselect `Authorize using browser`
  - Auth URL: Enter the /auth url you've collected earlier.
  - Access Token URL: Enter the /token url you've collected earlier.
  - Client ID: `<the-name-you-just-used>`
  - Client Secret `The secret of the Client you've noted`
  - Scope: `openid`
  - State: leave default
  - Client Authentication: `Send as Basic Auth header`

Click on "Get New Access Token" at the bottom of the page.
A popup should open with the login to Keycloak. 
Login to Keycloak with any credential from the previous assignments.

Postman should show a green succeeded icon if you've configured the Client correctly.
You will see a large string in the top of the page containing the token.
Take note of the token.

## Assignment 3 - Inspect the token
Copy this token,
Goto https://jwt.ms and inspect the token.

- Can you find the Issuer (the `iss` field) of this token?
- Can you see which _scopes_ we were given?

## Assignment 4 - Reconfigure Grafana
We want to reconfigure Grafana to use OAuth.  
To do this, we have to give Grafana the correct configuration parameters in a file called `grafana.ini`.
Your current Docker Compose setup has already started a Grafana instance that does not use OAuth.
Verify that it started correctly using the url `http://localhost:3000` and logging in using `admin/admin`.

In this directory update the `grafana.ini` file with the correct parameters you've collected.
This should include:
- `client_id` (the name of the client)
- `client_secret` (the secret you copied from the Credentials tab)
- `auth_url`
- `token_url`
- `api_url` (this points to the userinfo endpoint)
- `scopes` (the default is `openid`)

After changing the values, restart grafana using: `docker-compose restart grafana`.
> Do not restart your entire stack, you might kill Keycloak, which is configured with an in-memory database

Browse to `http://localhost:3000` and try to login using the `Login using OAuth` button at the bottom.
If you can log in successfully, you'll see the proper username in the top right corner of Grafana.

Can you find which permissions you were granted by Grafana in your Profile page?

## Assignment 5 - Use Roles (optional)
In the last assignment you've probably noticed you were not given many permissions by Grafana.  
To let Grafana grant you specific permissions, we need to configure a few more things.

- Create Roles in Keycloak specifically for Grafana.
- Enable Authorization on the Client to allow this information to be exposed in the Token(s).
- Configure Grafana to request those Roles when requesting a Token.
- Configure Grafana to grant permissions when it detects those Roles in the token.

First, we need to create the Roles.
- Go to the Client Configuration tab in Keycloak
- Open the `Roles` tab, and add 2 roles called `grafana-admin` and `grafana-editor`
- Go back to the `Mappers` tab and click "Add Builtin"
- Check the `client roles` and click `Add selected`
- Click on the new entry `client roles` and set:
  - Client: `<your client>`
  - Token Claim Name: `grafana_roles`
  - Enable "Add claim to ID Token"
  - Press "Save"

Go to Postman, and request a new Token the same way you did in Assignment 2.
When you check the token in `https://jwt.ms` you'll notice that nothing (much) has changed.

That's because we did not yet grant the user this Role!

- Go to Keycloak -> Users -> Click on `View all users`.
- Select the user you've been using, and open the `Role Mappings` tab.
- Under the `Client Roles` find your Client
- Select the `grafana-admin` role you just created, and click `Add selected`.


Repeat your Postman step, and inspect the token.

There should be a new/changed section in the token containing:
```json
{
  "grafana_roles": [
    "grafana-admin"
  ]
}
```
This means you've granted your self Grafana Admin permissions inside Keycloak!  
To make Grafana use this information, we need to tell it to inspect the token for that specific information.
We can do that by adding an option called `role_attribute_path` to the grafana.ini.

Use the following syntax:
```
role_attribute_path = contains(grafana_roles[*], 'role-name') && 'Admin' || contains(grafana_roles[*], 'role-name') && 'Editor' || 'Viewer'
```
But change the names of the roles to the one you created.

As you can see, this maps the `grafana_roles` part of the JSON to specific role in Grafana itself!
The `grafana_roles` value inside the token is called a `claim`.

If you log in to grafana again, you should now see that you have Admin permissions!
Verify that your Editor role works, by granting another user the `grafana-editor` role.
