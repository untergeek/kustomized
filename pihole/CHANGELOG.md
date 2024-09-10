# CHANGELOG

## 2024-09-10

### `README.md`

Updated instructions and details to reflect other changes.

### `patches/deployment.yaml`

#### Thoroughly documented all options

#### Added environment variables

- `BLOCKINGMODE`
- `CUSTOM_CACHE_SIZE`
- `EDNS0_ECS`
- `MOZILLA_CANARY`

All of these options are set to their respective default values.

### Removed

#### `base/volumes/masq-pvc.yaml`

Also removed the associated entries in `base/deployment.yaml`.

This project was derived from `pihole-ha`, where we wanted to share `/etc/dnsmasq.d`
with other nodes. No such volume sharing is necessary in this project, as the
ConfigMap for `02-mymasq.conf` can be mounted as its own file.