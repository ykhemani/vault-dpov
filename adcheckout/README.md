# Active Directory (AD) Secrets Engine - Privileged Account Check Out / Check In

## Use Case
A service account is a user account that is created explicitly to provide a security context for services running on Windows Server operating systems. Often, an organization has a limited number of service accounts based on their Client Access License (CAL) which they share among the team. As a result, credential rotation is a cumbersome process and it's difficult to audit who took what actions with those shared credentials.

The Active Directory secrets engine in HashiCorp Vault enables authorized humans and applications to check in and check out a select set of service account credentials. To use the shared service accounts, the requester first checks out the account. When done using the account, they simply check the service account back in for others to use. Whenever a service account is checked back in, Vault rotates its password.

This minimizes the number of accounts consumed and ensures that as an organization grows, they do not expand beyond the limit of their Client Access License. At the same time, the credentials are valid per check-out and user actions can be tied to whoever had a service account checked out. This lowers the operational burden and helps an organization meet its security requirements around auditability.


## Use Case Validation

### Environment setup

#### Vault access

Set your `VAULT_ADDR` and `VAULT_TOKEN` environment variables.

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

### Enable the AD Secrets Engine

The first command disables any secret engine already configured on at `${AD_SECRETS_PATH}`
```
vault secrets disable ${AD_SECRETS_PATH}
vault secrets enable -path=${AD_SECRETS_PATH} ad
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
### Create a Library of Service Accounts

A library is a set of service accounts that you want Vault to manage for check-out.

```
vault write ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT} \
  service_account_names=${AD_SERVICE_ACCOUNT_NAME1},${AD_SERVICE_ACCOUNT_NAME2} \
  ttl=2m \
  max_ttl=1h \
  disable_check_in_enforcement=false
```

List the libraries in the AD Secrets Engine.

```
vault list ${AD_SECRETS_PATH}/library
```

### Create a client policy to allow check out and check in of service accounts

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

Examine the policy. 

```
vault policy read priv-acct
```

### Check-out service accounts

Generate a token with the `priv-account` policy we created above.

While you would attach such a policy to a group or user in production, for the purposes of the POV, we will generate a direct token with the `priv-acct` policy we created above.

```
PRIV_ACCT_TOKEN=$(vault token create -format=json -policy="priv-acct" | jq -r ".auth.client_token")
```

#### Examine the resulting token.

```
echo ${PRIV_ACCT_TOKEN}
```

#### View status of the accounts in the library

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault read ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/status
```

#### Check out service account

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out ttl=15s
```

There are two service accounts. Let's check out the second one.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out
```

Let's try to check out one more. This should fail.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out
```

#### View the library status again.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault read ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/status
```

#### Check In a service account

When our work with a service account is complete, we can check it in.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-in service_account_names=privileged-account1@hashi.cloud
```

#### Check Out a service account

Because we checked in one of the two service accounts, we should be able to check it out again.

```
VAULT_TOKEN=${PRIV_ACCT_TOKEN} vault write -f ${AD_SECRETS_PATH}/library/${AD_SERVICE_ACCOUNT}/check-out
```

---


