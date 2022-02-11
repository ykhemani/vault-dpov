# MySQL / MariaDB Dynamic Database Credentials

## Use Case

Data protection is top priority, and database credential rotation is a critical part of any data protection initiative. Each application or microservice has a different set of data access requirements. When a system is attached, continuous credential rotation becomes necessary and needs to be automated.

The Database Secrets engine generates database credentials dynamically based on configured roles. With this approach, each copy of an application or microservice receives a unique set of database credentials for traceability. The administrator configures a TTL for the database credential so that they are automatically revoked when they are no longer used.

## Use Case Validation

### Environment setup

#### Authenticate with Vault

Set your `VAULT_ADDR` environment variable and authenticate with Vault.

For example:

```
vault login -method userpass username=${VAULT_USERNAME}
```

##### Expected Result:
```
$ vault login -method userpass username=${VAULT_USERNAME}
Password (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                    Value
---                    -----
token                  s.G5u65ofdbGHQxRON8w7ddE6Y
token_accessor         ZvM3dgPbJaj5XRNQqxcBwieU
token_duration         768h
token_renewable        true
token_policies         ["default" "policy-x"]
identity_policies      []
policies               ["default" "policy-x"]
token_meta_username    username
```

#### MySQL Setup

A MySQL database service has been configured in the POV environment.

Alternatively, adjust based on where the MySQL database server is running. The Vault servers must be able to reach the MySQL database, and the `${MYSQL_VAULT_DB_USER} user must have privileges to manage users and grants.

Set the following environment variables to help configure the Vault Database Secrets Engine for MySQL. Adjust appropriately for the database server.

```
export MYSQL_VAULT_PATH=mysql-demo
export MYSQL_EXAMPLE_ROLE_RO=demo-mysql-role-ro
export MYSQL_EXAMPLE_ROLE_RW=demo-mysql-role-rw
export MYSQL_DB_NAME=demodb
export MYSQL_DB_HOST=mysql.hashi.cloud
export MYSQL_DB_PORT=3306
export MYSQL_VAULT_DB_USER=vault
export MYSQL_VAULT_DB_PASSWORD=vault
export MYSQL_DATABASE=demodb
```

##### Expected Result:
```
$ export MYSQL_VAULT_PATH=mysql-demo
$ export MYSQL_EXAMPLE_ROLE_RO=demo-mysql-role-ro
$ export MYSQL_EXAMPLE_ROLE_RW=demo-mysql-role-rw
$ export MYSQL_DB_NAME=demodb
$ export MYSQL_DB_HOST=mysql.hashi.cloud
$ export MYSQL_DB_PORT=3306
$ export MYSQL_VAULT_DB_USER=vault
$ export MYSQL_VAULT_DB_PASSWORD=vault
$ export MYSQL_DATABASE=demodb
```

### Enable the Database Secrets Engine

The first command disables any secret engine already configured on at `${MYSQL_VAULT_PATH}`
```
vault secrets disable ${MYSQL_VAULT_PATH}
vault secrets enable -path ${MYSQL_VAULT_PATH} database
```

##### Expected Result:
```
$ vault secrets disable ${MYSQL_VAULT_PATH}
Success! Disabled the secrets engine (if it existed) at: mysql-demo/

$ vault secrets enable -path ${MYSQL_VAULT_PATH} database
Success! Enabled the database secrets engine at: mysql-demo/
```

### Configure the Database Connection

```
vault write ${MYSQL_VAULT_PATH}/config/${MYSQL_DB_NAME} \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(${MYSQL_DB_HOST}:${MYSQL_DB_PORT})/" \
    allowed_roles="${MYSQL_EXAMPLE_ROLE_RO},${MYSQL_EXAMPLE_ROLE_RW}" \
    username="${MYSQL_VAULT_DB_USER}" \
    password="${MYSQL_VAULT_DB_PASSWORD}"
```

##### Expected Result:
```
$ vault write ${MYSQL_VAULT_PATH}/config/${MYSQL_DB_NAME} \
>     plugin_name=mysql-database-plugin \
>     connection_url="{{username}}:{{password}}@tcp(${MYSQL_DB_HOST}:${MYSQL_DB_PORT})/" \
>     allowed_roles="${MYSQL_EXAMPLE_ROLE_RO},${MYSQL_EXAMPLE_ROLE_RW}" \
>     username="${MYSQL_VAULT_DB_USER}" \
>     password="${MYSQL_VAULT_DB_PASSWORD}"
```

### Create a Read-Only Role

```
vault write ${MYSQL_VAULT_PATH}/roles/${MYSQL_EXAMPLE_ROLE_RO} \
    db_name=${MYSQL_DB_NAME} \
    creation_statements="GRANT SELECT ON ${MYSQL_DB_NAME}.* TO '{{name}}'@'%' IDENTIFIED BY '{{password}}';" \
    default_ttl="5m" \
    max_ttl="10m"
```

##### Expected Result:
```
$ vault write ${MYSQL_VAULT_PATH}/roles/${MYSQL_EXAMPLE_ROLE_RO} \
>     db_name=${MYSQL_DB_NAME} \
>     creation_statements="GRANT SELECT ON ${MYSQL_DB_NAME}.* TO '{{name}}'@'%' IDENTIFIED BY '{{password}}';" \
>     default_ttl="5m" \
>     max_ttl="10m"
Success! Data written to: mysql-demo/roles/demo-mysql-role-ro
```

### Create a Role with All Privileges on the `${MYSQL_DB_NAME}` database

```
vault write ${MYSQL_VAULT_PATH}/roles/${MYSQL_EXAMPLE_ROLE_RW} \
    db_name=${MYSQL_DB_NAME} \
    creation_statements="GRANT ALL PRIVILEGES ON ${MYSQL_DB_NAME}.* TO '{{name}}'@'%' IDENTIFIED BY '{{password}}';" \
    default_ttl="5m" \
    max_ttl="10m"
```

##### Expected Result:
```
$ vault write ${MYSQL_VAULT_PATH}/roles/${MYSQL_EXAMPLE_ROLE_RW} \
>     db_name=${MYSQL_DB_NAME} \
>     creation_statements="GRANT ALL PRIVILEGES ON ${MYSQL_DB_NAME}.* TO '{{name}}'@'%' IDENTIFIED BY '{{password}}';" \
>     default_ttl="5m" \
>     max_ttl="10m"
Success! Data written to: mysql-demo/roles/demo-mysql-role-rw
```

### List the roles configured for this secret engine

```
vault list ${MYSQL_VAULT_PATH}/roles
```

##### Expected Result:
```
$ vault list ${MYSQL_VAULT_PATH}/roles
Keys
----
demo-mysql-role-ro
demo-mysql-role-rw
```

### Obtain Dynamic Database Credentials using the Read-Only Role

```
vault read ${MYSQL_VAULT_PATH}/creds/${MYSQL_EXAMPLE_ROLE_RO}
```

Run this command multiple times and note that each time you get a unique username and password.

##### Expected Result:
```
$ vault read ${MYSQL_VAULT_PATH}/creds/${MYSQL_EXAMPLE_ROLE_RO}
Key                Value
---                -----
lease_id           mysql-demo/creds/demo-mysql-role-ro/X5kY2rR6JAqaglxRgLKZvepq
lease_duration     5m
lease_renewable    true
password           r00RWj3egq-B7bDPYQPC
username           v-userpass-d-demo-mysql-D7RNkd61

$ vault read ${MYSQL_VAULT_PATH}/creds/${MYSQL_EXAMPLE_ROLE_RO}
Key                Value
---                -----
lease_id           mysql-demo/creds/demo-mysql-role-ro/9rp1JNtU8rf2lBpIiR8ozcZ8
lease_duration     5m
lease_renewable    true
password           ISdSQHtq-z5pfXpcWRz2
username           v-userpass-d-demo-mysql-mDtFcuWY
```

### Connect to the MySQL database with the Dynamic Database Credentials

```
mysql -h ${MYSQL_DB_HOST} -D ${MYSQL_DB_NAME} -u <username> -p
```

Read some data.

```
select * from example;
```

Disconnect from the database.

```
quit
```

##### Expected Result:
```
$ mysql -h ${MYSQL_DB_HOST} -D ${MYSQL_DB_NAME} -u v-userpass-d-demo-mysql-D7RNkd61 -p
Enter password: 

mysql> show tables;
+------------------+
| Tables_in_demodb |
+------------------+
| example          |
+------------------+
1 row in set (0.00 sec)

mysql> select * from example;
+----+---------------------+
| id | title               |
+----+---------------------+
|  1 | Pride and Prejudice |
|  2 | The Great Gatsby    |
|  3 | Jane Eyre           |
|  4 | A Moveable Feast    |
+----+---------------------+
4 rows in set (0.00 sec)

mysql> quit
Bye
```

### Obtain Dynamic Database Credentials using the Role that grants all privileges on `${MYSQL_DB_NAME}`

```
vault read ${MYSQL_VAULT_PATH}/creds/${MYSQL_EXAMPLE_ROLE_RW}
```

Run this command multiple times and note that each time you get a unique username and password.

##### Expected Result:
```
$ vault read ${MYSQL_VAULT_PATH}/creds/${MYSQL_EXAMPLE_ROLE_RW}
Key                Value
---                -----
lease_id           mysql-demo/creds/demo-mysql-role-rw/C63GZeQmGy1qeyy13C7qR5Rh
lease_duration     5m
lease_renewable    true
password           lPX-TcESMacVrTO5DHa2
username           v-userpass-d-demo-mysql-vCUYeM7I

$ vault read ${MYSQL_VAULT_PATH}/creds/${MYSQL_EXAMPLE_ROLE_RW}
Key                Value
---                -----
lease_id           mysql-demo/creds/demo-mysql-role-rw/Dp5kdwFFeh6VLmvnCWOxammI
lease_duration     5m
lease_renewable    true
password           wn4-vm36CBRtlTk6FO9-
username           v-userpass-d-demo-mysql-QZ7eWHBf
```

### Connect to the MySQL database with the Dynamic Database Credentials

```
mysql -h ${MYSQL_DB_HOST} -D ${MYSQL_DB_NAME} -u <username> -p
```

##### Expected Result:
```
$ mysql -h ${MYSQL_DB_HOST} -D ${MYSQL_DB_NAME} -u v-userpass-d-demo-mysql-QZ7eWHBf -p
Enter password: 

mysql> quit
Bye
```

### Examine Lease for the Dynamic Database Credentials

```
vault lease lookup <lease id>
```

##### Expected Result:
```
$ vault lease lookup mysql-demo/creds/demo-mysql-role-rw/Dp5kdwFFeh6VLmvnCWOxammI
Key             Value
---             -----
expire_time     2022-02-11T19:41:42.274272377Z
id              mysql-demo/creds/demo-mysql-role-rw/Dp5kdwFFeh6VLmvnCWOxammI
issue_time      2022-02-11T19:36:42.274272195Z
last_renewal    <nil>
renewable       true
ttl             3m38s
```

### Examine what users exist in the MySQL Database

```
mysql -h ${MYSQL_DB_HOST} -u ${MYSQL_VAULT_DB_USER} -e 'select user from mysql.user;' --password=${MYSQL_VAULT_DB_PASSWORD}
```

##### Expected Result:
```
$ mysql -h ${MYSQL_DB_HOST} -u ${MYSQL_VAULT_DB_USER} -e 'select user from mysql.user;' --password=${MYSQL_VAULT_DB_PASSWORD}
mysql: [Warning] Using a password on the command line interface can be insecure.
+----------------------------------+
| user                             |
+----------------------------------+
| root                             |
| v-userpass-d-demo-mysql-D7RNkd61 |
| v-userpass-d-demo-mysql-QZ7eWHBf |
| v-userpass-d-demo-mysql-mDtFcuWY |
| v-userpass-d-demo-mysql-vCUYeM7I |
| vault                            |
| vtest                            |
| mysql.session                    |
| mysql.sys                        |
| root                             |
+----------------------------------+
```

### Revoke Lease for the Dynamic Database Credentials

Revoke one of the leases.

```
vault lease revoke <lease id>
```

##### Expected Result:
```
$ vault lease revoke mysql-demo/creds/demo-mysql-role-rw/Dp5kdwFFeh6VLmvnCWOxammI
All revocation operations queued successfully!
```

### Examine what users exist in the MySQL Database

List the database users. Note that the one from the lease you revoked no longer exists.

```
mysql -h ${MYSQL_DB_HOST} -u ${MYSQL_VAULT_DB_USER} -e 'select user from mysql.user;' --password=${MYSQL_VAULT_DB_PASSWORD}
```

##### Expected Result:
```
$ mysql -h ${MYSQL_DB_HOST} -u ${MYSQL_VAULT_DB_USER} -e 'select user from mysql.user;' --password=${MYSQL_VAULT_DB_PASSWORD}
mysql: [Warning] Using a password on the command line interface can be insecure.
+----------------------------------+
| user                             |
+----------------------------------+
| root                             |
| v-userpass-d-demo-mysql-D7RNkd61 |
| v-userpass-d-demo-mysql-mDtFcuWY |
| v-userpass-d-demo-mysql-vCUYeM7I |
| vault                            |
| vtest                            |
| mysql.session                    |
| mysql.sys                        |
| root                             |
+----------------------------------+
```

### Revoke Lease based on Prefix

We can also revoke leases by specifying a prefix. Revoke all leases for the `${MYSQL_EXAMPLE_ROLE_RW}' role.

```
vault lease revoke -prefix ${MYSQL_VAULT_PATH}/creds/${MYSQL_EXAMPLE_ROLE_RW}
```

##### Expected Result:
```
$ vault lease revoke -prefix ${MYSQL_VAULT_PATH}/creds/${MYSQL_EXAMPLE_ROLE_RW}
All revocation operations queued successfully!
```

### Examine what users exist in the MySQL Database

List the database users. Note that all users created using the `${MYSQL_EXAMPLE_ROLE_RW}` role no longer exist.

```
mysql -h ${MYSQL_DB_HOST} -u ${MYSQL_VAULT_DB_USER} -e 'select user from mysql.user;' --password=${MYSQL_VAULT_DB_PASSWORD}
```

##### Expected Result:
```
$ mysql -h ${MYSQL_DB_HOST} -u ${MYSQL_VAULT_DB_USER} -e 'select user from mysql.user;' --password=${MYSQL_VAULT_DB_PASSWORD}
mysql: [Warning] Using a password on the command line interface can be insecure.
+----------------------------------+
| user                             |
+----------------------------------+
| root                             |
| v-userpass-d-demo-mysql-D7RNkd61 |
| v-userpass-d-demo-mysql-mDtFcuWY |
| vault                            |
| vtest                            |
| mysql.session                    |
| mysql.sys                        |
| root                             |
+----------------------------------+
```

### Policy for obtaining dynamic database credentials

The policy below could be used to allow a client to obtain dynamic database credentials using the read-only role created above.

```
path "${MYSQL_VAULT_PATH}/creds/${MYSQL_EXAMPLE_ROLE_RO}" {
  capabilities = ["read"]
}
```

---
