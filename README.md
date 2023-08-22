# vault-pki-cert-manager-part-2

---
id: fe612e45-ed64-4df9-a163-8364272386c0
name: Configure Vault as a certificate manager in Kubernetes with Helm
short_name: Configure Vault as CM in Kubernetes with Helm
description: Configure Vault as a certificate manager in Kubernetes with Helm.
products_used:
  - vault
default_collection_context: vault/kubernetes
---

Kubernetes configured to use Vault as a certificate manager enables your
services to establish their identity and communicate securely over the network
with other services or clients internal or external to the cluster.

Jetstack's [cert-manager](https://cert-manager.io/) enables Vault's [PKI secrets
engine](/vault/docs/secrets/pki) to dynamically generate
X.509 certificates within Kubernetes through an Issuer interface.

In this tutorial, you set up Vault with the Vault Helm chart, configure the PKI
secrets engine and Kubernetes authentication. Then install Jetstack's
cert-manager, configure it to use Vault, and request a certificate.

## Prerequisites

This tutorial requires the [Kubernetes command-line interface
(CLI)](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and the [Helm
CLI](https://helm.sh/docs/helm/) installed,
[Minikube](https://minikube.sigs.k8s.io), the Vault Helm charts, the sample web
application, and additional configuration to bring it all together.


This tutorial was last tested 29 Apr 2020 on a macOS 10.15.4 using this
configuration.

Docker version.

<CodeBlockConfig highlight="3">


```shell-session
$ docker version
Client: Docker Engine - Community
  Version:          19.03.8
  ## ...
```

</CodeBlockConfig>


Minikube version.

<CodeBlockConfig highlight="2">


```shell-session
$ minikube version
minikube version: v1.8.2
commit: eb13446e786c9ef70cb0a9f85a633194e62396a1
```

</CodeBlockConfig>


Helm version.

<CodeBlockConfig highlight="2">


```shell-session
$ helm version
version.BuildInfo{Version:"v3.1.2", GitCommit:"d878d4d45863e42fd5cff6743294a11d28a9abce", GitTreeState:"clean", GoVersion:"go1.14"}
```

</CodeBlockConfig>


These are recommended software versions and the output displayed may vary
depending on your environment and the software versions you use.

First, follow the directions for [installing
Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/), including
VirtualBox or similar.

Next, install [kubectl CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
and [helm CLI](https://github.com/helm/helm#install).

First we will enable ingress on Minikube

```shell-session
$ minikube addons enable ingress
```

We will create a deplyoment called ngnix-demo

```shell-session
$ kubectl create deployment nginx-demo --image=nginxdemos/hello
```

Verify the deployment is ready

```shell-session
$ kubectl get deployment
```

```shell-session
$ kubectl expose deployment nginx-demo --port=80
```




Next, retrieve the web application and additional configuration by cloning the
[hashicorp/vault-guides](https://github.com/hashicorp/vault-guides) repository
from GitHub.

```shell-session
$ git clone https://github.com/hashicorp/vault-guides.git
```

This repository contains supporting content for all of the Vault learn guides.
The content specific to this tutorial can be found within a sub-directory.

Go into the
`vault-guides/operations/provision-vault/kubernetes/minikube/external-vault`
directory.

```shell-session
$ cd vault-guides/operations/provision-vault/kubernetes/minikube/external-vault
```

<Note title="Working directory">

 This tutorial assumes that the remainder of commands are
executed within this directory.

</Note>

## Start Minikube

[Minikube](https://minikube.sigs.k8s.io/) is a CLI tool that provisions and
manages the lifecycle of single-node [Kubernetes
clusters](https://kubernetes.io/docs/concepts/#kubernetes-control-plane). These
clusters are run locally inside Virtual Machines (VM).

Start a Kubernetes cluster.

```shell-session
$ minikube start
😄  minikube v1.8.2 on Darwin 10.15.4
✨  Automatically selected the hyperkit driver
🔥  Creating hyperkit VM (CPUs=2, Memory=4000MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.17.3 on Docker 19.03.6 ...
🚀  Launching Kubernetes ...
🌟  Enabling addons: default-storageclass, storage-provisioner
⌛  Waiting for cluster to come online ...
🏄  Done! kubectl is now configured to use "minikube"
```

Verify the status of the Minikube cluster.

```shell-session
$ minikube status
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

<Note title="Additional waiting">

 Even if the previous step completed successfully, you
may have to wait for Minikube to be available. If you see an error, try again
after a few minutes.

</Note>

The host, kubelet, and apiserver report that they are running. The `kubectl`, a
command line interface (CLI) for running commands against Kubernetes cluster, is
also configured to communicate with this recently started cluster.

## Install the Vault Helm chart

Add the HashiCorp Helm repository.

```shell-session
$ helm repo add hashicorp https://helm.releases.hashicorp.com
"hashicorp" has been added to your repositories
```

Update all the repositories to ensure `helm` is aware of the latest versions.

```shell-session
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "hashicorp" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Install the latest version of the Vault server running in standalone mode with
the Vault Agent Injector service disabled.

```shell-session
$ helm install vault hashicorp/vault --set "injector.enabled=false"
NAME: vault
## ...
```

The Vault server runs in standalone mode on a single pod. By default the Helm
chart starts a Vault Agent Injector pod but that is disabled
`injector.enabled=false`.

Get all the pods within the default namespace.

```shell-session
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
vault-0   0/1     Running   0          87s
```

The `vault-0` pod is deployed. The Vault server running in the pod's container
reports that it is running but it is not ready (`0/1`). To ready the pod
requires that the Vault server is initialized and unsealed.

Get all the services within the default namespace.

```shell-session
$ kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP             9m5s
vault        ClusterIP   10.105.21.154   <none>        8200/TCP,8201/TCP   2m34s
```

The Vault Helm chart creates a Service that directs requests to the Vault pod.
This enables us to address the Vault server within the cluster with the address
`http://vault.default:8200`.

## Initialize and unseal Vault

Vault run in standalone mode starts
[uninitialized](/vault/docs/commands/operator/init)
and in the [sealed](/vault/docs/concepts/seal#why) state.
Prior to initialization the storage backend is not prepared to receive data.

Initialize Vault with one key share and one key threshold.

```shell-session
$ kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 \
      -format=json > init-keys.json
```

The [`operator init`](/vault/docs/commands/operator/init) command
generates a root key that it disassembles into key shares `-key-shares=1` and
then sets the number of key shares required to unseal Vault `-key-threshold=1`.
These key shares are written to the output as unseal keys in JSON format
`-format=json`. Here the output is redirected to a local file named
`init-keys.json`

View the unseal key found in `init-keys.json`.

```shell-session
$ cat init-keys.json | jq -r ".unseal_keys_b64[]"
hmeMLoRiX/trBTx/xPZHjCcZ7c4H8OCt2Njkrv2yXZY=
```

<Warning title="Insecure operation">

 Do not run an unsealed Vault in production with a
single key share and a single key threshold. This approach is only used here to
simplify the unsealing process for this demonstration.

</Warning>

Create a variable named `VAULT_UNSEAL_KEY` to capture the Vault unseal key.

```shell-session
$ VAULT_UNSEAL_KEY=$(cat init-keys.json | jq -r ".unseal_keys_b64[]")
```

After initialization, Vault is configured to know where and how to access the
storage, but does not know how to decrypt any of it.
[Unsealing](/vault/docs/concepts/seal#unsealing) is
the process of constructing the root key necessary to read the decryption key
to decrypt the data, allowing access to the Vault.

Unseal Vault running on the `vault-0` pod with the `$VAULT_UNSEAL_KEY`.

```shell-session
$ kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.4.0
Cluster Name    vault-cluster-3ecefcf2
Cluster ID      0998f747-af15-f994-cded-295836b718d6
HA Enabled      false
```

The `operator unseal` command reports that Vault is initialized and unsealed.

<Warning title="Insecure operation">

 Providing the unseal key with the command writes the
key to your shell's history. This approach is only used here to simplify the
unsealing process for this demonstration.

</Warning>

Get all the pods within the default namespace.

```shell-session
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          5m49s
```

The `vault-0` pod reports that it is ready `1/1`. Vault is ready for you to
login with the root token generated during the initialization.

View the root token found in `init-keys.json`.

```shell-session
$ cat init-keys.json | jq -r ".root_token"
s.XzExf8TjRVYKm85xMATa6Q7U
```

Create a variable named `VAULT_ROOT_TOKEN` to capture the root token.

```shell-session
$ VAULT_ROOT_TOKEN=$(cat init-keys.json | jq -r ".root_token")
```

Login to Vault running on the `vault-0` pod with the `$VAULT_ROOT_TOKEN`.

```shell-session
$ kubectl exec vault-0 -- vault login $VAULT_ROOT_TOKEN

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.P3Koh6BZikQPDxPSNwDzmKJ5
token_accessor       kHFYypyS2EcYpMyrsyXUQmNa
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

The Vault server is ready to be configured as a certificate store.

## Configure PKI secrets engine

First, start an interactive shell session on the `vault-0` pod.

```shell-session
$ kubectl exec --stdin=true --tty=true vault-0 -- /bin/sh
/ $
```

Your system prompt is replaced with a new prompt `/ $`. Commands issued at this
prompt are executed on the `vault-0` container.

Enable the PKI secrets engine at its default path.

```shell-session
$ vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/
```

By default the KPI secrets engine sets the time-to-live (TTL) to 30 days. A
certificate can have its lease extended to ensure certificate rotation on a
yearly basis (8760h).

Configure the max lease time-to-live (TTL) to `8760h`.

```shell-session
$ vault secrets tune -max-lease-ttl=8760h pki
Success! Tuned the secrets engine at: pki/
```

Vault can accept an existing key pair, or it can generate its own self-signed
root. In general, we recommend maintaining your root CA outside of Vault and
providing Vault a signed intermediate CA.

Generate a self-signed certificate valid for `8760h`.

```shell-session
$ vault write pki/root/generate/internal \
    common_name=example.com \
    ttl=8760h
```

**Example output:** 

<CodeBlockConfig hideClipboard>

```plaintext
Key              Value
---              -----
certificate      -----BEGIN CERTIFICATE-----
## ...
-----END CERTIFICATE-----
expiration       1619120269
issuing_ca       -----BEGIN CERTIFICATE-----
## ...
-----END CERTIFICATE-----
serial_number    65:37:b5:b3:91:6c:7b:d8:33:22:03:28:b1:58:ff:be:8a:72:a4:c0
```

</CodeBlockConfig>

Configure the PKI secrets engine certificate issuing and certificate revocation
list (CRL) endpoints to use the Vault service in the default namespace.

```shell-session
$ vault write pki/config/urls \
    issuing_certificates="http://vault.default:8200/v1/pki/ca" \
    crl_distribution_points="http://vault.default:8200/v1/pki/crl"
```

**Output:** 

<CodeBlockConfig hideClipboard>

```plaintext
Key                        Value
---                        -----
crl_distribution_points    [http://vault.default:8200/v1/pki/crl]
enable_templating          false
issuing_certificates       [http://vault.default:8200/v1/pki/ca]
ocsp_servers               []
```

</CodeBlockConfig>

Configure a role named `example-dot-com` that enables the creation of
certificates `example.com` domain with any subdomains.

```shell-session
$ vault write pki/roles/example-dot-com \
    allowed_domains=example.com \
    allow_subdomains=true \
    max_ttl=72h
```

**Output:** 

<CodeBlockConfig hideClipboard>

```plaintext
Key                                   Value
---                                   -----
allow_any_name                        false
allow_bare_domains                    false
allow_glob_domains                    false
allow_ip_sans                         true
allow_localhost                       true
allow_subdomains                      true
allow_token_displayname               false
allow_wildcard_certificates           true
allowed_domains                       [example.com]
allowed_domains_template              false
allowed_other_sans                    []
allowed_serial_numbers                []
allowed_uri_sans                      []
allowed_uri_sans_template             false
allowed_user_ids                      []
basic_constraints_valid_for_non_ca    false
client_flag                           true
cn_validations                        [email hostname]
code_signing_flag                     false
country                               []
email_protection_flag                 false
enforce_hostnames                     true
ext_key_usage                         []
ext_key_usage_oids                    []
generate_lease                        false
issuer_ref                            default
key_bits                              2048
key_type                              rsa
key_usage                             [DigitalSignature KeyAgreement KeyEncipherment]
locality                              []
max_ttl                               72h
no_store                              false
not_after                             n/a
not_before_duration                   30s
organization                          []
ou                                    []
policy_identifiers                    []
postal_code                           []
province                              []
require_cn                            true
server_flag                           true
signature_bits                        256
street_address                        []
ttl                                   0s
use_csr_common_name                   true
use_csr_sans                          true
use_pss                               false
```

</CodeBlockConfig>

The role, `example-dot-com`, is a logical name that maps to a policy used to
generate credentials. This generates a number of endpoints that are used by the
Kubernetes service account to issue and sign these certificates. A policy must
be created that enables these paths.

Create a policy named `pki` that enables read access to the PKI secrets engine
paths.

```shell-session
$ vault policy write pki - <<EOF
path "pki*"                        { capabilities = ["read", "list"] }
path "pki/sign/example-dot-com"    { capabilities = ["create", "update"] }
path "pki/issue/example-dot-com"   { capabilities = ["create"] }
EOF
```

**Output:** 

<CodeBlockConfig hideClipboard>

```plaintext
Success! Uploaded policy: pki
```

</CodeBlockConfig>

These paths enable the token to view all the roles created for this PKI secrets
engine and access the `sign` and `issues` operations for the `example-dot-com`
role.

Lastly, exit the `vault-0` pod.

```shell-session
$ exit
```

## Configure Kubernetes authentication

Vault provides a [Kubernetes
authentication](/vault/docs/auth/kubernetes) method
that enables clients to authenticate with a Kubernetes Service Account
Token.

First, start an interactive shell session on the `vault-0` pod.

```shell-session
$ kubectl exec --stdin=true --tty=true vault-0 -- /bin/sh
/ $
```

Your system prompt is replaced with a new prompt `/ $`. Commands issued at this
prompt are executed on the `vault-0` container.

Enable the Kubernetes authentication method.

```shell-session
$ vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
```

Configure the Kubernetes authentication method to use location of the Kubernetes
 API.

~> For the best compatibility with recent Kubernetes versions, ensure you
  are using Vault v1.9.3 or greater.

```shell-session
$ vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```

**Output:** 

<CodeBlockConfig hideClipboard>

```plaintext
Success! Data written to: auth/kubernetes/config
```

</CodeBlockConfig>

The environment variable `KUBERNETES_PORT_443_TCP_ADDR`
references the internal network address of the Kubernetes host.

~> You can validate the issuer name of your Kubernetes cluster using
[this method](/vault/docs/auth/kubernetes#discovering-the-service-account-issuer).

Finally, create a Kubernetes authentication role named `issuer` that binds
the `pki` policy with a Kubernetes service account named `issuer`.

```shell-session
$ vault write auth/kubernetes/role/issuer \
    bound_service_account_names=issuer \
    bound_service_account_namespaces=default \
    policies=pki \
    ttl=20m
```

**Output:** 

<CodeBlockConfig hideClipboard>

```plaintext
Success! Data written to: auth/kubernetes/role/issuer
```

</CodeBlockConfig>

The role connects the Kubernetes service account, `issuer`, in the `default`
namespace with the `pki` Vault policy. The tokens returned after authentication
are valid for 20 minutes. This Kubernetes service account name, `issuer`, is
created in the [Deploy Issuer and Certificate](#configure-an-issuer-and-generate-a-certificate)
section.

Lastly, exit the `vault-0` pod.

```shell-session
$ exit
```

## Deploy Cert Manager

Jetstack's cert-manager is a Kubernetes add-on that automates the management and
issuance of TLS certificates from various issuing sources. Vault can be
configured as one of those sources. The cert-manager requires the creation of
a set of Kubernetes resources that provide the interface to the certificate
creation.

Install Jetstack's cert-manager's version 1.12.3 resources.

```shell-session
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.12.3/cert-manager.crds.yaml
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
```

Create a namespace named `cert-manager` to host the cert-manager.

```shell-session
$ kubectl create namespace cert-manager
namespace/cert-manager created
```

Jetstack's cert-manager Helm chart is available in a repository that they
maintain. Helm can request and install Helm charts from these custom
repositories.

Add the `jetstack` chart repository.

```shell-session
$ helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories
```

Helm maintains a cached list of charts for every repository that it maintains.
This list needs to be updated periodically so that Helm knows about all
available charts and their releases. A repository recently added needs to be
updated before any chart is requested.

Update the local list of Helm charts.

```shell-session
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jetstack" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

The results show that the `jetstack` chart repository has retrieved an update.

Install the cert-manager chart version 0.11 in the `cert-manager` namespace.

```shell-session
$ helm install cert-manager \
    --namespace cert-manager \
    --version v1.12.3 \
   jetstack/cert-manager
```

**Output:** 

<CodeBlockConfig hideClipboard>

```plaintext
NAME: cert-manager
## ...
```

</CodeBlockConfig>

The cert-manager chart deploys a number of pods within the `cert-manager`
namespace.

Get all the pods within the `cert-manager` namespace.

```shell-session
$ kubectl get pods --namespace cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-66958f45fc-pdf64              1/1     Running   0          27s
cert-manager-cainjector-755bbf9c6b-gpgtg   1/1     Running   0          27s
cert-manager-webhook-76954fcbcd-w4lll      1/1     Running   0          27s
```

Wait until the pods prefixed with `cert-manager` are running and ready (`1/1`).

Thes pods now require configuration to interface with Vault.

## Configure an issuer and generate a certificate

The cert-manager enables you to define Issuers that interface with the Vault
certificate generating endpoints. These Issuers are invoked when a Certificate
is created.

When you [configured Vault's Kubernetes
authentication](#configure-kubernetes-authentication) a Kubernetes service
account, named `issuer`, was granted the policy, named `pki`, to the certificate
generation endpoints.

Create a service account named `issuer` within the default namespace.

```shell-session
$ kubectl create serviceaccount issuer
serviceaccount/issuer created
```

The service account generated a secret that is required by the Issuer automatically
in Kubernetes 1.23.
In Kubernetes 1.24+, you need to create the secret explicitly.

<CodeBlockConfig filename="issuer-secret.yaml">

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: issuer-token-lmzpj
  annotations:
    kubernetes.io/service-account.name: issuer
type: kubernetes.io/service-account-token
```

</CodeBlockConfig>

#### Kubernetes 1.24+ only

```shell-session
$ kubectl apply -f issuer-secret.yaml
secret/issuer-token-lmzpj created
```

Get all the secrets in the default namespace.

```shell-session
$ kubectl get secrets
default-token-mlm2n           kubernetes.io/service-account-token   3      13d
issuer-token-lmzpj            kubernetes.io/service-account-token   3      47s
sh.helm.release.v1.vault.v1   helm.sh/release.v1                    1      28m
vault-token-749nd             kubernetes.io/service-account-token   3      28m
```

The issuer secret is displayed here as the secret prefixed with `issuer-token`.

Create a variable named `ISSUER_SECRET_REF` to capture the secret name.

```shell-session
$ ISSUER_SECRET_REF=$(kubectl get secrets --output=json | jq -r '.items[].metadata | select(.name|startswith("issuer-token-")).name')
```

Define an Issuer, named `vault-issuer`, that sets Vault as a certificate
issuer.

```shell-session
$ cat > vault-issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: vault-issuer
  namespace: default
spec:
  vault:
    server: http://vault.default:8200
    path: pki/sign/example-dot-com
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: issuer
        secretRef:
          name: $ISSUER_SECRET_REF
          key: token
EOF
```

Create the `vault-issuer` Issuer.

```shell-session
$ kubectl apply --filename vault-issuer.yaml
issuer.cert-manager.io/vault-issuer created
```

The specification defines the signing endpoint and the authentication endpoint
and credentials.

- `metadata.name` sets the name of the Issuer to `vault-issuer`
- `spec.vault.server` sets the server address to the Kubernetes service created
  in the default namespace
- `spec.vault.path` is the signing endpoint created by Vault's PKI
  `example-dot-com` role
- `spec.vault.auth.kubernetes.mountPath` sets the Vault authentication endpoint
- `spec.vault.auth.kubernetes.role` sets the Vault Kubernetes role to `issuer`
- `spec.vault.auth.kubernetes/secretRef.name` sets the secret for the Kubernetes
  service account
- `spec.vault.auth.kubernetes/secretRef.key` sets the type to `token`.

Define a certificate named `example-com`.

```shell-session
$ cat > example-com-cert.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: vault-issuer
  commonName: www.example.com
  dnsNames:
  - www.example.com
EOF
```

The Certificate, named `example-com`, requests from Vault the certificate
through the Issuer, named `vault-issuer`. The common name and DNS names are
names within the allowed domains for the configured Vault endpoint.

Create the `example-com` certificate.

```shell-session
$ kubectl apply --filename example-com-cert.yaml
certificate.cert-manager.io/example-com created
```

View the details of the `example-com` certificate.

```shell-session
$ kubectl describe certificate.cert-manager example-com
Name:         example-com
Namespace:    default
## ...
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    10s   cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  10s   cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "example-com-bpfwz"
  Normal  Requested  9s    cert-manager-certificates-request-manager  Created new CertificateRequest resource "example-com-wfclm"
  Normal  Issuing    9s    cert-manager-certificates-issuing          The certificate has been successfully issued
```

The certificate reports that it has been issued successfully.

## Next steps

In this tutorial, you installed Vault configured the PKI secrets engine and
Kubernetes authentication. Then installed Jetstack's cert-manager, configured it
to use Vault, and requested a certificate.

Besides creation, these certificates can be revoked and removed. Learn more about
[Jetstack's cert-manager](https://cert-manager.io/) used in this tutorial and
explore Vault's PKI secrets engine as a certificate authority in the [Build Your
Own Certificate Authority](/vault/tutorials/secrets-management/pki-engine).
