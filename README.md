# 1. Install Vault:

```
helm install vault hashicorp/vault -n vault --create-namespace --values vault/vault-custom-values.yaml
```

go to Vault UI, enter 1 & 1 as numbers, copy tokens, unseal with unseal-token and login with root token

# 2. Configure Vault:

open a bash in the previously created vault container:

```
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```

within the container, do the following:
‚
```
vault auth enable kubernetes

vault write auth/kubernetes/config \
   kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

vault secrets enable -path=kvv2 kv-v2

vault policy write dev - <<EOF
path "kvv2/*" {
   capabilities = ["read"]
}
EOF

vault write auth/kubernetes/role/role1 \
   bound_service_account_names=default \
   bound_service_account_namespaces=app \
   policies=dev \
   audience=vault \
   ttl=24h

vault kv put kvv2/webapp/config username="static-user" password="static-password"

exit
```

# Install Vault Secrets Operator:

in your local CLI:

```
helm install vault-secrets-operator hashicorp/vault-secrets-operator -n vault-secrets-operator-system --create-namespace --values vault/vault-operator-values.yaml
```

now create a secret sync:

```
kubectl create ns app
kubectl apply -f vault/vault-auth-static.yaml
kubectl apply -f vault/static-secret.yaml
```

and check if the secret was created:

```
kubectl get secret secretkv -n app
kubectl get secret secretkv -n app -o jsonpath='{.data.password}' | base64 --decode
```

go back into the vault container:

```
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```

within the vault container, update the secret with a new password and check if it's synced (may take some time):

```
vault kv put kvv2/webapp/config username="static-user" password="static-password123"
exit

kubectl get secret secretkv -n app -o jsonpath='{.data.password}' | base64 --decode

```

# TEST THE SIDECAR AGENT INJECTOR

go back into the vault container:

```
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```

in the vault container, do:

```
vault secrets enable -path=internal kv-v2

vault kv put internal/database/config username="db-readonly-username" password="db-secret-password"

vault policy write internal-app - <<EOF
path "internal/data/database/config" {
   capabilities = ["read"]
}
EOF

vault write auth/kubernetes/role/internal-app \
      bound_service_account_names=internal-app \
      bound_service_account_namespaces=default,app \
      policies=internal-app \
      ttl=24h

exit
```

create a service account, deploy the app and check if the secret was injected:

```
kubectl create sa internal-app -n app

kubectl apply -f vault/deployment-website.yaml -n app

kubectl exec $(kubectl get pod -l app=website -n app -o jsonpath="{.items[0].metadata.name}") -n app --container website -- cat /vault/secrets/database-config.txt
```

go back into the vault container and update the secret:

```
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```

```
vault kv put internal/database/config username="db-readonly-username" pas
ssword="db-secret-password123"
exit
```

Check if the secret was updated:

```
sleep 30
kubectl exec $(kubectl get pod -l app=website -n app -o jsonpath="{.items[0].metadata.name}") -n app --container website -- cat /vault/secrets/database-config.txt
```

# cleanup

```
helm delete vault-operator -n vault-secret-operator-system
helm delete vault -n vault

kubectl delete namespace app
kubectl delete namespace vault
kubectl delete namespace vault-secret-operator-system
```
