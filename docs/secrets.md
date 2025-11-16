# Todo Secrets

Create manually:

```bash
kubectl create namespace cert-manager --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic cloudflare-api-token \
  -n cert-manager \
  --from-literal=api-token='<api-key-from-cloudflare>'
```
