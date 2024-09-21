# CHANGELOG

## 2024-09-20

A push to normalize the file structures among these different projects has meant
some changes happened. A few names were changed, but very little else is different
other than the change from a Deployment to a StatefulSet.

### `README.md` 

Updated instructions and details to reflect these changes.

Match the other projects by describing deployment with and without `overlays`.

### `base`

#### `ingress`

The Ingress definition manifest has been moved here.

#### `service`

The Service definition manifest has been moved here.

#### `statefulset`

The former Deployment has been recreated as a StatefulSet in order to have the
configuration be retained, even if the StatefulSet is deleted. This beats having
to re-upload and apply your backups each time you blast away the Deployment.

### `patches`

#### `configmap`

The `server.yml` file has been moved here. Note that because there is no ConfigMap
in the `base` definition, this one _must_ be configured properly for deployment
of the StatefulSet to work.

#### `ingress`

The Ingress `settings.yaml` file has been added. It will currently only apply
the TLS settings.

#### `statefulset`

The `settings.yaml` file here supersedes the old `deployment.yaml` patch file.
In addition to setting the same environment variables as before, it also has the
storage settings at the bottom of the file.

## 2024-09-10

### Initial Release

See README.md
