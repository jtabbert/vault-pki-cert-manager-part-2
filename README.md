# vault-pki-cert-manager-part-2

The purpose of this tutorial is to build on what we had accomplished in the last exercise.  Here we will generate a TLS certificate and apply this to the ngnix sample hello-world application

These are the software versions used in this tutorial

```shell-session
Minikube Version: v1.31.2
Ubuntu Version 22.04 LTS
Helm Version: v3.12.3
```

First we will enable ingress on Minikube

```shell-session
minikube addons enable ingress
```

We will create a deplyoment called ngnix-demo

```shell-session
kubectl create deployment nginx-demo --image=nginxdemos/hello
```

Verify the deployment is ready

```shell-session
kubectl get deployment
```
Expose the deployment

```shell-session
kubectl expose deployment nginx-demo --port=80
```

Apply the Ingress to allow traffic into our application

```shell-session
cat > ingress.yaml <<EOF 
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
kubectl apply -f ingress.yaml
```
Now we need to get the ingress IP address to update our hosts file

```shell-session
kubectl get ingress
```

Output

```shell-session
NAME         CLASS   HOSTS              ADDRESS          PORTS     AGE
nginx-demo   nginx   demo.example.com   192.168.59.100   80, 443   15m
```

Update our hosts file to point example.com to the address in the output of the command above

```shell-session
sudo nano /etc/hosts
```

At the bottom of the file add, replacing the IP address below with the output of the command "kubectl get ingress" 

```shell-session
192.168.59.100 demo.example.com
```

Now use curl to inspect the certificate.  Note the CN=Kubernetes Ingress Controller Fake Certificate.  This is a default certificate, we will replace this in the following steps.

```shell-session
curl --insecure -vvI https://demo.example.com 2>&1 | awk 'BEGIN { cert=0 } /^\* SSL connection/ { cert=1 } /^\*/ { if (cert) print }'
```

The Output should look somthing like this

```shell-session
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate
*  start date: Aug 22 15:37:42 2023 GMT
*  expire date: Aug 21 15:37:42 2024 GMT
*  issuer: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate

```

Now we will generate a certificate for "demo.example.com" and store this as a K8s secret called "demo-example-com-tls"

```shell-session
cat > demo-example-com-tls.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: demo-example-com-tls
  namespace: default
spec:
  secretName: demo-example-com-tls
  issuerRef:
    name: vault-issuer
  commonName: demo.example.com
  dnsNames:
  - demo.example.com
EOF
```

Apply the file to make the changes

```shell-session
kubectl apply -f demo-example-com-tls.yaml
```

Now we will apply the Ingress patch to use TLS and the HashiCorp Vault Generated Certificate

```shell-session
cat > ingress-patch.yaml <<EOF
spec:
  tls:
  - hosts:
    - demo.example.com
    secretName: demo-example-com-tls
EOF
```

We will apply the patch to the nginx-demo ingress controller

```shell-session
kubectl patch ingress nginx-demo --patch-file=ingress-patch.yaml
```

We can run the curl command again to see CN=example.com

```shell-session
curl --insecure -vvI https://demo.example.com 2>&1 | awk 'BEGIN { cert=0 } /^\* SSL connection/ { cert=1 } /^\*/ { if (cert) print }'
```

Output

```shell-session
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=demo.example.com
*  start date: Aug 22 19:28:47 2023 GMT
*  expire date: Aug 25 19:29:17 2023 GMT
*  issuer: CN=example.com
```

We can see now we are now using the Vault issued certificate.  We can also view this via a web browser.

## Next steps

In this tutorial, we built on our previous work. We leveraged Jetstack's cert-manager to automatically
Request a certificate from Vault. This provided us with a real world example of how to use Jetstack's cert-manager
along with HashiCorp Vault to automate the certificate lifecycle.

Besides creation, these certificates can be revoked and removed. Learn more about
[Jetstack's cert-manager](https://cert-manager.io/) used in this tutorial and
explore Vault's PKI secrets engine as a certificate authority in the [Build Your
Own Certificate Authority](/vault/tutorials/secrets-management/pki-engine).
