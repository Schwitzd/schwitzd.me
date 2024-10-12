+++
title = 'Let’s Encrypt certificates with Traefik'
date = 2024-09-01T15:42:50Z
draft = false
+++

In this article, we'll look at using [Traefik](https://traefik.io/) in [K3S](https://k3s.io/) alongside [cert-manager](https://cert-manager.io/) to act as an [ACME](https://en.wikipedia.org/wiki/Automatic_Certificate_Management_Environment) (Automatic Certificate Management Environment) client for acquiring certificates from [Let's Encrypt](https://letsencrypt.org/).

In order to obtain certificates, it is necessary to prove domain ownership to a Certificate Authority (CA). One method of verifying ownership is through a [DNS-01 challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge).

For this guide, we will use Cloudflare as our DNS provider, but the principles outlined can easily be adapted for use with other DNS providers.

## Getting started

Without delving too deeply into the wonderful world of certificate management, here is a simplified schematic of what goes on under the hood.

<!--
flowchart RL
    subgraph LE[Let's Encrypt]
        CA
    end

    subgraph Cloudflare
        TXT
    end

    subgraph Issuer
        Token[API Token]
    end

    subgraph Certificate
        TLS[TLS Secret]
    end

    subgraph Traefik
        Ingress
    end

    Issuer --\> |1| LE
    Token --\> |2| TXT
    LE --\> |3 - DNS-01| Cloudflare
    CA --\> |4| Issuer
    Certificate --\> |5| Issuer
    Traefik --\> |6| TLS
-->

{{< figure src="https://1drv.ms/i/s!AnwizapUFZTc6z7XqXEf5qlNfE3r?embed=1&width=660" align="center" caption="source: blog.stonegarden.dev" >}}

1. The `issuer` requests a `DNS-01` challenge from Let's Encrypt.
1. Using the Cloudflare API token, the `issuer` creates a `TXT` DNS record in response to the DNS-01 challenge.
1. Let's Encrypt verifies the `TXT` record to establish trust.
1. After verification, Let's Encrypt will now sign certificate requests from the `issuer`.
1. The `certificate` resource asks the `issuer` for a valid TLS certificate and stores it as a `secret`.
1. Traefik appends the TLS `Secret` to the appropriate `Ingress` and `IngressRoute` resources.

## Traefik & cert-manager

Traefik is already included in K3S out of the box (unless you have chosen to use something else), all you need to do is deploy it using the [Helm Chart](https://helm.sh/docs/topics/charts/).

```sh
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace --set crds.enabled=true
```

## Cloudflare DNS

Cert-manager provides excellent documentation on setting up [DNS-01 providers](https://cert-manager.io/docs/configuration/acme/dns01/). For [Cloudflare](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/), all that's required is an API token with the necessary permissions. This guide will walk you through the steps to obtain and configure that token.

## Get the certificate

Now that you've deployed both Traefik and Cert-Manager, and created the Cloudflare API token, it's time to configure the cluster to obtain certificates from Let's Encrypt. But first you need to make a strategic decision, issue a wildcard certificate to be used on all pods, or request a specific certificate for each service. I chose to issue certificates per `Ingress` or `IngressRoute`. Read this documentation instead if you want to use [wildcard certificate](https://github.com/traefik/traefik-helm-chart/blob/master/EXAMPLES.md#provide-default-certificate-with-cert-manager-and-cloudflare-dns).

First we create a secret with the API token we got from Cloudflare:

```yaml
apiVersion: v1
kind: Secret
metadata:
 name: cloudflare-api-token
 namespace: cert-manager
type: Opaque
stringData:
 api-token: <API Token>
```

The next decision is whether to create an `Issuer` or a `ClusterIssuer` resource:

* `Issuer`: This resource is namespace-scoped, meaning that it can only be used within the same namespace in which it is created.
* `ClusterIssuer`: This resource is cluster-scoped, meaning it can be used across all namespaces in the cluster.

To keep my cluster simple, I decided to use a `ClusterIssuer` resource that will take care of issuing certificates for the whole cluster.

For testing purposes, it may be a good idea to use the Let's Encrypt staging server instead of the production server, so that you don't have to limit the rate.
If you are sure about your configuration, replace with the production server: `https://acme-v02.api.letsencrypt.org/directory`.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
 name: le-clusterissuer
spec:
 acme:
   email: <email>
   server: https://acme-staging-v02.api.letsencrypt.org/directory
   privateKeySecretRef:
     name: cloudflare-key
   solvers:
     - dns01:
         cloudflare:
           apiTokenSecretRef:
             name: cloudflare-api-token
             key: api-token
```

This is an example of values to configure `ingress` with TLS on `Grafana` using the Helm chart.

```yaml
ingress:
  enabled: true
  ingressClassName: traefik
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: "websecure"
    cert-manager.io/cluster-issuer: "le-clusterissuer"
  path: /
  pathType: Prefix
  hosts:
    - grafana.example.com
  tls:
    - secretName: tls-grafana-ingress
      hosts:
        - grafana.example.com
```

## Useful commands

Below is a list of useful commands for troubleshooting:

```sh
# Retrive log from cert-manager pod
kubectl -n cert-manager logs pods/<cert-manager-pod-name>

# Retrieve detailed information about issuers in the specified namespace
kubectl describe ClusterIssuer

# List detailed information about certificate requests in the specified namespace
kubectl -n <namespace> get certificateRequest -o wide

# Display the certificates present in the specified namespace
kubectl -n <namespace> get certificates

# Show detailed information about a specific TLS secret in the specified namespace
kubectl -n <namespace> describe secret <tls-secret-name>
```

## Credits

I would like to thanks for inspiring me to write this article:

* [Wildcard Certificates with Traefik](https://blog.stonegarden.dev/articles/2023/12/traefik-wildcard-certificates/) (also to let me discover [Mermaid](https://mermaid.live))
* [Secure Web Applications with Traefik Proxy, cert-manager, and Let’s Encrypt](https://traefik.io/blog/secure-web-applications-with-traefik-proxy-cert-manager-and-lets-encrypt/)
