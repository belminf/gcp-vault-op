# gcp-vault-op
K8s Vault Operator deployment using Consul storage backend with GCP Cloud KMS sealing.

## Vault operator install
### Deploy helm chart
Deploy with `etcd-operator` as we're going to use it for the storage backend:

```
helm install stable/vault-operator --name vault --set etcd-operator.enabled=true
```
### Network policy
If you have network policies, you'll need to allow:

* Vault Operator -> Vault: For health checking
* Vault -> Etcd: Vault's storage backend

```
kubectl apply -f netpol.yaml
```

### Example vault

You'll need to create a `VaultService` resource which the Vault operator's custom controller will see and then create the appropriate Kubernetes resources for that Vault.

```
kubectl apply -f vault-example.yaml
```

You should then see a few pods appear:

* 3x Vault pods - `vault-example-$alphanum-$alphanum` (they will be `Running` but the `vault` container in the pod will not be `Ready` since we have not unsealed the Vault)
* 1x etcd pod - `vault-example-etcd-$alphanum`

You should also see all 3 Vault pods listed as sealed under `Vault Status`:
```
kubectl describe vault vault-example
```

To troubleshoot apparent issues, see:
* Logs from the `vault` container in the Vault pods
* Logs from the Vault operator pod

You'll then need to initialize the Vault. To do that, you'll need the `vault` CLI command install. You'll then port forward one of the Vault pods and run the initialize command:

```
# You owe it to yourself to properly set TLS in prod
# But this is just lab-level security so...
export VAULT_SKIP_VERIFY=1

# In a shell you could keep open
kubectl port-forward $(kubectl get vault vault-example -o jsonpath='{.status.vaultStatus.sealed[0]}') port-forward 8200

# Run the operator init
vault operator init
```

**Note:** Save the unseal tokens and the root token for later.

Now, to unseal the Vault, you'll need to use the unseal tokens:

```
# In a shell you could keep open
kubectl port-forward $(kubectl get vault vault-example -o jsonpath='{.status.vaultStatus.sealed[0]}') port-forward 8200

# Do this until you the command returns `Sealed: False`
# Each subsquent run should use a different unseal token

vault operator unseal
```

You need to do this for every node (3 in our example). Once you've unsealed all nodes, the following should return `<nil>`:

```
kubectl get vault vault-example -o jsonpath='{.status.vaultStatus.sealed}'
```
### Resources
* [GCP Cloud KMS seal type doc](https://www.vaultproject.io/docs/configuration/seal/gcpckms.html)
* [Hasicoprp Vault Guides](https://sourcegraph.com/github.com/hashicorp/vault-guides)
* [CoreOS Vault Operator](https://sourcegraph.com/github.com/coreos/vault-operator)
* [Hashicorp - Auto-unseal with GCP Cloud KMS](https://learn.hashicorp.com/vault/operations/autounseal-gcp-kms)
* [Vault Operator Helm Chart](https://github.com/helm/charts/tree/master/stable/vault-operator)
* [Hashicorp Consul Helm Chart](https://www.hashicorp.com/blog/announcing-the-consul-helm-chart)
