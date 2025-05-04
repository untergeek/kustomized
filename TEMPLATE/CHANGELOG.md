# CHANGELOG

## 2025-04-24

Initial Creation here: A StatefulSet template for simple apps, typically with a
single port, an ingress, a storage volume, environment variables, possibly a
configmap, a loadbalancer, TLS for the ingress.

This has been created to be as agnostic as possible. Names for the various bits
can be assigned in `settings/name.properties`. Most of the other settings are
are configured in `settings/server.properties`. Features and add-ons are enabled
in `kustomization.yaml` via `components`. It has been designed to be as modular
as possible. If at some future point you wanted to add a feature that isn't included
yet, the pattern here should make it easy to follow and do likewise. The big deal
here is the configuration files and the components. Knowing how this works should
make it relatively easy to drop a container name into `settings/server.properties`,
configure a few other values in the same file, and uncomment a few components in
`kustomization.yaml`. Create a namespace, hit `kustomize build /path/to/bundle`
and you should see some magic. If you don't change the names in
`settings/name.properties` they will be the same in every place you use this. But
who cares? If they're in their own different namespaces, then there should be no
problem.

Supposing you wanted to put multiple items in the same namespace? You can...if you
use `settings/name.properties` to rename everything so it won't collide. Building
and running it all will just take multiple Kustomization runs. You could potentially
concatenate them together into one huge manifest, but why complicate things.
