<!-- omit from toc -->
# [Auth Methods](https://developer.hashicorp.com/vault/docs/concepts/auth)

- [Token](#token)
  - [Regenerate a Root Token](#regenerate-a-root-token)
- [Approle](#approle)
- [Okta](#okta)
- [Userpass](#userpass)


Authentication in Vault is the process by which user or machine supplied information is verified against an internal or external system. Vault supports multiple auth methods including GitHub, LDAP, AppRole, and more. Each auth method has a specific use case.

Before a client can interact with Vault, it must authenticate against an auth method. Upon authentication, a token is generated. This token is conceptually similar to a session ID on a website. The token may have attached policy, which is mapped at authentication time. This process is described in detail in the policies concepts documentation.

See [Auth Methods](https://developer.hashicorp.com/vault/docs/auth) for a list of configurable authentication methods.

## [Token](https://developer.hashicorp.com/vault/docs/auth/token)

The token auth method is built-in and automatically available at `/auth/token`. It allows users to authenticate using a token, as well to create new tokens, revoke secrets by token, and more.

When any other auth method returns an identity, Vault core invokes the token method to create a new unique token for that identity.

The token store can also be used to bypass any other auth method: you can create tokens directly, as well as perform a variety of other operations on tokens such as renewal and revocation.

Please see the [token concepts](https://developer.hashicorp.com/vault/docs/concepts/tokens) page dedicated to tokens.

Types of Tokens:

- Service Tokens: (main type and the default).  Starts with `hvs.`
- Batch Tokens: Not written to storage back end. Used for high volume requests, replication etc.  Replicated to all clusters, start with `hvb.`
- Periodic Token: Is a type of service token but does not have a max TTL so it can be refreshed indefinitely.  Create with `-period=24h`.  Useful for an application where it would be very difficult to change the token.
- Use Limited Token: Is a type of service token but can be used only the number of times stated during creation.  Create with `-use-limit=2`.  These tokens still has a ttl so it can time out before the number of uses is hit.
- Orphan Token: A token that is not revoked when the parent token is revoked. Create with `-orphan`
- These flags can be set in auth as well.  Example: `vault write auth/approle/role/hcvop  policies="hcvop" period="72h"

See [slides 277-288](operations-training/01-Create-a-working-Vault-server-configuration-given-a-scenario.pdf) for more on tokens.

```bash
# List how many tokens are in vault at the moment
vault list auth/token/accessors
# token                hvs.x8blFYnSQnl7VI3Gyza4N4v6
# token_accessor       QbfF8ZvoqMBNRCxghYGJaR9J
# token                hvs.CAESICIvZ7Z2et4aXt23yt_oAiScwSz0tUpE-xYoXWvdbjJxGh4KHGh2cy5FVXNyOEhUNWtMb1haTmtqckN3M0N5Y2U
# token_accessor       g7kX9XPcwNEDF6U2TYPawJRj

# Create a new root token
vault token create

# Create a non root token tied to a policy
vault token create -ttl=5m -policy=training

# Create a token that never expires with a ttl of 24 hours
vault token create -period=24h -policy=auto-unseal-policy

# Create a token that has multiple policies
vault token create -policy=my-first-policy -policy=my-second-policy

# Create a performance replicable batch token
vault token create -type=batch -orphan=true -policy=my-policy

# View a token's details, if no token is given displays the currently logged in token
vault token lookup <token>

# Look up what a token can do on a path
vault token capabilities <token> <path>
vault token capabilities hvs.91Jf2g2jxPrs4kK59JN04346 kv/data/apps/webapp
# create, list, read, update

# Renew a token
vault token renew hvs.91Jf2g2jxPrs4kK59JN04346

# Log in using userpass via the API and save the token to a file
curl --request POST --data @password.json http://127.0.0.1:8200/v1/auth/userpass/login/bryan | \
  jq -r ".auth.client_token" > token.txt

# Log in using userpass via the API and save the token to a file
VAULT_TOKEN=$(curl --request POST --data @password.json http://127.0.0.1:8200/v1/auth/userpass/login/bryan | \
  jq -r ".auth.client_token")

# Write an API key into an apikey store called splunk
curl --header "X-Vault-Token: hvs.91Jf2g2jxPrs4kK59JN04346" \
  --request POST \
  --data '{"apiKey": "3281254858514"}'\
  http://127.0.0.1:8200/v1/secret/apikey/splunk

# Get the api key called splunk
curl --header "X-Vault-Token: hvs.91Jf2g2jxPrs4kK59JN04346" \
  --request GET \
  http://127.0.0.1:8200/v1/secret/data/apikey/splunk

# Token Accessors
# Caan be used to 
#  * Look up token properties
#  * Look up capabilities
#  * Renew Token
#  * Revoke Token
# Cannot be used to log into vault, get a secret, generate new tokens, get the current token, etc.
vault token lookup -accessor gFgh4kfjK459K4k599fKNMb4
vault token revoke -accessor gFgh4kfjK459K4k599fKNMb4
vault token renew -accessor gFgh4kfjK459K4k599fKNMb4

# Vault has a default TTL of 768 hours / 32 days, can be changed in config file with 'default_lease_ttl = 24h'
# This also applies to max-ttl
```

<a name="regenerate-a-root-token"></a>
### Regenerate a Root Token

- In case normal login is broken
- A root token is needed for a particular task

See [slides 305-314](operations-training/01-Create-a-working-Vault-server-configuration-given-a-scenario.pdf) for more on tokens.

```bash
### STEP 1
vault operator generate-root -init
#   - A One-Time-Password has been generated for you and is shown in the OTP field.
#   - You will need this value to decode the resulting root token, so keep it safe.
#   - Give the Nonce value to those that will be entering their key shards
# Nonce         99d9604b-069f-a767-dfa8-047438c47290
# Started       true
# Progress      0/3
# Complete      false
# OTP           2q5DsjyyMSVIaoc0teSXwrnxfr6A
# OTP Length    28

### STEP 2
# Have three of the vault operators that have a sharded key run the following command
vault operator generate-root
# Operation nonce: 99d9604b-069f-a767-dfa8-047438c47290
Unseal Key (will be hidden): *****************
# Nonce            99d9604b-069f-a767-dfa8-047438c47290
# Started          true
# Progress         3/3
# Complete         true
# Encoded Token    WgdGagkMITsMAWcjLjwrB0c1CjcBI149Lz1vLg
# NOTE: The last person will get the above Encoded Token, give this back to the admin.

# View status, what is the progress, etc.
vault operator generate-root -status

### STEP 3
# Once we have all three shareded keys entered we will recieve the "Encoded Token" shown above.
# We can now decrypt that token using the One Time Key we were given in step 1, -init
vault operator generate-root \
-otp="2q5DsjyyMSVIaoc0teSXwrnxfr6A" \
-decode="WgdGagkMITsMAWcjLjwrB0c1CjcBI149Lz1vLg"
# hvs.zfXBAR1jOSH73PYovQ0EIOYo
```

<a name="approle"></a>
## [Approle](https://developer.hashicorp.com/vault/docs/auth/approle)

The approle auth method allows machines or apps to authenticate with Vault-defined roles. The open design of AppRole enables a varied set of workflows and configurations to handle large numbers of apps. This auth method is oriented to automated workflows (machines and services), and is less useful for human operators.

**NOTES:**

- When working with a fleet of servers needing the same role, it is best practice to reuse the `role-id` but generate a new `secret-id` for each server
- Whenever possible a new `secret-id` should be injected during provisioning of the application, for example, during CI/CD Jenkins would call Vault to generate a new response wrapped `secret-id` and then inject the token into the application on deployment. The app could then use that token to unwrap the `secret-id`.  See [slide 253](operations-training/01-Create-a-working-Vault-server-configuration-given-a-scenario.pdf) for a diagram.

See [slides 251-265](operations-training/01-Create-a-working-Vault-server-configuration-given-a-scenario.pdf) for more on approle.

```bash
# Enable the approle authentication method, this will enable it under the approle path
vault auth enable approle

# Create a role, this depends on there already being a policy with the given name
vault write auth/approle/role/bryan policies=bryan token_ttl=20m

vault list auth/approle/role
# Keys
# ------
# bryan

# Retrieve the role-id
vault read auth/approle/role/bryan/role-id
# Key        Value
# ---        -----
# role_id    3454d712-a86c-91c7-f987-ca5066c85dbc
# NOTE: Every time we read this role-id we will get the same ID back, it never changes.

# Create the secret-id
vault write -force auth/approle/role/bryan/secret-id
# Key                   Value
# ---                   -----
# secret_id             2d2692fe-cd5e-18fc-4be0-19d61ebe4430
# secret_id_accessor    76136638-8862-9d69-4862-1ffb3d0e448b
# secret_id_num_uses    0
# secret_id_ttl         0s
# NOTE: Each time we generate a new secret-id we get a new one with a different ID each time.
# NOTE: Each token-id has an accessor that can be used just to renew / revoke etc.

# Log in using the aprole auth method and the new role we created above
vault write auth/approle/login role_id=3454d712-a86c-91c7-f987-ca5066c85dbc secret_id=2d2692fe-cd5e-18fc-4be0-19d61ebe4430
# Key                     Value
# ---                     -----
# token                   hvs.CAESIGLn79qMNr2ZKEBPFXe70t_pi1qnYqEdGsY7Y6RzhtUgGh4KHGh2cy4wNTUxeHJVRTJjZ0R3dXVnc1dnM3Bibzc
# token_accessor          FWrHkbTDeU5gmylgGJfnB6xo
# token_duration          20m
# token_renewable         true
# token_policies          ["default"]
# identity_policies       []
# policies                ["default"]
# token_meta_role_name    bryan
```

<a name="okta"></a>
## [Okta](https://developer.hashicorp.com/vault/docs/auth/okta)

```bash
# Enable okta and configure it
vault auth enable okta
vault write auth/okta/config base_url="okta.com" org_name="myorgname" api_token="oktaapitoken"
vault read auth/okta/config

# Set up a user
vault write auth/okta/users/bryan@krausen.io policies=bryan
vault write auth/okta/groups/mygroupname policies=bryan

# Should now be able to log into vault with okta
```

<a name="userpass"></a>
## [Userpass](https://developer.hashicorp.com/vault/docs/auth/userpass)

```bash
vault auth enable userpass
vault write auth/userpass/users/frank password="mypassword" policies="bryan"
vault list auth/userpass/users
vault read auth/userpass/users/frank
vault login -method=userpass username=frank
# This will return the standard user login data including the token
```
