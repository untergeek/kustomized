# Pi-hole Kubernetes HA

## It finally works!

Credit for the initial build goes to [Heracles31 in the PiHole Discourse
forums](https://discourse.pi-hole.net/t/pi-hole-high-availability-with-kubernetes/67505/4).

I needed to fork it and make it my own after that. It took me the better part of
two evenings to unravel this, but here it is:

**High-Availability Pi-Hole on Kubernetes** as a Kustomize deployable!

Well, the one thing that can't be HA is the WebUI. That would introduce write
contention, and we don't want that.

A few things I modified/changed/added:

* `initContainers` defined for the read-only pods so they won't try to mount the
  same volume as the read-write pod *while* that pod is still initializing.
* Added a `startupProbe` for both kinds of pods so that they'd come online sooner.
* `LoadBalancer IP` sharing. This requires you to know what's available in your
  pool, but makes things neater.
* Web server session affinity. I quickly discovered I could not log into the UI
  without this addition, even with only 1 pod hosting port 80.
* Put the initial web password in `.env.secret`
* What I *hope* is an easy to read, follow, patch, and deploy bundle.


## Assumptions

1. You have created the namespace to be used already. In `kustomization.yaml`,
   it defaults to `pihole-ha`, but you can change that if you prefer.
2. You have installed something that gives you `LoadBalancer`. For bare-metal
   deployers, this is usually metallb.
3. You have installed an Ingress, ostensibly Google's `ingress-nginx`, as there
   are annotations that depend on it.
4. You have a `StorageClass` that can provide a `PersistentVolumeClaim` with
   `ReadWriteMany` (`RWX`) capability. **This is absolutely necessary.** The
   pods will simply fail to start without this. [Longhorn](https://longhorn.io)
   is easy to install and can provide this capacity using local storage on your
   nodes (i.e., no extra drives necessary).
5. You have `HugePages` enabled. I found quickly that Pihole wanted HugePages or
   the UI would simply not work. There's a post somewhere in the Discourse forums
   explaining how to get that to work.

## Usage

Like all other Kustomize deployments:

```
kubectl apply -k kustomized/pihole-ha
```

## Configuration

The options *I* needed are in the `patches` subdirectory. Feel free to edit and
add as much as needed for your needs.

I will go through the options I've sanitized in order:

### `patches/deployments/read-*`

* `<LOADBALANCER_IP>` should be the IP you will be assigning to this deployment
* `<PIHOLE-UI.EXAMPLE.COM>` should be the FQDN that hosts the UI
* `<EXAMPLE.COM>` should be just the domain.tld part. This is so hosts from the same domain can more easily browse.
* `<ADD LOCAL or UPSTREAM DNS or comment these lines>` If you have your own local DNS and want those to resolve, add them here, separated by semicolons `;`
* Feel free to add any other environment variables you may need.

### `patches/services/*.yaml`

* `<LOADBALANCER_IP>` should be the IP you will be assigning to this deployment.

This is actually where it gets pinned and assigned. The `metallb.universe.tf/allow-shared-ip: key-to-share-<LOADBALANCER_IP>` part is what allows you to use the same IP for both the TCP and UDP ports.

### `patches/volumes/02-mymasq.conf`

* Replace `NS?.EXAMPLE.COM` and their associated IP addresses as desired
* Or comment out those lines.
* Add any other `dnsmasq` settings you may want to use here.

### Patches included `kustomization.yaml`

#### Images

```
images:
  - name: pihole/pihole
    newTag: 2024.07.0
```

Replace `2024.07.0` with the next Pihole image tag that comes out.

#### Volume Settings

```
value: "<YOUR STORAGECLASS NAME>"
```

Replace this with whatever `StorageClassName` you have that supports `ReadWriteMany`
PVC claims. Also feel free to re-size the volumes.


- `pvc-dnsmasq` is for `/etc/dnsmasq.d` which will probably never use the `10Mi` assigned.
- `pvc-etc` is for `/etc/pihole`, which contains all of the data files for Pihole.
  It will take 20Mi almost out of the box, so this defaults to `100Mi`, but you
  may want more or less.

#### Service Settings

This is where the files in `patches/services` get applied.

#### Ingress Settings

* `<PIHOLE-UI.EXAMPLE.COM>` should be the FQDN that hosts the UI, and match what
  was used in `patches/deployments/read-*`
* `<YOUR_TLS_CERT_SECRET>` should be a TLS secret that works for `<PIHOLE-UI.EXAMPLE.COM>`

#### Deployment Settings (for both `read-only` and `read-write`)

* `<YOUR TIMEZONE>` should be a canonical time zone, like `America/Chicago`.

## The finished product

```
kubectl -n pihole-ha get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/pihole-ro-847bff778f-9stth   1/1     Running   0          69s
pod/pihole-ro-847bff778f-qxqw4   1/1     Running   0          69s
pod/pihole-ro-847bff778f-tsgws   1/1     Running   0          69s
pod/pihole-rw-557d57865c-mgt79   1/1     Running   0          69s

NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                     AGE
service/pihole-tcp   LoadBalancer   10.103.243.218 <LOADBALANCER_IP>   53:31288/TCP,80:31814/TCP   69s
service/pihole-udp   LoadBalancer   10.111.234.174 <LOADBALANCER_IP>   53:31150/UDP                69s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pihole-ro   3/3     3            3           69s
deployment.apps/pihole-rw   1/1     1            1           69s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/pihole-ro-847bff778f   3         3         3       69s
replicaset.apps/pihole-rw-557d57865c   1         1         1       69s
```

## Errata

- Due to the load-balanced nature of this deployment strategy, it was necessary
  to add these `annotations` to the ingress. They permit the authentication to
  properly process and pin, otherwise you get a doom loop.

  ```
  nginx.ingress.kubernetes.io/affinity: cookie
  nginx.ingress.kubernetes.io/affinity-mode: persistent 
  ```

- The `FTLCONF_MAXDBDAYS` and `FTLCONF_MAXNETAGE` options are omitted from
  `base/deployments/read-only/deployment.yaml`. This is because the local databases
  cannot be cleaned on the read-only system. Leaving them set at the default (365
  days) means they will never attempt to clean up.