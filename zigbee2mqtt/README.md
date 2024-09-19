# Deploy zigbee2mqtt in Kubernetes using Kustomize

## Requirements

* A functional k8s cluster.
* An Ingress provider (I use `ingress-nginx`, not to be confused with `nginx-ingress`)
* Secrets defined in `.env.secret`:
  * `user=zigbee2mqtt`
    * `user` the username to connect to your MQTT broker
  * `password=CHANGEME`
    * `password` the password for `user` on your MQTT broker
  * `network_key='[0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15]'`
    * `network_key` is an encryption key for your Zigbee network. You really
      shouldn't operate without it. It is a series of 16 digits. Each value can
      be between 0 and 15. You can randomly set each individual value to whatever
      you like. I do not recommend using all one number, or the sequence above.
      Please note that the array is encapsulated in single quotes. This is
      because it must be a string value to become a Secret. Don't worry, it will
      be properly rendered in `secret.yaml` by our lovely initContainer.

### Optional

* A TLS secret suitable for an Ingress route. It will work the same way (mine does).
  This will enable you to do MQTTS. If you plan on securing your zigbee2mqtt
  frontend with TLS encryption, you need to either create a TLS secret before
  hand, or modify `patches/ingress/settings.yaml` to do the Let's Encrypt part
  for you automatically.

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

#### `nodeName`

This setting is documented inline in `kustomization.yaml`, but it bears further
explanation here.

[`nodeName`](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodename)
will pin the pod to run only on the k8s node where your USB Zigbee controller is
inserted. This is only needed if you are using a USB dongle! For network-based
controllers, like the [UZG-01](https://uzg.zig-star.com), this setting is
unnecessary.

The `nodeName` setting should be indented at the same level as the term `containers`, e.g.

```
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
will get really, really long, though.

#### `volumeMount` and `volumes` sections

If you are using a USB-based Zigbee device, you will need to uncomment and configure
these lines (i.e., set `/dev/ttyACM0` to wherever your device is in `/dev`).

```
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

```
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

```
namespace: zigbee2mqtt
```

Because we are using a StatefulSet here, you should create the namespace manually
before applying. Having a namespace automatically be created and destroyed would
defeat the retention of the persistent volume claim we want (and need) because
deleting/destroying a namespace triggers the deletion of everything in it, which
would include our PVC we want to retain.

#### `commonLabels`

This will apply the app label `zigbee2mqtt` to everything being created.

```
commonLabels:
  app: zigbee2mqtt
```

#### Images

This section allows us to apply a new image to upgrade our StatefulSet without
having to edit the `base` files.

```
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

```
## Patches
patches:
# ConfigMap settings:
- patch:
  path: patches/configmap/configuration.yaml
# StatefulSet settings:
- patch:
  path: patches/statefulset/settings.yaml
# Ingress settings:
- patch: |-
    - op: replace
      path: "/spec/rules/0/host"
      value: "Z2M.EXAMPLE.COM"
  target:
    kind: Ingress
    name: ingress
- patch:
  path: patches/ingress/settings.yaml
```

The Ingress settings are in two pieces. I did this for convenience, really.

If you do not plan on using a TLS certificate to secure your frontend, then
you only need the `- op:` type patch in `kustomization.yaml`.

```
# Ingress settings:
- patch: |-
    - op: replace
      path: "/spec/rules/0/host"
      value: "Z2M.EXAMPLE.COM"
  target:
    kind: Ingress
    name: ingress
- patch:
  path: patches/ingress/settings.yaml
```

Simply replace `Z2M.EXAMPLE.COM` with what you plan to host your zigbee2mqtt
frontend at.

If you are going to use TLS, then you should also edit `patches/ingress/settings.yaml`:

```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
spec:
  tls:
    - hosts:
      - "Z2M.EXAMPLE.COM"
      secretName: MY_TLS_SECRET
```


Again, replace `Z2M.EXAMPLE.COM` with what you plan to host your zigbee2mqtt
frontend at.

Replace `MY_TLS_SECRET` with the name of your TLS secret (must be in same
namespace).

Now, you could just as easily do both patches in one step, if you like. Just
remove or comment out the `- op: replace` if you put it in the patch file.

That would look like this in `patches/ingress/settings.yaml`:

```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
spec:
  rules:
    - host: Z2M.EXAMPLE.COM
  tls:
    - hosts:
      - "Z2M.EXAMPLE.COM"
      secretName: MY_TLS_SECRET
```

and require these changes in `kustomization.yaml` (just comment the inline patch):

```
# Ingress settings:
# - patch: |-
#     - op: replace
#       path: "/spec/rules/0/host"
#       value: "Z2M.EXAMPLE.COM"
#   target:
#     kind: Ingress
#     name: ingress
- patch:
  path: patches/ingress/settings.yaml
```

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

#### `patches/ingress/settings.yaml`

Be sure to use a unique domain name for every instance you deploy.

### Test

* (Optional) To preview generated configuration before deploying:

  `kubectl kustomize overlays/NAME`

### Apply

* Run the following command to build and deploy:

  `kubectl apply -k overlays/NAME`
