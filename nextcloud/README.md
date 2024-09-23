# Deploy Nextcloud in Kubernetes using Kustomize

A [NextCloud](https://nextcloud.com/) Kubernetes deployment with Apache, MariaDB,
Cron, and Redis. Based on the code found in [this article on
Medium](https://medium.com/@acheaito/nextcloud-on-kubernetes-19658785b565), and
subsequently modified.

## Requirements

* A functional k8s cluster.
* A dedicated namespace for this project (default is `nextcloud`).
* An Ingress provider (I use `ingress-nginx`, not to be confused with `nginx-ingress`)
* A TLS secret suitable for an Ingress route. You need to either create a TLS
  secret before hand, or modify `patches/nextcloud/ingress/settings.yaml` to do
  the Let's Encrypt part for you automatically.
* Secrets defined in `.env.secret`:
  * `MYSQL_DATABASE=nextcloud`
    * The MariaDB database name
  * `MYSQL_USER=nextcloud`
    * The MariaDB username
  * `MYSQL_PASSWORD=<plaintext_value_here>`
    * The MariaDB password for the above username
  * `MYSQL_ROOT_PASSWORD=<plaintext_value_here>`
    * The MariaDB root password
  * `NEXTCLOUD_ADMIN_PASSWORD=<plaintext_value_here>`
    * The default Nextcloud Admin password
  * `SMTP_HOST=smtp.example.com`
    * [OPTIONAL] for email support
  * `SMTP_NAME=user@example.com`
    * [OPTIONAL] for email support
  * `SMTP_PASSWORD=<plaintext_value_here>`
    * [OPTIONAL] for email support
  * `MAIL_FROM_ADDRESS=nextcloud`
    * [OPTIONAL] for email support
  * `MAIL_DOMAIN=example.com`
    * [OPTIONAL] for email support

## Deployment (without `overlays`)

### `.env.secret`

Change these values accordingly. The `MYSQL_DATABASE` and `MYSQL_USER` values
should probably remain as `nextcloud`, but the `_PASSWORD` values can be set to
whatever you like. The mail/SMTP values should be set if you plan on using email
functionality. This requires additional settings to be uncommented in
`patches/nextcloud/deployment/settings.yaml`.

```
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud
MYSQL_PASSWORD=<plaintext_value_here>
MYSQL_ROOT_PASSWORD=<plaintext_value_here>
NEXTCLOUD_ADMIN_PASSWORD=<plaintext_value_here>
SMTP_HOST=smtp.example.com
SMTP_NAME=user@example.com
SMTP_PASSWORD=<plaintext_value_here>
MAIL_FROM_ADDRESS=nextcloud
MAIL_DOMAIN=example.com
```

Update this file with the values required. 

### `kustomization.yaml`

```
namespace: nextcloud

commonLabels:
  app: nextcloud
```

Change `namespace` if you prefer. The default is `nextcloud`.

Any `commonLabels` you apply must never change. This is an unbreakable rule.

#### Images

Update each `newTag` vaue with the latest, or desired version for each image.


```
images:
  - name: nextcloud
    newTag: 30.0.0-apache
  - name: redis
    newTag: 7.4.0-alpine3.20
  - name: mariadb
    newTag: lts-noble
```
  
#### MariaDB

Update the amount of space you want to allocate to the MariaDB PVC, and specify
a `storageClassName` as needed.

```
# MariaDB patches
- patch: |-
    ## Storage
    - op: replace
      path: /spec/volumeClaimTemplates/0/spec/resources/requests/storage
      value: 10Gi
    ## Uncomment and add a value if you need to define a storageClassName
    # - op: add
    #   path: /spec/volumeClaimTemplates/0/spec/storageClassName
    #   value: ""
    ## StatefulSet
    ## Uncomment this to make it pull the latest lts-noble image Always
    # - op: replace
    #   path: /spec/template/spec/containers/0/imagePullPolicy
    #   value: Always
  target:
    kind: StatefulSet
    name: db
```

`10Gi` is the default value. Be advised, whatever you set is **NOT** upgradable
since MariaDB is now a StatefulSet. Choose wisely for your initial size. If you
have a large number of files and users, it may be prudent to set this larger.

Also, the image behind the `lts-noble` tag for the MariaDB image may change, but
you won't automatically receive that update because the default `imagePullPolicy`
for tags that are not `latest` is `IfNotPresent`. If you want to _always_ pull
the latest `lts-noble` tag, then uncomment the `imagePullPolicy` patch section.
This can also be done ad-hoc when you know there's been a change. Just uncomment
it for "this" update, and then recomment again.

#### Nextcloud

**Storage settings**

These patches will need to be edited and configured with your storage settings
before they can be used.

```
## App storage
- patch:
  path: patches/nextcloud/pv/settings.yaml
- patch:
  path: patches/nextcloud/pvc/settings.yaml
## External storage
## Only uncomment if base/external is also uncommented in the resources block.
# - patch:
#   path: patches/external/pv/settings.yaml
# - patch:
#   path: patches/external/pvc/settings.yaml
```

Uncomment `base/external` if you want to add the External storage lines.

```
resources:
  - base
  - base/external
```

**Deployment settings**

Carefully read the comments in this file as they explain settings that need to be
updated, or uncommented and used in conjunction with other settings and patches.

* `patches/nextcloud/deployment/settings.yaml`:
  * `MYSQL_HOST` replace `NAMESPACE` with your own. If you use `nextcloud` as
    your namespace, it will read `db.nextcloud.svc.cluster.local`
  * `REDIS_HOST` replace `NAMESPACE` with your own. If you use `nextcloud` as
    your namespace, it will read `redis.nextcloud.svc.cluster.local`
  * `OVERWRITECLIURL` should be whatever you use for `NEXTCLOUD.EXAMPLE.COM` in
    `patches/nextcloud/ingress/settings.yaml`.
  * `NEXTCLOUD_TRUSTED_DOMAINS` can be multiple space separated values. At the
    very least, you should have `localhost` and whatever you used for
    `NEXTCLOUD.EXAMPLE.COM`. Possible additional values include:
      * `app.NAMESPACE.svc.cluster.local`, where `NAMESPACE` is the actual
        name of your namespace in `kustomization.yaml`. The provided value
        is `nextcloud`.
      * `ingress-nginx-controller.NGINX-INGRESS-NAMESPACE.svc.cluster.local`,
        where `NGINX-INGRESS-NAMESPACE` is the namespace where your Nginx
        ingress controller was installed. If you followed their normal, helm
        installation procedures, this is ostensibly `ingress-nginx`, and the
        kubernetes metadata name is `ingress-nginx-controller`.
  * `SMTP_` settings. To use these, uncomment and ensure you have added
    the necessary passwords and settings to `.env.secret`.

* `patches/nextcloud/ingress/settings.yaml`:
  * Replace `NEXTCLOUD.EXAMPLE.COM` with your domain name.
  * Replace `<MY_TLS_SECRET>` with your own TLS secret.

* `pv/settings.yaml` & `pvc/settings.yaml` in `patches/nextcloud`:
  * Edit to fit your needs. This project uses an nfs mount. You may need to fork
    and edit if this is not how your environment is configured.

* OPTIONAL: `pv/settings.yaml` and `pvc/settings.yaml` in `patches/external/`:
  * Edit to fit your needs. This project uses an nfs mount. You may need to fork
    and edit if this is not how your environment is configured. This also requires
    uncommenting `- base/external` in `resources:` near the top of the
    `kustomization.yaml` file, as well as other related configuration items.

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
