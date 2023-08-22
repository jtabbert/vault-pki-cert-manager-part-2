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


```shell-session
$ cat > ingress.yaml <<EOF 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-demo
  annotations:
    cert-manager.io/issuer: "vault-issuer"
spec:
  tls:
  - hosts:
    - demo.example.com
    secretName: nginx-demo
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

```shell-session
$ kubectl apply -f ingress.yaml
```

## Next steps

In this tutorial, you installed Vault configured the PKI secrets engine and
Kubernetes authentication. Then installed Jetstack's cert-manager, configured it
to use Vault, and requested a certificate.

Besides creation, these certificates can be revoked and removed. Learn more about
[Jetstack's cert-manager](https://cert-manager.io/) used in this tutorial and
explore Vault's PKI secrets engine as a certificate authority in the [Build Your
Own Certificate Authority](/vault/tutorials/secrets-management/pki-engine).
