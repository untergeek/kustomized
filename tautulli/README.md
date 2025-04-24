# Deploy Tautulli in Kubernetes using Kustomize

This is hopefully a quick way to get a fully functional Tautulli instance running.

## Requirements 

* A functional k8s cluster.
* A dedicated namespace for this project (default is `tautulli`).
* A running Plex Media Server somewhere this Tautulli instance can reach.

### Optional

* To enable automatic TLS encryption using the `component`, you will need to have
  a Cert Manager installed and functional in your Kubernetes cluster. Then you
  need only uncomment the `component` in `kustomization.yaml` and make sure you
  have entered names and the proper Cert Manager, e.g. `letsencrypt-prod`

## Deployment

Pretty much everything was designed to work by putting the proper values into
`settings/server.properties`, and to enable any `components` you might need. The
rest is handled by `replacements`.

## Post Deployment Configuration

If you are running Plex Media Server inside Kubernetes as well, when it comes
time to connect, you will want to use the Kubernetes service DNS name to connect,
even if Tautulli finds a valid IP. Things can move around in Kubernetes, so use
the Kubernetes DNS name to keep things sane.

The naming convention is `METADATA_NAME.NAMESPACE.svc.cluster.local`. For me, that
looks like `pms-web.plex.svc.cluster.local`, but don't forget to append `:32400`:

`pms-web.plex.svc.cluster.local:32400`
