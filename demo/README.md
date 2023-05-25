# DEMO - Openshift GitOps - Argo CD with Vault Secrets

This demo includes the basics about setting up an Argo CD environment and using Vault Secrets.

## Prerequisites

- Openshift 4.12+
- Vault 1.13+ on Kubernetes (*namespace vault*)
- Red Hat GitOps Operator
- SeadledSecrets
- Oc CLI +4.12 - [Official Doc](https://docs.openshift.com/container-platform/4.12/cli_reference/openshift_cli/getting-started-cli.html)

## Setting Up

- Show SealedSecrets solution

```$bash
oc get pods -n sealedsecrets
```

- Create Secret in Vault Server

```$bash
oc project vault

oc exec -it vault-0 -- /bin/sh

  $ vault kv put secret/webapp/config username="static-user" \
    password="static-password"

## Visit Vault console
```

- Create Vault Credentials for Argo CD

```$bash

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

- Check App Namespace in order to see the respective secret created

```$bash
oc get all -n app-vault

oc get secrets -n app-vault
```

## Author

Asier Cidon @RedHat
