---
title: bhyve and KVM branded zones
category: info
layout: page
show_in_sidebar: true
---

## Introduction

With OmniOS r151028, support for bhyve and KVM branded zones has been
introduced. These branded zones are incredibly small and do not need to be kept
up-to-date using _pkg_ since they share files with the global zone. Their
purpose is to allow kvm and bhyve to be managed as a zone keeping them
isolated from the rest of the system and enabling protection from known
CPU vulnerabilities.

## Configuration

Zones are configured largely through the use of attributes which can make
them quite verbose to set up through `zonecfg`. You may be interested in
the work on [ZCage](https://github.com/cneira/zcage) which has support for
configuring these types of zone.

## Example

Most attributes have reasonable default value as described in the next
section so here's a complete example bhyve zone configuration for installing
FreeBSD from an _iso_ file. Since nothing is explicitly specified, this machine
will default to a single virtual CPU and 1GiB of RAM. The machine's serial
console is accessible via `zlogin`. It is also possible to configure VNC for
machines that use UEFI boot.

```terminal
omnios# dladm create-vnic -l net0 bhyve0
omnios# zfs create -V 30G rpool/bhyve0
omnios# zonecfg -z freebsd
create -b
set brand=bhyve
set zonepath=/data/zone/bhyve
set ip-type=exclusive
add net
    set allowed-address=10.0.0.112/24
    set physical=bhyve0
end
add device
    set match=/dev/zvol/rdsk/rpool/bhyve0
end
add attr
    set name=bootdisk
    set type=string
    set value=rpool/bhyve0
end
add fs
    set dir=/rpool/iso/debian-9.4.0-amd64-netinst.iso
    set special=/rpool/iso/debian-9.4.0-amd64-netinst.iso
    set type=lofs
    add options ro
    add options nodevices
end
add attr
    set name=cdrom
    set type=string
    set value=/rpool/iso/debian-9.4.0-amd64-netinst.iso
end
omnios# zoneadm -z freebsd install
omnios# zoneadm -z freebsd boot
omnios# zlogin -C freebsd
```

## Attributes

The following table shows the available attributes for bhyve and KVM zones
along with their default values. Attributes are added to the zone
configuration as shown in the example above; all attributes have the
_string_ type.

{:.bordered .responsive-table .bordered}
| Attribute                     | Default                | Syntax
| ---                           | ---                    | ---
| `acpi`<sup>1</sup>            | `on`                   | `on`,`off`
| `bootdisk`<sup>2</sup>        |                        | Eg. `rpool/hdd-bhyve0`
| `bootorder`                   | `cd`                   | [`c`][`d`][`n`]
| `bootrom`<sup>1,4</sup>       | `BHYVE_RELEASE_CSM`    | firmware image name
| `cdrom`<sup>3</sup>           |                        | `/path/to/image.iso`
| `console`<sup>6</sup>         | `/dev/zconsole`        | Eg. `socket,/tmp/vm.com1,wait`
| `disk`<sup>2</sup>            |                        | `/dev/zvol/rdsk/fast/swap`
| `diskif`                      | `virtio`               | `virtio`,`ahci`
| `hostbridge`<sup>1</sup>      | `i440fx`               | `i440fx`,`q35`,`amd`,`netapp`,`none`
| `netif`                       | `virtio`               | `virtio`,`e1000`
| `ram`                         | `1G`                   | <code><i>n</i>G</code>,<code><i>n</i>M</code>
| `type`                        | `generic`              | `generic`,`windows`
| `vcpus`<sup>7</sup>           | `1`                    | <code>[[cpus=]<i>numcpus</i>][,sockets=<i>n</i>][,cores=<i>n</i>][,threads=<i>n</i>]]</code>
| `vnc`<sup>5</sup>             |                        | `off`,`on`,`options`

### Notes

1. bhyve only;
2. You will also need to pass the underlying disk device through to the zone
   via a device entry as shown in the example above;
3. The ISO file needs passing through to the zone via a lofs mount as shown
   in the example above;
4. Available firmware files can be found in `/usr/share/bhyve/firmware/`;
5. Setting vnc to on is the same as setting it to `unix=/tmp/vm.vnc`;
6. You can connect to the virtual machine console from the global zone with
   `zlogin -C zonename`;
7. For KVM, the extended syntax can cause problems with the guest; it's best to stick to simple numbers here.


## VNC Access

If the `vnc` attribute is set to `on`, then a VNC server will be started
listening on a UNIX socket at `/tmp/vm.vnc` within the zone. Note that this
only functions for bhyve zones if the guest is booted via UEFI. In order to
connect the socket to a TCP port so that it can be accessed using a VNC client
one option is to use the mini `socat` utility that comes with the brand.

```terminal
omnios# /usr/lib/brand/bhyve/socat /data/zone/bhyve/root/tmp/vm.vnc 5905
```

> It is intended that future zone management tools incorporate this feature
> in an easy-to-use way.

