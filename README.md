# Kubernetes OpenID Connect Traefik Forward Authentication

This repository contains Kubernetes manifest files for deploying Traefik Forward
Authentication using OpenID Connect with Keycloak.

The `traefik-forward-auth` service that this deploys provides support for Google
as well as other OpenID Connect Providers, you can modify this configuration to
match other OIDC providers you integrate with.

## Traefik

[Traefik](https://containo.us/traefik/) is a Cloud Native Edge Router and
reverse proxy that can direct traffic between services based on routing rules.
Traefik provides a Ingress Controller that can be deployed into Kubernetes
clusters for these purposes. Traefik introduced a Kubernetes Custom Resource
Definition (CRD) for
[Ingress Routes](https://docs.traefik.io/providers/kubernetes-crd/), which is
what the configuration in this repository is based on.

### Traefik Forward Authentication

Traefik does not provide native authentication for things such as JWT, OpenID
Connect, etc. Traefik instead provides a
[ForwardAuth](https://docs.traefik.io/middlewares/forwardauth/) middleware that
can be used to delegate authentication to external service. Upon a successful
response Traefik will grant access to the requested resources.

### Deploying Traefik v2

See the Github repository
<https://github.com/sleighzy/k3s-traefik-v2-kubernetes-crd> for instructions and
configuration on deploying a k3s cluster with Traefik v2.

## K3s and K3d

[k3s](https://k3s.io/) is a lightweight, certified Kubernetes distribution, for
production workloads from Rancher Labs. k3s installs Traefik, version 1.7, as
the Ingress Controller, and a service loadbalancer (klippy-lb) by default so
that the cluster is ready to go as soon as it starts up. The instructions below
are using Traefik v2 so this cluster has been deployed without the default
Traefik 1.7 and Traefik v2 installed separately

[k3d](https://github.com/rancher/k3d) is tool developed by the folk at Rancher
to deploy k3s nodes into Docker containers. This provides the means to deploy
server and multiple worker nodes on your local machine, taking up very little
resource, each running within its own container.

k3s (using k3d) will be used as the Kubernetes distribution for the examples in
this repository.

## OpenID Connect

[OpenID Connect](https://openid.net/connect/) is extremely popular
authentication and authorization framework based on the OAuth 2.0 protocol.

## Keycloak

[Keycloak](https://www.keycloak.org/) is a widely used production Identity
Provider (IdP) and Identity Broker and natively supports authentication
mechanisms such as SAML and OpenID Connect. The instructions and configuration
in this repository will be integrating with Keycloak.

These instructions do not cover the actual deployment or configuration of
Keycloak. They are based on the following assumptions:

- that you have a client id configured in Keycloak, in my example this is
  `traefik-forward-auth`
- that you have the client secret for this client id
- that you have a user, or some form of identity, that you can authenticate with
  Keycloak

The
[k3s-keycloak-deployment](https://github.com/sleighzy/k3s-keycloak-deployment)
repository contains my manifest files and instructions for deploying Keycloak on
K3s.

## Traefik OpenID Connect Delegated Authentication Service

The GitHub repository <https://github.com/thomseddon/traefik-forward-auth> and
associated Docker image on Docker Hub is a Traefik Forward Authentication
implementation that uses OpenID Connect to authenticate with OpenID Connect
Providers. This implementation previously only supported Google as an OIDC
Provider, and a number of popular forks were made of this to extend this
support. This implementation however now offers full and extensive support for
other providers, and extended functionality. The instructions and configuration
below are based on this image given it is the original, now offers full support,
and is being maintained.

### Secrets

The `001-secrets.yaml` file needs to be updated with the client secret (from
Keycloak) and the random secret used for signing cookies.

```sh
$ echo -n '<my oidc client secret>' | base64
NWNhOWXXXXXXXXXXXXXXXXXXMjcw

$ openssl rand -hex 16 | base64
OWUwNTBmZTM5ODQwNjE1NzJmZGE1ZjQ2NGE4YjVkOTgK
```

### Configuration

The `001-config.yaml` file needs to be updated with the configuration for your
OIDC provider and other information.

Refer to the documentation on
<https://github.com/thomseddon/traefik-forward-auth> as to what the items are
and their values based on the configuration in the `001-config.yaml` file in
this repository.

### Deployment

Apply the manifests in order (prefixed by number) to install the secrets,
config, deployment and ingress route for the
[`traefik-forward-auth`](https://github.com/thomseddon/traefik-forward-auth)
delegated authentication service.

## Ingress vs IngresssRoute

Ingress Controllers such as Nginx, Traefik, etc. deployed within Kubernetes can
use rules to route traffic to services. Traefik as an Ingress Controller can use
the standard Ingress annotations and annotations to configure Traefik to use
forward auth for services. Traefik v2 however also has a
[Kubernetes CRD Ingress Controller](https://docs.traefik.io/routing/providers/kubernetes-crd/)
which is what this repository uses.

If using the new Traefik `IngressRoute` CRD then the `002-middlewares.yaml` file
should be applied. This middleware can then be referenced within the
`IngressRoute` definition for each service it should be applied to. This removes
a lot of the boiler-plate yaml for adding annotations to all your services.

If using Traefik with the normal `Ingress` annotations then the annotations
below would be applied to your service instead.

```yaml
metadata:
  name: whoami
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik
    ingress.kubernetes.io/auth-type: forward
    ingress.kubernetes.io/auth-url: http://traefik-forward-auth.kube-system.svc.cluster.local
    ingress.kubernetes.io/auth-response-headers: X-Forwarded-User
```

## Authentication for the whoami service

Using the classic `whoami` service deployment commonly used in Traefik and other
documentation the below `IngressRoute` can be applied. This specifies that
requests to the host `whoami.mydomain.io` will be routed to the `whoami` service
on port 80. This however also references the `traefik-forward-auth` middleware
that was applied in the `002-middlewares.yaml` file. If the session has not yet
been authenticated then the user will be redirected to Keycloak when attempting
to access `whoami.mydomain.io`. Once they have successfully provided their
credentials they will be redirected back to the `whoami` service.

```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: whoami
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`whoami.mydomain.io`)
      middlewares:
        - name: traefik-forward-auth
          namespace: kube-system
      services:
        - name: whoami
          port: 80
  tls:
    certResolver: godaddy
```

## Credits

Credits to the following for getting this going:

- <https://github.com/thomseddon/traefik-forward-auth/blob/master/examples/docker-compose-oidc.yml> -
  OIDC example with current configuration
- <https://github.com/thomseddon/traefik-forward-auth/issues/33> - initial k8s
  configuration, but seems to be out-of-date (or incorrect) with the exact
  configuration
- <https://geek-cookbook.funkypenguin.co.nz/ha-docker-swarm/traefik-forward-auth/keycloak/> -
  blog post on integrating with Keycloak using Traefik 1.7 and the funkypenguin
  fork of the traefik-forward-auth service
