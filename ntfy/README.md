# Deploy ntfy in Kubernetes using Kustomize

Have a Kubernetes cluster and want to run your own [ntfy](https://ntfy.sh/) server?

Hopefully this makes it simple for you!

## Requirements:

* A functional k8s cluster.
* A dedicated namespace for this project (default is `ntfy`).
* An Ingress provider (I use `ingress-nginx`, not to be confused with `nginx-ingress`)

## Optional TLS Support

* If you have `cert-manager` installed and functional, you can easily enable TLS
  in your Ingress by enabling `component/ingress/tls` in `kustomization.yaml`.

## Settings

See `settings/config.properties`:

```ini
# NTFY
environment=staging
storage_size=500Mi
timezone=UTC
log_level=INFO
debug=false

# Resources
requests_cpu=150m
requests_mem=150Mi
limits_cpu=500m
limits_mem=500Mi

# Ingress
ingress_fqdn=ntfy.example.com
```

* `environment` is used to set the `NODE_ENV` environment variable
* `storage_size` determines the size of `/var/ntfy/cache`
* `timezone` is used to set the `TZ` environment variable
* `log_level` is used to set the `NTFY_LOG_LEVEL` environment variable
* `debug` is used to set the `NTFY_DEBUG` environment variable

The Resources settings set the pod resource limits (should be self-explanatory)

* `ingress_fqdn` sets the host name in your Ingress manifest.

Optional (require components to be enabled):

```ini
# TLS component (must be enabled in kustomization.yaml)
# For production, you probably want letsencrypt-prod, or whatever you named it
ingress_cert_manager=letsencrypt-staging
ingress_tls=ntfy-tls-certificate

# OTel component (must be enabled in kustomization.yaml)
otel_endpoint=http://my-otel-collector.opentelemetry-operator-system.svc:4318
otel_service_name=ntfy
```

These settings will do nothing unless `components/ingress/tls` is enabled in
`kustomization.yaml`:

* `ingress_cert_manager` is the name of your `cert-manager` Issuer (cluster or not)
* `ingress_tls` is the (arbitrary) name of the TLS certificate for your Ingress

These settings will do nothing unless `components/statefulset/otel` is enabled in
`kustomization.yaml`, and you have selected a replacement container image that
supports OTel (at this time, my custom build Docker container is available at
`untergeek/ntfy:v2.11.0-symbols`)

* `otel_endpoint` is the target endpoint for the OTel data
* `otel_service_name` is the name to apply as the service name to all OTel data.

## Deployment (without `overlays`)

### `kustomization.yaml`:

#### Namespace

You can use the default namespace, `ntfy`, or change to use your own. In either
case, the namespace must be created before deployment.

#### Images

Currently, the most recent release is `v2.11.0`. To run a different release, replace
`v2.11.0` with the desired tag.

```yaml
images:
  - name: binwiederhier/ntfy
    newTag: v2.11.0
```

### Patch files:

#### `patches/statefulset.yaml`:

One of the few reasons you might need this is to add any other environment variables
you may want.


### `patches/configmap/server.yml`:

This file will be used to create a ConfigMap that will be mounted as
`/etc/ntfy/server.yml`, and contains the settings used to configure ntfy. At the
very least, you should edit `patches/configmap/server.yml` and configure:

```yaml
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

* Copy `kustomization.yaml`, and the `patches`, `replacements`, and `settings`
  directories files to `overlays/NAME`, or whatever directory structure you prefer.
* Each `overlays/NAME` should have its own `kustomization.yaml`, `patches`,
  `replacements`, and `settings` directories.
* Be sure to update the paths in the `resources` and `components` section of
  `kustomization.yaml` accordingly (since the relative paths have changed):

  ```yaml
  resources:
    - ../../base

  components:
    - ../../components/ingress/tls
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
