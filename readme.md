# Gitops SaaS FluxCD Operator

## Deploy FluxCD operator on OpenShift cluster

### Install the Flux CLI

```
curl -s https://fluxcd.io/install.sh | sudo bash
```

### Set the nonroot SCC

```
NS="flux-system"
oc adm policy add-scc-to-user nonroot system:serviceaccount:$NS:kustomize-controller
oc adm policy add-scc-to-user nonroot system:serviceaccount:$NS:helm-controller
oc adm policy add-scc-to-user nonroot system:serviceaccount:$NS:source-controller
oc adm policy add-scc-to-user nonroot system:serviceaccount:$NS:notification-controller
oc adm policy add-scc-to-user nonroot system:serviceaccount:$NS:image-automation-controller
oc adm policy add-scc-to-user nonroot system:serviceaccount:$NS:image-reflector-controller
```

### Export your credentials

```
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

### Run the Flux bootstrap command.

```
flux bootstrap github \
  --hostname=github.ibm.com \
  --ssh-hostname=github.ibm.com \
  --owner=$GITHUB_USER \
  --repository=gitops-saas \
  --branch=main \
  --path=./clusters/heliconia
```

```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=openshift-gitops \
  --branch=main \
  --path=./clusters/heliconia \
  --personal
```

## Encrypting secrets using sops and age

https://github.com/mozilla/sops
https://github.com/FiloSottile/age

### Prepare

1. Generate an age key with age using age-keygen:

```
age-keygen -o age.agekey
```

2. Create a secret with the age private key, the key name must end with .agekey to be detected as an age key:

```
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```

This secret will be deployed in the same cluster as FluxCD operator.

3. Store public key and creation rules in file .sops.yaml

### Use sops and the age public key to encrypt a Kubernetes secret:

1. Add following section to the FluxCD Kustomization

```
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: instana-nightly
  namespace: flux-system
spec:
  ...
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

2. Encrypting secrets
```
sops -e instana-nightly.yaml > instana-nightly.enc.yaml
```