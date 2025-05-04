# Deploy Pi-hole in Kubernetes using Kustomize

This is hopefully a quick way to get a fully functional Pi-hole instance running.

## Requirements 

* A functional k8s cluster.
* A dedicated namespace for this project (default is `pihole`).
* You can create a service using `LoadBalancer`. For bare-metal deployers, this is
  usually [metallb](https://metallb.io). This is the assumed provider.
* An Ingress provider (I use `ingress-nginx`, not to be confused with `nginx-ingress`)
* Secrets defined in `.env.secret`:
  * `password=changemelikerightnow`
    * The admin `password` for the Pi-hole UI.

### Optional

* A TLS secret suitable for an Ingress route. If you plan on securing your Pi-hole
  server with TLS encryption, you need to either create a TLS secret before hand,
  or modify `patches/ingress/settings.yaml` to do the Let's Encrypt part for you
  automatically.

## Deployment (without `overlays`)

The options *I* needed are in the `patches` subdirectory. Feel free to edit and
add as much as needed for your needs.

I will go through the options I've sanitized in order:

### `patches/statefulset/settings.yaml`

* `<LOADBALANCER_IP>` should be the IP you will be assigning to this deployment
* `<PIHOLE-UI.EXAMPLE.COM>` should be the FQDN that hosts the UI
* `<EXAMPLE.COM>` should be just the domain.tld part. This is so hosts from the
  same domain can more easily browse.
* `<ADD LOCAL or UPSTREAM DNS or comment these lines>` If you have your own local
  DNS and want those to resolve, add them here, separated by semicolons `;`
* Be sure to set the `storageClassName` if needed, and set the desired storage
  size in the `volumeClaimTemplates` section at the bottom.
* Feel free to add any other environment variables you may need.

### `patches/service/*/settings.yaml`

* `<LOADBALANCER_IP>` should be the IP you will be assigning to this deployment.

This is actually where it gets pinned and assigned. If you are using `metallb`,
the `metallb.universe.tf/allow-shared-ip: key-to-share-<LOADBALANCER_IP>` part
is what allows you to use the same IP for both the TCP and UDP ports.

### `patches/configmap/02-mymasq.conf`

* Add any `dnsmasq` settings you may want to use here.

### Patches in `kustomization.yaml`

* Be sure to set the `namespace` you choose here. It's set to `pihole` by default.

#### Images

```
images:
  - name: pihole/pihole
    newTag: 2024.07.0
```

Replace `2024.07.0` with the next Pihole image tag that comes out.

#### StatefulSet Settings

This is where the generated secret replaces the one in the `base` manifest.

This is also where the settings in `patches/statefulset/settings.yaml` are applied.

#### Service Settings

This is where the files in `patches/service/*/settings.yaml` get applied.

#### Ingress Settings

* `<PIHOLE-UI.EXAMPLE.COM>` should be the FQDN that hosts the UI, and match what
  was used in `patches/statefulset/settings.yaml`

If you plan to use TLS to secure your ingress, be sure to uncomment these lines:

```
# - patch:
#   path: patches/ingress/settings.yaml
```

and edit `patches/ingress/settings.yaml` to set `<YOUR_TLS_CERT_SECRET>` and
`<PIHOLE-UI.EXAMPLE.COM>`.

### The finished product

`<LOD_BAL_IP>` for you will be the `<LOADBALANCER_IP>` set in your patches.

```
kubectl -n pihole get all
NAME           READY   STATUS    RESTARTS   AGE
pod/pihole-0   1/1     Running   0          25m

NAME              TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                     AGE
service/tcp-svc   LoadBalancer   10.100.248.8    <LOD_BAL_IP>   53:32681/TCP,80:31202/TCP   25m
service/udp-svc   LoadBalancer   10.102.16.101   <LOD_BAL_IP>   53:30955/UDP                25m

NAME                      READY   AGE
statefulset.apps/pihole   1/1     25m
```

Whatever you set as `<PIHOLE-UI.EXAMPLE.COM>` will show up under `HOSTS`:

```
kubectl -n pihole get ingress
NAME      CLASS   HOSTS                      ADDRESS       PORTS     AGE
ingress   nginx   <PIHOLE-UI.EXAMPLE.COM>    <INGRESS_IP>  80, 443   25m
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

## Follow-up items

### Change the preset Admin password

This isn't terribly hard to do, thankfully.

```
$ kubectl -n NAMESPACE exec CONTAINERNAME-0 -ti -- pihole -a -p
```

In our case, if you're using the default namespace, it's `pihole`, and unless
you decided to go all out and edit the `base` StatefulSet manifest file, the
default container name is also `pihole`. 

The final part `pihole -a -p` is the command to change the password, which I have
personally tested and verified works even when called by `kubectl`:

```
$ kubectl -n pihole exec pihole-0 -ti -- pihole -a -p
Enter New Password (Blank for no password):
Confirm Password:
  [✓] New password set
```
