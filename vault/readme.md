# Vault Demo on Red Hat OpenShift via Helm
1. Prerequisites
2. Configure and start the OpenShift cluster
3. Install the Vault Helm chart
4. Configure Kubernetes authentication
5. Deployment: Request secrets directly from Vault
 - Create the secret
 - Create the service account
 - Define the read policy
 - Create a Kubernetes authentication role
 - Deploy the application
 6. Deployment: Secrets through annotations
 - Create the service account
 - Create the secret
 - Define the read policy
 - Create a Kubernetes authentication role
 - Deploy the application

## 1. Prerequisites
1. Verify the RedHat OpenShift version.
```
oc version
```
2. Verify the Helm version.
```
helm version
```
3. Retrieve the web app and additional configuration
```
git clone https://github.com/hashicorp-education/learn-vault-redhat-openshift.git
```
4. Go to the directory
```
cd learn-vault-redhat-openshift
```

## 2. Configure and start the OpenShift cluster
1. Create `vault` project.
```
oc new-project vault
```

## 3. Install the Vault Helm chart
1. Add the Hashicorp Helm repository.
```
helm repo add hashicorp https://helm.releases.hashicorp.com
```
2. Update all the repositories to ensure helm is aware of the latest versions.
```
helm repo update
```
3. Install the latest version of the Vault server running in development mode configured to work with OpenShift.
```
helm install vault hashicorp/vault \
    --set "global.openshift=true" \
    --set "server.dev.enabled=true" \
    --set "server.image.repository=docker.io/hashicorp/vault" \
    --set "injector.image.repository=docker.io/hashicorp/vault-k8s"
```
4. Display all the pods within the default namespace.
oc get pods

## 4. Configure Kubernetes authentication
1. Start an interactive shell session on the `vault-0` pod.
```
oc exec -it vault-0 -- /bin/sh
```
2. Enable the Kubernetes authentication method.
```
vault auth enable kubernetes
```
3. Configure the Kubernetes authentication method to use the location of the Kubernetes host. It will automatically use the pod's own identity to authenticate with Kubernetes when querying the token review API.
```
vault write auth/kubernetes/config kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```
4. Exit the `vault-0`` pod.
```
exit
```

## 5. Deployment: Request secrets directly from Vault
**Create the service account**
1. Display the service account defined in `service-account-webapp.yml`
```
cat service-account-webapp.yml
```
2. Apply the service account.
```
oc apply --filename service-account-webapp.yml
```
3. Get all the service accounts within the default namespace.
```
oc get serviceaccounts
```
**Create the secret**
1. Start an interactive shell session on the `vault-0` pod.
```
oc exec -it vault-0 -- /bin/sh
```
2. Create a secret at path `secret/webapp/config` with a username and password.
```
vault kv put secret/webapp/config username="static-user" password="static-password"
```
3. Get the secret at path `secret/webapp/config`.
```
vault kv get secret/webapp/config
```
**Define the read policy**
1. Write out the policy named `webapp` that enables the `read` capability for secrets at path `secret/data/webapp/config`.
```
vault policy write webapp - <<EOF
path "secret/data/webapp/config" {
  capabilities = ["read"]
}
EOF

```
**Create a Kubernetes authentication role**
1. Create a Kubernetes authentication role, named `webapp`, that connects the Kubernetes service account name and `webapp` policy.
```
vault write auth/kubernetes/role/webapp bound_service_account_names=webapp bound_service_account_namespaces=vault policies=webapp ttl=24h
```
2. Read the Kubernetes authentication role, named `webapp`, that connects the Kubernetes service account name and `webapp` policy.
```
vault read auth/kubernetes/role/webapp
```
3. Exit the `vault-0` pod.
```
exit
```
**Deploy the application**
1. Display the webapp deployment defined in `deployment-webapp.yml`.
```
cat deployment-webapp.yml
```
2. Apply the webapp deployment.
```
oc apply --filename deployment-webapp.yml
```
3. Display all the pods within the `vault` namespace.
```
oc get pods
```
4. Perform a curl request at http://localhost:8080 on the webapp pod.
```
oc exec \
    $(oc get pod --selector='app=webapp' --output='jsonpath={.items[0].metadata.name}') \
    --container app -- curl -s http://localhost:8080 ; echo

```

## 6. Deployment: Secrets through annotations
**Create the service account**
1. Display the service account defined in `service-account-issues.yml`.
```
cat service-account-issues.yml
```
2. Apply the service account.
```
oc apply --filename service-account-issues.yml
```
3. Get all the service accounts within the default namespace.
```
oc get serviceaccounts
```
**Create the secret**
1. Start an interactive shell session on the `vault-0`` pod.
```
oc exec -it vault-0 -- /bin/sh
```
2. Create a secret at path `secret/issues/config`` with a username and password
```
vault kv put secret/issues/config username="annotation-user" \
    password="annotation-password"
```
3. Get the secret at path `secret/issues/config`.
```
vault kv get secret/issues/config
```
**Define the read policy**
1. Write the policy named `issues` that enables the `read capability` for secrets at path `secret/data/issues/config``.
```
vault policy write issues - <<EOF
path "secret/data/issues/config" {
  capabilities = ["read"]
}
EOF

```
**Create a Kubernetes authentication role**
1. Create a Kubernetes authentication role, named issues, that connects the Kubernetes service account name and issues policy.
```
vault write auth/kubernetes/role/issues \
    bound_service_account_names=issues \
    bound_service_account_namespaces=vault \
    policies=issues \
    ttl=24h
```
2. Exit the `vault-0` pod.
```
exit
```
**Deploy the application**
1. Display the issues deployment defined in `deployment-issues.yml`.
```
cat deployment-issues.yml
```
2. Apply the issues deployment.
```
oc apply --filename deployment-issues.yml
```
3. Display all the pods within the `vault` namespace.
```
oc get pods
```
4. Display the logs of the `vault-agent` container in the `issues` pod.
```
oc logs $(oc get pod -l app=issues -o jsonpath="{.items[0].metadata.name}") --container vault-agent
```
5. Display the secret written to the `issues` container.
```
oc exec $(oc get pod -l app=issues -o jsonpath="{.items[0].metadata.name}") --container issues -- cat /vault/secrets/issues-config.txt ; echo
```
