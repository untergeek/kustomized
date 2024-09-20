# CHANGELOG

## 2024-09-20

A push to normalize the file structures among these different projects has meant
some changes happened. A few names were changed, but very little else is different
other than the change from a Deployment to a StatefulSet.

### `README.md` 

Updated instructions and details to reflect these changes.

Match the other projects by describing deployment with and without `overlays`.

### `base`

#### `configmap`

The `02-mymasq.conf` ConfigMap has been moved here.

#### `ingress`

The Ingress definition manifest has been moved here.

#### `service`

The Service definitions for the TCP and UDP services have been moved to sub-directories
here.

#### `statefulset`

The former Deployment has been recreated as a StatefulSet in order to have the
configuration be retained, even if the StatefulSet is deleted. This beats having
to re-upload and apply your backups each time you blast away the Deployment.

### `patches`

#### `configmap`

The `02-mymasq.conf` patch file has been moved here.

#### `ingress`

The Ingress `settings.yaml` file has been added. It will currently only apply
the TLS settings.

#### `service`

The former `tcp.yaml` and `udp.yaml` patch files have been relocated to `tcp`
and `udp` subdirectories here, each with its own `settings.yaml` file.

#### `statefulset`

The `settings.yaml` file here supersedes the old `deployment.yaml` patch file.
In addition to setting the same environment variables as before, it also has the
storage settings at the bottom of the file.

## 2024-09-10

### `README.md`

Updated instructions and details to reflect other changes.

### `base/services/tcp.yaml` & `base/services/udp.yaml`

Changed `externalTrafficPolicy` to `Local` as we have only 1 pod, and would
hopefully like to preserve the client IP address with this setting.

### `patches/deployment.yaml`

#### Thoroughly documented all options

#### Added environment variables

- `BLOCKINGMODE`
- `CUSTOM_CACHE_SIZE`
- `EDNS0_ECS`
- `MOZILLA_CANARY`

All of these options are set to their respective default values.

### Removed

#### `base/volumes/masq-pvc.yaml`

Also removed the associated entries in `base/deployment.yaml`.

This project was derived from `pihole-ha`, where we wanted to share `/etc/dnsmasq.d`
with other nodes. No such volume sharing is necessary in this project, as the
ConfigMap for `02-mymasq.conf` can be mounted as its own file.
