# bank-vault-deploy-crd-charts

A helmchart to Install a CRD Declaration before Using the `Vault` Resource in a given Cloud Platform deployment.

## Bank Vault
[Bank Vaults](https://github.com/banzaicloud/bank-vaults) is a thick, tricky, shifty right with a fast and intense tube for experienced surfers only.
Think heavy steel doors, secret unlocking combinations and burly guards with smack-down attitude.

### Challenge

The `--dry-run` flag of `helm install` and `helm upgrade` is not currently supported for CRDs.

The purpose of "Dry Run" is to validate that the output of the chart will actually work if sent to the server. But CRDs are a modification of the server's behavior.

Helm cannot install the CRD on a dry run, so the discovery client will not know about that `Custom Resource` (CR), and validation will **fail**.

### Solution
as a work-around this limitation this chart will host a pre-configured definition of expected server bahavior to setup a statefull set of Bank Vaults.

Helm is optimized to load as many resources into Kubernetes as fast as possible. By design, Kubernetes can take an entire set of manifests and bring them all online (this is called the reconciliation loop).

![](https://i.imgur.com/DWjeHjr.png)

But there's a difference with **CRDs**.

For a CRD, the declaration must be registered before any resources of that CRDs kind(s) can be used. And the registration process sometimes takes a few seconds.

----
:pushpin:   **Please Take this into consideration in your Pipeline Deployment Order**.
----

### Dependencies
1. An existing [1Password Connect and the 1Password Connect Kubernetes Operator](https://github.com/1Password/connect-helm-charts/tree/main/charts/connect) server in the cluster to be used for secret injection.
2. [1Password connect](https://developer.1password.com/docs/connect/get-started#step-1-set-up-a-secrets-automation-workflow) `Token` with reading rights to lookup `Secrets` in the 1password remote vault.

---
### Reconcile states

Operator controllers work one level of abstraction higher than the Kubernetes controllers. The Kubernetes controllers reconcile built-in kinds like Deployment and Job into lower-level kinds like Pods. Custom controllers reconcile CRDs like [Bank-vault](https://github.com/banzaicloud/bank-vaults) into workload kinds like [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) and [Service.](https://kubernetes.io/docs/concepts/services-networking/) So, a custom controller's current state becomes a Kubernetes controller's desired state.

Both `kinds` of controllers reconcile between the desired and current state, but it takes two rounds of transformation to deploy a workload for an operator's custom resource:

1. The operator’s controller transforms the custom resource into a set of ***managed resources*** (that is, the workload) that are the operator’s current state but are also the control plane’s **desired state**.

2. The Kubernetes controllers transform the ***managed resources*** into running pods (aka the operand) to the **current state** .

![](https://i.imgur.com/xoxlAll.png)



### Deployed Resources

To get an overview of managed resources, between `cluster desired state` &  `Operator desired state`, can be desplayed:
#### Kubernettes Managed resources

```bash=
kubectl get all --namespace=vault
```

```bash=
NAME                                   READY   STATUS    RESTARTS   AGE
pod/vault-0                            3/3     Running   0          4d14h
pod/vault-configurer-bcb87fd54-cz6jv   1/1     Running   0          4d14h
pod/vault-operator-6b65dd5cd8-69wks    1/1     Running   0          4d14h

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                               AGE
service/vault              ClusterIP   < $INT_IP >    <none>        8200/TCP,8201/TCP,9091/TCP,9102/TCP   4d14h
service/vault-0            ClusterIP   < $INT_IP >    <none>        8200/TCP,8201/TCP,9091/TCP            4d14h
service/vault-configurer   ClusterIP   < $INT_IP >    <none>        9091/TCP                              4d14h
service/vault-operator     ClusterIP   < $INT_IP >    <none>        80/TCP,8383/TCP                       4d14h

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vault-configurer   1/1     1            1           4d14h
deployment.apps/vault-operator     1/1     1            1           4d14h

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/vault-configurer-bcb87fd54   1         1         1       4d14h
replicaset.apps/vault-operator-6b65dd5cd8    1         1         1       4d14h

NAME                     READY   AGE
statefulset.apps/vault   1/1     4d14h

```

#### Operator Managed/Required objects
Pre-Configured & Namespace scoped `service account` for the operator to achive the desired state.

```bash=
kubectl get serviceaccounts,role,rolebindings,secrets --namespace=vault
```
```bash=
NAME                            SECRETS   AGE
serviceaccount/default          1         14d
serviceaccount/vault            1         4d22h
serviceaccount/vault-operator   1         4d22h

NAME                                   CREATED AT
role.rbac.authorization.k8s.io/vault   2022-04-29T11:34:38Z

NAME                                          ROLE         AGE
rolebinding.rbac.authorization.k8s.io/vault   Role/vault   4d22h

NAME                                                 TYPE                                  DATA   AGE
secret/sh.helm.release.v1.vault-operator.v1          helm.sh/release.v1                    1      4d22h
secret/sh.helm.release.v1.vault-operator.v2          helm.sh/release.v1                    1      4d21h
secret/sh.helm.release.v1.vault-operator.v3          helm.sh/release.v1                    1      44h
secret/vault-configurer                              Opaque                                1      4d22h
secret/vault-operator-token-dzdmc                    kubernetes.io/service-account-token   3      4d22h
secret/vault-raw-config                              Opaque                                1      4d22h
secret/vault-token-p2644                             kubernetes.io/service-account-token   3      4d22h
secret/vault-unseal-keys                             Opaque                                7      4d22h
```

Using [Shamir Algorithim](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing) to `Initialize & Unseal` the vault in case of a disaster or cluster Migration.

:pushpin:  **[ Important ]:**
Keys are Managed by the Operator and are stored in a `kind: PersistentVolume` inside the cluster.

```bash=
kubectl describe secret vault-unseal-keys
Name:         vault-unseal-keys
Namespace:    vault
Labels:       app.kubernetes.io/name=vault
              vault_cr=vault
Annotations:  <none>

Type:  Opaque

Data
====
vault-unseal-1:  66 bytes
vault-unseal-2:  66 bytes
vault-unseal-3:  66 bytes
vault-unseal-4:  66 bytes
vault-root:      26 bytes
vault-test:      10 bytes
vault-unseal-0:  66 bytes
```

