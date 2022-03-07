+++

showpagemeta = true

title = "oAuth2Proxy, Azure AD and Traefik"

date =  "2022-03-04T18:58:33+01:00"

draft = false

categories = ["might-be-useful"]

+++

## Summary

[oAuth2Proxy](https://artifacthub.io/packages/helm/oauth2-proxy/oauth2-proxy) is a nice component for Kubernetes that
adds an authentication layer, transparently, on top of your workload.

The deployment of oAuth2Proxy is quite straightforward, via Helm, but when it comes to the integration with the
authentication providers, some complexity arises, mainly due to the fact the documentation is not very complete. To be
fair, oAuth2Proxy supports
[a lot of backends](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider), and while some of
these are well documented, for some others the documentation is very limited.

Let's talk about the latter category, in particular authentication with **Azure AD provider**.

This notes will describe how to configure oAuth2Proxy with Azure AD and expose its login endpoint via a Kubernetes
ingress resource.

When it comes to Ingress component in Kubernetes, I like Traefik. It's lightweight, supports a lot of features, is
pretty flexible and it integrates perfectly with the Kubernetes ecosystem.


## Architecture Diagram

How actually the authentication works?

oAuth2Proxy supports 2 mode of working: AuthServer or ReverseProxy. Since in our setup we already have a reverse proxy -
Traefik - we will use oAuth2Proxy only as AuthServer. In this way oAuth2Proxy will check if the provided token
(if any) is valid otherwise will trigger a redirect to start the authentication process in Azure AD.

See the diagram below:

{{<mermaid align="center">}} graph LR; U[User] -->|Request| T(Traefik Ingress)
T -->|forward auth| OA2P(oAuth2Proxy)
T -->|token is valid| W(Workload)
T -->|token is not valid| AZ[Redirect with 30x to Azure AD]
{{< /mermaid >}}


## Prerequisites

These notes start with the assumption you already have a properly configured **Azure AD App Registration**.    
In addition, we will assume Traefik is already installed in the cluster.

## Conventions used in this notes

1. We want to configure oAuth2Proxy with the domain: `example.com`
2. Our login endpoint will be: `login.example.com`
3. Our application endpoint is: `myapp.example.com`
4. The app registration we pre-configured in Azure AD give us the following information:
    - `<app_id>`
    - `<app_password>`
    - `<tenant_id>`
5. We generated a `<strong_cookie_secret>` via any or the procedures [described here].

[described here]: https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview#generating-a-cookie-secret

## oAuth2Proxy configuration

Although oAuth2Proxy seems to support native 'azuread' as provider, I noticed it works better with the standard 'oidc'
provider.

With Helm, you can ship the configuration via `values.yaml` file. Use the following configuration:

----
values.yaml

```yaml
config:
  clientID: "<app_id>"
  clientSecret: "<app_password>"
  cookieSecret: "<strong_cookie_secret>"
  configFile: |-
    cookie_domains = ".example.com"
    cookie_refresh = "1h"
    email_domains = "*"
    footer = "oAuth2 Protected Area"
    oidc_email_claim = "upn"
    oidc_issuer_url = "https://login.microsoftonline.com/<tenant_id>/v2.0"
    provider = "oidc"
    reverse_proxy = true
    set_authorization_header = true
    set_xauthrequest = true
    silence_ping_logging = true
    skip_jwt_bearer_tokens = true
    skip_provider_button = true
    upstreams = [ "static://202" ]
    whitelist_domains = ".example.com:*"

ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.allow-http: "false"
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: "websecure"
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.tls.options: "default"
    traefik.ingress.kubernetes.io/router.middlewares: oauth2proxy-headers@kubernetescrd
  path: /
  hosts:
    - login.<common_domain>
  tls:
    - hosts:
        - login.<common_domain>
```

_NOTE: if you check the values.yaml you will see we mention a Traefik middleware (we will create it later):
`oauth2proxy-headers@kubernetescrd`_

let's deploy oAuth2Proxy with the usual helm commands:

```shell
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
helm repo update
helm upgrade --install -n oauth2proxy --create-namespace oauth2proxy oauth2-proxy/oauth2-proxy \
  --values values.yaml
```

## oAuth2Proxy and Traefik integration

In the diagram above we mention that Traefik will use the [Forward Auth mechanism] to check whether a request is allowed
to be propagated. This in Traefik is achieved via the creation of a Middleware. This approach will additionally require
another middleware to properly set/propagate the required headers. In this case we will use the [Headers] middleware

[Forward Auth mechanism]: https://doc.traefik.io/traefik/middlewares/http/forwardauth/

[Headers]: https://doc.traefik.io/traefik/middlewares/http/headers/

From a separation of concern perspective, we can safely consider the **Headers** middleware strictly related to the
oAuth2Proxy functionality, so we will deploy that middleware in the oauth2proxy namespace.

When it comes to protect your application, then the other middleware can be part of your application namespace. This is
convenient when you want to protect multiple applications with oAuth2.

For sake of simplicity, in this example we will install both middlewares in the `oauth2proxy` namespace.

Let's create these 2 middlewares. See the code below

```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: headers
  namespace: oauth2proxy
spec:
  headers:
    sslRedirect: true
    stsSeconds: 315360000
    browserXssFilter: true
    contentTypeNosniff: true
    forceSTSHeader: true
    sslHost: example.com
    stsIncludeSubdomains: true
    stsPreload: true
    frameDeny: true
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: forwardauth
  namespace: oauth2proxy
spec:
  forwardAuth:
    address: "https://login.example.com/?rd=https%3A%2F%2Fmyapp.example.com"
    trustForwardHeader: true
    authResponseHeaders:
      - X-Auth-Request-Access-Token
      - Authorization
      - X-Auth-Request-Redirect
      - X-Auth-Request-Email
      - X-Auth-Request-User
      - X-Auth-Request-Groups
      - X-Auth-Request-Username
      - Set-Cookie
```

NOTES:

- As we mentioned above, if you need to protect multiple endpoints, you only need to create multiple `forwardauth`
  middlewares, one per each namespace where the endpoint is installed.
- Make sure you keep the urlencoded version for the parameter `rd=` in the `address` property of the `forwardauth`.

## Your application and oAuth2Proxy

Once we created the middlewares, Traefik will automatically discover them and use the `headers` one for the ingress of
oAuth2Proxy.

Now, to instruct our application to requires authentication we can simply add an annotation on the ingress of our
application with:

```yaml
traefik.ingress.kubernetes.io/router.middlewares: oauth2proxy-forwardauth@kubernetescrd
```

## Final Notes

Traefik uses a special construct to refer to Kubernetes CRD. If you checked above you saw middleware references such as
`auth2proxy-forwardauth@kubernetescrd`.  
This name represents the structure `<namespace>-<resource_name>@kubernetescrd`.


## What about creating an Azure AD App Registration?

This topic was out of scope of these notes, but you can find an excellent tutorial in this
[YouTube video](https://www.youtube.com/watch?v=59YwW8FrLm8). 

