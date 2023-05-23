# This project describes how to deploy an empty nginx web server (in a local loop) using SSL self-signed certificates in k3s (k8s)
## Works for local clusters (with internet access)

1. First, install cert-manager using helm chart

Installation Instructions:
```bash
https://cert-manager.io/docs/installation/helm/
```

2. Check that `cert-manager` is installed:

```bash
kubectl get all --namespace cert-manager
```

Check that all the pods have started and are working correctly

3. Next, create a `certificate authority` (CA) on the wizard.

Create a certificate authority private key

```bash
openssl genrsa -out ca.key 4096
```

4. Generate a CA certificate

```bash
openssl req -new -x509 -sha256 -days 365 -key ca.key -out ca.crt
```
Convert the contents of the generated files (`ca.key` and `ca.crt`) to `base64` to transfer the values in the manifest from kind secret to manifest (nginx1-ca-secret.yml)

```bash
cat ca.crt | base64 -w 0
cat ca.key | base64 -w 0
```


## Now let's create manifest files to run nginx with TLS


1. Create the file `nginx1-ca-secret.yml` and paste the converted text into


```bash
data:
tls.crt: # content of ca.crt (converted to base64)
tls.key: # content of ca.key (converted to base64)
```

#### !! Make sure that secret is in the same namespace as cent-manager

2. Create a `cluster issuer` object using `test-nginx-clusterissuer.yml`

3. Add namespace `test-nginx`

4. Create a new certificate `test-nginx-cert.yml` for your projects
5. Add the `tls` section to the 'ingress' object `test-nginx-ingress.yml`
6. Execute the command `kubectl apply -f` for all created files

!!! Before performing the following steps , look at the logs of created objects , make sure that all objects work without errors

7. Apply the manifest files `deployment`, `service`, and `Middleware' (for redirecting http to https)

####  !! Be careful when copying the key, it sometimes does not highlight the '=' character at the end

In order to be able to log in from all nodes via https and use the certificate accordingly, it is necessary (`CentOS7')

8. Copy the certificate file (usually with the extension `.crt` or `.pem`) to the server

An example in the CentOS 7 operating system, for example, with the command

``bash
sudo cp /etc/pki/tls/certs/your_file.crt /etc/pki/ca-trust/source/anchors/
``

If there is no file on the path `(etc/pki/tls/certs/your_file.crt)` then add it

9. Update the certificate store by running the following command:

```bash
sudo update-ca-trust
```

10. Restart the OpenSSL services and the browser to apply the changes. For example, to restart the OpenSSL service, run:

```bash
sudo systemctl restart httpd or restart the deployment in the k3s cluster
``

11. Check that everything is working by running `curl https://mynginx.local.home `

If you do everything locally, do not forget to update the record on the local `DNS` or `/etc/hosts`

## Architecture Diagram

```
                Cert-Manager Objects                        Nginx1 Objects

               ┌───────────────────────────┐                    ┌─────────────────────────────────────┐
Created CA     │ kind: Secret              │                    │                                     │
private key ──►│ name: test-nginx-ca-secret│◄───────────┐       │ kind: Ingress                       │
and cert       │ tls.key: **priv key**     │         	│       │ name: test-nginx-ingress            │
               │ tls.crt: **cert**         │         	│       │ tls:                                │
               └───────────────────────────┘        	│       │   - hosts:                          │
                                                        │       │     - mynginx.local.home            │
               ┌──────────────────────────────────┐     │  ┌────┼───secretName: test-nginx-tls-secret │
               │                                  │     │  │    │                                     │
               │ kind: ClusterIssuer              │     │  │    └─────────────────────────────────────┘
           ┌───┤►name: test-nginx-clusterissuer   │     │  │
           │   │ secretName: test-nginx-ca-secret─┼─────┘  │
           │   │                                  │        │
           │   └──────────────────────────────────┘        │
           │                                          	   │
           │   ┌───────────────────────────────────┐       │
           │   │                                   │   	   │
           │   │ kind: Certificate                 │       │
           │   │ name: test-nginx-cert             │       │
           └───┼─issuerRef:                        │       │
               │   name: test-nginx-clusterissuer  │       │
               │   kind: ClusterIssuer             │       │
               │ dnsNames:                         │       │
               │   - mynginx.local.home            │       │
           ┌───┼─secretName: test-nginx-tls-secret │       │
           │   │                                   │       │
           │   └──────────┬────────────────────────┘       │
           │              │                                │
           │              │ will be created                │
           │              ▼ and managed automatically      │
           │   ┌───────────────────────────────┐           │
           │   │                               │           │
           │   │ kind: Secret                  │           │
           └───┤►name: test-nginx-tls-secret◄──┼───────────┘
               │                               │
               └───────────────────────────────┘
```

