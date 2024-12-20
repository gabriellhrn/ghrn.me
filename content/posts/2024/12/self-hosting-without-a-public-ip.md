---
title: "Self-hosting without a public IP"
date: 2024-12-20T09:31:37+01:00
categories: ['tech']
tags: ['cloudflare', 'kubernetes', 'self-hosting', 'tutorial']
---

Some time ago I installed [k3s](https://k3s.io/) on a couple of Raspberry Pi to
learn more about Kubernetes. I ended up using this cluster as a home lab. I
would use it to test some software I wrote, to try out tools or services that
could be deployed on Kubernetes, and I used it to control my home network too.
I deployed PiHole to this cluster and configured my router to use PiHole as the
DNS server. It worked pretty well for a while until the Pi became noisy
because their fans started to fail.

While that setup was up and running, I wanted to be able to access the services
running on the cluster even when I wasn't home. The problem is that I don't own
a public IP and I didn't want to buy one, not only because I'd have to pay for it,
but mainly because I actually didn't want to expose services that were probably
not safe to the public Internet.

I looked into a couple of options, but ended up going with Cloudflare. I was
already using Cloudflare's free plan to manage my DNS and I found out I could
use Cloudflare Tunnel to expose my services through Cloudflare's global
network and Cloudflare Access for authentication.

In this post, we'll deploy a web server and a database to Kubernetes, and
expose them to the Internet, through Cloudflare. **Note that this setup is not
recommended for production!** I'm not going to configure authentication or make
the service reliable. This is for learning purposes.

# The setup

I'm assuming you already have Kubernetes running. If you don't, spin up one with
[microk8s](https://microk8s.io/) or [minikube](https://minikube.sigs.k8s.io/docs/).
I tested this setup on microk8s running on my laptop. I'm also assuming you
already have a domain managed by Cloudflare. If you don't, you can create a
Cloudflare account and add your domain. You can use Cloudflare Tunnel, Access
and DNS with the free plan.

This setup works by deploying a lightweight daemon called `cloudflared` to your
private network, in this case to a local Kubernetes cluster. This daemon can
connect to resources in the private network, such as HTTP servers, and will
establish an outbound-only connection to Cloudflare's global network, making it
possible to connect to private resources through Cloudflare.

## Creating the tunnel

We'll first create the tunnel, then deploy `cloudflared` to the cluster. I'm
using the [CLI](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-local-tunnel/#1-download-and-install-cloudflared),
but the same can be done through the [Cloudflare Dashboard](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/).

Let me log in to Cloudflare first:

```bash
$ cloudflared tunnel login
```

And now I create a tunnel for my home lab:

```bash
$ cloudflared tunnel create home-lab
...
Created tunnel home-lab with id <tunnel-uuid>
```

We'll need the tunnel UUID for later steps, but you can fetch it later by
running:

```bash
$ cloudflared tunnel info
```

We can also use this command to check if the tunnel has been created successfully.

Alright, now that we have a tunnel, we need to deploy `cloudflared` and some
sample services to Kubernetes, then we can connect `cloudflared` to the tunnel
we just created.

## Deploying sample services

We're going to deploy a web server and a database. We'll connect cloudflared
to the web server via HTTP and to the database via TCP.

Create a file called `nginx.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
```

Deploy nginx to Kubernetes by running:

```bash
$ kubectl create -f nginx.yaml
```

This will create an `nginx` deployment and an `nginx` service listening on port 80.
You can check if it was deployed correctly by doing port forwarding:

```bash
$ kubectl port-forward svc/nginx 8080:80
```

And using curl to access the service:

```bash
$ curl localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

Now we'll deploy the database. Create a file called `postgres.yaml` with the
following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:latest
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres
                  key: password
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
```

We'll first create a secret with a password:

```bash
$ kubectl create secret generic postgres --from-literal=password=<my-password>
secret/postgres created
```

And now we deploy the service:

```bash
$ kubectl create -f postgres.yaml
```

This will create a `postgres` deployment and a `postgres` service listening on
port 5432. Test it with port forwarding:

```bash
$ kubectl port-forward svc/postgres 5432:5432
```

Access it using `psql`:

```bash
$ psql -h localhost -U postgres -p 5432
...

postgres=#
```

Now that we have some sample services running on Kubernetes, we can deploy
`cloudflared` and set up some DNS routes.

## Deploying the daemon

We'll deploy `cloudflared` as a deployment and we'll use a `ConfigMap` to
configure the daemon, including the ingress rules to route the traffic to the
right service. Note that the `cloudflared` deployment won't create the routes.
You have to create them yourself.

We'll create a file called `cloudflared.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
spec:
  selector:
    matchLabels:
      app: cloudflared
  replicas: 2
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
      - name: cloudflared
        image: cloudflare/cloudflared:2024.12.2
        # Run the tunnel at startup with config from config.yaml.
        # config.yaml is the ConfigMap.
        args:
        - tunnel
        - --config
        - /etc/cloudflared/config.yaml
        - run
        livenessProbe:
          httpGet:
            path: /ready
            port: 2000
          failureThreshold: 1
          initialDelaySeconds: 10
          periodSeconds: 10
        volumeMounts:
        # We mount the ConfigMap at /etc/cloudflared/config.yaml.
        - name: config
          mountPath: /etc/cloudflared
          readOnly: true
        # We mount credentials.json (from Secret)
        # at /etc/cloudflared/creds/credentials.json.
        - name: creds
          mountPath: /etc/cloudflared/creds
          readOnly: true
      volumes:
      # Create a volume for credentials.json. It will be read by the daemon
      # during startup.
      - name: creds
        secret:
          secretName: home-lab
      # Create a volume for config.yaml. It will be read by the daemon during
      # startup.
      - name: config
        configMap:
          name: cloudflared
          items:
          - key: config.yaml
            path: config.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared
data:
  # We use this file to configure cloudflared.
  config.yaml: |
    # The name of the tunnel to use. It has to be created previously.
    # To create a tunnel:
    #   cloudflared tunnel create <tunnel-name>
    tunnel: home-lab
    credentials-file: /etc/cloudflared/creds/credentials.json
    metrics: 0.0.0.0:2000
    no-autoupdate: true
    # Route these domains to the local services. The DNS names need to be
    # created previously and routed to the tunnel above.
    # To route a domain to the tunnel:
    #   cloudflared tunnel route dns <tunnel-name> <domain>
    ingress:
    # For HTTP services, we prefix the domain with http://
    - hostname: www.example.com
      service: http://nginx:80
    # This is a TCP connection to a database.
    # We prefix the domain with tcp://
    - hostname: pg.example.com
      service: tcp://postgres:5432
    # The file must contain a catch-all rule. Here we respond with 404 if nothing
    # else matches.
    - service: http_status:404
```

<!-- vale Vale.Spelling = NO -->
If you look at the config file, you'll notice two hostnames in the ingress part:
<!-- vale Vale.Spelling = YES -->

```yaml
...
    ingress:
    - hostname: www.example.com
      service: http://nginx:80
    - hostname: pg.example.com
      service: tcp://postgres:5432
...
```

Update them with your own domain and take note of them. We'll configure DNS
records for these subdomains later on.

We can deploy `cloudflared` with:

```bash
$ kubectl create -f cloudflared.yaml
```

Let's take a look at the logs and see if the daemon is running fine:

```bash
$ kubectl logs deploy/cloudflared
...
INF Starting metrics server on [::]:2000/metrics
INF Registered tunnel connection connIndex=0 connection=<uuid>
...
```

With the daemon running we can create DNS records to point to the tunnel. We
need to create DNS records for those domains configured in the ingress. The
command will be `cloudflared tunnel route dns <tunnel-name> <hostname>`:

```bash
$ cloudflared tunnel route dns home-lab www.example.com
INF Added CNAME www.example.com which will route to this tunnel tunnelID=<uuid>
$ cloudflared tunnel route dns home-lab pg.example.com
INF Added CNAME pg.example.com which will route to this tunnel tunnelID=<uuid>
```

And with that we can now access our local resources through Cloudflare:

```bash
$ curl https://www.example.com
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

For the database, we need to do port forwarding with `cloudflared access`:

```bash
$ cloudflared access tcp --hostname pg.example.com --url localhost:5432
INF Start Websocket listener host=localhost:5432
```

On another shell:

```bash
$ psql -h localhost -U postgres -p 5432
Password for user postgres:
...

postgres=#
```

Exposing Postgres like this is particularly useful, because we can now use
the tunnel as a jumphost. It's a good practice to keep databases in private
networks, and Cloudflare Tunnel for connectivity and Access for authentication
is an option that is safe and simple to set up, while providing authentication
with identity providers (the free version allows you to authenticate with
GitHub). We can also configure Access to request a reason for the access request
and require an approval from someone before granting the access. I'll cover a
setup with Access in another post.

# Final thoughts

I like using Tunnel and Access because they're simple to set up, they provide a
safe solution, and they're included in Cloudflare's free plan. However, I wish
Cloudflare would let me manage my tunnels, domains and access applications from
Kubernetes too. That would make it easier to automate everything. But I believe
it shouldn't be too hard to create a
[controller](https://kubernetes.io/docs/concepts/architecture/controller/)
and use the SDK to manage those resources. Something to look into.
