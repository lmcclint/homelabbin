# cert-manager with Porkbun (DNS-01)

Pattern B: k0s manages its own certs using cert-manager and a Porkbun DNS-01 webhook. Caddy on Synology also uses Porkbun DNS-01, but each plane issues independently.

## Install cert-manager (if not already)
Follow cert-manager docs for your k8s.

## Install the Porkbun webhook
Use the Porkbun webhook project and set `groupName` to `acme.porkbun` (matches the ClusterIssuer here). Example Helm install (adjust version):

```bash
helm repo add porkbun-webhook https://kjaleshire.github.io/cert-manager-webhook-porkbun/
helm repo update
helm upgrade --install cert-manager-webhook-porkbun porkbun-webhook/cert-manager-webhook-porkbun \
  --namespace cert-manager \
  --create-namespace \
  --set groupName=acme.porkbun
```

## Create the Porkbun API secret
Create a Secret in the `cert-manager` namespace with your API keys:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: porkbun-credentials
  namespace: cert-manager
stringData:
  api-key: "<PORKBUN_API_KEY>"
  secret-api-key: "<PORKBUN_API_SECRET_KEY>"
```

Apply it:

```bash
kubectl apply -f secret.yaml
```

## Create the ClusterIssuer
Edit `cluster-issuer.yaml` and set your email, then:

```bash
kubectl apply -f cluster-issuer.yaml
```

## Request certs
- Wildcard (recommended for apps under `lab.2bit.name`):

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: lab-2bit-wildcard
  namespace: default
spec:
  secretName: lab-2bit-wildcard-tls
  issuerRef:
    name: le-dns-porkbun
    kind: ClusterIssuer
  dnsNames:
    - "*.lab.2bit.name"
    - "lab.2bit.name"
```

- Or annotate your Ingress to use the issuer and let cert-manager request per-host certs.

Notes
- With DNS-01, your public zone in Porkbun must exist so ACME can verify TXT records. Your service endpoints can remain private on RFC1918 and only in internal DNS.
- Rotate API keys periodically and scope them to DNS only.
