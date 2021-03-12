# KubeDB: PostgreSQL with TLS


## Enable Cert-Manager Managed TLS

- We will need a cert-manager Issuer object. For this reason, First we are going to create a pair of `ca.crt` and `ca.key`.

```
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./ca.key -out ./ca.crt -subj "/CN=postgres/O=kubedb"
// Create demo namespace if not present
$ kubectl create ns demo

// Add cert-manager if not present
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.yaml

$ kubectl create secret tls postgres-ca  --cert=ca.crt --key=ca.key --namespace=demo

$ kubectl apply -f issuer.yaml

$ kubectl get issuer -n demo
NAME                 READY   AGE
postgres-ca-issuer   True    1m

```
- the certificates will be created by default specification.
- Apply the yaml:

```
$ kubectl apply -f demo-pg-with-tls.yaml
```

```
// Check the configuration 
$ kubectl exec -it -n demo demo-pg-0 -- cat var/pv/data/postgresql.conf

```


- Now, the server is ready to accept tls enabled connection.