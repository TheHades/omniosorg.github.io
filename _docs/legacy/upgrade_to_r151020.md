---
title: Upgrading to r151020
category: info
show_in_sidebar: false
---

# Upgrading to r151020.

One may upgrade any of r151014, r151016, r151018, or r151019/bloody to
r151020.

## Upgrading to OpenSSH from SunSSH

SunSSH has been end-of-lifed in r151020. To this end, either prior to
the upgrade switch to OpenSSH:

```
# /usr/bin/pkg install --no-backup-be --reject pkg:/network/ssh --reject pkg:/network/ssh/ssh-key --reject pkg:/service/network/ssh --reject pkg:/service/network/ssh-common pkg:/network/openssh pkg:/network/openssh-server
```

Or add all of the `--reject` options shown above to any commands below for
linked-image deployments. If you use non-linked image zones (ipkg), you
must update to SunSSH PRIOR to reboot.

Switching from SunSSH to OpenSSH will require commenting-out `MaxAuthTriesLog`
and `RhostsAuthentication` options from `/etc/ssh/sshd_config`.
Also both `ListenAddress` directives MUST be commented out
(SunSSH has one uncommented by default). There will be an
`sshd_config.new` to consult, which comes from a fresh, unhindered
installation of OpenSSH.

Also, if you have entries in `/etc/pam.conf` you may need to update them. For example,
one may need to `s/sshd-kbdint/sshd/` to re-enable their two-factor authentication.

**ALL INSTALLATIONS WITH ZONES MUST** update all zones (including
global) to OpenSSH pre-upgrade. Even linked-image zones will require
per-zone installation, because of how SunSSH/OpenSSH were mediated prior
to SunSSH's end-of-life.

## A REMINDER ABOUT NUMBER OF BOOT ENVIRONMENTS

There is a known issue with grub and its memory management that limits
the number of boot environment entries in . This limit has been
shrinking as grub gets more features and bugfixes. We hope to eliminate
GRUB in the next LTS (OmniOS r151022).

If you have more than 32 Boot Environments, please go through and
destroy ones you don't need anymore **PRIOR TO UPGRADING**. (Use the
[beadm](http://illumos.org/man/1m/beadm) command - e.g. `beadm destroy BENAME`). The
experimental limits with the r151020 version of grub suggests 35 is the
upper limit of number of boot environments before grub cannot boot the
system. If you have more than 32 boot environments, we recommend
deleting enough to not break grub.

Performing the Upgrade (lipkg zones from a r151014 or later stable)
-------------------------------------------------------------------

**WARNING**: For the linked-images to upgrade, you MUST switch all zones
*pre-upgrade* to OpenSSH for this to work if you have not already. See
above for how to switch to OpenSSH. Please make sure all zones have
OpenSSH installed prior to following any directions below.

If you have moved to linked-image (lipkg)
non-global zones exclusively, the upgrade process can be simpler, modulo
some setup.

* If you wish to avoid a window of on-rpool log overflow between the old and new boot environments, disable any services that log in appropriate zones (including global)
* Make sure the global zone can reach the network
* Create a backup boot environment for safety (being careful of the number of BEs):
  ```
  # beadm create <appropriate-backup-name>
  ```
* Change the publisher (as root or with privilege) in every zone root, including global. For example, going from r151018 to r151020:
  ```
  # /usr/bin/pkg set-publisher -G http://pkg.omniti.com/omnios/r151018/ -g http://pkg.omniti.com/omnios/r151020/ omnios
  # /usr/bin/pkg -R /zones/zone1/root set-publisher -G http://pkg.omniti.com/omnios/r151018/ -g http://pkg.omniti.com/omnios/r151020/ omnios
  # /usr/bin/pkg -R /zones/zone2/root set-publisher -G http://pkg.omniti.com/omnios/r151018/ -g http://pkg.omniti.com/omnios/r151020/ omnios
  .  .  .
  ```
* If you have non-OmniOS IPS publishers, some of those packages may not yet be aware of r151020 and block the upgrade (e.g. some ms.omniti.com packages). Uninstall these ones prior to the update
* Perform the update, optionally specifying the new BE name, and if you are on SunSSH, adding the `--reject` arguments as well:
  ```
  # /usr/bin/pkg update {--be-name new-BE-name}
  ```
* Many times a BE is named after the release it's becoming.  For example:
  ```
  # /usr/bin/pkg update --be-name r151020
  ```
* Reboot

**NOTE**: Once past a release migration, linked-image zones still offers
great convenience, because a single “pkg update” in the global zone easy
updates all linked-image zones.

## Performing the Upgrade (all other cases, including lipkg if you wish)

**WARNING**: For the detach/attach method of upgrade, you MUST switch
all zones *pre-upgrade* to OpenSSH for this to work if you have not
already. See above for how to switch to OpenSSH. Please make sure all
zones have OpenSSH installed prior to following any directions below.

If you have non-global native (ipkg) zones, they must be shutdown and
detached at this time. **Even if you are used to updating zones after
`pkg upgrade` by using `pkg -R`, you MUST perform the upgrade this way,
because of the signature-policy changes.** (Bloody users, even
post-r151014 bloody, must do this too because of the signature policy
changes.) Once the global zone is updated and rebooted, the zones will
be upgraded as they are re-attached to the system. This is not necessary
for s10-branded zones or KVM guests.

After shutting down the zones gracefully (`zlogin <zonename>; shutdown -i5 -g0 -y`):

```
# /usr/sbin/zoneadm -z <zonename> detach
```

It would also be a good idea to take a ZFS snapshot of the zone root in
case it's needed for rollback (such as if there are issues with the zone
upgrade.)

```
# /usr/sbin/zfs snapshot -r <zoneroot>@<old-release>
```

where <zoneroot> is the name of the ZFS dataset whose
mountpoint corresponds to the value of *zonepath* in the zone's
configuration. There are child datasets under this one, so we use the `-r`
option to recursively snapshot all.

Because each OmniOS release has its own dedicated repo, you will first
need to set the package publisher to the repository for r151020:

```
# /usr/bin/pkg unset-publisher omnios
# /usr/bin/pkg set-publisher -P --set-property signature-policy=require-signatures -g http://pkg.omniti.com/omnios/r151020/ omnios
```

**NOTE: The “require-signatures” is new since r151014, and if you are
upgrading this way it is up to you, the administrator, to make this
change effective. Since you are upgrading from r151014 or later, it's
strictly not needed, but stating it again will not hurt. The zone attach
code will automatically match zone's publisher policies with the global
zone. New zone creation after updating to r151020 will continue to apply
the global zone's publisher policies on a per-publisher basis. If you
have existing other publishers with or without signature policies, those
publishers' signature policies will propagate into non-global zones.**

Update the global zone. The argument `--be-name` is optional, but it's nice to use a
name that's more meaningful than “omnios-N”. Add a `-nv` after the
`update` sub-command to do a dry run if you're unsure of what will
happen. **REMEMBER** - You must change to OpenSSH first before using
this method.

```
# /usr/bin/pkg update --be-name=omnios-r151020 entire@11,5.11-0.151020
```

This will create a new BE and install r151020 packages into it. When it
is complete, reboot your system. The new BE will now be the default
option in GRUB.

Once booted into your new r151020 BE, if you don't have non-global
zones, you are done with the upgrade.

If you have non-global native (ipkg) zones, you can either modify them
to become linked-image zones (lipkg), or
continue on with non-linked images. If you wish to make a zone be a
linked-image one, change the brand PRIOR to attachment:

**NOTE: This is optional. Use only if you want <zonename> to be linked-image.**

```
# /usr/sbin/zonecfg -z <zonename> set brand=lipkg
```

Attach each one with the option, which will upgrade the zone's core
packages to match the global zone.

```
# /usr/sbin/zoneadm -z <zonename> attach -u
```

Assuming the attach succeeds, the zone may be booted as usual:

```
# /usr/sbin/zoneadm -z <zonename> attach -u
```

## OOPS, I didn't pay attention and now my zones won't attach

If for some reason you didn't move all of your zones to OpenSSH prior to
detaching, you can use a heavier-weight approach.

First, force-attach the zone. Normally this is dangerous, but if you're
in an unattachable state, this is your only choice:

```
# /usr/sbin/zoneadm -z <zonename> attach -F
```

Next, use the `-R` option to pkg(1M) and force-upgrade the zone with all
of the `--reject` flags:

```
# /usr/bin/pkg -R /zones/<zonename>/root update --reject pkg:/network/ssh --reject pkg:/network/ssh/ssh-key --reject pkg:/service/network/ssh --reject pkg:/service/network/ssh-common pkg:/network/openssh pkg:/network/openssh-server
```

Finally, detach and attach the zone to reality check its integrity:

```
# /usr/sbin/zoneadm -z <zonename> detach
# /usr/sbin/zoneadm -z <zonename> attach -u
```

The attach should be quick, as the upgrade already happened. The zone is
now updated.

