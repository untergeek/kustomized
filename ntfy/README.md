# Deploy ntfy in Kubernetes using Kustomize

Have a Kubernetes cluster and want to run your own [ntfy](https://ntfy.sh/) server?

Hopefully this makes it simple for you!

## Requirements:

* A functional k8s cluster.
* A dedicated namespace for this project (default is `ntfy`).
* An Ingress provider (I use `ingress-nginx`, not to be confused with `nginx-ingress`)

## Optional

* A TLS secret suitable for an Ingress route. If you plan on securing your ntfy
  server with TLS encryption, you need to either create a TLS secret before hand,
  or modify `patches/ingress/settings.yaml` to do the Let's Encrypt part for you
  automatically.

## Deployment (without `overlays`)

### `kustomization.yaml`:

#### Namespace

You can use the default namespace, `ntfy`, or change to use your own. In either
case, the namespace must be created before deployment.

#### Images

Currently, the most recent release is `v2.11.0`. To run a different release, replace
`v2.11.0` with the desired tag.

```
images:
  - name: binwiederhier/ntfy
    newTag: v2.11.0
```

#### Ingress settings

Replace `<NTFY.EXAMPLE.COM>` with the FQDN you plan to use.

```
# Ingress settings:
- patch: |-
    - op: replace
      path: "/spec/rules/0/host"
      value: "<NTFY.EXAMPLE.COM>"
  target:
    kind: Ingress
    name: ingress
```

### Patch files:

#### `patches/statefulset/settings.yaml`:

Update/Add any environment variables you may need.

You can also update the resource settings, and the volume storage settings at the
bottom of the file.

#### `patches/ingress/settings.yaml`:

To use TLS to secure your traffic, replace `NTFY.EXAMPLE.COM` with the FQDN you
plan to use.

You should also have created a TLS secret in advance that matches the FQDN. Replace
`<MY_TLS_SECRET>` with the name of that TLS secret. It should be in the same
namespace, e.g. `ntfy`, unless you've gone to the trouble to create a service
account and can share secrets across namespaces.


```
  tls:
    - hosts:
      - "NTFY.EXAMPLE.COM"
      secretName: <MY_TLS_SECRET>
```

### `patches/configmap/server.yml`:

This file will be used to create a ConfigMap that will be mounted as
`/etc/ntfy/server.yml`, and contains the settings used to configure ntfy. At the
very least, you should edit `patches/configmap/server.yml` and configure:

```
base_url: https://ntfy.example.com/
```

which should be the FQDN you set in your Ingress.

Any other settings are at your discretion.

### Test

* (Optional) To preview generated configuration before deploying:

  `kubectl kustomize .`

### Apply

* Run the following command to build and deploy:

  `kubectl apply -k .`

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

* (Optional) To preview generated configuration before deploying:

  `kubectl kustomize overlays/NAME`

### Apply

* Run the following command to build and deploy:

  `kubectl apply -k overlays/NAME`
