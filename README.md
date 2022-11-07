# Openshift GitOps - Argo CD with Vault Secrets

This repository includes a set of examples to integrate ArgoCD and Vault for creating secrets in Openshift following a secure gitops model.

## Prerequisites

- Openshift 4.11+
- Vault 1.12.0+

## Vault

Vault secures, stores, and tightly controls access to tokens, passwords, certificates, API keys, and other secrets in modern computing.

First of all, it is required to deploy Vault in Openshift in order to save the private information that it will be required to use during the rest of the examples. 

Please follow the next steps, included in the [official documentation](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-openshift), in order to install and configure Vault properly:

```$bash
oc login -u kubeadmin -p xxxxx https://api.my.domain.com:6443

helm repo add hashicorp https://helm.releases.hashicorp.com

oc new-project vault

helm install vault hashicorp/vault \
    --set "global.openshift=true" \
    --set "server.dev.enabled=true"

oc exec -it vault-0 -- /bin/sh

  vault auth enable kubernetes

  vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

  exit
```

### APP 1

Once vault is installed, it is time to deploy an application to test the Vault implementation. This use case is requesting secret information directly from Vault.

Please follow the next steps to create this application:

```$bash
oc create sa webapp

oc exec -it vault-0 -- /bin/sh

  $ vault kv put secret/webapp/config username="static-user" \
    password="static-password"

  $ vault kv get secret/webapp/config

  $ vault policy write webapp - <<EOF
  path "secret/data/webapp/config" {
    capabilities = ["read"]
  }
  EOF

  $ vault write auth/kubernetes/role/webapp \
    bound_service_account_names=webapp \
    bound_service_account_namespaces=vault \
    policies=webapp \
    ttl=24h

  $ exit

oc apply --filename deployment-webapp.yml

oc exec \
    $(oc get pod --selector='app=webapp' --output='jsonpath={.items[0].metadata.name}') \
    --container app -- curl -s http://localhost:8080 

```

### APP 2

Once vault is installed, it is time to deploy a second application in order to test the Vault implementation using config files injection.
 
Please follow the next steps to create an application to test this feature:

```$bash

oc exec -it vault-0 -- /bin/sh

  $ vault kv put secret/issues/config username="annotation-user" \
    password="annotation-password"

  $ vault kv get secret/issues/config

  $ vault policy write issues - <<EOF
    path "secret/data/issues/config" {
    capabilities = ["read"]
    }
  EOF

  $ vault write auth/kubernetes/role/issues \
    bound_service_account_names=issues \
    bound_service_account_namespaces=vault \
    policies=issues \
    ttl=24h

  $ exit

oc apply --filename deployment-issues.yml

oc logs \
    $(oc get pod -l app=issues -o jsonpath="{.items[0].metadata.name}") \
    --container vault-agent

oc exec \
    $(oc get pod -l app=issues -o jsonpath="{.items[0].metadata.name}") \
    --container issues -- cat /vault/secrets/issues-config.txt
```

## Git

In order to have a GitOps model, it is required to create a public GitHub or GitLab repo with a secret.yaml file as unique content (E.g. [secret.yaml](./secret.yaml)). 

This repository will be the source of trust that contains a secret with an annotation that references a Vault secret (*avp.kubernetes.io/path: "secret/data/webapp/config"*). This reference allows ArgoCD, through the respective plugin and vault credentials, to find the required secret in Vault and create a secret in Openshift using this information.

NOTE: It is important to know that this secret is using the references to the *APP 1* Vault secret created before. For this reason, it is required to deploy *APP 1* or create the respective Vault secret at least.

## ArgoCD

Once the previous resources are created and configured properly, it is time to create an Argo CD instance. This new Argo CD instance has to be configured to include *Argo CD Vault Plugin* integration. Please follow the [link](https://argocd-vault-plugin.readthedocs.io/en/stable/) for more information about this integration.

First of all, in order to allow Argo CD to access Vault information, it is required to create a specific token. Please follow the next steps to generate the required token in Vault:

```$bash

oc exec -it vault-0 -- /bin/sh

  $ vault token create
  ...
  Key                Value
  ---                -----
  token              hvs.GZp6u2t007HJJboiXjDUDN0I  <------- Token to access Vault secrets information
  ...
```

NOTE: It is required to update the file **vault.env** in order to update the required information.
 
Regarding the steps to deploy Argo CD in Openshift and create the respective Argo CD application that handles the creation of the secret, once the Red Hat GitOps is installed and the information required to access Vault is included in the respective environment variables file, are included in the following procedure:

```$bash
oc new-project argocd

oc create secret generic argocd-vault-plugin-credentials --from-env-file=vault.env

oc apply -f argocd.yaml

oc apply -f argocd-app.yaml
```

Once the Argo CD instance and the ArgoCD application are created, it will be possible to review the information included in the respective secret with the information saved in Vault:

```$bash
oc project argocd

oc get secret example-secret -o yaml
...
apiVersion: v1
data:
  username: c3RhdGljLXVzZXI=
kind: Secret
...
```

## Author

Asier Cidon @RedHat
