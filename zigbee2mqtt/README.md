# Deploy zigbee2mqtt in Kubernetes using Kustomize

## Requirements

* A functional k8s cluster.
* An Ingress provider (I use `ingress-nginx`, not to be confused with `nginx-ingress`)
* Secrets defined in `settings/.env.secrets`:
  * `user=zigbee2mqtt`
    * `user` the username to connect to your MQTT broker
  * `password=CHANGEME`
    * `password` the password for `user` on your MQTT broker
  * `network_key='[0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15]'`
    * `network_key` is an encryption key for your Zigbee network. You really
      shouldn't operate without it. It is a series of 16 digits. Each value can
      be between 0 and 15. You can randomly set each individual value to whatever
      you like. I do not recommend using all one number, or the sequence above.
      Please note that the array is encapsulated in single quotes. This is
      because it must be a string value to become a Secret. Don't worry, it will
      be properly rendered in `secret.yaml` by our lovely initContainer.

## HUGE NOTE

It is important to note that each time the StatefulSet starts or restarts, the 
initContainer called `init-configuration` will test whether 
`/app/data/configuration.yaml` exists. If it does not exist, the ConfigMap
derived `configuration.yaml` will be copied to `/app/dataconfiguration.yaml`. If it
does exist, it will not be overwritten. Environment variable values will overwrite
what is in `configuration.yaml` each time, but the file will otherwise never be
overwritten. This is in part because Zigbee2MQTT allows the end user to reconfigure
values in the configuration file via the UI, and these changes would otherwise not
be preserved between restarts.

## Settings Files

The values in these files are used to set most needed values.

### `.env.secrets`

Required:

```ini
# You definitely need to change the password and network_key
user=zigbee2mqtt
password=CHANGEME
network_key=GENERATE or '[0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15]'
mqtt_server=mqtt://mqtt.mosquitto.svc.cluster.local:1883
serial_port=CHANGEME to /dev/tty.XXXXXX or tcp://10.0.0.1:6638
```

* `user` is the MQTT username
* `password` is the MQTT password
* `mqtt_server` is the MQTT server IP address or URL
* `network_key` is the Zigbee network key, if you are migrating. Otherwise you can
  leave this as `GENERATE` and a random network key will be generated on first run.
  **HUGE NOTE:** Do not include spaces in the key. The secret sometimes gets munged
  by the `initContainer` that processes it. Keep it shorter for safety.
* `serial_port` is the filesystem path to the serial port, e.g. `/dev/ttyACM0`, or
  the tcp socket definition for a network connection.
  **HUGE NOTE:** Do NOT use single or double quotes around this value. It will break.

Optional:

These are only used if you have enabled the APM component, which will only work
if you are using an APM-enabled container image.

```ini
apm_token=CHANGEME
apm_url=CHANGEME
```

* `apm_token` is the Bearer authorization token used to connect to the APM URL.
* `apm_url` is the URL of the APM server

### `z2m.properties`

```ini
timezone=UTC
log_level=info
baudrate=115200
ingress_cert_manager=letsencrypt-staging
ingress_fqdn=Z2M.EXAMPLE.COM
ingress_tls=z2m-tls-certificate
ingress_basic_auth_msg=Authentication Required - Zigbee2MQTT UI
```

Required:

* `timezone` is your local timezone in canonical format, e.g. `America/Chicago`
* `log_level` is the log level you want to set
* `baudrate` is the transmission speed of your Zigbee device

Optional:

* `ingress_cert_manager` is only used if you have enabled the TLS component. It
  is the cert manager you want to use from `cert-manager`.
* `ingress_tls` is only used if you have enabled the TLS comonent. It is the name
  you wish to give to your TLS certificate. Arbitrary, used by `cert-manager`.
* `ingress_basic_auth_msg` is only used if you have enabled the basic_auth component.
  This message will be displayed when you attempt to visit the UI for Zigbee2MQTT,
  and basic auth credentials are requested.

### `auth`

This file will only be used if you enable the auth component.

```
z2muser:$apr1$MKqyDzD3$wuIpgAowG7NEi.uUAcCD50
```

This is an Apache `htpasswd` file. You can create your own via:

```shell
htpasswd -c FILENAME USERNAME
New password:
Re-type new password:
Adding password for user USERNAME
```

More passwords can be added to the same file by omitting the `-c` flag:

```shell
htpasswd FILENAME USER2
New password:
Re-type new password:
Adding password for user USER2
```

You absolutely should replace `auth` with your own file!

If you want to test with the values in this file:

* Username: `z2muser`
* Password: `zigbee2mqtt`

### `apm.properties`

This file will only be used if you enable the apm component, which will only work
if you are using an APM enabled container image.

```ini
# You must change the container image to one that supports APM for these to work
node_env=production
apm_service_name=zigbee2mqtt
apm_service_node_name=z2m-node
apm_verify_cert=false
apm_disable_instrumentations=redis
```

* `node_env` must be set to `production` in order for APM data to ship.
* `apm_service_name` is the `service.name` which will be set for all data.
* `apm_service_node_name` is the name assigned to the node. It is useful to
  manually set this here, otherwise it will be the container id.
* `apm_verify_cert` should probably stay false here to guarantee your APM
  server's self-signed certificate can be used to secure traffic, even if you
  don't have the CA for it.
* `apm_disable_instrumentation` allows you to put a comma-separated list of
  Node.js modules to ignore instrumentation data for. It can't be empty without
  an error resulting, so `redis` is placed here because Zigbee2MQTT makes no use
  of Redis.


## Deployment (without `overlays`)

### Edit `patches/configmap/configuration.yaml`

Some settings are documented in comments in the patch file. All settings are
documented at [zigbee2mqtt.io](https://zigbee2mqtt.io). Do check in
`patches/statefulset/settings.yaml` before including a setting that is already
being set as an environment variable.

### Edit `patches/ingress/settings.yaml`

Basic settings are documented in comments in the patch file. Any other settings
are your own choice and responsibility.

### Edit `patches/statefulset/settings.yaml`

#### `image`

In the event that you want to use an APM-enabled container image, you will need
to specify it here. At this time, the only supporting image is at
`untergeek/zigbee2mqtt:1.41.0-apm`, in which the only changes from the out of the
box image of the same version are as follows:

1. The top 5 lines of `index.js` are now:

```javascript
require('elastic-apm-node').start({
  active: process.env.NODE_ENV === 'production'
})

const semver = require('semver');
```

2. `npm install elastic-apm-node --save && \` was injected at line 13 of
`docker/Dockerfile` such that this particular `RUN` command now shows:

```Dockerfile
RUN apk add --no-cache --virtual .buildtools make gcc g++ python3 linux-headers git npm && \
    npm install elastic-apm-node --save && \
    npm ci --production --no-audit --no-optional --no-update-notifier && \
    # Serialport needs to be rebuild for Alpine https://serialport.io/docs/9.x.x/guide-installation#alpine-linux
    npm rebuild --build-from-source && \
    apk del .buildtools
```

3. The original line 40 of the Dockerfile has been commented out and an explanatory line
added, such that it now reads:

```Dockerfile
# Set this in your Container definition, not here
# ENV NODE_ENV production
```

The reason being that `NODE_ENV` should not be hard coded, in my opinion.

If `NODE_ENV` is not set to `production`, zigbee2mqtt will not attempt to send
APM data to the configured URL.

The resulting container image was published with this command:

```shell
docker buildx build \
   --file ./docker/Dockerfile \
   --platform linux/amd64 \
   --tag untergeek/zigbee2mqtt:1.41.0-apm \
   --build-arg VERSION=1.41.0-apm \
   --build-arg COMMIT=acd5932c \
   --push \
   .
```

Making use of this image requires you to specify it in this patch file:

```yaml
spec:
  containers:
    - name: zigbee2mqtt
      image: untergeek/zigbee2mqtt:1.41.0-apm
```

If you specify this in the patch file, you can update it using `images` in the
`kustomization.yaml` file:

```yaml
images:
  - name: untergeek/zigbee2mqtt
    newTag: 1.41.0-apm
```

#### `nodeName`

This setting is documented inline in `kustomization.yaml`, but it bears further
explanation here.

[`nodeName`](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodename)
will pin the pod to run only on the k8s node where your USB Zigbee controller is
inserted. This is only needed if you are using a USB dongle! For network-based
controllers, like the [UZG-01](https://uzg.zig-star.com), this setting is
unnecessary.

The `nodeName` setting should be indented at the same level as the term `containers`, e.g.

```yaml
spec:
  nodeName: my-k8-node-name
  containers:
    - name: zigbee2mqtt
```

#### Environment Variables

Comments inline explain much. Feel free to add any extra environment variables
as desired. The [zigbee2mqtt documentation](https://www.zigbee2mqtt.io/guide/configuration/#environment-variables)
explains:

> It is possible to override the values in `configuration.yaml` via environment
> variables. The name of the environment variable should start with
> `ZIGBEE2MQTT_CONFIG_` followed by the path to the property you want to set in
> uppercase split by a `_`.
>
> In case you want to for example override:
> ```
> mqtt:
>   base_topic: zigbee2mqtt
> ```
> set `ZIGBEE2MQTT_CONFIG_MQTT_BASE_TOPIC` to the desired value.

So, ideally, use environment variables where it makes sense. Some of those names
will get really, really long, though:

```yaml
spec:
  containers:
    - name: zigbee2mqtt
      env:
        - name: ZIGBEE2MQTT_CONFIG_MQTT_BASE_TOPIC
          value: mytopic
```

#### `volumeMount` and `volumes` sections

If you are using a USB-based Zigbee device, you will need to uncomment and configure
these lines (i.e., set `/dev/ttyACM0` to wherever your device is in `/dev`).

```yaml
          volumeMounts:
            - name: z2m-data
              mountPath: /app/data
            ## Uncomment these if you need them
            # - name: z2m-udev
            #   mountPath: /run/udev
            # - name: zigbee-device
            #   mountPath: /dev/ttyACM0
      ## Uncomment these if you need them
      # volumes:
      #   - name: z2m-udev
      #     hostPath:
      #       path: /run/udev
      #   - name: zigbee-device
      #     hostPath:
      #       path: /dev/ttyACM0
```

If you are uncommenting these lines, unless you only have 1 k8s node in your
cluster, you will also need to make use of the `nodeName` directive already
described.

#### `volumeClaimTemplates` section

```yaml
  ## Uncomment this if you need to change anything, like specifying a non-default
  ## storageClassName, or using more or less storage.
  # volumeClaimTemplates:
  #   - metadata:
  #       name: z2m-data
  #     spec:
  #       storageClassName: MY_STORAGECLASS
  #       accessModes: [ "ReadWriteOnce" ]
  #       resources:
  #         requests:
  #           storage: 100Mi
```

### `kustomization.yaml` Configuration

#### `namespace`

You can change the namespace here.

```yaml
namespace: zigbee2mqtt
```

Because we are using a StatefulSet here, you should create the namespace manually
before applying. Having a namespace automatically be created and destroyed would
defeat the retention of the persistent volume claim we want (and need) because
deleting/destroying a namespace triggers the deletion of everything in it, which
would include our PVC we want to retain.

#### `commonLabels`

This will apply the app label `zigbee2mqtt` to everything being created.

```yaml
commonLabels:
  app: zigbee2mqtt
```

#### Images

This section allows us to apply a new image to upgrade our StatefulSet without
having to edit the `base` files.

```yaml
images:
  - name: koenkk/zigbee2mqtt
    newTag: 1.40.1
  - name: busybox
    newTag: "1.36"
```

The `busybox` image is used in our `initContainer` that does an initial copy of
the `configuration.yaml` ConfigMap to the persistent volume used by our StatefulSet.

#### Patches

Most of the patch files have already been covered already. This is where they
are applied.

```yaml
### Patches
patches:
## ConfigMap settings:
- patch:
  path: patches/configmap/configuration.yaml
## StatefulSet settings:
- patch:
  path: patches/statefulset/settings.yaml
```

#### Replacements

Values extracted from the `settings/*` files are applied in the various `replacements`
in the `patches` as well as several of the `components`. 

### Test

* (Optional) To preview generated configuration before deploying:

  `kubectl kustomize .`

### Apply

* Run the following command to build and deploy:

  `kubectl apply -k .`
         
## Deployment using `overlays`

### Copy the basic structure to each `overlay`

* Copy `kustomization.yaml`, and the `settings`, `patches`, and `replacements`
  directories to `overlays/NAME`, or whatever directory structure you prefer.
* Each `overlays/NAME` should have its own `kustomization.yaml`, `settings`,
  `patches`, and `replacements` subdirectories.
* Be sure to update the `resources` section of `kustomization.yaml` to be able to
  reach the `base` directory:

  ```yaml
  resources:
    - ../../base
  ```

* Enable any `components` you may want in `kustomization.yaml` by uncommenting them.

### Edit each `overlay` path accordingly

Apply the same configuration steps as above for each `overlay/NAME` path you
create. 

Use your judgment, but at the very least, you should probably ensure that you
are using different values for these settings in:

#### `kustomization.yaml`

* `namespace` -- A **must.** Absolutely necessary.

#### `patches/configmap/configuration.yaml`

* `mqtt.client_id` to keep them unique.
* `channel` numbers (prevent collisions)
* `network_key` value (again, prevent collisions)

#### `patches/statefulset/settings.yaml`

* `nodeName`, if you have multiple k8s nodes each running a USB Zigbee controller

### Test

* (Optional) To preview generated configuration before deploying:

  `kubectl kustomize overlays/NAME`

### Apply

* Run the following command to build and deploy:

  `kubectl apply -k overlays/NAME`

## Backup and Restore of your zigbee2mqtt data

Since this is a StatefulSet, the Persistent Volume should not be deleted in the
event that you delete your StatefulSet. This is a deliberate choice. You should
be able to backup or restore settings to this volume so long as it exists.

Unless you've modified the `base` configuration, the pod name should _always_ be
`zigbee2mqtt-0`. If you've modified the StatefulSet and chosen a different container
name, then it will be that value followed by a `-0`. For the sake of continuity,
this guide assumes that the pod name is `zigbee2mqtt-0`.

### Command-line Backups

#### Backup

This isn't backup so much as it is, "copy files from the volume attached to the pod."

The format of the command is:

```shell
kubectl -n NAMESPACE cp PODNAME:/PATH/TO/SOURCE /PATH/TO/DESTINATION
```

where

* `NAMESPACE` is the namespace where `PODNAME` is running
* `PODNAME` is the name of the pod
* `/PATH/TO/SOURCE` is the path to the source directory (or specific file) on
  the pod.
* `/PATH/TO/DESTINATION` is the local path where you want the entire source
  directory, or the specific file to go.

#### Restore

This isn't restore so much as it is, "copy files to the volume attached to the pod."

The format of the command is:

```shell
kubectl -n NAMESPACE cp /PATH/TO/SOURCE PODNAME:/PATH/TO/DESTINATION
```

where

* `NAMESPACE` is the namespace where `PODNAME` is running
* `PODNAME` is the name of the pod
* `/PATH/TO/SOURCE` is the path to the local directory (or specific file) to be
  copied to the pod.
* `/PATH/TO/DESTINATION` is the target path on the pod where you want the entire
  source directory (or specific file) to go.

### Backup

Backup can be done while the `zigbee2mqtt-0` pod is running. In fact, you cannot
backup data from the volume _unless_ it is attached to a pod.

#### Full directory backup

Using the defaults, our command might look like this (also creating the destination):

```shell
OUTPUT=./dump; \
mkdir -p $OUTPUT; \
kubectl -n zigbee2mqtt cp zigbee2mqtt-0:/app/data $OUTPUT
```

If we do this on a running system, the contents of `./dump` would look like:

```shell
$ ls -1 ./dump
configuration.yaml
coordinator_backup.json
database.db
devices.yaml
groups.yaml
secret.yaml
state.json
```

#### Single file backup

Using the defaults, our command might look like this:

```shell
FILENAME=configuration.yaml; \
kubectl -n zigbee2mqtt cp zigbee2mqtt-0:/app/data/$FILENAME $FILENAME.bak
```

This will copy `configuration.yaml` from the pod to `configuration.yaml.bak` in
the present working directory.

### Restore

**NOTE:** You should never attempt to restore data to this volume while it is
attached to the `zigbee2mqtt-0` pod. It is a running system, and overwriting or
changing files will undoubtedly lead to bad outcomes.

1. Ensure that the StatefulSet is not running.
2. Prepare the `mount_pvc.yaml` manifest in the same path as this README to mount
   the PVC in a separate pod. You shouldn't have to modify anything unless you have
   changed from the default in `base`.
3. Apply the manifest, manually specifying the `namespace`:

   ```shell
   kubectl -n NAMESPACE apply -f mount_pvc.yaml
   ```

4. Ensure the pod is live, and connect to it:

   ```shell
   kubectl -n NAMESPACE exec mount-pvc -ti -- ls /app/data
   ```

   This should show the contents of the PVC are in the exact same mount point as
   they would be in the StatefulSet pod.

At this point you can either copy an entire file over, or a full directory. The
procedure is slightly different for the full directory, so that will follow the
single file copy example.

#### Copy a single file

Using the defaults, our command might look like this:

```shell
kubectl -n NAMESPACE cp SOURCEFILE mount-pvc:/app/data/DESTFILE
```

This does allow for file renaming. For example:

```shell
kubectl -n NAMESPACE cp configuration.yaml.bak \
    mount-pvc:/app/data/configuration.yaml
```

This will overwrite the existing `configuration.yaml`.

#### Copy a whole directory

This is trickier since you can't use the contents of an entire directory as the
SOURCEFILE. The command won't accept `PATH/*` arguments. And just setting `PATH`
or `PATH/` will copy `PATH` to the `DESTINATION` such that you will have
`DESTINATION/PATH` instead of the contents of `PATH` being at `DESTINATION`.

The way around this is using tar streams. It's not as daunting as it sounds.

1. Navigate into the directory containing the contents you want to restore. You
   must be at the root of the path whose contents will be at `DESTINATION`, in
   our case, `/app/data` on the pod. If you don't, you'll end up copying whatever
   is in the directory you are in.
2. The command is this:

   ```shell
   tar cf - . | kubectl -n NAMESPACE exec mount-pvc -i -- tar xf - -C /app/data
   ```

Let's break that down a bit.

* `tar cf - .` captures the present working directory, `.` as a tar stream, which
  is piped to:
* `kubectl`
  * `-n NAMESPACE` (our namespace)
  * `exec` (going to execute a command on a pod)
  * `mount-pvc` (the pod name)
  * `-i` (the execution will be interactive [the stream])
  * `--` (`kubectl` now requires this to mean everything that follows will be
    part of the command and arguments to be executed in the pod)
  * `tar xf - -C /app/data` (extract the tar stream contents into `/app/data`)

I ran a few experiments and this will overwrite existing files on the pod.

You can now verify what's in the volume:

* `ls`:

  ```shell
  kubectl -n NAMESPACE exec mount-pvc -ti -- ls -l /app/data
  # ... ls output follows
  ```

* `cat`:

  ```shell
  kubectl -n NAMESPACE exec mount-pvc -ti -- cat /app/data/FILENAME
  # ... contents of FILENAME follow
  ```

#### Post-restore cleanup

At this point, you know you've restored things, so you need to delete the
`mount-pvc` pod by deleting the manifest:

```shell
kubectl -n NAMESPACE delete -f mount_pvc.yaml
```

This will delete the `mount-pvc` pod, but will leave the volume intact.

You can now recreate your zigbee2mqtt StatefulSet, and it will use the files
you restored as configuration data. Since we had to delete the StatefulSet in
order to stop the pod to do the file restore, we need to re-apply our Kustomization:

```shell
kubectl apply -k .
```

or

```shell
kubectl apply -k overlays/NAME
```

### Use Velero

I recently discovered [Velero](https://velero.io/docs/v1.15/). I now use this to
be able to restore the entire `z2m-data` persistent volume on a cron-timed basis
to my MinIO S3-compatible data store. I won't include all of the details of how
to install Velero, or schedule backups, or how to restore, leave alone how to set
up MinIO. If you have the drive to do all that, you can make good use of it for
more deployments than just Zigbee2MQTT. Once configured, this is literally all
you need to add as an annotation:

`backup.velero.io/backup-volumes: z2m-data`

In context, it looks like this:

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zigbee2mqtt
spec:
  template:
    metadata:
      annotations:
        backup.velero.io/backup-volumes: z2m-data
```

The beauty of this is that you can restore to that point in time very easily. In
fact, it's much easier to restore with Velero than using the `kubectl cp` method
shared above. It's just a lot more work and such to install MinIO or other S3. The
Velero install itself is stupid simple.
