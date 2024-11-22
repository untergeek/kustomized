# CHANGELOG

## 2024-11-22

Moved a lot of stuff around and started using `components` to add functionality
for TLS in ingress, as well as basic_auth.

Added means to configure via configuration files in `settings/*`. New files include

* `settings/apm.properties`
* `settings/auth`
* `settings/z2m.properties`

`.env.secret` has been renamed to `settings/.env.secrets`

Switched to using `replacements` as much as possible.

If you use a container image that supports Elastic's APM, then you can enable
this functionality by way of configuring `settings/apm.properties` and enabling
the APM path via `components`. This is provided in `kustomization.yaml` but
commented out. At this time, the only supporting image is at
`untergeek/zigbee2mqtt:1.41.0-apm`, in which the only changes from the out of the
box image of the same version are as follows:

1. The top 5 lines of `index.js` are now:

```javascript
require('elastic-apm-node').start({
  active: process.env.NODE_ENV === 'production'
})

const semver = require('semver');
```

2. `npm install elastic-apm-node --save && \` was injected at line 13 of
`docker/Dockerfile` such that this particular `RUN` command now shows:

```Dockerfile
RUN apk add --no-cache --virtual .buildtools make gcc g++ python3 linux-headers git npm && \
    npm install elastic-apm-node --save && \
    npm ci --production --no-audit --no-optional --no-update-notifier && \
    # Serialport needs to be rebuild for Alpine https://serialport.io/docs/9.x.x/guide-installation#alpine-linux
    npm rebuild --build-from-source && \
    apk del .buildtools
```

3. The original line 40 of the Dockerfile has been commented out and an explanatory line
added, such that it now reads:

```Dockerfile
# Set this in your Container definition, not here
# ENV NODE_ENV production
```

The reason being that `NODE_ENV` should not be hard coded, in my opinion.

If `NODE_ENV` is not set to `production`, zigbee2mqtt will not attempt to send
APM data to the configured URL.

The resulting container image was published with this command:

```bash
docker buildx build \
   --file ./docker/Dockerfile \
   --platform linux/amd64 \
   --tag untergeek/zigbee2mqtt:1.41.0-apm \
   --build-arg VERSION=1.41.0-apm \
   --build-arg COMMIT=acd5932c \
   --push \
   .
```

## 2024-09-20

Added `mount_pvc.yaml` for aiding in backup/restore of data from the persistent
volume claim mounted at `/app/data`.

Added rudimentary instructions to the README describing how to backup your data,
and how to restore data using another pod.

## 2024-09-19

Initial release.
