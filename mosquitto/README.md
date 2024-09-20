# Deploy Mosquitto MQTT broker in Kubernetes using Kustomize

## Requirements

* A functional k8s cluster.
* [metallb](https://metallb.io) _must_ already be installed. You will want this,
  trust me.

### Optional

* A TLS secret suitable for an Ingress route. It will work the same way (mine does).
  This will enable you to do MQTTS.
* Super-secret bonus level: Create a second instance using `overlays` in a
  different subnet and bridge the two together. I do this so that my Home Assistant
  can reside in my regular subnet, but my IoT devices and such can write to an
  MQTT broker in their own, separate subnet, but Home Assistant can see those
  events as though they were in the same subnet, or the same broker.

## Deployment (without `overlays`)

### Edit `patches/configmap/*.yaml`

The following ConfigMaps need to be updated to suit your needs.

#### `mosquitto.yaml`

```
data:
  mosquitto.conf: |
    listener 1883 0.0.0.0
    protocol mqtt
    log_dest stdout

    persistence true
    persistence_file persistence.db
    persistence_location /mosquitto/data

    password_file /mosquitto/config/password.txt
    acl_file /mosquitto/config/acl.txt
```

You can add/remove anything you want from this. Suggested bits are commented at
the bottom, such as MQTTS and certificates:

```
## Add any configuration changes as appropriate. If you plan on running MQTTS, add the following:
#    listener 1883 0.0.0.0
#    cafile /etc/ssl/certs/ca-certificates.crt
#    keyfile /mosquitto/certs/tls.key
#    certfile /mosquitto/certs/tls.crt
## Edit & uncomments sections of kustomization.yaml and the associated patches:
# patches/statefulset/settings.yaml: Uncomment and configure the secret
```

As stated, you will need to enable MQTTS in the `kustomization.yaml` file by
uncommenting the mentioned patch, and configuring the TLS secret.

#### `password.yaml`

This ConfigMap should contain the lines extracted from a file created by `mosquitto_passwd`.

```
data:
  password.txt: |
    mosquitto:$7$101$8aeh1N+nM3x+Rxne$kcBYBWXXZckiYa0IF23kEf6P6mLMN2NlGBBOjZ2RFCE8IsliMxW/ZaMk3tmR35tf0u5Hj430z3ehYHI2fHDdnw==
# this is just mosquitto:mosquitto

# Add/remove any password lines as appropriate. Generate a file using mosquitto_passwd
```

This mirrors the `base/configmap/acl.yaml` default. You should replace the provided
`mosquitto:` line with your own password contents, preserving the proper indentation.

#### `acl.yaml`

This ACL file should also be replaced with your own configuration. You probably
need to add the same users you added to the password ConfigMap here, with the
appropriate topic permissions per user.

```
data:
  acl.txt: |
    topic read $SYS/#

    # This affects all clients.
    pattern write $SYS/broker/connection/%c/state

    user mosquitto
    topic readwrite #

# Add any acl additions/changes as appropriate
```

### Update `kustomization.yaml` with changes

#### `volumeClaimTemplates` section

```
## volumeClaimTemplates settings
# - patch: |-
#     - op: add
#       path: /spec/volumeClaimTemplates/0/spec/storageClassName
#       value: MYSTORAGECLASS
#     - op: replace
#       path: /spec/volumeClaimTemplates/0/spec/resources/requests/storage
#       value: 1Gi  # This is already the default
#   target:
#     kind: StatefulSet
#     name: mosquitto
```

In the first `- op:` block, set `MYSTORAGECLASS` if using something other than the
default, otherwise remove these lines or keep them commented.

In the second `- op:` block, set the value to something you expect to use. `1Gi`
is the default, but my use case does not have very many persistent values, so
I set this to `100Mi` in my case (and that will not likely ever fill).

#### MQTTS and TLS cert settings

If you plan on using TLS certificates, you'll need to uncomment this `- patch:` block.

```
## MQTTS port and TLS cert settings -- Needed if you plan on using MQTTS.
## Update patches/configmap/mosquitto.yaml accordingly
## Also update the MQTTS -op section in the mqtt service settings below.
# - patch:
#   path: patches/statefulset/settings.yaml
#   target:
#     kind: StatefulSet
#     name: mosquitto
```

You will also need to edit `patches/statefulset/settings.yaml` to set the `secret`:

```
secretName: MYSECRET
```

Replace `MYSECRET` with your TLS certificate secret name (should be in the same)
namespace.

#### Services

Set `MY DESIRED LOADBALANCER IP` and uncomment this `- patch:` block here:

```
### Services
## mqtt settings
# - patch: |-
#   target:
#     kind: Service
#     name: mqtt
#     - op: add
#       path: /metadata/annotations/metallb.universe.tf~1loadBalancerIPs
#       value: "MY DESIRED LOADBALANCER IP"
```

If you plan on using TLS certificates and MQTTS, you'll need to uncomment this
`- op:` block.

```
##### Only uncomment the following -op section if you're planning on using MQTTS as well.
##### Update patches/configmap/mosquitto.yaml and patches/statefulset/settings.yaml accordingly
#     - op: add
#       path: /spec/ports/-
#       value:
#         name: mqtts
#         port: 8883
#         targetPort: 8883
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

### Test

* (Optional) To preview generated configuration before deploying:

  `kubectl kustomize overlays/NAME`

### Apply

* Run the following command to build and deploy:

  `kubectl apply -k overlays/NAME`

## Bonus: Bridging with a second instance via `overlay`

Create a second instance in `overlay` with the full `patches` directory and
`kustomization.yaml` file. Name this path bridge, or iot, or whatever you will.

### ConfigMaps in `overlay/OTHER`

#### `mosquitto.yaml`

Add these bits after the regular configuration options:

```
    # Bridge

    connection iot-bridge
    address mqtt.mosquitto.svc.cluster.local:1883
    topic # both 2
    bridge_insecure true
    cleansession false
    notifications false
    remote_clientid mosquitto-iot
    remote_password CHANGEME
    remote_username bridger
    start_type automatic
    try_private true

# This mosquitto.conf has a "bridge" definition. In other words, in addition to being its
# own entity, it bridges topics between the instances. In our case, `topic # both 2` means
# we are effectively mirroring all topics between the two. The `remote_username` and the
# `remote_password` must be in existence in the `password.yaml` ConfigMap on the leader.
#
# The `address` in the Bridge section must be properly defined in order to work.
#
# address mqtt.NAMESPACE.svc.cluster.local:1883
# 
# where `NAMESPACE` is the namespace where the leader instance is running
#
# If we had a leader mosquitto instance in namespace `mymosquitto`, then this line would be:
#
# address mqtt.mymosquitto.svc.cluster.local:1883
```

Hopefully this makes sense. This portion must be added after the other settings
in your `mosquitto.yaml` ConfigMap definition. Make changes as needed.

#### Don't forget the other bits

* This other instance _must_ be in a different `NAMESPACE` to avoid pod/service/etc
  collisions. Be sure to set that in `kustomization.yaml`.
* If you're doing MQTTS in this instance, don't forget to set the appropriate
  settings.
* If it's in a different subnet, make sure you have an IP available in `metallb`.

### Test

* (Optional) To preview generated configuration before deploying:

  `kubectl kustomize overlays/OTHER`

### Apply

* Run the following command to build and deploy:

  `kubectl apply -k overlays/OTHER`
