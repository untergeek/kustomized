# Deploy Nextcloud in Kubernetes using Kustomize

A [NextCloud](https://nextcloud.com/) Kubernetes deployment with Apache, MariaDB,
Cron, and Redis. Based on the code found in [this article on
Medium](https://medium.com/@acheaito/nextcloud-on-kubernetes-19658785b565)

## Requirements

* A functional k8s cluster.
* A kubernetes storage manager (I use [OpenEBS](https://www.openebs.io/)).
* An Ingress provider (I use `ingress-nginx`, not to be confused with `nginx-ingress`)

## Deployment (without `overlays`)

* Update `.env.secret` with values
  * These values will be used in various places, so this is mentioned first.
* Update `kustomization.yaml` with changes
  * Images section
    * Update each image with the latest, or desired version
  * MariaDB patches
    * Update the amount of space you want to allocate to the MariaDB PVC
    * If you have properly set the `MYSQL_` values in `.env.secret`, you should
      not have to change any of the `deployment` patches.
  * Nextcloud patches
    * If you edit the files `patches/*.yaml` files below, you should not have to
      change any values using `path: filename.yaml` syntax.
    * For `/spec/template/spec/containers/0/env/3/value` and
      `/spec/template/spec/containers/0/env/4/value`:
      * Replace `NAMESPACE` with the actual name of your namespace in
        `kustomization.yaml`. The provided value is `nextcloud`.
    * For `add` ops:
      * `OVERWRITECLIURL` should be whatever you used for `HOST.EXAMPLE.COM` in
        `patches/ingress.yaml`.
      * `NEXTCLOUD_TRUSTED_DOMAINS` can be multiple space separated values. At the
        very least, you should have `localhost` and whatever you used for
        `HOST.EXAMPLE.COM`. Possible additional values include:
          * `app.NAMESPACE.svc.cluster.local`, where `NAMESPACE` is the actual
            name of your namespace in `kustomization.yaml`. The provided value
            is `nextcloud`.
          * `ingress-nginx-controller.NGINX-INGRESS-NAMESPACE.svc.cluster.local`,
            where `NGINX-INGRESS-NAMESPACE` is the namespace where your Nginx
            ingress controller was installed. If you followed their normal, helm
            installation procedures, this is ostensibly `ingress-nginx`, and the
            kubernetes metadata name is `ingress-nginx-controller`.
      * If you do not want the `SMTP_` settings, you can comment those out, as
        they are `add` in the `kustomization.yaml` file.
* Update `patches/*.yaml` files:
  * `patches/ingress.yaml`. Replace `HOST.EXAMPLE.COM` with your domain name.
    Replace `secretName: my-tls-secret` with your own TLS secret.
  * `patches/volumes/pv.yaml` and `patches/volumes/pvc.yaml` to fit your needs.
    This setup uses an nfs mount. You may need to fork and edit if this is not
    how your environment is configured.
  * OPTIONAL: `patches/volumes/external-pv.yaml` and `patches/volumes/external-pvc.yaml`
    to fit your needs. This setup uses an nfs mount. You may need to fork and
    edit if this is not how your environment is configured. This also requires
    uncommenting `- base/external` in `resources`, as well as other related
    configuration items in `kustomization.yaml`.
* (Optional) To preview generated configuration before deploying:
    `kubectl kustomize .`
* Run the following command to build and deploy:
    `kubectl apply -k .`
         
## Deployment using `overlays`

* Copy `kustomization.yaml`, `.env.secret`, and the various `patches/*` files to
  `overlays/NAME`, or whatever directory structure you prefer.

* Be sure to update the `resources` section of `kustomization.yaml` to be able to
reach the `base` directory:

```
resources:
  - ../../base
#  - ../../base/external
```

* (Optional) To preview generated configuration before deploying:
    `kubectl kustomize overlays/NAME`
* Run the following command to build and deploy:
    `kubectl apply -k overlays/NAME`