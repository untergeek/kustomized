# Deploy metallb "pools" in Kubernetes using Kustomize

This means `IPAddressPool`s, `L2Advertisement`s, and `BGPAdvertisement`s.

This does NOT install or configure `metallb`.

## Requirements

* A functional k8s cluster.
* [metallb](https://metallb.io) _must_ already be installed. There are a lot of
  configuration knobs and dials to turn with an installation. This project isn't
  for deploying and configuring `metallb`, but rather the IP pools and advertisements.

## Deployment (without `overlays`)

**NOTE:** Do **not** change the namespace from `metallb-system`. I believe this
is a requirement for `metallb` itself.

### Update `kustomization.yaml` with changes

#### `IPAddressPool` section

* The base `IPAddressPool` name is `pool`. To have more than one pool, you
  will need to rename it. You will probably want to do so anyway. Choose the
  new name in the `/metadata/name` `rename` `op`.
* The IP address pool itself is a single line from an array in the `base`
  definition. You can add more here if you like, but the default is to replace
  just the one entry. Feel free to use CIDR notation, as the `AvoidBuggyIPs`
  flag is set to true (which will avoid `.0` and `.255` addresses). Replace
  `/spec/addresses/0` `value` with your IP range.

#### `L2Advertisement`

* Edit `patches/advertisement.yaml` and put in whatever changes you need. The
  existing example sets an ethernet interface and 3 nodes. There are many more
  configuration options available to you. Do not change the `/metadata/name` in
  this patch! The renames will take place later in `kustomization.yaml` as the
  patch file must match the name. Yes, we could set `target` and get around that.
  It's problematic to troubleshoot this sometimes as folks don't know what they've
  done, so we keep it fairly pure here.
* Edit the `value` for `/metadata/name`. This is the `rename` `op` for the this
  `L2Advertisement`.
* Edit the `value` for `/spec/ipAddressPools/0`. This value should match what you
  renamed `/metadata/name` to in the `IPAddressPool` section.

#### `BGPAdvertisement`

This one requires replacing the `L2Advertisement` section with the commented out
`BGPAdvertisement` section. Everything else should remain similar to the previous
instructions.

**NOTE:** This also requires commenting out the `l2` line and uncommenting the
`bgp` one:

```
  # - base/advertisements/l2/advertisement.yaml
  - base/advertisements/bgp/advertisement.yaml
```

* Edit `patches/advertisement.yaml` and put in whatever changes you need. The
  existing example sets an ethernet interface and 3 nodes. There are many more
  configuration options available to you. Do not change the `/metadata/name` in
  this patch! The renames will take place later in `kustomization.yaml` as the
  patch file must match the name. Yes, we could set `target` and get around that.
  It's problematic to troubleshoot this sometimes as folks don't know what they've
  done, so we keep it fairly pure here.
* Edit the `value` for `/metadata/name`. This is the `rename` `op` for the this
  `BGPAdvertisement`.
* Edit the `value` for `/spec/ipAddressPools/0`. This value should match what you
  renamed `/metadata/name` to in the `IPAddressPool` section.

#### Both L2 and BGP at the same time

As `metallb` themselves tell you, you can use both advertisement types at once,
for the same `IPAddressPool`. To do so requires both lines will need to be
uncommented:

```
  - base/advertisements/l2/advertisement.yaml
  - base/advertisements/bgp/advertisement.yaml
```

You will also need to copy `patches/advertisement.yaml` to another file, e.g.
`patches/bgp_advertisement.yaml`. So long as you can distinguish between the two,
that's all that matters.

You would then need to uncomment both sections in `kustomization.yaml`, and ensure
that the correct `patches/FILENAME.yaml` patches the correct advertisement type.

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
* For complex deployments, this could look like `overlays/GROUP/SUBGROUP`, where
  `GROUP` could be `teamname`, and `SUBGROUP` could be `prod`, `qa`, or `dev`.
  This allows fine granular control over which blocks of IPs are allocated to
  which teams and environments.
* Each `NAME` or `SUBGROUP` should have its own `kustomization.yaml` and `patches`
  subdirectory.
* Be sure to update the `resources` section of `kustomization.yaml` to be able to
reach the `base` directory. 

For `NAME` (one level)

```
resources:
  - ../../base
  - ../../base/advertisements/l2
```

For `GROUP/SUBGROUP` (two levels)

```
resources:
  - ../../../base
  - ../../../base/advertisements/l2
```

### Edit each `overlay` path accordingly

Apply the same configuration steps as above for each `overlay` path you create.

### Test

* (Optional) To preview generated configuration before deploying:

`kubectl kustomize overlays/NAME`

or

`kubectl kustomize overlays/GROUP/SUBGROUP`

### Apply

* Run the following command to build and deploy:

`kubectl apply -k overlays/NAME`

or

`kubectl apply -k overlays/GROUP/SUBGROUP`
