# Changelog

## 2024-09-07

### Fix ConfigMap naming in `base/deployments/read-*/deployment.yaml`

The filename and ConfigMap key were both supposed to be `02-mymasq.conf`, rather
than `02-localdns.conf`. This has been corrected.

### Reordered entries in `patches/deployments/read-*/*.yaml`

Now they start the same, and the `read-write` patch adds all extra env vars after.

### Set `DNSMASQ_LISTENING` to `single`

The `all` setting was not assigning `interface=eth0` to `/etc/pihole.conf`, which
appears to be necessary for `gravity` to filter addresses. Everything seemed to
be making it through.

### Added `FTLCONF_IGNORE_LOCALHOST`

Without `FTLCONF_IGNORE_LOCALHOST` set to `yes`, the Pihole query log is flooded
with `dig` requests to `@127.0.0.1` (localhost) to resolve `cloudflare.com` every
5 seconds