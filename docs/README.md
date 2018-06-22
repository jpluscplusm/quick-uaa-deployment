# Playtime with UAA

## Install uaa CLI

There is a nice new [UAA CLI](https://github.com/cloudfoundry-incubator/uaa-cli) `uaa` that interacts with the UAA and returns JSON.

At the time of writing, the only method for installation is:

```
go get -u github.com/cloudfoundry-incubator/uaa-cli
cd $GOPATH/src/github.com/cloudfoundry-incubator/uaa-cli
make && make install
uaa -h
```

## Spin up UAA

See https://github.com/starkandwayne/uaa-deployment to deploy UAA locally via VirtualBox or to any cloud:

```
uaa-deployment up
```

Once our UAA is running we can view the target URL and some admin-level authentication:

```
uaa-deployment info
```

The output might look like:

```
UAA:
  url: https://192.168.50.6:8443
  client: uaa_admin
  client_secret: nnb2tbev0j82gxdz65xc
  username: admin
  password: 2rbaswzllkuy51ymzahz
```

Visit the URL and you will be redirected to the `/login` page:

![uaa-web-user-login](images/uaa-web-user-login.png)

Try to login with both pairs of credentials - client/client_secret, and username/password.

You will discover that you cannot authenticate with client/client_secret. These are not user credentials.

Rather they are credentials for an application to talk to the UAA API - either on behalf of itself (say, to register new UAA users or new UAA clients), or on behalf of a UAA user (say, to ask for the user's personal information, or their authorized permissions within the organization).

Succesfully logging in with the `admin` username/password will look like:

![uaa-web-user-success](images/uaa-web-user-success.png)

You'll see a corporate logo (the default is "Cloud Foundry"), the title "Where to?", and .... a void of emptiness. This homepage of the UAA can be filled with "tiles" - icons/names for your internal corporate applications that users can possible use. We will revisit this later.

## Authenticating a third-party application

The web interface above is the UAA itself. As a user (such as `admin`), you can login. That is, you can authenticate that you are `admin`, or later when we create more users you can authenticate that you are one of those users. Conceptually, each human knows how to authenticate as a single UAA user.

The `uaa` CLI is a separate application from the UAA. The UAA refers to third-party applications as "clients". We want to configure this separate application to target our UAA, to authenticate as a valid application, and to interact with the UAA's API.

That the `uaa` CLI has a similar name to the UAA API server is confusing. The `uaa` CLI is just one of infinite applications that might want to interact with the UAA API - either directly as a client, or on behalf of the UAA's users.

For convenience the `uaa-deployment auth` command will target and authentication with your UAA:

```
uaa-deployment auth-client
```

Alternately, you can run the `uaa` commands directly:

```
uaa target https://192.168.50.6:8443 --skip-ssl-validation
uaa get-client-credentials-token uaa_admin -s nnb2tbev0j82gxdz65xc
```

This time we use the client/client_secret values. We are giving the `uaa` application permission to interact with the UAA API as a peer: one application talking to another application.

You can now use the `uaa` CLI to interact with your UAA.

To see a list of user accounts in JSON:

```
uaa list-users
```

You can filter the attributes of each user:

```
uaa list-users --attributes id,userName
```

To see the various scopes that the `admin` user is allowed to access:

```
uaa list-users | jq -r ".resources[] | select(.userName == \"admin\").groups[].display"
```

The output might look like:

```
user_attributes
roles
scim.me
uaa.user
cloud_controller.write
cloud_controller.read
approvals.me
openid
uaa.offline_token
password.write
bosh.admin
oauth.approvals
profile
notification_preferences.write
notification_preferences.read
cloud_controller_service_permissions.read
```

To see the list of groups in JSON:

```
uaa list-groups
uaa list-groups | jq -r ".resources[].displayName" | sort
```

As a logged in user - by any of the authentication commands - we potentially can ask the UAA "who am I?"

To see a list of application clients in JSON:

```
uaa list-clients

uaa list-clients | jq -r ".[] | {client_id, authorized_grant_types, scope}"
```

We've introduced one client so far - `uaa_client` - which we're using currently to allow the `uaa` CLI application to talk to the UAA API. To view its configuraton:

```
uaa get-client uaa_admin
```

The output might look like:

```json
{
  "client_id": "uaa_admin",
  "scope": [
    "uaa.none"
  ],
  "resource_ids": [
    "none"
  ],
  "authorized_grant_types": [
    "client_credentials"
  ],
  "authorities": [
    "clients.read",
    "password.write",
    "clients.secret",
    "clients.write",
    "uaa.admin",
    "scim.write",
    "scim.read"
  ],
  "lastModified": 1529652956499
}
```

We have authenticated our `uaa` application so we should be able to ask "who am I?":

```
uaa userinfo
```

This command fails, so run the command again with `--verbose` to see the error message:

```
uaa userinfo --verbose
```

The output will include an error:

```
GET /userinfo?scheme=openid HTTP/1.1
...
{"error":"access_denied","error_description":"Invalid token does not contain resource id (openid)"}
```

The reason we cannot invoke `uaa userinfo` is that we are currently authenticated as a client, not on behalf of a user.

That is, the `uaa` application is talking directly to the UAA - one application to another. Peers.

The `uaa userinfo` command assumes that the `uaa` application is talking to the UAA on behalf of an authenticated user.

## Creating users

```
uaa create-user drnic \
  --email drnic@starkandwayne \
  --givenName "Dr Nic" \
  --familyName "Williams" \
  --password drnic_secret
```

Once created, we can lookup the user with their username:

```
uaa get-user drnic
```

The JSON output might be similar to:

```json
{
  "id": "87fde4a5-17f3-4667-a5e2-fff62220c73e",
  "meta": {
    "created": "2018-06-22T09:27:10.655Z",
    "lastModified": "2018-06-22T09:27:10.655Z"
  },
  "userName": "drnic",
  "name": {
    "familyName": "Williams",
    "givenName": "Dr Nic"
  },
  "emails": [
    {
      "value": "drnic@starkandwayne",
      "primary": false
    }
  ],
  "groups": [
    {
      "value": "5a201c79-3265-46a8-873d-8631facdb2a1",
      "display": "user_attributes",
      "type": "DIRECT"
    },
    {
      "value": "b07d8fda-aaba-4f3e-9f5c-dca9f7c99e9f",
      "display": "roles",
      "type": "DIRECT"
    },
...
```

Visit the UAA web UI, logout from `admin` user, and login as `drnic`:

![uaa-web-normal-user-login](images/uaa-web-normal-user-login.png)

In the top right corner, select "Account Settings" and note that the new user has not yet granted any third-party client applications permission to access their UAA account:

![uaa-web-user-profile](images/uaa-web-user-profile.png)

## Authenticating on behalf of a user - via local password

For the `uaa` CLI application to act on behalf of a user - with the permission of the user - then user will need to authenticate themselves. That is, to prove that they are who they claim to be. The simplest method is for a user to provide the secret password for their UAA user account.

In this section, the user will give their username (who they claim to be) and their password (their proof that it is them) to the `uaa` client application, rather than to the UAA.

The `uaa` CLI will forward the username/password to the UAA API to get authorization to act on behalf of the user.

For a UAA client to be allowed to authorize users with the UAA it needs a UAA client to exist with an `authorized_grant_types` list that includes `password`.

We can use the `uaa` - authenticated as the `uaa_client` client - to create a new UAA client:

```
uaa-deployment auth-client
uaa create-client our_uaa_cli -s our_uaa_cli_secret \
  --authorized_grant_types password,refresh_token \
  --scope "openid"  \
  --authorities uaa.none \
  --access_token_validity 120 \
  --refresh_token_validity 86400
```

A user can now provide the `uaa` CLI permission to interact with the UAA on its behalf:

```
uaa get-password-token uaa_cli -s uaa_cli_secret -u drnic -p drnic_secret
```

To demonstrate that the `uaa` is now operating on behalf of the `drnic` user:

```
uaa userinfo
```

The JSON output will be like:

```json
{
  "user_id": "87fde4a5-17f3-4667-a5e2-fff62220c73e",
  "sub": "87fde4a5-17f3-4667-a5e2-fff62220c73e",
  "user_name": "drnic",
  "given_name": "Dr Nic",
  "family_name": "Williams",
  "email": "drnic@starkandwayne",
  "phone_number": null,
  "previous_logon_time": 1529661057132,
  "name": "Dr Nic Williams"
}
```

That is, the `uaa` CLI has authenticated as `drnic` user, and is authorized to look up that user's personal information. User Authentication & Authorization. UAA. Boomshakalaka.

## Authenticating on behalf of a user - via web UI

The risk of a user providing their username/password to a UAA third-party client application is that the client application stores the raw username/password and reuses them later without the user's permission.

This risk can be avoided by the third-party client delgate the login process to the UAA user interface.

First, register a new UAA client that is designed to allow the `uaa` to be used by normal UAA users.

```
uaa-deployment auth-client
uaa create-client uaa-cli-authcode -s uaa-cli-authcode \
  --authorized_grant_types authorization_code,refresh_token \
  --redirect_uri http://localhost:9876 \
  --scope openid
```

Now, to allow a user to authenticate:

```
uaa get-authcode-token uaa-cli-authcode -s uaa-cli-authcode --port 9876
```

```
Launching browser window to https://drnic-uaa.starkandwayne.com:8443/oauth/authorize?client_id=uaa-cli-authcode&redirect_uri=http%3A%2F%2Flocalhost%3A9876&response_type=code where the user should login and grant approvals
Starting local HTTP server on port 9876
Waiting for authorization redirect from UAA...
```

![uaa-app-auth-ack](images/uaa-app-auth-ack.png)

After clicking "Authorize" the browser changes to:

![uaa-app-auth-success](images/uaa-app-auth-success.png)

More importantly, our `uaa get-authcode-token` command has automatically now completed:

```
Local server received request to GET /?code=xfnUz8RUON
Calling UAA /oauth/token to exchange code xfnUz8RUON for an access token
Stopping local HTTP server on port 9876
Access token added to active context.
```

If we visit the "Account Settings" in the UAA web UI we can see that `drnic` has a record of the new third-party application that has been previously authorized. It also documents that the application only has permission to use UAA API operations that only require the `openid` scope.

![uaa-web-user-profile-authorized-client](images/uaa-web-user-profile-authorized-client.png)

Running `uaa get-authcode-token` again will automatically authenticate the user via the browser:

```
uaa get-authcode-token uaa-cli-authcode -s uaa-cli-authcode --port 9876
```

The `uaa` CLI is now acting on behalf of a user.

The `openid` scope allows the UAA API calls for:

```
uaa userinfo
```

But `drnic` cannot use `uaa` CLI to perform other API requests that have not been permitted to `drnic`, that `uaa` has requested from `drnic`, or that `drnic` has not permitted to `uaa` application:

```
$ uaa list-clients --verbose
{"error":"insufficient_scope","error_description":"Insufficient scope for this resource","scope":"uaa.admin clients.read clients.admin zones.uaa.admin"}

$ uaa list-users --verbose
{"error":"insufficient_scope","error_description":"Insufficient scope for this resource","scope":"uaa.admin scim.read zones.uaa.admin"}
$ uaa get-user drnic --verbose
{"error":"insufficient_scope","error_description":"Insufficient scope for this resource","scope":"uaa.admin scim.read zones.uaa.admin"}

$ uaa create-user foo --email foo@bar.com --givenName Foo --familyName Bar --verbose
{"error":"insufficient_scope","error_description":"Insufficient scope for this resource","scope":"uaa.admin scim.write scim.create zones.uaa.admin"}
```

## User permissions

Each new user is automatically added as a member of various groups:

```
uaa get-user drnic | jq -r ".groups[].display" | sort
```

The output might be similar to:

```
approvals.me
oauth.approvals
openid
password.write
profile
roles
scim.me
uaa.offline_token
uaa.user
user_attributes
```

To start to learn what authorizations/priveleges each group provides:

```
uaa list-groups | jq -r ".resources[] | {displayName, description}"
```

An interesting selection of the output is:

```json
{
  "displayName": "openid",
  "description": "Access profile information, i.e. email, first and last name, and phone number"
}
{
  "displayName": "password.write",
  "description": "Change your password"
}
{
  "displayName": "uaa.user",
  "description": "Act as a user in the UAA"
}
{
  "displayName": "scim.userids",
  "description": "Read user IDs and retrieve users by ID"
}
{
  "displayName": "scim.invite",
  "description": "Send invitations to users"
}
{
  "displayName": "uaa.none",
  "description": "Forbid acting as a user"
}
```

Comparing the two lists, we see that our `drnic` user will be granted permission to:

* `openid` - Access profile information, i.e. email, first and last name, and phone number
* `password.write` - Change your own password
* `uaa.user` - Act as a user in the UAA

From the sample list of available authorization groups, we can note that `drnic` user is not in the following groups:

* `scim.userids` - cannot read the user IDs nor retrieve users by ID
* `scim.invite` - is not allowed to send invites to other users
* `uaa.node` - is not forbidden from acting as a user

In the subsequent sections we will allow our new users to "login" and authorize the `uaa` CLI to interact with the UAA on their behalf.

## Client permissions

Third-party client applications are also scoped in their ability to interact with the UAA API.

Consider our original client `uaa_admin`:

```
uaa get-client uaa_admin
```

The output shows that `uaa_admin` client has many authorities, but scope `uaa.none`:

```json
{
  "client_id": "uaa_admin",
  "scope": [
    "uaa.none"
  ],
  "resource_ids": [
    "none"
  ],
  "authorized_grant_types": [
    "client_credentials"
  ],
  "authorities": [
    "uaa.admin",
    "clients.read",
    "clients.secret",
    "clients.write",
    "scim.write",
    "scim.read"
    "password.write",
  ],
  "lastModified": 1529652956499
}
```

Alternately, the `uaa-cli-authcode` client has a scope `openid` but authorities `uaa.none`:

```json
{
  "client_id": "uaa-cli-authcode",
  "scope": [
    "openid"
  ],
  "resource_ids": [
    "none"
  ],
  "authorized_grant_types": [
    "refresh_token",
    "authorization_code"
  ],
  "redirect_uri": [
    "http://localhost:9876"
  ],
  "authorities": [
    "uaa.none"
  ],
  "lastModified": 1529653539556
}
```