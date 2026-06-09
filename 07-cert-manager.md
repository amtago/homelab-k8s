# 07 — cert-manager

## Overview

cert-manager automates the issuance and renewal of TLS certificates from **Let's Encrypt** using the ACME protocol. All HTTPS services in the cluster get their certificates through cert-manager, with Traefik handling TLS termination.

---

## Architecture

```
cert-manager watches Certificate resources
        │
        ▼
Sends ACME challenge to Let's Encrypt
        │
        ├── HTTP-01: Traefik serves /.well-known/acme-challenge/
        │   (requires port 80 open on the VPS)
        │
        └── DNS-01: Updates Route 53 TXT record
            (works even behind firewall — recommended for wildcards)
        │
        ▼
Let's Encrypt issues certificate
        │
        ▼
cert-manager stores it as a Kubernetes Secret
        │
        ▼
Traefik IngressRoute references the Secret for TLS
```

---

## Pods

| Pod | Role | Age | Restarts |
|-----|------|-----|----------|
| `cert-manager-67c98b89c8-cbztj` | Core controller; watches Certificate/CertificateRequest resources | 106d | 0 ✅ |
| `cert-manager-cainjector-5c5695d979-fzcb4` | Injects CA bundles into ValidatingWebhookConfiguration resources | 106d | 0 ✅ |
| `cert-manager-webhook-7f9f8648b9-6g85g` | Validates cert-manager API objects via admission webhook | 106d | 0 ✅ |

All three pods are healthy with zero restarts — cert-manager is stable.

---

## Installation

cert-manager is installed via Helm and must be in place **before** Rancher is deployed.

### Install via Helm

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

### Verify

```bash
kubectl get pods -n cert-manager
kubectl get crds | grep cert-manager
```

---

## ClusterIssuers

A `ClusterIssuer` is a cluster-wide resource that defines how certificates are obtained. This cluster uses **Let's Encrypt** with HTTP-01 challenge validation via Traefik.

### Production ClusterIssuer (HTTP-01)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your@email.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
```

### Staging ClusterIssuer (for testing — no rate limits)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your@email.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
```

> ✅ Always test with the **staging** issuer first. Let's Encrypt production has a rate limit of **5 failed validations per hour** per domain. Staging certificates are not trusted by browsers but won't hit rate limits.

```bash
kubectl apply -f clusterissuer-staging.yaml
kubectl apply -f clusterissuer-prod.yaml

# Verify
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod
```

---

## Certificate Resources

A `Certificate` resource tells cert-manager to obtain a certificate for a specific domain and store it as a Secret.

### Example — Rancher

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rancher-tls
  namespace: cattle-system
spec:
  secretName: rancher-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - rancher.ewtst.com
```

### Example — Harbor

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: harbor-tls
  namespace: harbor
spec:
  secretName: harbor-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - harbor.ewtst.com
```

### Example — Wildcard via DNS-01 (Route 53)

For a wildcard cert covering all subdomains, HTTP-01 won't work — DNS-01 is required.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-ewtst
  namespace: cert-manager
spec:
  secretName: wildcard-ewtst-tls
  issuerRef:
    name: letsencrypt-prod-dns
    kind: ClusterIssuer
  dnsNames:
    - "*.ewtst.com"
    - "ewtst.com"
```

For DNS-01 with Route 53, the ClusterIssuer needs AWS credentials:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod-dns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your@email.com
    privateKeySecretRef:
      name: letsencrypt-prod-dns-key
    solvers:
      - dns01:
          route53:
            region: us-east-1
            hostedZoneID: <YOUR_ROUTE53_ZONE_ID>
            accessKeyIDSecretRef:
              name: route53-credentials
              key: access-key-id
            secretAccessKeySecretRef:
              name: route53-credentials
              key: secret-access-key
```

---

## Certificate Lifecycle

```
Certificate resource created
        │
        ▼
cert-manager creates CertificateRequest
        │
        ▼
cert-manager creates Order (ACME)
        │
        ▼
cert-manager creates Challenge
        │
        ▼ HTTP-01: Traefik serves token at /.well-known/acme-challenge/<token>
          DNS-01:  Route 53 TXT record created
        │
        ▼
Let's Encrypt validates and issues cert
        │
        ▼
cert-manager stores cert in Secret (secretName in Certificate spec)
        │
        ▼
Traefik IngressRoute uses the Secret for TLS
        │
        ▼
cert-manager auto-renews ~30 days before expiry (90-day certs)
```

---

## Integrating with Traefik

Traefik references the TLS secret created by cert-manager in an `IngressRoute`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: harbor-ingress
  namespace: harbor
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`harbor.ewtst.com`)
      kind: Rule
      services:
        - name: harbor-portal
          port: 80
  tls:
    secretName: harbor-tls   # must match Certificate.spec.secretName
```

---

## Checking Certificate Status

```bash
# List all certificates and their ready status
kubectl get certificates --all-namespaces

# Detailed certificate info (expiry, issuer, etc.)
kubectl describe certificate rancher-tls -n cattle-system

# Check certificate requests
kubectl get certificaterequests --all-namespaces

# Check ACME orders
kubectl get orders --all-namespaces

# Check ACME challenges
kubectl get challenges --all-namespaces
```

---

## Troubleshooting

### Certificate stuck in `False` / not Ready

```bash
# Step 1: Check certificate events
kubectl describe certificate <name> -n <namespace>

# Step 2: Check the CertificateRequest
kubectl get certificaterequests -n <namespace>
kubectl describe certificaterequest <name> -n <namespace>

# Step 3: Check the Order
kubectl get orders -n <namespace>
kubectl describe order <name> -n <namespace>

# Step 4: Check the Challenge
kubectl get challenges -n <namespace>
kubectl describe challenge <name> -n <namespace>

# Step 5: Check cert-manager logs
kubectl logs -n cert-manager \
  $(kubectl get pod -n cert-manager -l app=cert-manager -o name) --tail=100
```

### HTTP-01 challenge failing

Common causes:
- Port 80 is firewalled on the VPS → open it: `sudo ufw allow 80/tcp`
- Traefik is not handling port 80 → check Traefik entrypoints config
- DNS not yet propagated → wait and retry
- Wrong `ingressClassName` in solver → must match Traefik's class

```bash
# Verify Traefik is listening on port 80
kubectl get svc -n traefik

# Check Traefik logs
kubectl logs -n traefik $(kubectl get pod -n traefik -o name) --tail=50
```

### Let's Encrypt rate limit hit

If you hit the production rate limit (5 certificates per registered domain per week), switch to staging temporarily:

```bash
# Delete the failed certificate
kubectl delete certificate <name> -n <namespace>

# Re-create with staging issuer
# Change issuerRef.name to letsencrypt-staging in the Certificate yaml
kubectl apply -f certificate-staging.yaml
```

### Certificate not renewing

cert-manager renews certificates automatically when they have less than 30 days remaining. To force a renewal:

```bash
# Delete the secret — cert-manager will re-issue
kubectl delete secret <secretName> -n <namespace>

# Or annotate to trigger renewal
kubectl annotate certificate <name> -n <namespace> \
  cert-manager.io/issuer-name=letsencrypt-prod --overwrite
```

### Check certificate expiry dates

```bash
kubectl get secrets --all-namespaces \
  -o jsonpath='{range .items[?(@.type=="kubernetes.io/tls")]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{end}' | \
  while read ns name; do
    echo -n "$ns/$name: "
    kubectl get secret $name -n $ns \
      -o jsonpath='{.data.tls\.crt}' | \
      base64 -d | \
      openssl x509 -noout -enddate 2>/dev/null || echo "error reading cert"
  done
```

---

## Useful Commands Summary

```bash
# Health check
kubectl get pods -n cert-manager
kubectl get clusterissuer
kubectl get certificates --all-namespaces

# Full debug chain
kubectl get certificates,certificaterequests,orders,challenges --all-namespaces

# cert-manager version
kubectl get pod -n cert-manager -l app=cert-manager \
  -o jsonpath='{.items[0].spec.containers[0].image}'

# Upgrade cert-manager
helm upgrade cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set crds.enabled=true
```
