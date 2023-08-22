# vault-pki-cert-manager-part-2

The purpose of this tutorial is to build on what we had accomplished in the last exercise.  Here we will generate a TLS certificate and apply this to the ngnix sample hello-world application


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
Now we need to get the ingress IP address to update our hosts file

```shell-session
$ kubectl get ingress
```

Output

```shell-session
NAME         CLASS   HOSTS              ADDRESS          PORTS     AGE
nginx-demo   nginx   demo.example.com   192.168.59.100   80, 443   15m
```

Update our hosts file to point example.com to the address in the output of the command above

```shell-session
$ sudo nano /etc/hosts
```

At the bottom of the file add

```shell-session
$ 192.168.59.100 demo.example.com
```

Now use curl to inspect the certificate.  Note the CN=Kubernetes Ingress Controller Fake Certificate.  This is a default certificate, we will replace this in the following steps.

```shell-session
$ curl --insecure -vvI https://demo.example.com 2>&1 | awk 'BEGIN { cert=0 } /^\* SSL connection/ { cert=1 } /^\*/ { if (cert) print }'
```


```shell-session
$ cat > demo-example-com-tls.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: demo-example-com
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

```shell-session
$ kubectl apply -f demo-example-com-tls.yaml
```


Now we will apply the Ingress patch to use TLS and the HashiCorp Vault Generated Certificate

```shell-session
$ cat > ingress-patch.yaml <<EOF
spec:
  tls:
  - hosts:
    - demo.example.com
    secretName: demo-example-com-tls
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
