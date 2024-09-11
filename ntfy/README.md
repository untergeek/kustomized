# Deploy ntfy in Kubernetes using Kustomize

Have a Kubernetes cluster and want to run your own [ntfy](https://ntfy.sh/) server?

Hopefully this makes it simple for you!

## Requirements:

* A functional k8s cluster.
* An Ingress provider (I use `ingress-nginx`, not to be confused with `nginx-ingress`)
* [OPTIONAL] If you plan on securing your ntfy server with TLS encryption, you
  need to either create a TLS secret before hand, or modify `patches/ingress.yaml`
  to do the Let's Encrypt part for you automatically.

## Configuration:

### `kustomization.yaml`:

#### Namespace:

You can use the default namespace, `ntfy`, or change to use your own.

For example, to use the namespace `myntfy`, edit and replace `ntfy` here:

```
namespace: ntfy
```

so it becomes:

```
namespace: myntfy
```

You will also need to uncomment `- base/namespace.yaml` in the `resources` section:

```
resources:
  - base
#  - base/namespace.yaml
```

will become:

```
resources:
  - base
  - base/namespace.yaml
```

And then edit the `Namespace setting` section and replace `NEW VALUE`
in `patches:` so that

```
Namespace setting
- patch: |-
    - op: replace
      path: "/metadata/name"
      value: "NEW VALUE"
  target:
    kind: Namespace
    name: ntfy
```

replace `NEW VALUE` with `myntfy`:

```
      value: "myntfy"
```

Note that this section is commented by default. You will need to uncomment it to
change the namespace.

#### Images:

Currently, the most recent release is `v2.11.0`. To run a different release, replace
`v2.11.0` with the desired tag.

```
images:
  - name: binwiederhier/ntfy
    newTag: v2.11.0
```

#### Ingress settings:

Replace `<NTFY.EXAMPLE.COM>` with the FQDN you plan to use.

```
# Ingress settings:
- patch: |-
    - op: replace
      path: "/spec/rules/0/host"
      value: "<NTFY.EXAMPLE.COM>"
  target:
    kind: Ingress
    name: ingress
```

### Patch files:

#### `patches/deployment.yaml`:

Update/Add any environment variables you may need. You can also update the resource
settings.

#### `patches/ingress.yaml`:

To use TLS to secure your traffic, replace `NTFY.EXAMPLE.COM` with the FQDN you
plan to use.

You should also have created a TLS secret in advance that matches the FQDN. Replace
`<MY_TLS_SECRET>` with the name of that TLS secret. It should be in the same
namespace, e.g. `ntfy`, unless you've gone to the trouble to create a service
account and can share secrets across namespaces.


```
  tls:
    - hosts:
      - "NTFY.EXAMPLE.COM"
      secretName: <MY_TLS_SECRET>
```

### `patches/server.yml`:

This file will be used to create a ConfigMap that will be mounted as
`/etc/ntfy/server.yml`, and contains the settings used to configure ntfy. At the
very least, you should edit `patches/server.yml` and configure:

```
base_url: https://ntfy.example.com/
```

which should be the FQDN you set in your Ingress.

Any other settings are at your discretion.
