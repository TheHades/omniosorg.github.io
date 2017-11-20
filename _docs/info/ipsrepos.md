---
title: IPS Repositories
category: info
show_in_sidebar: true
---

# IPS Repositories

OmniOS takes a *layer cake* approach to packaging. The core OS contains
the packages needed to build the OS, plus a few small frills (more
shells, tmux/screen, etc.). Users are encouraged to either create their
own package repositories for additional software they want to run (and where
they like it to be installed) or use repositories published by other users.

Maintainers of add-on repositories are encouraged to share their work with the
community. If you wish to have your repository listed here, please get in
touch [in the Lobby](https://gitter.im/omniosorg/Lobby).

## Repos

{:.bordered .responsive-table}
| URL                                        | Publisher    | Signed | Build Scripts                                                     | Notes                                       |
|--------------------------------------------|--------------|--------|-------------------------------------------------------------------|---------------------------------------------|
| <https://pkg.omniosce.org/r151024/core/>   | omnios       | yes    | [r151024](https://github.com/omniosorg/omnios-build/tree/r151024) | Core OS components (stable)
| <https://pkg.omniosce.org/r151024/extra/>  | extra.omnios | yes    | [r151024](https://github.com/omniosorg/omnios-extra/tree/r151024) | Additional packages (stable)
| <https://pkg.omniosce.org/r151022/core/>   | omnios       | yes    | [r151022](https://github.com/omniosorg/omnios-build/tree/r151022) | Core OS components (LTS)
| <https://pkg.omniosce.org/r151022/extra/>  | extra.omnios | yes    | [r151022](https://github.com/omniosorg/omnios-extra/tree/r151022) | Additional packages (LTS)
| <https://pkg.omniosce.org/bloody/core/>    | omnios       | no     | [master](https://github.com/omniosorg/omnios-build)               | Core OS components (unstable)
| <https://pkg.omniosce.org/bloody/extra/>   | extra.omnios | no     | [master](https://github.com/omniosorg/omnios-extra)               | Additional packages (unstable)

## Unofficial Extras

{:.bordered .responsive-table}
| URL                                      | Publisher          | Maintainer                             | Build Scripts                                                               | Notes                                                                        |
|------------------------------------------|--------------------|----------------------------------------|-----------------------------------------------------------------------------|------------------------------------------------------------------------------|
| <http://pkg.cs.umd.edu/>                 | cs.umd.edu         | Sergey Ivanov                          |                                                                             |                                                                              |
| <http://pkg.omniti.com/omniti-ms/>       | ms.omniti.com      | OmniTI                                 | [omniti-ms](https://github.com/omniti-labs/omniti-ms)                       | Non-core packages used in OmniTI's managed services environments             |
| <http://pkg.omniti.com/omniti-perl/>     | perl.omniti.com    | OmniTI                                 | [omnios-build-perl](https://github.com/omniti-labs/omnios-build-perl)       | Perl module dists designed to work with omniti/runtime/perl                  |
| <http://pkg.niksula.hut.fi/>             | niksula.hut.fi     | pkg@niksula.hut.fi                     | <https://github.com/niksula/omnios-build>                                   | Signed packages; see the [instructions](http://pkg.niksula.hut.fi/)          |
| <http://scott.mathematik.uni-ulm.de/>    | uulm.mawi          | Steffen Kram                           | [stefri/omnios-build](https://github.com/stefri/omnios-build)               |                                                                              |
| <http://sfe.opencsw.org/localhostomnios> | localhostomnios    | SFE Community                          | <https://sourceforge.net/p/pkgbuild/code/HEAD/tree/spec-files-extra/trunk/> | Open for contribution                                                        |
| <https://ips.qutic.com/>                 | application        | [qutic development](https://qutic.com) | <https://github.com/jfqd/omnios-userland>                                   | Userland packages; pull requests welcome                                     |          
| <http://www.opencsw.org/>                | SysV packages      | OpenCSW                                | <https://sourceforge.net/p/gar/code/HEAD/tree/>                             | A collection of SysV packages (i.e. for use with the old pkgadd(1M) command) |

Note that ms.omniti.com and perl.omniti.com hold packages built
specifically for OmniTI's own use. While there is nothing secret or
astonishing therein, non-OmniTI users may wish to see the
[template branch](https://github.com/omniti-labs/omnios-build/tree/template)
which may be used as the basis to build one's own packages.

