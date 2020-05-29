---
categories:
- 技术文章
- 云计算
date: 2016-12-17T09:53:44+08:00
description: ""
keywords:
- ceph, rbd, openstack
title: Explore ceph rbd
author: jieteki
url: ""
---

It's now common case when use ceph as the storage backend for openstack,
cinder, glance, nova all these componets can share the same ceph cluster, make it extreamly fast for nova to boot an
instance from glance.

<!--more-->

\# source openrc admin admin

EXPLORE IMAGES
--------------

list glance images, there are three by default

```sh
stack@starya:~/devstack$ glance image-list
+--------------------------------------+---------------------------------+
| ID                                   | Name                            |
+--------------------------------------+---------------------------------+
| 4e7659d7-c5d5-4036-ba41-b8d8415b6eeb | cirros-0.3.4-x86_64-uec         |
| 1ac00d53-3475-4eb8-a4d2-83def42e469d | cirros-0.3.4-x86_64-uec-kernel  |
| 52372aa2-f8af-4a0d-aea1-3f6da6ca67c9 | cirros-0.3.4-x86_64-uec-ramdisk |
```

list rbd pools, the imajeez pool is used by glance. volumeuh pool is used by cinder, vmz pool is used by nova.

NOTICE:

everything in a image from the rbd view. so volumes are images in volumeuh, instance images are images in vmz, glance images are images in imajeez.

```sh
stack@starya:~$ sudo ceph df
GLOBAL:
    SIZE       AVAIL      RAW USED     %RAW USED
    10230M     10036M         193M          1.89
POOLS:
    NAME         ID     USED       %USED     MAX AVAIL     OBJECTS
    data         0           0         0        10036M           0
    metadata     1           0         0        10036M           0
    rbd          2           0         0        10036M           0
    backeups     3           0         0        10036M           0
    imajeez      4      33091k      0.32        10036M          12
    vmz          5           0         0        10036M           0
    volumeuh     6          32         0        10036M           5
```

list pool images in imajeez, three images. corespond to glance image list, the image name here is image id in glance view.

```sh
stack@starya:~/devstack$ rbd ls imajeez
1ac00d53-3475-4eb8-a4d2-83def42e469d
4e7659d7-c5d5-4036-ba41-b8d8415b6eeb
52372aa2-f8af-4a0d-aea1-3f6da6ca67c9
```

show glance image detail. direct\_url is a rbd:// protocol, ended with snap. which means the image in glance is a snapshot of image in rbd. the snapshot is created by default.

```sh
stack@starya:~/devstack$ glance image-show 4e7659d7-c5d5-4036-ba41-b8d8415b6eeb
+------------------+----------------------------------------------------------------------------------+
| Property         | Value                                                                            |
+------------------+----------------------------------------------------------------------------------+
| checksum         | eb9139e4942121f22bbc2afc0400b2a4                                                 |
| container_format | ami                                                                              |
| created_at       | 2016-08-27T10:14:41Z                                                             |
| direct_url       | rbd://58bd8c9c-d62d-4747-9d5c-                                                   |
|                  | ebdc02c9b7d8/imajeez/4e7659d7-c5d5-4036-ba41-b8d8415b6eeb/snap                   |
| disk_format      | ami                                                                              |
| id               | 4e7659d7-c5d5-4036-ba41-b8d8415b6eeb                                             |
| kernel_id        | 1ac00d53-3475-4eb8-a4d2-83def42e469d                                             |
| min_disk         | 0                                                                                |
| min_ram          | 0                                                                                |
| name             | cirros-0.3.4-x86_64-uec                                                          |
| owner            | 270f4f7cc4184db4a2cb48357d414629                                                 |
| protected        | False                                                                            |
| ramdisk_id       | 52372aa2-f8af-4a0d-aea1-3f6da6ca67c9                                             |
| size             | 25165824                                                                         |
| status           | active                                                                           |
| tags             | []                                                                               |
| updated_at       | 2016-08-27T10:14:41Z                                                             |
| virtual_size     | None                                                                             |
| visibility       | public                                                                           |
+------------------+----------------------------------------------------------------------------------+
```

get info of the rbd image, and it’s snaps

```sh
stack@starya:~/devstack$ rbd info imajeez/4e7659d7-c5d5-4036-ba41-b8d8415b6eeb
rbd image '4e7659d7-c5d5-4036-ba41-b8d8415b6eeb':
	size 24576 kB in 3 objects
	order 23 (8192 kB objects)
	block_name_prefix: rbd_data.372283fd80f
	format: 2
	features: layering, striping
	stripe unit: 4096 kB
	stripe count: 1
stack@starya:~/devstack$ rbd snap ls imajeez/4e7659d7-c5d5-4036-ba41-b8d8415b6eeb
SNAPID NAME     SIZE
     4 snap 24576 kB
```

could NOT delete imajeez image, because it has snapshot

```sh
stack@starya:~/devstack$ rbd rm imajeez/4e7659d7-c5d5-4036-ba41-b8d8415b6eeb
2016-08-28 17:23:34.139435 7f11e5f6f7c0 -1 librbd: image has snapshots - not removing
Removing image: 0% complete...failed.
rbd: image has snapshots - these must be deleted with 'rbd snap purge' before the image can be removed.
```

EXPLORE VOLUMES AND SNAPSHOTS
-----------------------------



create three volumes, and there snapshots

![](http://image.bitbut.com/257BC4C435262D0D789E3430378BD3B7.jpg)

![](http://image.bitbut.com/039F06063E6A339E2CF743D3D25C2907.jpg)

```sh
vol-a ====> snap-a
vol-a ====> snap-b
vol-m ====> snap-m
cirros-0.3.4-x86_64-uec =====> vol-cirros
```

list in cinder

```sh
stack@starya:~/devstack$ cinder list
+--------------------------------------+-----------+------------+------+-------------+----------+-------------+
|                  ID                  |   Status  |    Name    | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+------------+------+-------------+----------+-------------+
| 48eef4de-e725-489c-ae06-725e0cfc78c6 | available | vol-cirros |  2   |     ceph    |   true   |             |
| 554830b3-81fb-4423-b186-c41974e0b899 | available |   vol-m    |  1   |     ceph    |  false   |             |
| a60139fa-835c-4f83-80f7-7a32e9507f44 | available |   vol-a    |  3   |     ceph    |  false   |             |
+--------------------------------------+-----------+------------+------+-------------+----------+-------------+
```

list in rbd, notice the volume ID and rbd NAME are relevant

```sh
stack@starya:~/devstack$ rbd ls volumeuh
volume-48eef4de-e725-489c-ae06-725e0cfc78c6
volume-554830b3-81fb-4423-b186-c41974e0b899
volume-a60139fa-835c-4f83-80f7-7a32e9507f44
```

let’s see the vol-a has two snapshots in rdb

```sh
stack@starya:~/devstack$ rbd info volumeuh/volume-a60139fa-835c-4f83-80f7-7a32e9507f44
rbd image 'volume-a60139fa-835c-4f83-80f7-7a32e9507f44':
	size 3072 MB in 768 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.37a475b31fd5
	format: 2
	features: layering, striping
	stripe unit: 4096 kB
	stripe count: 1
stack@starya:~/devstack$ rbd snap ls volumeuh/volume-a60139fa-835c-4f83-80f7-7a32e9507f44
SNAPID NAME                                             SIZE
     2 snapshot-7712c4ac-b325-446b-bb27-65c58235e175 1024 MB
     3 snapshot-de943b3b-6039-4776-b560-0e36b76eda86 1024 MB
```

can not delete a snapshot from rbd

```sh
stack@starya:~/devstack$ rbd snap rm volumeuh/volume-a60139fa-835c-4f83-80f7-7a32e9507f44@snapshot-de943b3b-6039-4776-b560-0e36b76eda86
rbd: snapshot 'snapshot-de943b3b-6039-4776-b560-0e36b76eda86' is protected from removal.
2016-08-28 18:17:50.566728 7f86038957c0 -1 librbd: removing snapshot from header failed: (16) Device or resource busy
```

but can delete from horizon. after delete, ls snapshot again.

```sh
stack@starya:~/devstack$ rbd snap ls volumeuh/volume-a60139fa-835c-4f83-80f7-7a32e9507f44
SNAPID NAME                                             SIZE
     2 snapshot-7712c4ac-b325-446b-bb27-65c58235e175 1024 MB
```

create another volume from the snapshot, then the volume become the snapshot’s children

```sh
stack@starya:~/devstack$ rbd children volumeuh/volume-a60139fa-835c-4f83-80f7-7a32e9507f44@snapshot-7712c4ac-b325-446b-bb27-65c58235e175
volumeuh/volume-72690574-9cac-45c5-9a84-d040af987ed0
```

![](http://image.bitbut.com/044B59EA2734D6372F749EF90125A5A1.png)

Note

The terms “parent” and “child” mean a Ceph block device snapshot (parent), and the corresponding image cloned from the snapshot (child). These terms are important for the command line usage below.

GETTING START WITH LAYERING
---------------------------

Ceph block device layering is a simple process. You must have an image. You must create a snapshot of the image. You must protect the snapshot. Once you have performed these steps, you can begin cloning the snapshot.

![](http://image.bitbut.com/3BB62904DD4E6216F7D0F432FBE85026.png)

let’s see the info of the volume just created from snapshot

```sh
stack@starya:~/devstack$ rbd info volumeuh/volume-72690574-9cac-45c5-9a84-d040af987ed0
rbd image 'volume-72690574-9cac-45c5-9a84-d040af987ed0’:
	size 1024 MB in 256 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.42cf22a520f
	format: 2
	features: layering, striping
	parent: volumeuh/volume-a60139fa-835c-4f83-80f7-7a32e9507f44@snapshot-7712c4ac-b325-446b-bb27-65c58235e175
	overlap: 1024 MB
	stripe unit: 4096 kB
 	stripe count: 1
```

delete the snapshot from horizon, shows ERROR snapshot is busy. because it has children.

```sh
2016-08-28 18:48:33.507 ERROR cinder.volume.manager [req-3b0a5b92-bdc6-477f-96bc-06e2e866b000 d31901d9095549419b6bc06b0c48d39e 270f4f7cc4184db4a2cb48357d41462
9] Delete snapshot failed, due to snapshot busy.
```

flatten the volume, then get info again, the volume is concret volume, and has no relation to the snapshot anymore.

```sh
stack@starya:~/devstack$ rbd flatten volumeuh/volume-72690574-9cac-45c5-9a84-d040af987ed0
Image flatten: 100% complete…done.
stack@starya:~/devstack$ rbd info volumeuh/volume-72690574-9cac-45c5-9a84-d040af987ed0
rbd image 'volume-72690574-9cac-45c5-9a84-d040af987ed0':
	size 1024 MB in 256 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.42bf36ce8449
	format: 2
	features: layering, striping
	stripe unit: 4096 kB
 	stripe count: 1
```

delete the snapshot from horizon, snapshot deleted successfuly.

```sh
2016-08-28 18:50:10.239 INFO cinder.volume.manager [req-4079a7f6-ae1e-40a3-8205-f6b47a889b66 d31901d9095549419b6bc06b0c48d39e 270f4f7cc4184db4a2cb48357d414629] Delete snapshot completed successfully
```


ROLLBACK
--------

rollback snapshot1 means overriting volume1 with snapshot1, but clone snapshot1 to volume2 is faster and prefered.

```
rbd --pool rbd snap rollback --snap snapname foo
    rbd snap rollback rbd/foo@snapname
```

NOTICE:
Rolling back an image to a snapshot means overwriting the current version of the image with data from a snapshot. The time it takes to execute a rollback increases with the size of the image. It is **faster to clone** from a snapshot **than to rollback** an image to a snapshot, and it is the preferred method of returning to a pre-existing state.



links:

<http://docs.ceph.com/docs/hammer/rbd/rbd-snapshot/><br/>
<http://www.cnblogs.com/sammyliu/p/4838138.html>

