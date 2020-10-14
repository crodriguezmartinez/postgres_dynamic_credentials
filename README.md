# postgres_dynamic_credentials

### Docker:

#### pull db image
docker pull postgres:latest

#### start db
docker run \
      --name postgres \
      --env POSTGRES_USER=root \
      --env POSTGRES_PASSWORD=rootpassword \
      --detach  \
      --publish 5432:5432 \
      postgres

#### connect to db
docker exec -it postgres psql

#### create role in db
CREATE ROLE ro NOINHERIT;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO "ro";
#### disconnect
\q

### Vault

#### Policy for db admin:

#### Mount secrets engines
path "sys/mounts/*" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}

#### Configure the database secrets engine and create roles
path "database/*" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}

#### Manage the leases
path "sys/leases/+/database/creds/readonly/*" {
  capabilities = [ "create", "read", "update", "delete", "list", "sudo" ]
}

path "sys/leases/+/database/creds/readonly" {
  capabilities = [ "create", "read", "update", "delete", "list", "sudo" ]
}

#### Policy for the apps that will be retrieving the secrets:

#### Get credentials from the database secrets engine 'readonly' role.
path "database/creds/readonly" {
  capabilities = [ "read" ]
}

#### Write policies in Vault:

vault write policy admin_policy /etc/vault.d/admin_policy.hcl
vault write policy apps_policy /etc/vault.d/apps_policy.hcl

#### Create admin and apps user and map policies to them:

vault write auth/userpass/users/admin \
    password=admin-password \
    policies=admin_policy
    
vault write auth/userpass/users/apps \
    password=apps-password \
    policies=apps_policy
    
#### Login with admin user and enable DB secrets engine:

vault login -method=userpass \
  username=admin \
  password=admin-password
  
vault secrets enable database

#### Enable Postgres DB plugin:

vault write database/config/postgresql \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/postgres?sslmode=disable" \
    allowed_roles=readonly \
    username="root" \
    password="rootpassword"
    
#### Create DB role:

vault write database/roles/readonly \
> db_name=postgresql \
> creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
>         GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
> default_ttl=1h \
> max_ttl=24h

#### Read dynamic secret in Vault as app, and check it exists in the DB:

vault login -method=userpass username=apps password=apps-password

vault read database/creds/readonly

```
Key                Value
---                -----
lease_id           database/creds/readonly/PQbFr2dUdJ2vnHQ2CvnJVvQV
lease_duration     1h
lease_renewable    true
password           A1a-Jnf9jXXZOUsznU3L
username           v-userpass-readonly-hzwKvpA3tUqy69pzvz9R-1602662337
```

docker exec -it postgres psql

```
                       usename                       |        valuntil        
-----------------------------------------------------+------------------------
 root                                                | 
 v-userpass-readonly-hzwKvpA3tUqy69pzvz9R-1602662337 | 2020-10-14 08:59:02+00
(2 rows)

```



