# CHANGELOG

## 2024-09-23

A push to normalize the file structures among these different projects has meant
some changes happened. A few names were changed, but very little else is different
except as outlined. Notably, inline patches which were very verbose and hard to
read have been superseded by patch files in most cases.

Pre-change and post-change manifests were compared and the only changes were the
image tag for Nextcloud (bumped to 30.0.0), and the secret names, which have now
been unified (no need to replace them everywhere now).

### Breaking Change

MariaDB is now a `StatefulSet` rather than a `Deployment`. I had a bad experience
with an upgrade which made it clear this is the better way. Since I am the only
one using this Kustomization right now, it shouldn't affect anyone else.

### `README.md` 

Updated instructions and details to reflect these changes.

Match the other projects by describing deployment with and without `overlays`.

### `base`

Too numerous to enumerate all changes. Suffice to say that all all previous files
have been renamed and relocated. The notable exception to this is that MariaDB is
now a `StatefulSet` rather than a `Deployment`, and that the `secrets.yaml`
file that was part of the MariaDB configuration, which has been removed in lieu
of a  secretGenerator placed in `kustomization.yaml`, which reads from the new
file, `.env.secretbase`.

The reason for this change is that rolling restarts and such were found to be
untrustworthy for the database component. I tried an upgrade and an issue with
my PVC prevented an upgrade. A StatefulSet for just the MariaDB component made
much more sense, since that is one thing that should persist between redeployments
of the main Nextcloud app container.

### `patches`

#### `external`

The `external-pv.yaml` and `external-pvc.yaml` files have been renamed and
relocated here.

#### `nextcloud`

##### `deployment`

The `settings.yaml` file supersedes the old `deployment.yaml` patch file.

##### `ingress`

The Ingress `settings.yaml` file supersedes the old `ingress.yaml` patch file.

##### `pv`

The `settings.yaml` file supersedes the old `pv.yaml` patch file.

##### `pvc`

The `settings.yaml` file supersedes the old `pvc.yaml` patch file.

## 2024-09-10

### Initial Release

See README.md
