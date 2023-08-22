# vault-pki-cert-manager-part-2


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
Expose the deployment

```shell-session
$ kubectl expose deployment nginx-demo --port=80
```

Apply the Ingress to allow traffic into our application

```shell-session
$ cat > ingress.yaml <<EOF 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-demo
spec:
  rules:
    - host: demo.example.com
      http:
        paths:
          - pathType: Prefix
            path: "/"
            backend:
             service:
               name: nginx-demo
               port:
                 number: 80
EOF 
```

Apply the ingress

```shell-session
$ kubectl apply -f ingress.yaml
```

Use curl to inspect the certificate.  Note the CN=Kubernetes Ingress Controller Fake Certificate

```shell-session
$ curl --insecure -vvI https://demo.example.com 2>&1 | awk 'BEGIN { cert=0 } /^\* SSL connection/ { cert=1 } /^\*/ { if (cert) print }'
```

Now we will apply the Ingress patch to use HashiCorp Vault & the "cert-manager.io/issuer" 

```shell-session
$ cat > ingress-patch.yaml <<EOF
metadata:
  annotations:
    cert-manager.io/issuer: "vault-issuer"
spec:
  tls:
  - hosts:
    - demo.example.com
    secretName: nginx-demo
EOF
```

We will apply the patch to the nginx-demo ingress controller

```shell-session
$ kubectl patch ingress nginx-demo --patch-file=ingress-patch.yaml
```

We can run the curl command again to see CN=example.com

```shell-session
$ curl --insecure -vvI https://demo.example.com 2>&1 | awk 'BEGIN { cert=0 } /^\* SSL connection/ { cert=1 } /^\*/ { if (cert) print }'
```

## Next steps

In this tutorial, we built on our previous work. We leveraged Jetstack's cert-manager to automatically
Request a certificate from Vault. This provided us with a real world example of how to use Jetstack's cert-manager
along with HashiCorp Vault to automate the certificate lifecycle.

Besides creation, these certificates can be revoked and removed. Learn more about
[Jetstack's cert-manager](https://cert-manager.io/) used in this tutorial and
explore Vault's PKI secrets engine as a certificate authority in the [Build Your
Own Certificate Authority](/vault/tutorials/secrets-management/pki-engine).
