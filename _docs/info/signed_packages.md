---
title: Signed Packages
category: info
show_in_sidebar: false
---

# Signed Packages

With the [r151008 release cycle](/legacy/releasenotes/r151008), OmniTI
started providing cryptographically signed packages in the main release
repository. Our intention is to provide signed packages for all new
packages published to any release repo.

With the [r151014 release cycle](/legacy/releasenotes/r151014) we take the
further step with ISO and Kayak installations that the “omnios”
publisher will now **require** signed OmniOS packages. If someone
upgrades from pre-r151014 to r151014, they must follow the
[documented upgrade instructions](/legacy/upgrade_to_r151014) to make
**require-signatures** the signature policy for the “omnios” publisher.
This policy will not affect other publishers' default settings.

Note that only packages in the stable release repos will be signed.
Packages in the unstable (bloody) repo will not be signed.

This document describes what this means for the end user.

## Installer Changes

As of r151008, OmniOS release install images came with the OmniTI
Certificate Authority (CA) set as an approved authority for the omnios
publisher and **verify** that packages from the omnios publisher will be
signed with a cert issued by the OmniTI CA. (Unsigned packages are still
allowed with the **verify** policy, but signed packages will be
inspected under that same policy.)

As of r151014, OmniOS release install images change the signature policy
to **require-signatures**. This means that, by default, all packages
subsequently installed from the omnios publisher will be
cryptographically verified and will fail to install if the hash of the
package's manifest contents does not match the signature. Other
publishers that you wish to add will not be subject to this restriction
(they will default to **verify**, which only rejects upon invalid
signatures, not for lack of signatures) unless you choose to make them
so.

## Changing the Defaults

Both publishers and images (i.e. collections of publishers that supply
software to an instance of OmniOS) have a property (signature-policy)
that governs whether and to what extent package signatures should be
checked. People who [upgrade to r151014](/legacy/upgrade_to_r151014) are
expected to change the omnios publisher's policy per the documented
upgrade instructions.

### Signature Policy

In a default r151014 or later install, packages from the omnios
publisher *must* have valid signatures in order to be installed
(signature-policy **require-signatures**). Signatures in packages from
other publishers will be verified *if they exist*, but are not required
(image signature-policy **verify**). Use the following commands to view
the properties of the omnios publisher and the local image,
respectively:

```
$ pkg publisher omnios
$ pkg property
```

If you wish to require *all* packages from *all* publishers configured
in the local image to be signed, set the *image's* signature-policy
property:

```
# pkg set-property signature-policy require-signatures
```

**WARNING** - Do not do this unless you are sure all your publishers,
especially ones in non-global zones who inherit policies from the global
zone, have signed packages AND you have their root CA certs installed
properly.

See the pkg(1) man page for details on publisher and image properties.
Publisher properties are set via the **set-publisher** sub-command.
Image properties, as shown above, are set via the **set-property**
sub-command.

### Chains of Trust

#### Manifest Signing

Certificates used for signing must be issued by a trusted authority. By
default, pkg(1) trusts CA certs that have been either:

* set for the publisher: 
  ```
  pkg set-publisher --approve-ca-cert /path/to/ca-cert.pem publishername
  ```
* placed into a dedicated directory for this purpose. For OmniOS, it's `/etc/ssl/pkg/`

If you wish to configure pkg(1) to trust different global CAs, you may
configure the local image with a different trust directory:

```
# pkg set-property trust-anchor-directory /path/to/cert/dir
```

#### SSL Transport

A different property is used to govern trust when communicating with
repositories over HTTPS. This is the “ca-path” image property. By
default, this is also . You may change this path to point at an
alternate directory.

```
# pkg set-property ca-path /path/to/cert/dir
```

NOTE: SSL transport is not implemented for OmniOS currently by default.

## FAQ

### What if I don't care about requiring signed packages?

You may change the signature-policy property on all publishers and the local image
if you wish. Use the value “ignore”. Do so at your own risk, however

```
# pkg set-property signature-policy ignore
```

### How do I trust the install media?

SHA1 and MD5 checksums are published for all ISO, USB and Kayak ZFS images.
You may retrieve them over HTTPS from the installation page
in order to verify that they are coming from OmniTI.

### How do I get my existing r151012 install to have the same settings as a new one?

To convert an existing install to behave like a new install, follow these steps.
This includes removing and re-adding the omnios publisher. This is because signed
manifests have the same timestamp as their unsigned predecessors, and pkg(1) doesn't
notice the change. Removing the publisher clears all local copies of the repository
catalog, allowing the new, signed manifests to be seen.

```
# pkg update pkg
# pkg unset-publisher omnios
# pkg set-publisher -g http://pkg.omniti.com/omnios/release/ omnios
# pkg update web/ca-bundle
# openssl sha1 /etc/ssl/pkg/OmniTI_CA.pem
# # Compare the sha1 against the one listed at <secure URL>
# pkg set-publisher --approve-ca-cert /etc/ssl/pkg/OmniTI_CA.pem omnios
# pkg set-publisher --set-property signature-policy=require-signatures omnios
```

