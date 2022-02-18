# Active Directory (AD) Secrets Engine - Privileged Account Check Out / Check In

## Use Case
A service account is a user account that is created explicitly to provide a security context for services running on Windows Server operating systems. Often, an organization has a limited number of service accounts based on their Client Access License (CAL) which they share among the team. As a result, credential rotation is a cumbersome process and it's difficult to audit who took what actions with those shared credentials.

The Active Directory secrets engine in HashiCorp Vault enables authorized humans and applications to check in and check out a select set of service account credentials. To use the shared service accounts, the requester first checks out the account. When done using the account, they simply check the service account back in for others to use. Whenever a service account is checked back in, Vault rotates its password.

This minimizes the number of accounts consumed and ensures that as an organization grows, they do not expand beyond the limit of their Client Access License. At the same time, the credentials are valid per check-out and user actions can be tied to whoever had a service account checked out. This lowers the operational burden and helps an organization meet its security requirements around auditability.


## Use Case Validation

### Environment setup

#### Vault access

Set your `VAULT_ADDR` environment variable and authenticate with Vault.

For example:

```
vault login -method userpass username=${VAULT_USERNAME}
```

##### Expected Result
```
$ vault login -method userpass username=${VAULT_USERNAME}
Password (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                    Value
---                    -----
token                  s.PtSDldJPQAbbzDFgJoVRSOae
token_accessor         p7b1BVp05haugYq1ul3sLNfQ
token_duration         768h
token_renewable        true
token_policies         ["default" "policy-x"]
identity_policies      []
policies               ["default" "policy-x"]
token_meta_username    username
```

#### AD Setup

An Active Directory service has been configured in the POV environment.

Set the following environment variables to help configure the Vault AD Secrets Engine.

```
export AD_SECRETS_PATH=ad
export AD_SERVER=dc.hashi.cloud
export AD_URL="ldaps://${AD_SERVER}:636"
export AD_BIND_DN="cn=vault-reset,cn=Users,dc=hashi,dc=cloud"
export AD_BIND_PASSWORD="P@ssword1"
export AD_USER_DN="dc=hashi,dc=cloud"
```

The environment variables below reference the privileged accounts Vault will manage.

```
export AD_SERVICE_ACCOUNT="priv-acct"
export AD_SERVICE_ACCOUNT_NAME1="privileged-account1@hashi.cloud"
export AD_SERVICE_ACCOUNT_NAME2="privileged-account2@hashi.cloud"
```

##### Expected Result
```
$ export AD_SECRETS_PATH=ad
$ export AD_SERVER=dc.hashi.cloud
$ export AD_URL="ldaps://${AD_SERVER}:636"
$ export AD_BIND_DN="cn=vault-reset,cn=Users,dc=hashi,dc=cloud"
$ export AD_BIND_PASSWORD="P@ssword1"
$ export AD_USER_DN="dc=hashi,dc=cloud"
$ export AD_SERVICE_ACCOUNT="priv-acct"
$ export AD_SERVICE_ACCOUNT_NAME1="privileged-account1@hashi.cloud"
$ export AD_SERVICE_ACCOUNT_NAME2="privileged-account2@hashi.cloud"
```

### Enable the AD Secrets Engine

The first command disables any secret engine already configured on at `${AD_SECRETS_PATH}`
```
vault secrets disable ${AD_SECRETS_PATH}
vault secrets enable -path=${AD_SECRETS_PATH} ad
```

##### Expected Result
```
$ vault secrets disable ${AD_SECRETS_PATH}
Success! Disabled the secrets engine (if it existed) at: ad/
$ vault secrets enable -path=${AD_SECRETS_PATH} ad
Success! Enabled the ad secrets engine at: ad/
```

### Configure the AD Secrets Engine

The environment variables we set above will facilitate the secret engine configuration.

```
vault write ${AD_SECRETS_PATH}/config \
  binddn=${AD_BIND_DN} \
  bindpass=${AD_BIND_PASSWORD} \
  url=${AD_URL} \
  userdn=${AD_USER_DN} \
  insecure_tls=true
```

##### Expected Result
```
$ vault write ${AD_SECRETS_PATH}/config \
>   binddn=${AD_BIND_DN} \
>   bindpass=${AD_BIND_PASSWORD} \
>   url=${AD_URL} \
>   userdn=${AD_USER_DN} \
>   insecure_tls=true
Success! Data written to: ad/config
```

### Create a Library of Service Accounts

A library is a set of service accounts that you want Vault to manage for check-out.

In this example, we are creating a library with two service accounts, identified by the `${AD_SERVICE_ACCOUNT_NAME1}` and `${AD_SERVICE_ACCOUNT_NAME2}` environment variables. We are specifying a default `ttl` of 2 minutes and a `max_ttl` of 1 hour. This is how long a single service account in this library may be checked out. In practice, you would set these values according to the intended use for these service accounts.

```
vault write ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT} \
  service_account_names=${AD_SERVICE_ACCOUNT_NAME1},${AD_SERVICE_ACCOUNT_NAME2} \
  ttl=2m \
  max_ttl=1h \
  disable_check_in_enforcement=false
```

##### Expected Result
```
$ vault write ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT} \
>   service_account_names=${AD_SERVICE_ACCOUNT_NAME1},${AD_SERVICE_ACCOUNT_NAME2} \
>   ttl=2m \
>   max_ttl=1h \
>   disable_check_in_enforcement=false
Success! Data written to: ad/library/priv-acct
```

List the libraries in the AD Secrets Engine.

```
vault list ${AD_SECRETS_PATH}/library
```

##### Expected Result
```
$ vault list ${AD_SECRETS_PATH}/library
Keys
----
priv-acct
```

### Create a client policy to allow check out and check in of service accounts

It is always best practice to use an approach of granting least-privilege when authoring policies. 

We can create the policy by referencing a file, or as in the example below, by providing the policy inline.

```
vault policy write priv-acct -<<EOF
# To check-out a service account which is a part of priv-acct library
path "ad/library/priv-acct/check-out" {
  capabilities = [ "update" ]
}

# To allow the extension of TTL
path "sys/leases/renew" {
  capabilities = [ "update" ]
}

# To check back in a service account
path "ad/library/priv-acct/check-in" {
  capabilities = [ "update" ]
}

# If you want to allow the client to see the library status
path "ad/library/priv-acct/status" {
  capabilities = [ "read" ]
}
EOF
```

##### Expected Result
```
$ vault policy write priv-acct -<<EOF
> # To check-out a service account which is a part of priv-acct library
> path "ad/library/priv-acct/check-out" {
>   capabilities = [ "update" ]
> }
> 
> # To allow the extension of TTL
> path "sys/leases/renew" {
>   capabilities = [ "update" ]
> }
> 
> # To check back in a service account
> path "ad/library/priv-acct/check-in" {
>   capabilities = [ "update" ]
> }
> 
> # If you want to allow the client to see the library status
> path "ad/library/priv-acct/status" {
>   capabilities = [ "read" ]
> }
> EOF
Success! Uploaded policy: priv-acct
```

Examine the policy. It should match the policy we created above.

```
vault policy read priv-acct
```

##### Expected Result
```
$ vault policy read priv-acct
# To check-out a service account which is a part of priv-acct library
path "ad/library/priv-acct/check-out" {
  capabilities = [ "update" ]
}

# To allow the extension of TTL
path "sys/leases/renew" {
  capabilities = [ "update" ]
}

# To check back in a service account
path "ad/library/priv-acct/check-in" {
  capabilities = [ "update" ]
}

# If you want to allow the client to see the library status
path "ad/library/priv-acct/status" {
  capabilities = [ "read" ]
}
```

### Check-out service accounts

Generate a token with the `priv-account` policy we created above.

You would attach such a policy to a group or user in practice. However, for the purposes of the POV, we will generate a direct token with the `priv-acct` policy we created above.

```
PRIV_ACCT_TOKEN=$(vault token create -format=json -policy="priv-acct" | jq -r ".auth.client_token")
```

##### Expected Result
```
$ PRIV_ACCT_TOKEN=$(vault token create -format=json -policy="priv-acct" | jq -r ".auth.client_token")
```

#### Examine the resulting token.

```
echo ${PRIV_ACCT_TOKEN}
```

##### Expected Result
```
$ echo ${PRIV_ACCT_TOKEN}
s.HsQ7eyA4c2K0yRlV3bBnHBr2
```

#### View status of the accounts in the library

Note that there are two accounts available in this library, and that both are available.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault read ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/status
```

##### Expected Result
```
$ VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault read ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/status
Key                                Value
---                                -----
privileged-account1@hashi.cloud    map[available:true]
privileged-account2@hashi.cloud    map[available:true]
```

#### Check out service account

Using the `${PRIV_ACCT_TOKEN}` token, let's check out a service account in this library. Let's override the default `ttl` and specify a `ttl` of 30 seconds.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out ttl=30s
```

##### Expected Result
```
$ VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out ttl=30s
Key                     Value
---                     -----
lease_id                ad/library/priv-acct/check-out/R7rvtT8QFCLWgD3rY7O3th2T
lease_duration          30s
lease_renewable         true
password                ?@09AZHMzvEpznKhi4yQvgVVPAs06BrxrvNUaUriPmQZUaF0F9lBiWQB9CLFkoX7
service_account_name    privileged-account1@hashi.cloud
```

There are two service accounts. Let's check out the second one.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out
```

##### Expected Result
```
$ VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out
Key                     Value
---                     -----
lease_id                ad/library/priv-acct/check-out/KB0b7ujK6jmJBy1Aztxw7Tsu
lease_duration          2m
lease_renewable         true
password                ?@09AZtUAuBVrbykG36E4LpnDoifFmwKM446EOvJcapuERTcMa1YcIWawDnFyoBB
service_account_name    privileged-account2@hashi.cloud
```

Let's try to check out one more. This should fail.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out
```

##### Expected Result
```
$ VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out
Error writing data to ad/library/priv-acct/check-out: Error making API request.

URL: PUT http://10.0.101.135:8200/v1/ad/library/priv-acct/check-out
Code: 400. Errors:

* No service accounts available for check-out.
```

#### View the library status again.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault read ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/status
```

##### Expected Result
```
$ VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault read ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/status
Key                                Value
---                                -----
privileged-account1@hashi.cloud    map[available:false borrower_client_token:089dd931fb02b66b0eb1bd032c1e043c567dfb58 borrower_entity_id:df20dcb9-534e-3ba5-f10e-39853b0e15d6]
privileged-account2@hashi.cloud    map[available:false borrower_client_token:089dd931fb02b66b0eb1bd032c1e043c567dfb58 borrower_entity_id:df20dcb9-534e-3ba5-f10e-39853b0e15d6]
```

Note that both of the service accounts in this library are not available because they are checked out. The library status also shows the borrower's entity id, which can be queried to determine who has checked out a given service account.

#### Check In a service account

When our work with a service account is complete, we can check it in.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-in service_account_names=privileged-account1@hashi.cloud
```

##### Expected Result
```
$ VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-in service_account_names=privileged-account1@hashi.cloud
Key          Value
---          -----
check_ins    [privileged-account1@hashi.cloud]
```

#### Check Out a service account

Because we checked in one of the two service accounts, we should be able to check it out again.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out
```

##### Expected Result
```
$ VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out
Key                     Value
---                     -----
lease_id                ad/library/priv-acct/check-out/hmngfI3jvYGnaM8aUGJfP5cq
lease_duration          2m
lease_renewable         true
password                ?@09AZtTKl2zcvGb9uiJlheaI0Vn9BP4JG0UrTxpnnXatXPNdrcYEjxXYdY4rlZl
service_account_name    privileged-account1@hashi.cloud
```
---
