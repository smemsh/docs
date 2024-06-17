LVM Thin Data Usage
==============================================================================

Newer ``thin-provisioning-tools`` packages have a ``/usr/sbin/thin_ls``
which can be used to find out the total dirty blocks usage of a
copy-on-write LVM snapshot.  It's to be given a thin pool meta device::

  $ sudo thin_ls -m /dev/mapper/*_tmeta
  no current metadata snap

but then complains that it can't operate on live metadata without using
a snapshot (that's ``-m``).  We can make the snapshot by finding the
thinpool ``tpool`` mapping::

  $ ls /dev/mapper/*-tpool
  raidthin4-pool0-tpool

and using ``dmsetup`` messages to make a [specific metadata] snapshot::

  $ sudo dmsetup message /dev/mapper/*-tpool 0 reserve_metadata_snap

Now we can list the specific fields we find interesting, as shown in
``thin_ls(8)``::

  $ sudo thin_ls -m -o DEV,MAPPED,EXCLUSIVE,SHARED,CREATE_TIME,SNAP_TIME \
    /dev/mapper/*_tmeta
  DEV MAPPED EXCLUSIVE SHARED CREATE_TIME SNAP_TIME
    1 859GiB     29GiB 830GiB           0        11
    2 626GiB     12GiB 614GiB           1         1
    3 625GiB   4200MiB 621GiB           2         2
    4 625GiB   1829MiB 623GiB           3         3
    5 618GiB   3209MiB 615GiB           4         4
    6 770GiB   3618MiB 766GiB           5         5
    7 772GiB   2127MiB 769GiB           6         6
    8 771GiB   7670MiB 764GiB           7         7
    9 773GiB     10GiB 763GiB           8         8
   10 843GiB   1115MiB 842GiB           9         9
   11 843GiB    481MiB 843GiB          10        10
   12 846GiB   3577MiB 843GiB          11        11

then release the snapshot when done::

  $ sudo dmsetup message /dev/mapper/*-tpool 0 release_metadata_snap

.
