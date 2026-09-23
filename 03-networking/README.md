# Networking notes

## nginx Service and port 80

`nginx-service.yaml` is deliberately configured as a `NodePort` Service.

Do not change it to `LoadBalancer` while it exposes Service port `80` on this
single-node K3s cluster. K3s ServiceLB creates a helper Pod that binds the
LoadBalancer Service port on the node. Traefik already owns node ports `80`
and `443`, so a second LoadBalancer Service on port `80` cannot be scheduled
and will remain pending.

```text
node:80 -> Traefik LoadBalancer -> Traefik Pod
node:31911 -> nginx-service NodePort -> nginx Pod
```

To expose nginx on standard HTTP/HTTPS ports, add an Ingress rule for it and
let Traefik route the request. Alternatively, use an unused Service port if a
separate K3s LoadBalancer Service is required.
