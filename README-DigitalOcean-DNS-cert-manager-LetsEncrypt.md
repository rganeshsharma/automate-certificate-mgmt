# DigitalOcean DNS + cert-manager + Let's Encrypt

End-to-end guide for delegating a public domain to DigitalOcean DNS and automating browser-trusted TLS certificate issuance and renewal for an RKE2/Kubernetes platform using cert-manager, Let's Encrypt, DNS-01, and Gateway API.

---

## Table of Contents

1. [Architecture](#1-architecture)
2. [What We Are Building](#2-what-we-are-building)
3. [Prerequisites](#3-prerequisites)
4. [Terminology: AWS vs DigitalOcean](#4-terminology-aws-vs-digitalocean)
5. [Repository Layout](#5-repository-layout)
6. [Step 1 - Add the Domain to DigitalOcean DNS](#6-step-1---add-the-domain-to-digitalocean-dns)
7. [Step 2 - Delegate the Domain at the Registrar](#7-step-2---delegate-the-domain-at-the-registrar)
8. [Step 3 - Verify DNS Delegation](#8-step-3---verify-dns-delegation)
9. [Step 4 - Create DNS with OpenTofu](#9-step-4---create-dns-with-opentofu)
10. [Step 5 - Create the Wildcard Application DNS Record](#10-step-5---create-the-wildcard-application-dns-record)
11. [Step 6 - Install cert-manager](#11-step-6---install-cert-manager)
12. [Step 7 - Create a DigitalOcean DNS API Token](#12-step-7---create-a-digitalocean-dns-api-token)
13. [Step 8 - Store the DNS Token in Kubernetes](#13-step-8---store-the-dns-token-in-kubernetes)
14. [Step 9 - Create a Let's Encrypt Staging ClusterIssuer](#14-step-9---create-a-lets-encrypt-staging-clusterissuer)
15. [Step 10 - Request a Staging Wildcard Certificate](#15-step-10---request-a-staging-wildcard-certificate)
16. [Step 11 - Validate DNS-01](#16-step-11---validate-dns-01)
17. [Step 12 - Create the Production ClusterIssuer](#17-step-12---create-the-production-clusterissuer)
18. [Step 13 - Request the Production Wildcard Certificate](#18-step-13---request-the-production-wildcard-certificate)
19. [Step 14 - Configure Gateway API TLS](#19-step-14---configure-gateway-api-tls)
20. [Step 15 - Create HTTPRoutes](#20-step-15---create-httproutes)
21. [Step 16 - Verify HTTPS End to End](#21-step-16---verify-https-end-to-end)
22. [How Automatic Renewal Works](#22-how-automatic-renewal-works)
23. [Certificate Lifecycle](#23-certificate-lifecycle)
24. [Production Security Recommendations](#24-production-security-recommendations)
25. [Troubleshooting](#25-troubleshooting)
26. [Useful Commands](#26-useful-commands)
27. [GitOps Evolution](#27-gitops-evolution)
28. [Complete Traffic Flow](#28-complete-traffic-flow)
29. [Final Production Architecture](#29-final-production-architecture)
30. [References](#30-references)

---

# 1. Architecture

```text
                           INTERNET
                              |
                              | DNS query
                              v
                     +------------------+
                     | Domain Registrar |
                     +--------+---------+
                              |
                              | NS delegation
                              v
                  +-------------------------+
                  |   DigitalOcean DNS      |
                  | Authoritative DNS Zone  |
                  +-----------+-------------+
                              |
                   *.dev.example.com
                              |
                              v
                 +---------------------------+
                 | DigitalOcean Load Balancer|
                 | Public IP                 |
                 +-------------+-------------+
                               |
                         TCP 80 / 443
                               |
                               v
                 +---------------------------+
                 | Envoy Gateway             |
                 | Kubernetes Gateway API    |
                 | TLS termination           |
                 +-------------+-------------+
                               |
                         HTTPRoute routing
                               |
             +-----------------+------------------+
             |                 |                  |
             v                 v                  v
          Argo CD           Harbor              vLLM
```

Certificate automation:

```text
cert-manager
     |
     | ACME
     v
Let's Encrypt
     |
     | DNS-01 challenge
     v
_acme-challenge.dev.example.com
     |
     | DigitalOcean DNS API
     v
Temporary TXT record
     |
     | successful validation
     v
Let's Encrypt issues certificate
     |
     v
Kubernetes TLS Secret
     |
     v
Gateway API HTTPS listener
```

---

# 2. What We Are Building

The target platform uses:

- A real public domain such as `example.com`
- DigitalOcean DNS as the authoritative DNS provider
- `dev.example.com` as the platform environment
- `*.dev.example.com` as a wildcard application namespace
- A DigitalOcean Load Balancer as the public entry point
- RKE2 as Kubernetes
- Gateway API with Envoy Gateway
- cert-manager for certificate lifecycle management
- Let's Encrypt as the public Certificate Authority
- ACME DNS-01 validation
- A wildcard certificate covering:
  - `dev.example.com`
  - `*.dev.example.com`
- Automatic certificate renewal

Example application names:

```text
argocd.dev.example.com
harbor.dev.example.com
grafana.dev.example.com
prometheus.dev.example.com
keycloak.dev.example.com
vllm.dev.example.com
api.dev.example.com
```

Throughout this guide replace:

```text
example.com
```

with your real domain.

---

# 3. Prerequisites

You should have:

- A domain registered with any registrar
- A DigitalOcean account
- A DigitalOcean API token for infrastructure provisioning
- OpenTofu installed
- kubectl installed
- Helm 3 installed
- An RKE2/Kubernetes cluster
- Gateway API CRDs installed
- Envoy Gateway or another Gateway API implementation installed
- A public DigitalOcean Load Balancer
- TCP ports 80 and 443 reachable where required

Check local tooling:

```bash
tofu version
kubectl version --client
helm version
```

Verify Kubernetes:

```bash
kubectl get nodes
```

---

# 4. Terminology: AWS vs DigitalOcean

| AWS | DigitalOcean |
|---|---|
| Route 53 Hosted Zone | DigitalOcean Domain / DNS Zone |
| Route 53 A Record | DigitalOcean A Record |
| Route 53 CNAME | DigitalOcean CNAME |
| Route 53 TXT | DigitalOcean TXT |
| Route 53 NS | DigitalOcean Nameservers |
| ALB/NLB | DigitalOcean Load Balancer |
| ACM | cert-manager + Let's Encrypt |
| Route 53 API | DigitalOcean DNS API |

The registrar and DNS provider do not need to be the same company.

The registrar owns the domain registration.

The authoritative DNS provider answers DNS queries for the domain.

---

# 5. Repository Layout

Recommended repository:

```text
domain-platform/
├── README.md
├── opentofu/
│   └── dns/
│       ├── versions.tf
│       ├── providers.tf
│       ├── variables.tf
│       ├── main.tf
│       ├── outputs.tf
│       └── terraform.tfvars.example
│
├── cert-manager/
│   ├── namespace.yaml
│   ├── clusterissuer-staging.yaml
│   ├── clusterissuer-production.yaml
│   ├── certificate-staging.yaml
│   └── certificate-production.yaml
│
├── gateway-api/
│   ├── gateway.yaml
│   └── routes/
│       ├── argocd.yaml
│       ├── harbor.yaml
│       └── vllm.yaml
│
└── .gitignore
```

Never commit:

```text
terraform.tfstate
terraform.tfstate.backup
*.tfstate
*.tfstate.*
*.tfvars
.env
secrets.yaml
```

Example `.gitignore`:

```gitignore
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.*
*.tfvars
.env
secrets.yaml
```

You may choose to commit `.terraform.lock.hcl` in an actual infrastructure repository to pin provider selections. If so, remove it from the example `.gitignore`.

---

# 6. Step 1 - Add the Domain to DigitalOcean DNS

Assume the domain is:

```text
example.com
```

In DigitalOcean:

```text
Networking
  -> Domains
  -> Add a domain
```

Add the apex/root domain:

```text
example.com
```

Do not add every application as a separate domain.

Do not use:

```text
argocd.dev.example.com
harbor.dev.example.com
```

as individual DigitalOcean domains.

They are DNS records belonging to the `example.com` zone.

If the domain already serves production traffic, first recreate all existing records in DigitalOcean before changing the nameservers.

---

# 7. Step 2 - Delegate the Domain at the Registrar

DigitalOcean's authoritative nameservers are:

```text
ns1.digitalocean.com
ns2.digitalocean.com
ns3.digitalocean.com
```

At the registrar where the domain was purchased:

```text
Domain
  -> DNS / Nameservers
  -> Custom Nameservers
```

Replace the registrar's existing nameservers with:

```text
ns1.digitalocean.com
ns2.digitalocean.com
ns3.digitalocean.com
```

This operation is called **DNS delegation**.

After delegation:

```text
Root DNS
   |
   v
Registrar delegation
   |
   v
ns1.digitalocean.com
ns2.digitalocean.com
ns3.digitalocean.com
   |
   v
DigitalOcean becomes authoritative for example.com
```

The registrar still owns the registration.

DigitalOcean now hosts the DNS zone.

---

# 8. Step 3 - Verify DNS Delegation

Check nameservers:

```bash
dig NS example.com
```

Expected:

```text
example.com.    NS    ns1.digitalocean.com.
example.com.    NS    ns2.digitalocean.com.
example.com.    NS    ns3.digitalocean.com.
```

Query a specific authoritative server:

```bash
dig @ns1.digitalocean.com example.com NS
```

You can also use:

```bash
nslookup -type=NS example.com
```

DNS changes can require some time to propagate due to caching and TTLs.

Do not proceed with ACME DNS-01 troubleshooting until the public internet sees DigitalOcean as authoritative.

---

# 9. Step 4 - Create DNS with OpenTofu

The initial registrar nameserver change is generally manual.

After delegation, manage the DNS zone and DNS records through OpenTofu.

## `versions.tf`

```hcl
terraform {
  required_version = ">= 1.8.0"

  required_providers {
    digitalocean = {
      source  = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
  }
}
```

Use the exact provider version appropriate for your environment and pin it through the lock file.

## `providers.tf`

```hcl
provider "digitalocean" {
  token = var.do_token
}
```

## `variables.tf`

```hcl
variable "do_token" {
  description = "DigitalOcean API token"
  type        = string
  sensitive   = true
}

variable "domain_name" {
  description = "Root DNS domain"
  type        = string
}

variable "load_balancer_ip" {
  description = "Public IP address of the platform Load Balancer"
  type        = string
}
```

Prefer environment variables rather than writing the token into a `.tfvars` file:

```bash
export TF_VAR_do_token="$DIGITALOCEAN_TOKEN"
```

## `main.tf`

If OpenTofu is responsible for creating the DigitalOcean DNS zone:

```hcl
resource "digitalocean_domain" "main" {
  name = var.domain_name
}
```

Create a wildcard application record:

```hcl
resource "digitalocean_record" "dev_wildcard" {
  domain = digitalocean_domain.main.id
  type   = "A"
  name   = "*.dev"
  value  = var.load_balancer_ip
  ttl    = 300
}
```

Optionally create the environment apex:

```hcl
resource "digitalocean_record" "dev" {
  domain = digitalocean_domain.main.id
  type   = "A"
  name   = "dev"
  value  = var.load_balancer_ip
  ttl    = 300
}
```

## `outputs.tf`

```hcl
output "domain" {
  value = digitalocean_domain.main.name
}

output "wildcard_record" {
  value = "*.${digitalocean_record.dev_wildcard.name}.${digitalocean_domain.main.name}"
}
```

## `terraform.tfvars.example`

```hcl
domain_name      = "example.com"
load_balancer_ip = "203.0.113.10"
```

Initialize:

```bash
tofu init
```

Format:

```bash
tofu fmt -recursive
```

Validate:

```bash
tofu validate
```

Plan:

```bash
tofu plan
```

Apply:

```bash
tofu apply
```

---

# 10. Step 5 - Create the Wildcard Application DNS Record

The key DNS record is:

```text
*.dev.example.com -> LOAD_BALANCER_PUBLIC_IP
```

Example:

```text
*.dev.example.com -> 203.0.113.10
```

Now all of these resolve to the same Load Balancer:

```text
argocd.dev.example.com
harbor.dev.example.com
grafana.dev.example.com
vllm.dev.example.com
api.dev.example.com
```

Verify:

```bash
dig argocd.dev.example.com A
dig harbor.dev.example.com A
dig vllm.dev.example.com A
```

All should resolve to the Load Balancer IP.

A wildcard DNS record is independent from a wildcard TLS certificate.

You need both:

```text
Wildcard DNS
*.dev.example.com
        |
        v
Load Balancer
```

and:

```text
Wildcard TLS certificate
*.dev.example.com
        |
        v
Gateway TLS termination
```

---

# 11. Step 6 - Install cert-manager

As of September 2026, cert-manager documentation lists `v1.21.1` as the current release.

Use the official OCI chart:

```bash
helm upgrade --install cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --version v1.21.1 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Verify:

```bash
kubectl get pods -n cert-manager
```

Expected components:

```text
cert-manager
cert-manager-cainjector
cert-manager-webhook
```

Verify CRDs:

```bash
kubectl get crd | grep cert-manager
```

Important CRDs include:

```text
certificates.cert-manager.io
certificaterequests.cert-manager.io
challenges.acme.cert-manager.io
clusterissuers.cert-manager.io
issuers.cert-manager.io
orders.acme.cert-manager.io
```

Check:

```bash
kubectl get clusterissuers
```

At this stage there may not be any issuers yet.

---

# 12. Step 7 - Create a DigitalOcean DNS API Token

DNS-01 requires cert-manager to temporarily create DNS TXT records.

The flow is:

```text
cert-manager
     |
     | DigitalOcean API
     v
DigitalOcean DNS
     |
     +--> create TXT
          _acme-challenge.dev.example.com
```

Therefore cert-manager needs a DigitalOcean API token with enough permission to modify DNS records.

Use a dedicated credential for certificate automation rather than reusing a broad personal infrastructure token whenever your DigitalOcean IAM model allows you to reduce its scope.

Do not commit the token to Git.

Do not put the raw token inside:

```text
README.md
values.yaml
ClusterIssuer YAML
Certificate YAML
OpenTofu source files
```

---

# 13. Step 8 - Store the DNS Token in Kubernetes

Create the Secret imperatively so the raw credential never exists in the Git repository:

```bash
kubectl create secret generic digitalocean-dns \
  --namespace cert-manager \
  --from-literal=access-token="$DO_CERT_MANAGER_TOKEN"
```

Verify the Secret exists:

```bash
kubectl get secret digitalocean-dns -n cert-manager
```

Do not print the Secret value unnecessarily.

The equivalent YAML shape is:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: digitalocean-dns
  namespace: cert-manager
type: Opaque
stringData:
  access-token: REDACTED
```

Do **not** commit a manifest containing the real token.

For a mature GitOps implementation use one of:

- External Secrets Operator
- HashiCorp Vault / OpenBao
- SOPS
- Sealed Secrets
- another supported enterprise secret-management system

---

# 14. Step 9 - Create a Let's Encrypt Staging ClusterIssuer

Always test ACME automation against Let's Encrypt staging first.

Create:

```text
cert-manager/clusterissuer-staging.yaml
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: admin@example.com

    server: https://acme-staging-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-staging-account-key

    solvers:
      - dns01:
          digitalocean:
            tokenSecretRef:
              name: digitalocean-dns
              key: access-token

        selector:
          dnsZones:
            - "example.com"
```

Replace:

```text
admin@example.com
example.com
```

with your values.

Apply:

```bash
kubectl apply -f cert-manager/clusterissuer-staging.yaml
```

Check:

```bash
kubectl get clusterissuer letsencrypt-staging
```

Detailed status:

```bash
kubectl describe clusterissuer letsencrypt-staging
```

Expected:

```text
Ready: True
```

---

# 15. Step 10 - Request a Staging Wildcard Certificate

Create the namespace that owns the Gateway if it does not exist:

```bash
kubectl create namespace envoy-gateway-system
```

Use the actual namespace where your Gateway object will exist.

Create:

```text
cert-manager/certificate-staging.yaml
```

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dev-example-com-staging
  namespace: envoy-gateway-system
spec:
  secretName: dev-example-com-staging-tls

  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
    group: cert-manager.io

  dnsNames:
    - "dev.example.com"
    - "*.dev.example.com"
```

Apply:

```bash
kubectl apply -f cert-manager/certificate-staging.yaml
```

Watch:

```bash
kubectl get certificate -n envoy-gateway-system -w
```

Inspect:

```bash
kubectl describe certificate \
  dev-example-com-staging \
  -n envoy-gateway-system
```

---

# 16. Step 11 - Validate DNS-01

cert-manager creates ACME resources automatically.

Inspect them:

```bash
kubectl get certificaterequests -A
kubectl get orders -A
kubectl get challenges -A
```

Describe the challenge:

```bash
kubectl describe challenge -A
```

During validation a TXT record appears similar to:

```text
_acme-challenge.dev.example.com
```

Check it:

```bash
dig TXT _acme-challenge.dev.example.com
```

or against DigitalOcean directly:

```bash
dig @ns1.digitalocean.com TXT _acme-challenge.dev.example.com
```

The process is:

```text
Certificate
     |
     v
CertificateRequest
     |
     v
ACME Order
     |
     v
ACME Challenge
     |
     v
cert-manager calls DigitalOcean DNS API
     |
     v
TXT _acme-challenge.dev.example.com
     |
     v
Let's Encrypt verifies TXT record
     |
     v
Certificate issued
     |
     v
TLS Secret created
```

Once successful:

```bash
kubectl get certificate \
  dev-example-com-staging \
  -n envoy-gateway-system
```

Expected:

```text
READY   True
```

Check resulting Secret:

```bash
kubectl get secret \
  dev-example-com-staging-tls \
  -n envoy-gateway-system
```

Because this certificate came from the Let's Encrypt staging CA, browsers will not trust it.

That is expected.

Staging proves the automation works without consuming production issuance attempts.

---

# 17. Step 12 - Create the Production ClusterIssuer

Create:

```text
cert-manager/clusterissuer-production.yaml
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    email: admin@example.com

    server: https://acme-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-production-account-key

    solvers:
      - dns01:
          digitalocean:
            tokenSecretRef:
              name: digitalocean-dns
              key: access-token

        selector:
          dnsZones:
            - "example.com"
```

Apply:

```bash
kubectl apply -f cert-manager/clusterissuer-production.yaml
```

Verify:

```bash
kubectl get clusterissuer
```

Expected:

```text
NAME                       READY
letsencrypt-staging        True
letsencrypt-production     True
```

Inspect if necessary:

```bash
kubectl describe clusterissuer letsencrypt-production
```

---

# 18. Step 13 - Request the Production Wildcard Certificate

Create:

```text
cert-manager/certificate-production.yaml
```

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: dev-example-com
  namespace: envoy-gateway-system
spec:
  secretName: dev-example-com-tls

  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
    group: cert-manager.io

  dnsNames:
    - "dev.example.com"
    - "*.dev.example.com"
```

Apply:

```bash
kubectl apply -f cert-manager/certificate-production.yaml
```

Watch:

```bash
kubectl get certificate -n envoy-gateway-system -w
```

Verify:

```bash
kubectl get certificate dev-example-com -n envoy-gateway-system
```

Expected:

```text
READY   True
```

Inspect:

```bash
kubectl describe certificate dev-example-com -n envoy-gateway-system
```

Check Secret:

```bash
kubectl get secret dev-example-com-tls -n envoy-gateway-system
```

The TLS Secret contains:

```text
tls.crt
tls.key
```

Do not manually copy certificate files around the cluster.

cert-manager owns the certificate lifecycle.

---

# 19. Step 14 - Configure Gateway API TLS

Example Gateway:

```text
gateway-api/gateway.yaml
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: platform-gateway
  namespace: envoy-gateway-system
spec:
  gatewayClassName: envoy

  listeners:
    - name: http
      protocol: HTTP
      port: 80
      hostname: "*.dev.example.com"
      allowedRoutes:
        namespaces:
          from: All

    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.dev.example.com"

      tls:
        mode: Terminate
        certificateRefs:
          - group: ""
            kind: Secret
            name: dev-example-com-tls

      allowedRoutes:
        namespaces:
          from: All
```

Apply:

```bash
kubectl apply -f gateway-api/gateway.yaml
```

Inspect:

```bash
kubectl get gateway -A
```

Detailed status:

```bash
kubectl describe gateway \
  platform-gateway \
  -n envoy-gateway-system
```

The HTTPS flow is now:

```text
Client
   |
   | HTTPS
   v
DigitalOcean Load Balancer
   |
   v
Envoy Gateway :443
   |
   | uses Kubernetes Secret
   v
dev-example-com-tls
```

TLS is terminated at the Gateway.

Backend applications can remain ClusterIP Services.

---

# 20. Step 15 - Create HTTPRoutes

## Argo CD

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: argocd
  namespace: argocd
spec:
  parentRefs:
    - name: platform-gateway
      namespace: envoy-gateway-system
      sectionName: https

  hostnames:
    - "argocd.dev.example.com"

  rules:
    - backendRefs:
        - name: argocd-server
          port: 80
```

## Harbor

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: harbor
  namespace: harbor
spec:
  parentRefs:
    - name: platform-gateway
      namespace: envoy-gateway-system
      sectionName: https

  hostnames:
    - "harbor.dev.example.com"

  rules:
    - backendRefs:
        - name: harbor
          port: 80
```

Adjust the Harbor backend Service and port to match your actual Helm deployment.

## vLLM

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: vllm
  namespace: inference
spec:
  parentRefs:
    - name: platform-gateway
      namespace: envoy-gateway-system
      sectionName: https

  hostnames:
    - "vllm.dev.example.com"

  rules:
    - backendRefs:
        - name: vllm
          port: 8000
```

Apply:

```bash
kubectl apply -f gateway-api/routes/
```

Because the wildcard DNS record already exists, creating a new hostname usually requires no new DNS record:

```text
newapp.dev.example.com
        |
        v
*.dev.example.com
        |
        v
Load Balancer
```

You only need a new HTTPRoute.

---

# 21. Step 16 - Verify HTTPS End to End

Check DNS:

```bash
dig argocd.dev.example.com
```

Check TCP/TLS:

```bash
curl -Iv https://argocd.dev.example.com
```

Inspect certificate:

```bash
openssl s_client \
  -connect argocd.dev.example.com:443 \
  -servername argocd.dev.example.com \
  </dev/null
```

Show certificate metadata:

```bash
echo | openssl s_client \
  -connect argocd.dev.example.com:443 \
  -servername argocd.dev.example.com \
  2>/dev/null \
  | openssl x509 -noout \
      -issuer \
      -subject \
      -dates \
      -ext subjectAltName
```

You should see SANs containing:

```text
DNS:dev.example.com
DNS:*.dev.example.com
```

The issuer should be part of the Let's Encrypt production chain.

Browser test:

```text
https://argocd.dev.example.com
https://harbor.dev.example.com
https://vllm.dev.example.com
```

These should no longer produce a self-signed-certificate warning.

---

# 22. How Automatic Renewal Works

This is the main reason for using cert-manager.

You do **not** manually:

```text
download a certificate
copy tls.crt
copy tls.key
replace a Secret
restart a web server
track expiration dates
```

Instead:

```text
Certificate resource
       |
       v
cert-manager controller
       |
       | monitors certificate lifetime
       |
       v
renewal required
       |
       v
new ACME Order
       |
       v
new DNS-01 challenge
       |
       v
temporary DigitalOcean TXT record
       |
       v
Let's Encrypt validation
       |
       v
new certificate
       |
       v
same Kubernetes TLS Secret updated
       |
       v
Gateway consumes renewed certificate
```

The desired state remains:

```yaml
kind: Certificate
spec:
  secretName: dev-example-com-tls
```

cert-manager handles the lifecycle behind that declaration.

Do not create cron jobs to renew Let's Encrypt certificates when cert-manager is already managing them.

---

# 23. Certificate Lifecycle

The key Kubernetes objects are:

```text
ClusterIssuer
     |
     v
Certificate
     |
     v
CertificateRequest
     |
     v
Order
     |
     v
Challenge
     |
     v
DNS TXT record
     |
     v
Let's Encrypt
     |
     v
TLS Secret
```

Useful commands:

```bash
kubectl get clusterissuer
kubectl get certificate -A
kubectl get certificaterequest -A
kubectl get order -A
kubectl get challenge -A
```

---

# 24. Production Security Recommendations

## 24.1 Never Commit DigitalOcean API Tokens

Bad:

```yaml
stringData:
  access-token: dop_v1_REAL_TOKEN
```

committed to Git.

Good:

```text
Git
 |
 | contains Secret reference only
 v
ClusterIssuer
 |
 v
Kubernetes Secret / External Secret
 |
 v
actual token
```

---

## 24.2 Separate Infrastructure and cert-manager Credentials

Prefer separate credentials for:

```text
OpenTofu infrastructure provisioning
```

and:

```text
cert-manager DNS validation
```

This reduces blast radius.

---

## 24.3 Use Minimum Required Permissions

The cert-manager credential only needs the permissions required to perform its DNS validation duties.

Avoid giving cluster workloads an unrestricted infrastructure administration credential when a narrower credential can be used.

---

## 24.4 Prefer an External Secret Manager for Production

Eventually:

```text
OpenBao / Vault
       |
       v
External Secrets Operator
       |
       v
Kubernetes Secret
       |
       v
cert-manager
```

This is preferable to storing long-lived credentials manually in Kubernetes.

---

## 24.5 Separate Staging and Production Issuers

Always keep:

```text
letsencrypt-staging
letsencrypt-production
```

as distinct resources.

Use staging while debugging.

Move to production only once DNS-01 succeeds.

---

## 24.6 Do Not Use Self-Signed Certificates for Public Services

Self-signed CA:

```text
good for:
internal labs
air-gapped environments
private corporate PKI
```

Let's Encrypt:

```text
good for:
internet-accessible domains
browser-trusted TLS
automated public certificates
```

---

## 24.7 Consider CAA Records

CAA records can restrict which Certificate Authorities may issue certificates for your domain.

Example concept:

```text
CAA example.com -> permit Let's Encrypt
```

Implement this only after understanding all CAs used by the domain so you do not accidentally block legitimate issuance.

---

# 25. Troubleshooting

## Problem 1 - ClusterIssuer is not Ready

Check:

```bash
kubectl describe clusterissuer letsencrypt-production
```

Controller logs:

```bash
kubectl logs \
  -n cert-manager \
  deployment/cert-manager
```

---

## Problem 2 - Certificate remains Pending

```bash
kubectl describe certificate \
  dev-example-com \
  -n envoy-gateway-system
```

Then:

```bash
kubectl get certificaterequest -A
kubectl get order -A
kubectl get challenge -A
```

Describe the current Challenge:

```bash
kubectl describe challenge -A
```

---

## Problem 3 - DNS-01 TXT record not appearing

Check public DNS:

```bash
dig TXT _acme-challenge.dev.example.com
```

Check authoritative DigitalOcean DNS directly:

```bash
dig @ns1.digitalocean.com \
  TXT _acme-challenge.dev.example.com
```

Check cert-manager logs:

```bash
kubectl logs \
  -n cert-manager \
  deployment/cert-manager \
  --tail=200
```

Possible causes:

```text
invalid DigitalOcean token
insufficient token permissions
wrong Secret name
wrong Secret key
wrong DNS zone
domain not delegated to DigitalOcean
DNS propagation/caching
incorrect ClusterIssuer selector
```

---

## Problem 4 - Wrong Nameservers

```bash
dig NS example.com
```

If the result does not return DigitalOcean nameservers, fix delegation at the registrar.

Expected:

```text
ns1.digitalocean.com
ns2.digitalocean.com
ns3.digitalocean.com
```

---

## Problem 5 - DNS resolves but site does not open

Check:

```bash
dig argocd.dev.example.com
```

Then:

```bash
kubectl get svc -A
kubectl get gateway -A
kubectl get httproute -A
```

Check Gateway conditions:

```bash
kubectl describe gateway \
  platform-gateway \
  -n envoy-gateway-system
```

Check HTTPRoute:

```bash
kubectl describe httproute argocd -n argocd
```

Verify Load Balancer firewall rules permit the required inbound traffic.

---

## Problem 6 - Browser sees wrong certificate

Inspect the certificate presented by the endpoint:

```bash
echo | openssl s_client \
  -connect argocd.dev.example.com:443 \
  -servername argocd.dev.example.com \
  2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates
```

Then inspect the Kubernetes Secret:

```bash
kubectl get secret \
  dev-example-com-tls \
  -n envoy-gateway-system
```

Confirm the Gateway references:

```yaml
certificateRefs:
  - name: dev-example-com-tls
```

and not the staging Secret.

---

## Problem 7 - Certificate works but Gateway does not accept the Secret

Gateway certificate references are namespace-sensitive.

The simplest design is:

```text
Gateway
  namespace: envoy-gateway-system

Certificate
  namespace: envoy-gateway-system

TLS Secret
  namespace: envoy-gateway-system
```

This avoids cross-namespace certificate references.

If you intentionally reference objects across namespaces, implement the appropriate Gateway API authorization model such as `ReferenceGrant` where required.

---

## Problem 8 - Wildcard certificate does not cover deeper names

This certificate:

```text
*.dev.example.com
```

covers:

```text
argocd.dev.example.com
harbor.dev.example.com
vllm.dev.example.com
```

It does **not** cover:

```text
api.ml.dev.example.com
foo.bar.dev.example.com
```

A wildcard only covers one DNS label at that wildcard position.

---

# 26. Useful Commands

## DNS

```bash
dig NS example.com
dig A dev.example.com
dig A argocd.dev.example.com
dig TXT _acme-challenge.dev.example.com
```

## cert-manager

```bash
kubectl get pods -n cert-manager
kubectl get clusterissuer
kubectl get certificate -A
kubectl get certificaterequest -A
kubectl get order -A
kubectl get challenge -A
```

## Gateway API

```bash
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A
```

## Secrets

```bash
kubectl get secret \
  dev-example-com-tls \
  -n envoy-gateway-system
```

## TLS

```bash
curl -Iv https://argocd.dev.example.com
```

```bash
openssl s_client \
  -connect argocd.dev.example.com:443 \
  -servername argocd.dev.example.com
```

---

# 27. GitOps Evolution

The first implementation can use:

```text
Helm CLI
kubectl apply
```

Once Argo CD is available, move platform resources to GitOps.

Target:

```text
GitHub
  |
  v
Argo CD
  |
  +--> cert-manager Helm deployment
  |
  +--> ClusterIssuer
  |
  +--> Certificate
  |
  +--> Gateway
  |
  +--> HTTPRoutes
```

Do not store the DigitalOcean token directly in Git.

Production secret flow:

```text
Vault/OpenBao
     |
     v
External Secrets Operator
     |
     v
digitalocean-dns Secret
     |
     v
cert-manager
```

OpenTofu remains responsible for cloud infrastructure:

```text
OpenTofu
  |
  +--> VPC
  +--> firewall
  +--> CPU nodes
  +--> GPU nodes
  +--> volumes
  +--> Load Balancer
  +--> DigitalOcean DNS
```

Argo CD manages Kubernetes platform configuration:

```text
Argo CD
  |
  +--> Cilium
  +--> Gateway API
  +--> Envoy Gateway
  +--> cert-manager
  +--> ClusterIssuers
  +--> Certificates
  +--> Harbor
  +--> observability
  +--> inference stack
```

This creates a clean responsibility boundary:

```text
OpenTofu = cloud infrastructure

Argo CD = Kubernetes desired state

cert-manager = certificate lifecycle

Let's Encrypt = public Certificate Authority

DigitalOcean DNS = authoritative DNS + ACME DNS-01 records
```

---

# 28. Complete Traffic Flow

Request:

```text
https://vllm.dev.example.com/v1/chat/completions
```

### Step 1 - DNS

```text
vllm.dev.example.com
        |
        v
*.dev.example.com
        |
        v
203.0.113.10
```

### Step 2 - Load Balancer

```text
203.0.113.10:443
      |
      v
DigitalOcean Load Balancer
```

### Step 3 - Gateway

```text
Load Balancer
     |
     v
Envoy Gateway
     |
     +--> TLS termination
          using dev-example-com-tls
```

### Step 4 - Route

```text
Host:
vllm.dev.example.com

      |
      v

HTTPRoute/vllm
```

### Step 5 - Service

```text
HTTPRoute
   |
   v
Service/vllm:8000
   |
   v
vLLM Pods
```

---

# 29. Final Production Architecture

```text
                             INTERNET
                                |
                                | HTTPS
                                v
                     vllm.dev.example.com
                                |
                                | DNS
                                v
              +--------------------------------+
              | DigitalOcean Authoritative DNS |
              |                                |
              | *.dev.example.com              |
              |            |                   |
              +------------+-------------------+
                           |
                           v
              +-------------------------------+
              | DigitalOcean Load Balancer    |
              | Public IP                     |
              +---------------+---------------+
                              |
                            :443
                              |
                              v
                  +-----------------------+
                  | Envoy Gateway         |
                  | Gateway API           |
                  |                       |
                  | TLS Termination       |
                  +-----------+-----------+
                              |
                              v
                    +-------------------+
                    | HTTPRoute         |
                    +---------+---------+
                              |
                              v
                    +-------------------+
                    | Kubernetes Service|
                    +---------+---------+
                              |
                              v
                           Workload


Certificate automation:

                  +-----------------------+
                  | cert-manager          |
                  +-----------+-----------+
                              |
                              | ACME
                              v
                  +-----------------------+
                  | Let's Encrypt         |
                  +-----------+-----------+
                              |
                              | DNS-01
                              v
                  +-----------------------+
                  | DigitalOcean DNS API  |
                  +-----------+-----------+
                              |
                              v
          _acme-challenge.dev.example.com TXT
                              |
                              v
                    Domain validated
                              |
                              v
                    Certificate issued
                              |
                              v
                    Kubernetes TLS Secret
                              |
                              v
                       Envoy Gateway
```

---

# 30. References

Official documentation:

- DigitalOcean DNS Quickstart  
  https://docs.digitalocean.com/products/networking/dns/getting-started/quickstart/

- DigitalOcean DNS registrar delegation  
  https://docs.digitalocean.com/products/networking/dns/getting-started/dns-registrars/

- cert-manager installation with Helm  
  https://cert-manager.io/docs/installation/helm/

- cert-manager ACME configuration  
  https://cert-manager.io/docs/configuration/acme/

- cert-manager DNS-01 configuration  
  https://cert-manager.io/docs/configuration/acme/dns01/

- Let's Encrypt challenge types  
  https://letsencrypt.org/docs/challenge-types/

---

# Summary

The complete platform certificate flow is:

```text
1. Own domain
       |
2. Add apex domain to DigitalOcean DNS
       |
3. Delegate registrar nameservers to DigitalOcean
       |
4. Verify authoritative DNS
       |
5. Provision wildcard DNS -> Load Balancer
       |
6. Install cert-manager
       |
7. Create dedicated DigitalOcean DNS API token
       |
8. Store token as Kubernetes Secret
       |
9. Configure Let's Encrypt staging ClusterIssuer
       |
10. Validate DNS-01 automation
       |
11. Configure Let's Encrypt production ClusterIssuer
       |
12. Request *.dev.example.com wildcard certificate
       |
13. cert-manager stores certificate in TLS Secret
       |
14. Gateway API references TLS Secret
       |
15. HTTPRoutes expose applications
       |
16. cert-manager automatically renews certificates
```

The only long-term manual domain operation should normally be the registrar-side nameserver delegation.

After that:

```text
OpenTofu
    -> DigitalOcean infrastructure and DNS

cert-manager
    -> TLS issuance and renewal

Let's Encrypt
    -> publicly trusted CA

Gateway API
    -> HTTPS termination and application routing

Argo CD
    -> future GitOps reconciliation
```

This gives the platform fully automated, publicly trusted HTTPS without buying certificates or manually renewing them.
