# Deploy Nginx Reverse Proxy in Kubernetes using Kustomize

This should be a very quick & dirty way to reverse proxy a web server or service
_not_ hosted in Kubernetes using an Nginx container and an Ingress controller.

These are light weight and low resource enough that if you need multiple, you
ought to run one `overlay` per host you are reverse-proxying. They're also
delightfully self-cleaning since they create and destroy the entire namespace.

## Requirements 

* A functional k8s cluster.
* An Ingress provider (I use `ingress-nginx`, not to be confused with `nginx-ingress`)

### Optional

* A TLS secret suitable for an Ingress route. If you plan on securing your Pi-hole
  server with TLS encryption, you need to either create a TLS secret before hand,
  or modify `patches/ingress/settings.yaml` to do the Let's Encrypt part for you
  automatically.

## Deployment (without `overlays`)

The options *I* needed are in the `patches` subdirectory. Feel free to edit and
add as much as needed for your needs.

I will go through the options I've sanitized in order:

### `patches/configmap/default.conf`

Paste in your Nginx reverse proxy configuration. The included example should be
sufficient to get you started. There are many more options than can be discussed
here. It should suffice here that the proxy is just doing simple pass-through to
the endpoint behind.

### `patches/configmap/nginx.conf`

Edit this to suit. The defaults are kind of aggressive for a small proxy, so I
made them less aggressive with only 4 workers and 128 connections per worker. If
you know what you're doing, then you can edit these accordingly. If you are only
proxying a lightweight, low-traffic webserver, then these should be fine.

### `patches/deployment/settings.yaml`

Make any changes here, as needed--particularly to the CPU and memory `resources`.
The defaults here should suffice for a lightweight, low-traffic webserver, but
you can dial those settings up however you need.

### `patches/ingress/settings.yaml`

If you plan on using TLS to terminate, set `<MY_TLS_SECRET>` and
`<REVERSE.EXAMPLE.COM>`. You will also need to uncomment the patch section in
`kustomization.yaml`.

### Patches in `kustomization.yaml`

Replace `<REVERSE-PROXY>` with your chosen namespace. This must be done both for
`namespace:` at the top of the file and in the `patches:` section for replacing
namespace:

```
## Namespace settings
## Change <REVERSE-PROXY> to match whatever you put in namespace above
- patch: |-
    - op: replace
      path: /metadata/name
      value: <REVERSE-PROXY>
```

#### Ingress Settings

`<REVERSE.EXAMPLE.COM>` should be the FQDN that hosts the UI, and match what was
used in `patches/statefulset/settings.yaml`

If you plan to use TLS to secure your ingress, be sure to uncomment these lines:

```
# - patch:
#   path: patches/ingress/settings.yaml
```

and edit `patches/ingress/settings.yaml` to set `<MY_TLS_SECRET>` and
`<REVERSE.EXAMPLE.COM>`.

### Test

(Optional) To preview generated configuration before deploying:

* `kubectl kustomize .`

### Apply

Run the following command to build and deploy:

* `kubectl apply -k .`

## Deployment using `overlays`

### Copy the basic structure to each `overlay`

* Copy `kustomization.yaml`, and the various `patches/*` files to `overlays/NAME`,
  or whatever directory structure you prefer.
* Each `overlays/NAME` should have its own `kustomization.yaml` and `patches`
  subdirectory.
* Be sure to update the `resources` section of `kustomization.yaml` to be able to
  reach the `base` directory:

  ```
  resources:
    - ../../base
  ```

### Edit each `overlay` path accordingly

Apply the same configuration steps as above for each `overlay/NAME` path you
create.

### Test

(Optional) To preview generated configuration before deploying:

* `kubectl kustomize overlays/NAME`

### Apply

Run the following command to build and deploy:

* `kubectl apply -k overlays/NAME`
