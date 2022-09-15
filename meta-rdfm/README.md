# `meta-rdfm` Yocto layer

Copyright (c) 2021-2022 [Antmicro](https://www.antmicro.com)

Antmicro's collection of Yocto layers for machine learning and computer vision applications.

## Description

The `meta-rdfm` Yocto layer provides set of recipes for tools to perfrom `Over The Air (OTA)` system updates.

Devices support can be extended by [meta-rdfm-tegra](../meta-rdfm-tegra) sublayer. Consider including it your in `bblayers.conf` if you are working with jetson family devices.

## Layer dependencies

The `meta-rdfm` layer depends on `meta-mender-core` sublayer presented in [meta-mender](https://github.com/mendersoftware/meta-mender) Yocto layer.

We recommend to use the `7221ae1a7e12bf8416cca630a829336c0d45faf6` revision of `meta-mender-core` to ensure compatibility between meta-layers.

## local.conf configuration

In order to use `meta-rdfm` with your project you have to define set of [features](https://docs.mender.io/system-updates-yocto-project/image-customization/features) you want to include in your `local.conf`:

```
# Defines rdfm features to be used
INHERIT += "mender-full"
```

**Note:** By default `meta-rdfm` layers sets `ext4` rootfs to be read-only, it's required by delta updates.
You can disable it by removing this line from `conf/layer.conf`:
```
EXTRA_IMAGECMD:ext4:append = "${@bb.utils.contains('IMAGE_FEATURES', 'read-only-rootfs', ' -O ^64bit -O ^has_journal', '', d)}"
```

For additional configuration features see [Glossary](#glossary)

## How to use

You must first build two images using `meta-rdfm` layer features in the way described in the [meta-antmicro sample use case](../README.md#meta-antmicro-sample-use-case) section, one for the initial flashing and one for the OTA update.

Once the board has been flashed with first image, you can install the initial update.

### Installing an update

Make sure that you have a working Ethernet connection with the target device.

On your host device, go to directory containing built images and run server:
```
cd <folder-containing-rdfm-image>
python -m http.server 12345
```

Open running `http.server` and copy link to built `.rdfm` image that will be used for an update.

On target device establish Ethernet connection with host device and run update installation using `rdfm-client`:
```
rdfm install <link-to-rdfm-image>
```

Reboot target device and verify if partition was properly installed:
```
df -h
```

You can list intstalled `provides` using:
```
rdfm show-provides
```

### Generating a delta update

The `meta-rdfm` layer supports OTA updates using deltas. In order to generate one you have to have two already built `.rdfm` images you want to perform an update with.

Install dependencies:
```
sudo apt install rdiff
sudo apt install jq
```

Clone `rdfm-artifact` on host device and build it:
```
cd <folder-containing-rdfm-images>
git clone git@github.com:antmicro/rdfm-artifact.git
cd rdfm-artifact
make
cp rdfm-artifact ../
cd ..
```

Create `deltas.sh` script with the following content:
```
#!/bin/sh

set -ex

OLD="$1"
NEW="$2"

SIGFILE="/tmp/rootfs-$$.sig"

checksum=$(tar -xOf "$OLD" header.tar.gz |tar -xzOf- headers/0000/type-info |jq -r '.artifact_provides."rootfs-image.checksum"')
tar -xOf "$OLD" data/0000.tar.gz | tar -xzOf- | /usr/bin/rdiff signature -R rollsum -b 4096 - "$SIGFILE"
./rdfm-artifact modify --delta-compress "$SIGFILE" --depends rootfs-image.checksum:"$checksum" "$NEW"

rm "$SIGFILE"
```

> **Warning:** This script generates an update with dependency on `rootfs-image.checksum.`
> If you don’t want to have a dependency on image checksum for update, consider removing next from script:
>
> ``--depends rootfs-image.checksum:"$checksum"``

Add execution rights to script and run it:
```
chmod +x deltas.sh
./deltas.sh <image-to-be-updated>.rdfm <image-with-update>.rdfm
```

The freshly generated image with delta update will be located in `<image-with-update>.rdfm` file and can be installed the same way it was described in [Installing an update section](#installing-an-update).

## Glossary

Besides the already existing parameters in meta-mender, the meta-rdfm layer introduces new variables that affect the build process.

### RDFM_ARTIFACT_NAME

Specifies the name of the generated artifact. Required for the `rdfm` image building process. Should be specified on the command line, and not in the configuration:
```
RDFM_ARTIFACT_NAME = "release-1"
```

### RDFM_CLEAR_PROVIDES

Specifies whether to clear provides for an output `.rdfm` image. By default, `rdfm-artifact` adds all provides to the `clear provides` section. This default behavior can cause issues when we use `depends` for image updating.

In order to force `meta-rdfm` clear provides, the `RDFM_CLEAR_PROVIDES` should be set to non-empty value in your local.conf or command line:
```
RDFM_CLEAR_PROVIDES = "1"
```

### RDFM_NO_PROVIDES_LOCALLY

Specifies whether to write `provides` to the installation `.ext4` image.
By default, the `.rdfm` image stores all the meta-data including `provides` in its header file, which is not part of the built installation image.
This behavior can cause issues if we try to update the system without having any connection to the server, since all the meta-data is stored exclusively on the server.
Default path for `provides` writing is ``/etc/mender/provides_info``.

When `RDFM_NO_PROVIDES_LOCALLY` is set to non-empty value, it specifies not to write any ``provides`` to ``/etc/mender/provides_info`` file:

```
RDFM_NO_PROVIDES_LOCALLY = "1"
```
