# SOPS + FluxCD

## Prerequisites

- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) - `brew install kubectl`
- [FluxCD CLI](https://fluxcd.io/docs/cmd/) - `brew install fluxcd/tap/flux`
- [age](https://github.com/FiloSottile/age) - `brew install age`
- [SOPS](https://github.com/mozilla/sops) - `brew install sops`

## Generate age key

Start by adding the `*.agekey` file to your `.gitignore` to avoid committing it to your repository:

```sh
echo "*.agekey" >> .gitignore
```

Generate an age private key:

```sh
age-keygen -o age.agekey
```

Store the public key as an environment variable:

```sh
export AGE_PUBLIC_KEY="<AGE_PUBLIC_KEY>"
```

Store the private key as an environment variable:

```sh
export SOPS_AGE_KEY="$(cat age.agekey)"
```

Save the public key to your flux repository for future encryption of secrets:

```sh
echo ${AGE_PUBLIC_KEY} > ./clusters/${CLUSTER_NAME}/.sops.age.pub
```

Create a secret with the age private key:

```sh
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```

## Configure SOPS

Create a `.sops.yaml` configuration file in the root of your Git repository:

```sh
cat <<EOF > ./.sops.${CLUSTER_NAME}.yaml
creation_rules:
  - path_regex: '.*.yaml'
    encrypted_regex: '^(data|stringData)$'
    age:
      - ${AGE_PUBLIC_KEY}
EOF
```

Create a kustomization for reconciling the secrets on the cluster:

```sh
cat <<EOF > ./clusters/${CLUSTER_NAME}/sops.yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: sops
  namespace: flux-system
spec:
  interval: 5m
  path: ./clusters/${CLUSTER_NAME}
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
EOF
```

## Test generating an encrypted secret

Create a Kubernetes secret manifest file named `secret.yaml`:

```sh
kubectl -n default create secret generic supersecret \
--from-literal=user=sebastian \
--from-literal=password=password \
--dry-run=client \
-o yaml > secret.yaml
```

Encrypt the secret:

```sh
sops --age=${AGE_PUBLIC_KEY} \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place secret.yaml
```

Test decrypting the secret:

```sh
sops --decrypt secret.yaml
```

## Clean Up

Store the private key in a safe place and delete it from your file system:

```sh
rm age.agekey
```
