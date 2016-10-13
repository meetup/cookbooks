# SSL Sidekick Proxy

There are many approaches for adding SSL to your application, here we'll talk about the sidekick approach where we create a lightweight proxy that handles your SSL termination.

What we'll be creating is:

* A secret to house our SSL cert and key
* A config housing our configuration file for Nginx.
* A pod containing:
  * An Nginx container configured to:
    * Redirect traffic on 80 to 443
    * Require SSL on 443
    * Proxy traffic on 443 to our application.
  * Our application container
* A service to expose our pod externally.

What we're assuming:
* kubectl is installed
* envtpl is installed for templating
* The Kubernetes namespace exists.

![Architecture Overview](https://docs.google.com/drawings/d/1B5P1jZsDT8iNpSp3wpRQOsvbND_emg7ibuDVDC2O8Sw/pub?w=532&h=396)

## The Secret (not the book)
Since we can't store secrets in a repository what we typically do is store a template that we can reuse later.

Secrets can be injected via volumes or environment variables, we'll be using volumes.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-certs
  namespace: my-ns
type: Opaque
data:
  tls.crt: {{ TLS_CRT_BASE64 }}
  tls.key: {{ TLS_KEY_BASE64 }}
```

With this definition, we'll be mounting this into our Nginx container.  The volume resulting will be a directory with two files: tls.crt and tls.key.

Now to fill it, the values must be Base64 with no line breaks, well use `entpl` for our example.

```bash
$ TLS_CRT_BASE64=$(cat tls.crt | base64 -w 0) \
TLS_KEY_BASE64=$(cat tls.key | base64 -w 0) \
envtpl < secret.yaml | kubectl apply -f -
```

## The Config

Configs, similar to secrets, can be mounted as volumes or inject
as env vars.  We'll be mounting again.

Configs arn't required to be Base64 so we can just hard code in
our configuration.

We'll start by breaking down our nginx configuration:

First we listen on 80 and redirect everything permanently
for our domain.

```
server {
  listen       80;
  server_name  my-domain.lol;
  return 301 https://$host$request_uri;
}
```

On 443, we'll require TLS and expect our secret to be mounted
to `/etc/nginx/certs`.  For simplicity, I'm leaving out any
extra headers that might be used for our application.

In this case our application runs on port 8081.

```
server {
  listen              443 ssl;
  server_name         my-domain.lol;
  ssl_certificate     /etc/nginx/certs/tls.crt;
  ssl_certificate_key /etc/nginx/certs/tls.key;
  ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers         HIGH:!aNULL:!MD5;
  location / {
    proxy_pass   http://127.0.0.1:8081;
    proxy_set_header Host $host;
  }
}
```

We can take these configurations and place them all into
our ConfigMap using multiline features of yaml.

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: my-nginx-config
  namespace: my-ns
data:
  default.conf: |
    server {
      listen       80;
      server_name  my-domain.lol;
      return 301 https://$host$request_uri;
    }
    server {
      listen              443 ssl;
      server_name         my-domain.lol;
      ssl_certificate     /etc/nginx/certs/tls.crt;
      ssl_certificate_key /etc/nginx/certs/tls.key;
      ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
      ssl_ciphers         HIGH:!aNULL:!MD5;
      location / {
        proxy_pass   http://127.0.0.1:8081;
        proxy_set_header Host $host;
      }
    }
```

This definition is not templated so we can just apply as is.

```
$ kubectl apply -f config.yaml
```

## The Pod (actually a deployment)

We want our service to rescheduled on failure so we'll be
using a Deployment descriptor.

The deployment descriptor will describe our Pod with our
two containers.  Since we're mounting the configuration
and secret for Nginx, we can use the official Nginx image.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-deployment
  namespace: my-ns
spec:
  replicas: 2
  template:
    metadata:
      labels:
        name: my-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:1.11
        imagePullPolicy: Always
        volumeMounts:
          - name: nginx-config
            mountPath: /etc/nginx/conf.d
          - name: nginx-certs
            mountPath: /etc/nginx/certs
            readOnly: true
      - name: my-app
        image: my-app:1
        imagePullPolicy: Always
      volumes:
        - name: nginx-config
          configMap:
            name: my-nginx-config
        - name: nginx-certs
          secret:
            secretName: my-certs
```

Deploy via kubectl

```sh
$ kubectl apply -f deployment.yaml
```

## The Service

Finally, the service is what actually makes our Pod
accessible.  It'll make it accessible in two ways, first internally via the domain name `my-service.my-ns`.   Second,
since we're defining the type as LoadBalancer, Kubernetes
will provision a GCP LoadBalancer to direct traffic to
this service and also give it an eternal IP address.

By default, the IP is ephemeral. If we want to setup external
DNS, we can allocate a static IP in GCP and put that into
the definition.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: my-ns
spec:
  type: LoadBalancer
  ports:
    - name: redirect
      port: 80
      targetPort: 80
    - name: ssl
      port: 443
      targetPort: 443
  selector:
    name: my-deployment
```

### Done. for now...
