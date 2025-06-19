# Generic Deployment using Kustomize

This is hopefully a quick way to build a viable Deployment. A template, if
you will. I tried to make as many "add-ons" into `components` as I could, so all
you have to do is create a namespace, edit a lines of config files, configure
an image, and you're off.

## Requirements 

* A functional k8s cluster.
* A namespace for your project.

### Configure the Deployment

Replace `OWNER/IMAGE:TAG` in `base/deployment/manifest.yaml`. It is also a good idea to set the same values in `kustomization.yaml` in the `images` section so it is easy to upgrade from there.

Settings in `settings/server.properties`:

* `app_requests_cpu=100m`
* `app_requests_mem=256Mi`
* `app_limits_cpu=1000m`
* `app_limits_mem=1Gi`

### Configure the Service

Settings in `settings/name.properties`:

* `svc_name=app-svc`
* `svc_port_name=http`

Settings in `settings/server.properties`:

* `svc_type=ClusterIP`
* `svc_protocol=TCP`
* `svc_port=80`
* `svc_targetPort=80`

## Components

### configmap

Need to use a config file for your app? And mount it in a particular location?
Use the `configmap` component.

Settings in `settings/name.properties`:

* `configmap_name=app-cm`

Settings in `settings/server.properties`:

* `cm_mountPath=/path/to/subpath_dir`
* `cm_subPath=example.conf`

Add the file contents in `patches/configmap/app-cm.yaml` under `example.conf: |-`. If you do change `example.conf` here, be sure to also change the value of `cm_subPath` accordingly.

### envvars

Rather than make a huge list of environment variables via patches, you can import them all as a single file using the `envvars` component. This makes use of a `configMapRef`, like this:

```yaml
spec:
  template:
    spec:
      containers:
        - name: app-container
          envFrom:
          - configMapRef:
              name: app-env
```

Simply put `KEY=VALUE` in `settings/app.env` for as many environment variables as you need.

### ingress

Want to add an ingress? Enable the `ingress` component.

Settings in `settings/name.properties`:

* `ingress_name=app-ingress`

Settings in `settings/server.properties`:

* `ingressClassName=nginx`
* `ingress_fqdn=host.example.com`

Want it to use TLS (well, duh)? Enable the `ingress/tls` component. However, this depends on `CertManager` being able to automatically generate certificates via the named `ingress_cert_manager` (ostensibly something staging or prod).

Settings in `settings/server.properties`:

* `ingress_cert_manager=letsencrypt-staging`
* `ingress_tls=app-tls-certificate`

### loadbalancer

Do you have `metallb` installed and configured? You can add a LoadBalancer to your deployment by enabling the `loadbalancer` component.

Settings in `settings/server.properties`:

* `loadbalancer_ip=10.0.0.1`

### storage

Does your app need a PersistentVolumeClaim? Enable the `storage/app` component.

Settings in `settings/name.properties`:

* `pvc_name=app-pvc`

Settings in `settings/server.properties`:

* `appVol_storage=500Mi`
* `appVol_accessMode=ReadWriteOnce`
* `appVol_mountPath=/path/to/storage`

Does your PVC need to use a different StorageClass than the default? Enable the `storage/storageClass` component.

Settings in `settings/server.properties`:

* `appVol_storageClassName=default`


## Deployment (without `overlays`)

Whether you've enabled any components, make sure you've configured all necessary values in `settings/server.properties`, `settings/name.properties`, `app.env`, etc.

### Images

Replace with the next image tag that comes out.

### View the Kustomize build

* (Optional) To preview generated configuration before deploying:

  `kubectl kustomize .`

  or

  `kustomize build .`

### Apply the Kustomization

* Run the following command to build and deploy:

  `kubectl apply -k .`

## Deployment using `overlays`

### Copy the Kustomize bits to each `overlay`

* Copy `kustomization.yaml`, and the various `patches/*`, `replacements/*`, and `settings/*` files and/or paths to `overlays/NAME`,
  or whatever directory structure you prefer.
* Each `overlays/NAME` should have its own `kustomization.yaml` and a copy of the aforementioned files and paths.
* Be sure to update the `resources` section of `kustomization.yaml` to be able to
  reach the `base` and `components` directories:

  ```yaml
  resources:
    - ../../base

  components:
    - ../../<component path1>
    - ../../<component path2>
    ...
    - ../../<component pathN>
  ```

### Edit each `overlay` path accordingly

Apply the same configuration steps as above for each `overlay/NAME` path you
create.

### View the overlay Kustomize build

* (Optional) To preview generated configuration before deploying:

  `kubectl kustomize overlays/NAME`

  or

  `kustomize build overlays/NAME`

### Apply the overlay

* Run the following command to build and deploy:

  `kubectl apply -k overlays/NAME`
