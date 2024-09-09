# Pi-hole via Kubernetes Kustomize

This is hopefully a quick way to get a fully functional Pihole instance running.

## Assumptions

1. You have installed something that gives you `LoadBalancer`. For bare-metal
   deployers, this is usually `metallb`.
2. You have installed an `Ingress` controller, ostensibly Google's `ingress-nginx`.
3. You've created a namespace already (default is `pihole`).

Why don't I include namespace creation here? Chiefly because I often have TLS
secrets in the namespace that I don't want to have to keep recreating or copying
every time I want to blast the deployment away and test again. I know. I'll
eventually get around to using service accounts and/or secret mirroring.

## Usage

### Installation

Like any Kustomize deployments:

```
kubectl apply -k kustomized/pihole
```

### Removal

Like any Kustomize deployments:

```
kubectl delete -k kustomized/pihole
```


## Configuration

The options *I* needed are in the `patches` subdirectory. Feel free to edit and
add as much as needed for your needs.

I will go through the options I've sanitized in order:

### `patches/deployment.yaml`

* `<LOADBALANCER_IP>` should be the IP you will be assigning to this deployment
* `<PIHOLE-UI.EXAMPLE.COM>` should be the FQDN that hosts the UI
* `<EXAMPLE.COM>` should be just the domain.tld part. This is so hosts from the same domain can more easily browse.
* `<ADD LOCAL or UPSTREAM DNS or comment these lines>` If you have your own local DNS and want those to resolve, add them here, separated by semicolons `;`
* Feel free to add any other environment variables you may need.

### `patches/services/*.yaml`

* `<LOADBALANCER_IP>` should be the IP you will be assigning to this deployment.

This is actually where it gets pinned and assigned. The `metallb.universe.tf/allow-shared-ip: key-to-share-<LOADBALANCER_IP>` part is what allows you to use the same IP for both the TCP and UDP ports.

### `patches/volumes/02-mymasq.conf`

* Add any `dnsmasq` settings you may want to use here.

### Patches included `kustomization.yaml`

* Be sure to set the `namespace` you choose here. It's set to `pihole` by default.

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

Replace this with whatever `StorageClassName` you are using for PVC claims. Also
feel free to re-size the volumes. I doubt you'll need more than a mb or two for
`pvc-dnsmasq`, as it only holds 1 file.


- `pvc-dnsmasq` is for `/etc/dnsmasq.d` which will probably never use the `10Mi` assigned.
- `pvc-etc` is for `/etc/pihole`, which contains all of the data files for Pihole.
  It will take 20Mi almost out of the box, so this defaults to `100Mi`, but you
  may want more or less.

#### Service Settings

This is where the files in `patches/services` get applied.

#### Ingress Settings

* `<PIHOLE-UI.EXAMPLE.COM>` should be the FQDN that hosts the UI, and match what
  was used in `patches/deployment.yaml`
* `<YOUR_TLS_CERT_SECRET>` should be a TLS secret that works for `<PIHOLE-UI.EXAMPLE.COM>`

#### Deployment Settings

* `<YOUR TIMEZONE>` should be a canonical time zone, like `America/Chicago`.

## The finished product

`<LOD_BAL_IP>` for you will be the `<LOADBALANCER_IP>` set in your patches.

```
kubectl -n pihole get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/pihole-5499f67b55-4nkrz   1/1     Running   0          25m

NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                     AGE
service/pihole-tcp   LoadBalancer   10.100.248.8    <LOD_BAL_IP>   53:32681/TCP,80:31202/TCP   25m
service/pihole-udp   LoadBalancer   10.102.16.101   <LOD_BAL_IP>   53:30955/UDP                25m

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pihole   1/1     1            1           25m

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/pihole-5499f67b55   1         1         1       25m
```

Whatever you set as `<PIHOLE-UI.EXAMPLE.COM>` will show up under `HOSTS`:

```
kubectl -n pihole get ingress
NAME             CLASS   HOSTS                      ADDRESS       PORTS     AGE
pihole-ingress   nginx   <PIHOLE-UI.EXAMPLE.COM>    <INGRESS_IP>  80, 443   25m
```