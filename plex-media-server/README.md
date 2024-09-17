# Deploy Plex Media Server in Kubernetes using Kustomize

**HUGE WARNING:** This is an ALPHA release and is not ready for general consumption.

Have a Kubernetes cluster and want to run your own [Plex Media Server](https://plex.tv/)?

Hopefully this makes it simple for you!

## Requirements:

* A functional k8s cluster.
* An Ingress provider (I use `ingress-nginx`, not to be confused with `nginx-ingress`)
* [OPTIONAL] If you plan on securing your ntfy server with TLS encryption, you
  need to either create a TLS secret before hand, or modify `patches/ingress.yaml`
  to do the Let's Encrypt part for you automatically.

## Configuration:

DO NOT USE unless you know what you're doing and are willing to make the effort
to fix it yourself.

At the very least, the `volumeClaimTemplates` need to be put back into the
`StatefulSet` instead of a dedicated pv + pvc.