---
description: Configuration for hardware acceleration for transcoding with VA-API and NVENC
---

# Hardware Transcoding

This document describes setting up transcoding in Memories, specifically using hardware acceleration. Hardware acceleration can significantly improve transcoding performance, especially for high resolution videos. Memories supports transcoding using **CPU**, **VA-API** and **NVENC**.

## Overview

Newer Intel processors come with a feature called QuickSync that can significantly boost transcoding performance (4-5x improvement over x264 is common). QuickSync can be used for hardware accelerated transcoding using the VA-API in ffmpeg.

Note: VA-API acceleration may also work with some AMD GPUs.

To configure VAAPI, you need to have `/dev/dri` available to the Nextcloud instance with the `www-data` in the group owning the drivers. You also need the correct drivers and a compatible version of ffmpeg installed (older versions may not work with modern hardware).

NVIDIA GPUs support hardware transcoding using NVENC.

!!! tip "Hardware acceleration is optional"

    Hardware acceleration is optional and not required for Memories to function. If you do not have hardware acceleration, Memories will use the CPU for transcoding.

!!! bug "Filing bugs related to transcoding"

    If you have issues with hardware transcoding, reach out for [help](/faq/#getting-help). Make sure you include details about your setup such as how the transcoder is set up, the version of each component and **the logs from the transcoder**.

## External Transcoder

!!! success "Recommmended Configuration"

    The easiest and recommended way to use hardware transcoding in a docker environment is to use an external transcoder.
    This setup utilizes a separate docker container that contains the hardware drivers and ffmpeg.
    If you cannot do this, other installation methods are also possible (see below).

[go-vod](https://github.com/pulsejet/memories/tree/master/go-vod), the transcoder of Memories, comes with a pre-built Docker image based on `linuxserver/ffmpeg`. The docker image connects to your Nextcloud instance and pulls the go-vod binary on startup. To set up an external transcoder, follow these steps.

1. Use a `docker-compose.yml` that runs the go-vod container and mounts the Nextcloud data directories to it. You must specify `NEXTCLOUD_HOST` to match the name of your Nextcloud container.

    ```yaml
    services:
      server:
        image: nextcloud
        volumes:
          - ncdata:/var/www/html

      go-vod:
        image: radialapps/go-vod
        restart: always
        depends_on:
          - server
        environment:
          - NEXTCLOUD_HOST=https://your-nextcloud-url
          # - NEXTCLOUD_ALLOW_INSECURE=1 # (self-signed certs or no HTTPS)
          - NVIDIA_VISIBLE_DEVICES=all
        devices:
          - /dev/dri:/dev/dri # VA-API (omit for NVENC)
        volumes:
          - ncdata:/var/www/html:ro
        # runtime: nvidia # (NVENC)
    ```

    !!! info "Device and volume bindings"
        In this example, the VA-API devices in `/dev/dri` are passed to the container, along with the Nextcloud data directory (as readonly). All volumes must be mounted at the same location as the Nextcloud container.

    !!! question "What to set in `NEXTCLOUD_HOST`?"
        The `NEXTCLOUD_HOST` environment variable must be set to the URL of your Nextcloud instance. If you are using a reverse proxy, you must set this to the URL of the reverse proxy. If you are using a self-signed certificate or http, you must also set `NEXTCLOUD_ALLOW_INSECURE=1`. This URL is used to download the transcoder binary and to connect to the Nextcloud instance.

    !!! tip "Setup for NVENC"
        If you want to use NVENC instead of VA-API, uncomment the `runtime` line and remove the `devices` section above. You will need to install the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) on your host.

1. You can now configure the go-vod connect address in the Memories admin panel to point to the external container. go-vod uses port `47788` by default, so in our example the **connection address** would be set to **`go-vod:47788`**.

1. Finally, turn on **enable external transcoder** in the admin panel. This will initiate a test of the transcoder and show the result.

Your external transcoder should now be functional. You can check the transcoding logs by running `docker compose logs -f go-vod`.

!!! tip "Usage with Nextcloud AIO"

    With AIO, you will need to put the container into the `nextcloud-aio` network. Also the `datadir` of AIO needs to be mounted at the same place as in its Nextcloud container into the go-vod container. Usually this would be `nextcloud_aio_nextcloud_data:/mnt/ncdata:ro` or `$NEXTCLOUD_DATADIR:/mnt/ncdata:ro`.

    If you are not using NVENC, you can also skip these steps and use the **memories community container** instead. Relevant documentation can be found [here](https://github.com/nextcloud/all-in-one/tree/main/community-containers/memories), and general directions on using community containers [here](https://github.com/nextcloud/all-in-one/tree/main/community-containers).

!!! info "Usage without Docker Compose"

    You can run a similar setup without `docker-compose`. Make sure that the Nextcloud and go-vod containers are in the same network and that the Nextcloud data directories are mounted at the same locations in both containers.

## Internal Transcoder

Memories ships with an internal transcoder binary that you can directly use. In this case, you must install the drivers and ffmpeg on the same host as Nextcloud, and Memories will automatically handle starting and communicating with go-vod. This is also the default setup when you enable transcoding without hardware acceleration.

!!! danger "Advanced usage only"

    In most cases, it is easier to use an external transcoder when you need hardware acceleration.
    The internal transcoder is only suitable for CPU transcoding or if you do not use Docker.

!!! tip "NVENC"

    These instructions mostly focus on VA-API. For NVENC, you may find further useful
    pointers in [this](https://github.com/pulsejet/memories/blob/master/go-vod/build-ffmpeg-nvidia.sh) build script.

### Bare Metal

If you are running Nextcloud on bare metal, you can install the drivers and ffmpeg directly on the host. If you are running nextcloud in a Virtual Magine or LXC container configuration, you will also need to pass through the hardware resource to the nextcloud machine. Some helpful guides can be found for [Proxmox VM](https://pve.proxmox.com/wiki/PCI_Passthrough) / [LXC Container](https://gist.github.com/packerdl/a4887c30c38a0225204f451103d82ac5?permalink_comment_id=4471564). 

On the Nextcloud machine, you need to make sure that the `www-data` user has access to the `/dev/dri` devices. You can do this by adding the `www-data` user to the appropriate groups.

```bash
## Ubuntu
sudo apt-get update
sudo apt-get install -y intel-media-va-driver-non-free ffmpeg # install VA-API drivers
sudo usermod -aG video www-data # add www-data to the video group (may be different)

## Alpine
apk update
apk add --no-cache bash ffmpeg libva-utils libva-vdpau-driver libva-intel-driver intel-media-driver mesa-va-gallium
```

In some cases, along with adding `www-data` to the appropriate groups, you may also need to set the permissions of the device manually:

```bash
sudo chmod 666 /dev/dri/renderD128
```

You can run a test using a sample video file to check if VA-API is working correctly for the `www-data` user:

```bash
# It may be best to run the following test from within your
# Nextcloud data directory (e.g. /mnt/ncdata/<user>/files/)

# download sample or or use any other video file
wget https://github.com/pulsejet/memories-assets/raw/main/sample.mp4
chown www-data:www-data sample.mp4

# check if VA-API is working
sudo -u www-data \
  ffmpeg -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 -hwaccel_output_format vaapi \
  -i 'sample.mp4' -vcodec h264_vaapi \
  output-www-data.mp4
```

!!! warning "Beware of old ffmpeg and driver versions"

    Some package repositories distribute old ffmpeg versions that do not support some modern hardware. (e.g., the VA-API driver installed by `apt` in the current debian image used by Nextcloud only supports up to 10th generation Intel Ice Lake CPUs). To ensure you have a compatible version, you may want to remove your existing ffmpeg version and build the drivers and ffmpeg from source.  [This script](https://github.com/pulsejet/memories/blob/master/go-vod/build-ffmpeg.sh) for VA-API or [this one](https://github.com/pulsejet/memories/blob/master/go-vod/build-ffmpeg-nvidia.sh) for NVENC might be useful.

### Docker

!!! danger "Use an external transcoder"

    If you need hardware transcoding and use Docker, it can be significantly easier to use an external transcoder. See [above](#external-transcoder) for instructions.
    The instructions below, as a result, are mostly historical and **not recommended** for normal usage.

If you use Docker and want to use the internal transcoder, you need to:

1. Pass the `/dev/dri` device to the Nextcloud container. In `docker-compose.yml`:
   ```yaml
   nc:
     build: .
     restart: always
     devices:
       - /dev/dri:/dev/dri
   ```
1. Make sure the right drivers are installed. This can be done using a custom Dockerfile as illustrated below.
   ```Dockerfile
   FROM nextcloud:stable-fpm

   RUN apt-get update && \
       apt-get install -y lsb-release && \
       echo "deb http://ftp.debian.org/debian $(lsb_release -cs) non-free" >> \
          /etc/apt/sources.list.d/intel-graphics.list && \
       apt-get update && \
       apt-get install -y intel-media-va-driver-non-free ffmpeg && \
       rm -rf /var/lib/apt/lists/*

   COPY start.sh /
   CMD /start.sh
   ```

1. The `start.sh` should add the user to the video group and start php-fpm.
   ```bash
   #!/bin/bash
   GID=`stat -c "%g" /dev/dri/renderD128`
   groupadd -g $GID render2 || true # sometimes this is needed
   GROUP=`getent group $GID | cut -d: -f1`
   usermod -aG $GROUP www-data

   php-fpm
   ```

If you're using the `linuxserver/nextcloud` image, based on Alpine, you can add the following to the `docker-compose.yml` file to install the VA-API drivers and use the internal transcoder directly.

```yaml
devices:
  - /dev/dri:/dev/dri
environment:
  - DOCKER_MODS=linuxserver/mods:universal-package-install
  - INSTALL_PACKAGES=libva|libva-intel-driver|intel-media-driver|mesa-va-gallium
```

## Troubleshooting

### Basic Steps

If you have trouble with trancoding, try the following steps:

1. Check the admin panel for any errors. It may be possible that Memories cannot connect to the transcoder or you have a go-vod version mismatch.

1. Check the JS console and the logs of the transcoder. See [below](#logging) for instructions.

1. The admin panel lists a few options that work around driver bugs. For instance, if your portrait videos are rotated on VA-API or your NVENC stream hangs, try enabling these workarounds.

1. If you are using the internal transcoder, make sure you are running a new enough version of ffmpeg (shown in the admin panel). Generally you would need at least ffmpeg v5.x for most modern hardware but many operating systems ship with v4.x. One troubleshooting step is to build ffmpeg and the hardware drivers from source.

1. It may be helpful to run a manual test of ffmpeg in the same environment as the transcoder. See [above](#bare-metal) for instructions. Note that the transcoder output / logs contain the full ffmpeg command used for each transcode.

### Logging

When running an **external transcoder**, the logs go to the container's stdout. You can view them using 

```bash
docker compose logs -f go-vod     # for docker-compose
docker logs -f <container-name>   # if not using docker-compose
```

When using the **internal transcoder**, the logs go to `/tmp/go-vod/<instance-id>.log`, where `<instance-id` is a unique ID for your Nextcloud instance that can be found in `config.php`. You can view them as illustrated below.

```bash
tail -f /tmp/go-vod/<instance-id>.log # bare metal

docker exec -it <container-name> cat /tmp/go-vod/<instance-id>.log # Docker
```
