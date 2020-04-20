---
title: Migrate entire pool to new pool
category: info
show_in_sidebar: true
---

## Migrate entire pool to new pool
If you have a pool which contains a lot of datasets and volumes backing LUN's migrating the pool in case of rebalancing, changing structure, or converting from (stripped) mirror to raidz(n) or vice versa this recipe can be of interest for you.

The idea behind the method of migrating an entire pool in the article is based on the idea that the contents of the pool you want to migrate is temporarily stored on another pool. This other pool can live locally on the same server as the pool you intend to migrate and therefore called a local temporary pool or it can live on other server in which case it is called a remote temporary pool. Each time the commands you are supposed to do differs whether the temporary pool is local or remote you choose the commands with the header:

Temporary pool local: **Local temporary pool**
Temporary pool remote: **Remote temporary pool**

In description below the following convention is:
local-host = Host where local pool exists.
remote-host = Host where remote pool exists.

*Please remember that you cannot change method along the line. If you choose* **Local temporary pool** *at the beginning you have to follow this route from start to end.*

you can migrate to both a local or a remote pool but I strongly recommend using a local pool because of the speed advantages which greatly influence downtime. I will however, show commands for both local and remote migration.

Since I do not use SMB then there is no example with SMB but I am very confident that it will work exactly the same way as a NFS share.

The current setup is shown below. For the sake of simplicity I use single disk pools but any pool structure, mirror, and raidz(n) can use this recipe.


``` terminal
local-host $ zpool status tank1 tank2
  pool: tank1
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        tank1       ONLINE       0     0     0
          c1t3d0    ONLINE       0     0     0

errors: No known data errors

  pool: tank2
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        tank2       ONLINE       0     0     0
          c1t4d0    ONLINE       0     0     0

errors: No known data errors
```

before rebooting into single user mode you need to save a list LUN's in case you need to change the name of the pool. As can be seen below every LUN is given a GUID which is referred to by the target and view and this GUID use an absolute path to point to the volume so this needs to be changed in that case. More about that later.

``` terminal
local-host $ sudo sbdadm list-lu

Found 1 LU(s)

              GUID                    DATA SIZE           SOURCE
--------------------------------  -------------------  ----------------
600144f9977e98a82116f276d021eba3  34359738368          /dev/zvol/rdsk/tank1/vm-100-disk-0
```

It is also advisable to save the NFS configuration.

``` terminal
local-host $ zfs get sharenfs tank1/nfs
NAME       PROPERTY  VALUE                                SOURCE
tank1/nfs  sharenfs  rw=@192.168.2.92,root=@192.168.2.92  local
```

The current contents on tank1 which we want to migrate.

``` terminal
local-host $ zfs list -r tank1
NAME                  USED  AVAIL  REFER  MOUNTPOINT
tank1                1.58G  13.4G    24K  /tank1
tank1/nfs             879M  13.4G   879M  /nfs
tank1/vm-100-disk-0   734M  13.4G   734M  -
```

When all this is done it is now time to reboot into single user mode.

After reboot into single user mode we start with creating a snapshot of the entire pool we want to migrate.

``` terminal
local-host $ sudo zfs snapshot -r tank1@replication

local-host $ zfs list -r -t snapshot tank1
NAME                              USED  AVAIL  REFER  MOUNTPOINT
tank1@replication                    0      -    24K  -
tank1/nfs@replication                0      -   879M  -
tank1/vm-100-disk-0@replication      0      -   734M  -
```

After making the a snapshot of the entire pool when temporarily send the entire pool contents to another pool. The pool needs to be configured prior to sending the data and it needs to have a size at least equal to the size of the pool we migrate.

**Local temporary pool**

``` terminal
local-host $ sudo zfs send -R tank1@replication | sudo zfs recv -Fdu tank2

local-host $ zfs list -r tank2
NAME                  USED  AVAIL  REFER  MOUNTPOINT
tank2                1.58G  13.4G    24K  /tank2
tank2/nfs             879M  13.4G   879M  /nfs
tank2/vm-100-disk-0   734M  13.4G   734M  -

local-host $ zfs list -r -t snapshot tank2
NAME                              USED  AVAIL  REFER  MOUNTPOINT
tank2@replication                    0      -    24K  -
tank2/nfs@replication                0      -   879M  -
tank2/vm-100-disk-0@replication      0      -   734M  - z
```

**Remote temporary pool**

``` terminal
local-host $ sudo zfs send -R tank1@replication | remote-host $ sudo zfs recv -Fdu tank2

remote-host $ zfs list -r tank2
NAME                  USED  AVAIL  REFER  MOUNTPOINT
tank2                1.58G  13.4G    24K  /tank2
tank2/nfs             879M  13.4G   879M  /nfs
tank2/vm-100-disk-0   734M  13.4G   734M  -

remote-host $ zfs list -r -t snapshot tank2
NAME                              USED  AVAIL  REFER  MOUNTPOINT
tank2@replication                    0      -    24K  -
tank2/nfs@replication                0      -   879M  -
tank2/vm-100-disk-0@replication      0      -   734M  - z
```

After we have insured our destination pool contains the same as our source pool we can safely destroy the replication snapshots on both source and destination pools as well as destroying the old source pool and recreate a new source pool. As stated at the beginning if you want to use another name for the new source pool at extra operation is needed which is shown at the end.

**Local temporary pool**

``` terminal
local-host $ sudo zfs destroy -r tank1@replication
local-host $ sudo zfs destroy -r tank2@replication

local-host $ sudo zpool destroy -f tank1

local-host $ sudo zpool create tank1 c1t3d0
```

**Remote temporary pool**

``` terminal
local-host $ sudo zfs destroy -r tank1@replication
remote-host $ sudo zfs destroy -r tank2@replication

local-host $ sudo zpool destroy -f tank1

local-host $ sudo zpool create tank1 c1t3d0
```

We should now have a new pool which awaits data from our temporary pool so we basically reverse the operation used prior.

**Local temporary pool**

``` terminal
local-host $ sudo zfs snapshot -r tank2@replication

local-host $ sudo zfs send -R tank2@replication | zfs recv -Fdu tank1

local-host $ zfs list -r tank1
NAME                  USED  AVAIL  REFER  MOUNTPOINT
tank1                1.58G  13.4G    24K  /tank2
tank1/nfs             879M  13.4G   879M  /nfs
tank1/vm-100-disk-0   734M  13.4G   734M  -

local-host $ zfs list -r -t snapshot tank1
NAME                              USED  AVAIL  REFER  MOUNTPOINT
tank1@replication                    0      -    24K  -
tank1/nfs@replication                0      -   879M  -
tank1/vm-100-disk-0@replication      0      -   734M  -
```

**Remote temporary pool**
``` terminal
remote-host $ sudo zfs snapshot -r tank2@replication

remote-host $ sudo zfs send -R tank2@replication | ssh <local-host> sudo zfs recv -Fdu tank1

local-host $ zfs list -r tank1
NAME                  USED  AVAIL  REFER  MOUNTPOINT
tank1                1.58G  13.4G    24K  /tank2
tank1/nfs             879M  13.4G   879M  /nfs
tank1/vm-100-disk-0   734M  13.4G   734M  -

local-host $ zfs list -r -t snapshot tank1
NAME                              USED  AVAIL  REFER  MOUNTPOINT
tank1@replication                    0      -    24K  -
tank1/nfs@replication                0      -   879M  -
tank1/vm-100-disk-0@replication      0      -   734M  -
```

We have now transferred our data back so we can now safely remove the replication snapshots in both pools.

**Local temporary pool**

``` terminal
local-host $ sudo zfs destroy -r tank1@replication
local-host $ sudo zfs destroy -r tank2@replication
```

**Remote temporary pool**

``` terminal
local-host $ sudo zfs destroy -r tank1@replication
remote-host $ sudo zfs destroy -r tank2@replication
```

To prevent mounting conflicts or having NFS datasets active and mounted on our temporary pool when must make some changes.

**Local temporary pool**

``` terminal
local-host $ sudo zfs set sharenfs=off tank2/nfs
local-host $ sudo zfs set mountpoint=/nfs2 tank2/nfs
```

**Remote temporary pool**

``` terminal
remote-host $ sudo zfs set sharenfs=off tank2/nfs
remote-host $ sudo zfs set mountpoint=/nfs2 tank2/nfs
```

If you changed the name of the pool you need to perform this extra step to have your LUN's available for your targets and view on the new pool. This is where the saved list of LUN's is needed. So for every LUN you need to perform the following:

``` terminal
local-host $ sudo stmfadm create-lu -p guid=600144f9977e98a82116f276d021eba3 /dev/zvol/rdsk/<name_for_new_tank1>/vm-100-disk-0
```

You can now reboot into multi user mode again and every thing should be available again.

To ensure everything is fine you can perform the checks below and compare the output to the output you created at the beginning using the same commands.


``` terminal
local-host $ sudo sbdadm list-lu

Found 1 LU(s)

              GUID                    DATA SIZE           SOURCE
--------------------------------  -------------------  ----------------
600144f9977e98a82116f276d021eba3  34359738368          /dev/zvol/rdsk/tank1/vm-100-disk-0

local-host $ zfs get sharenfs tank1/nfs
NAME       PROPERTY  VALUE                                SOURCE
tank1/nfs  sharenfs  rw=@192.168.2.92,root=@192.168.2.92  local
```

You can now safely destroy your temporary pool or stored it as a safety backup.

**Local temporary pool**

``` terminal
local-host $ sudo zpool destroy -f tank2
```

**Remote temporary pool**

``` terminal
remote-host $ sudo zpool destroy -f tank2
```

### Contact

Any problems or questions, please [get in touch](/about/contact.html).