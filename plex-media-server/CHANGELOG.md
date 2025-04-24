# CHANGELOG

## 2025-04-23

This is fully functional, so I'm releasing it in BETA.

Edit the settings in `settings/server.properties` to suit. At this time, this 
project assumes the end user has their media in an NFS fileshare.  You're welcome
to add other patches to this to accommodate something other than NFS. Pull requests
accepted.

You can enable or disable TLS support (you'll need Cert Manager running in your
cluster, as that is how TLS is supported in the `components` in this project) by
uncommenting the appropriate `component` entry and making sure the configuration
is accurate in your `server.properties` file.

The Nvidia hardware decoding functionality _will_ work if you provide a Docker image
that has the appropriate Nvidia libraries that match the kernel version of the node.
For me, I cloned the [pms-docker GitHub repo](https://github.com/plexinc/pms-docker)
and made a few edits to their Dockerfile. Since my Kubernetes cluster is running
on Ubuntu 22.04 at the moment, I changed the corresponding line at the very top
of the Dockerfile from `FROM ubuntu:20.04` to `FROM ubuntu:22.04`

Then, in the `apt install` section of the Dockerfile, I added the two packages
required:

```
    apt-get install -y \
      tzdata \
      curl \
      xmlstarlet \
      uuid-runtime \
      libnvidia-encode-550=550.120-0ubuntu0.22.04.1 \
      nvidia-utils-550=550.120-0ubuntu0.22.04.1 \
      unrar && \
```

These are what provide a _working_ `nvidia-smi` output (proof that the driver is
working as expected):

```
$ nvidia-smi
Thu Apr 24 00:32:56 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.120                Driver Version: 550.120        CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Quadro P620                    Off |   00000000:21:00.0 Off |                  N/A |
| 34%   38C    P8             N/A /  N/A  |       5MiB /   2048MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

If this is working on the host node, it will work in the container if the same OS
and packages/libraries are installed. You should be able to `kubectl exec` your pod
with the same command and see the same output:

```
$ kubectl -n plex exec plex-app-0 -- nvidia-smi
Wed Apr 23 18:34:49 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.120                Driver Version: 550.120        CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Quadro P620                    Off |   00000000:21:00.0 Off |                  N/A |
| 34%   38C    P8             N/A /  N/A  |       5MiB /   2048MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

If you do, then hardware transcoding will work in Plex from inside a container.
There may be other settings you have to do for your specific environment, and
I'm not going to cover them all. Try your favorite search engine or LLM for that.
The message is simply, "If `nvidia-smi` runs in your container, you're golden."

For this, you may need to host your own image repository, or push to hub.docker.com
or the like. Follow the instructions in the `kustomization.yaml` to uncomment the
inline patch to allow the `images` block to work with your image moving forward.
It's to override the default system image name (otherwise the `images` block can't
override the version because it doesn't see the images as even related).

### NOTE

When using NFS, you may need to set `plex_gid` and `plex_uid` in `server.properties`
to match the UID and GID of the filesystem. So long as you have read access, things
should work. If you want Plex to be able to delete files, then you'll need to ensure
the permissions are correct.


## 2024-09-17

First push. ALPHA release, not ready for general consumption.