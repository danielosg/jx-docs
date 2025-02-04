---
title: K3s
linktitle: K3s
type: docs
description: Setup Jenkins X on a K3s cluster running locally
date: 2021-11-22
publishdate: 2021-11-22
lastmod: 2021-11-22
weight: 50
aliases:
  - /v3/admin/platform/k3s
---

---

**NOTE**

Ensure you are logged into GitHub else you will get a 404 error when clicking the links below

This guide will walk you though how to setup Jenkins X on your laptop using [k3s](https://k3s.io/)

### Prerequisites

#### K3s

Make sure you have created a cluster using k3s.

If you dont have an existing k3s cluster, you can install one by running:

```bash
curl -sfL https://get.k3s.io | sh -
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/k3s-config
# Set it also in the bashrc or zshrc file, or you can flatten both of these configs into a single file
KUBECONFIG=~/.kube/config:~/.kube/k3s-config
```

To verify that k3s has been installed successfully, and configured run:

```bash
kubectl get nodes
```

This value of the node will be used later during installation and configuring of Jenkins X.

Check [k3s install guide](https://rancher.com/docs/k3s/latest/en/installation/) for more installation options.

#### Vault

Make sure you have vault running in a docker container with kubernetes auth enabled.

```bash
docker run --cap-add=IPC_LOCK -p 8200:8200 -e 'VAULT_DEV_ROOT_TOKEN_ID=myroot' -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200' --net host vault:latest
```

In another terminal run:

```bash
export VAULT_ADDR='http://0.0.0.0:8200'
vault auth enable kubernetes
```

#### Github

- Create a git bot user (different from your own personal user) e.g. https://github.com/join and generate a a personal access token, this will be used by Jenkins X to interact with git repositories. e.g. https://github.com/settings/tokens/new?scopes=repo,read:user,read:org,user:email,write:repo_hook,delete_repo,admin:repo_hook
- This bot user needs to have write permission to write to any git repository used by Jenkins X. This can be done by adding the bot user to the git organisation level or individual repositories as a collaborator Add the new bot user to your Git Organisation, for now give it Owner permissions, we will reduce this to member permissions soon.

### Jenkins X v3 installation

- Generate a cluster git repository from the [jx3-k3s-vault](https://github.com/jx3-gitops-repositories/jx3-k3s-vault) template, by clicking [here](https://github.com/jx3-gitops-repositories/jx3-k3s-vault/generate)
- Edit the value of the vault url in the `jx-requirements.yaml` file.
  Replace with `"http://<replace with k3s node name>:8200"`
- Commit and push your changes:

```bash
git add .
git commit -m "fix: set vault url"
git push origin main
```

- Set the GIT_USERNAME (bot username) and GIT_TOKEN (bot personal access token) env variable and run:

```bash
jx admin operator --username $GIT_USERNAME --token $GIT_TOKEN --url <url of the cluster git repo> --set "jxBootJobEnvVarSecrets.EXTERNAL_VAULT=\"true\"" --set "jxBootJobEnvVarSecrets.VAULT_ADDR=http://<replace with k3s node name>:8200"
```

> Note:
> The first job will fail as it cannot authenticate against vault. Once the secret-infra namespace has been created, we can configure the kubernetes backend

### Vault configuration

Remember to run the following commands in a terminal where you have set the value of `VAULT_ADDR`

- Create a vault config

```bash
VAULT_HELM_SECRET_NAME=$(kubectl -n secret-infra get secrets --output=json | jq -r '.items[].metadata | select(.name|startswith("kubernetes-external-secrets-token-")).name')
TOKEN_REVIEW_JWT=$(kubectl -n secret-infra get secret $VAULT_HELM_SECRET_NAME --output='go-template={{ .data.token }}' | base64 --decode)
KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)
KUBE_HOST=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}')
vault write auth/kubernetes/config \
        token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
        kubernetes_host="$KUBE_HOST" \
        kubernetes_ca_cert="$KUBE_CA_CERT" \
        disable_iss_validation=true
```

- Create a vault role:

```bash
vault write /auth/kubernetes/role/jx-vault bound_service_account_names='*' bound_service_account_namespaces=secret-infra token_policies=jx-policy token_no_default_policy=true disable_iss_validation=true
```

- Create a policy attached to vault role:

```bash
vault policy write jx-policy - <<EOF
path "secret/*" {
  capabilities = ["sudo", "create", "read", "update", "delete", "list"]
}
EOF
```

### Set up ingress and webhook

- Get the external IP of the traefik service (loadbalancer)

```bash
kubectl get svc -A | grep LoadBalancer
kube-system   traefik          LoadBalancer   <cluster-ip>    <external-ip>    80:31123/TCP,443:31783/TCP   40m
```

- Edit the jx-requirements.yaml file by editing the ingress domain:

```bash
jx gitops requirements edit --domain <external-ip>.nip.io
```

- Next, download and install [ngrok](https://ngrok.com/). Run this in a new terminal window/tab:

```bash
ngrok http 8080
```

- Once this tunnel is open, paste the ngrok url in the hook field in the helmfiles/jx/jxboot-helmfile-resources-values.yaml file in the cluster git repository.
- commit and push the changes.

```bash
git add .
git commit -m "chore: new ngrok ip"
git push origin main
```

- once Jenkins X is installed run the following command to enable webhooks via ngrok

```bash
kubectl port-forward svc/hook 8080:80
```

- Once the bootjob has succeeded, you should see:

```bash
HTTP Requests
-------------

POST /hook                     200 OK
```

### Next steps

- <a href="/v3/develop/create-project/" class="btn bg-primary text-light">Create or import projects</a>
