# DEMO - Openshift GitOps - Argo CD with Vault Secrets

This demo includes the basics about setting up an Argo CD environment and using Vault Secrets.

## Prerequisites

- Openshift 4.12+
- Vault 1.13+ on Kubernetes (*namespace vault*)
- Red Hat GitOps Operator
- Helm 
- oc CLI

## Setting Up

- Create Secret in Vault Server

```$bash
oc project vault

oc exec -it vault-0 -- /bin/sh

  $ vault kv put secret/webapp/config username="static-user" \
    password="static-password"

  $ vault kv get secret/webapp/config

  $ exit
```

- Create Vault Credentials for Argo CD

```$bash

oc exec -it vault-0 -- /bin/sh

  $ vault token create
  ...
  Key                Value
  ---                -----
  token              hvs.GZp6u2t007HJJboiXjDUDN0I  <------- Token to access Vault secrets information
  ...

  $ exit

vi vault.env
```

- Create App Namespace

```$bash
oc new-project app-vault

oc get all

oc get secrets
```

- Create Argo CD Instance

```$bash
oc new-project argocd

oc create secret generic argocd-vault-plugin-credentials --from-env-file=vault.env

oc apply -f argocd.yaml
```

- Create Argo CD Application

```$bash
oc apply -f argocd-app.yaml
```

- Access Argo CD console ans Sync Application

```$bash
oc get secret argocd-cluster -o jsonpath='{.data.admin\.password}' -n argocd | base64 -d
oc get route argocd-server -n argocd
```

## Author

Asier Cidon @RedHat
