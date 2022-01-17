# Dynamic Secret Creation [PostgreSQL]
# Using Mutation Webhook to inject Sidecar
![Alt text](https://assets.openshift.com/hubfs/Integrating%20Hashicorp%20Vault%20in%20OpenShift%204.png)
Deploy our test database
```
kubectl create ns postgres
kubectl -n postgres apply -f ./hashicorp/vault/example-apps/dynamic-postgresql/postgres.yaml
kubectl -n postgres apply -f ./hashicorp/vault/example-apps/dynamic-postgresql/pgadmin.yaml

kubectl -n postgres exec -it <podname> bash
psql --username=postgresadmin postgresdb
\du
```
Enable the database engine
```
vault secrets enable database
```

## Configure DB Credential creation

```
vault write database/config/postgresdb \
    plugin_name=postgresql-database-plugin \
    allowed_roles="sql-role" \
    connection_url="postgresql://{{username}}:{{password}}@postgres.postgres:5432/postgresdb?sslmode=disable" \
    username="postgresadmin" \
    password="admin123"

 vault write database/roles/sql-role \
    db_name=postgresdb \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```
When a user or application requests credentials, Vault will execute the SQL statement defined in the creation_statements parameter. This example, creates a role in the database wizard which allows select access to all tables.

The creation_statements are PostgreSQL standard SQL statements. SQL statements can contain template variables which are dynamically substituted at runtime. If you look at the create SQL statement below, you will see three template variables {{name}}, {{password}} and {{expiration}}:

When Vault rotates root credentials, it connects to the database using its existing root credentials. It then generates a new password for the configured user. Vault saves the password but you cannot retrieve it. This process removes the paper trail associated with the original password. Should you need to access the database then it is always possible to ask Vault to generate credentials. Run the following command to rotate the root credentials:

vault write --force /database/rotate-root/wizard

After running this command, you can check that Vault has rotated the credentials by trying to login using psql using the original credentials:

kubectl exec -it $(kubectl get pods --selector "app=postgres" -o jsonpath="{.items[0].metadata.name}") -c postgres -- bash -c 'PGPASSWORD=admin123 psql -U postgresadmin'

## Example Application

Create a policy to control access to secrets
Vault applies policy based on the path of the secret. For example, the path for database secrets is "database/creds/<role>". For the role created earlier, you can access the database secret at database/creds/db-app.
```
cat <<EOF > /home/vault/postgres-app-policy.hcl
path "database/creds/sql-role" {
  capabilities = ["read"]
}
EOF

vault policy write postgres-app-policy /home/vault/postgres-app-policy.hcl
```
Enable Kubernetes Auth method
```
vault auth enable kubernetes
vault write auth/kubernetes/config \
token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

Bind our role to a service account for our application
```
vault write auth/kubernetes/role/sql-role \
   bound_service_account_names=dynamic-postgres \
   bound_service_account_namespaces=vault-example \
   policies=postgres-app-policy \
   ttl=1h
```
```
#test 
vault read database/creds/sql-role
```
```
kubectl -n vault-example apply -f .\hashicorp\vault\example-apps\dynamic-postgresql\deployment.yaml
k apply -f docker-development-youtube-series/hashicorp/vault/example-apps/dynamic-postgresql/deployment.yaml -n vault-example
deployment.apps/dynamic-postgres created
serviceaccount/dynamic-postgres created
configmap/pgadmin-config created
deployment.apps/pgadmin created
configmap/postgres-config created
deployment.apps/postgres created
service/postgres created

k exec -it dynamic-postgres-ccb489896-kl4zw -n vault-example -- bash
Defaulted container "app" out of: app, vault-agent, vault-agent-init (init)
cat /vault/secrets/sql-role
{"db_connection": "host=postgres.postgress port=5432 user=v-kubernet-sql-role-IftB5Yb95EhRTX8KqenV-1642072738 password=bKDnuqO8-1cL1SQBQcFS dbname=postgresdb sslmode=disable"

/ $ vault read database/creds/sql-role
Key                Value
---                -----
lease_id           database/creds/sql-role/Ti3U5KYUjj88snjcpRyX37Xa
lease_duration     1h
lease_renewable    true
password           mlq0mnsUrvD-wzcPMGWB
username           v-root-sql-role-09wP4CvaJUNOUJYEp0KW-1642073420
/ $ vault read database/roles/sql-role
Key                      Value
---                      -----
creation_statements      [CREATE ROLE "{{name}}" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';         GRANT SELECT ON ALL TABLES IN SCHEMA public TO "{{name}}";]
db_name                  postgresdb
default_ttl              1h
max_ttl                  24h
renew_statements         []
revocation_statements    []
rollback_statements      []
```
