---
layout: post
title:  "Ceph 簡單指令操作"
description: "This artical contains simple commands for Ceph management and operation"
date: 2017-05-26 06:00:00
comments: true
published: true
tags: 
  - Ceph
  - SDS
categories: 
  - Ceph
---


Monitoring and Health
=====================

剛安裝完，首先先檢查 ceph cluster 的狀態：

```bash
# 檢視 ceph cluster status
$ ceph -s
    cluster cd6fcb41-c373-48fb-aab3-f8d330a26ccb
     health HEALTH_WARN
            too few PGs per OSD (16 < min 30)
     monmap e1: 3 mons at {ceph01=10.102.41.101:6789/0,ceph02=10.102.41.102:6789/0,ceph03=10.102.41.103:6789/0}
            election epoch 8, quorum 0,1,2 ceph01,ceph02,ceph03
     osdmap e36: 12 osds: 12 up, 12 in
            flags sortbitwise,require_jewel_osds
      pgmap v84: 64 pgs, 1 pools, 0 bytes data, 0 objects
            405 MB used, 2174 GB / 2174 GB avail
                  64 active+clean


```
> 也可以用 `ceph -w` 檢視即時的狀態


檢查健康狀態：

```bash
$ ceph health detail
HEALTH_WARN too few PGs per OSD (16 < min 30)
too few PGs per OSD (16 < min 30)
```


接著檢視目前 ceph cluster 提供了多少容量可用：

```bash
# 檢視目前可用的容量、每個 pool 的使用狀況 & quota 等資訊
$ ceph df
GLOBAL:
    SIZE      AVAIL     RAW USED     %RAW USED 
    2174G     2174G         405M          0.02 
POOLS:
    NAME     ID     USED     %USED     MAX AVAIL     OBJECTS 
    rbd      0         0         0          724G           0

# 連到 CRUSH tree, 顯示 weight, variance, capacity ... etc
$ ceph osd df tree
ID WEIGHT  REWEIGHT SIZE  USE    AVAIL %USE VAR  PGS TYPE NAME       
-1 2.12384        - 2174G   405M 2174G 0.02 1.00   0 root default    
-2 0.70795        -  724G   135M  724G 0.02 1.00   0     host ceph01 
 0 0.17699  1.00000  181G 35200k  181G 0.02 1.02  11         osd.0   
 3 0.17699  1.00000  181G 34708k  181G 0.02 1.00  19         osd.3   
 6 0.17699  1.00000  181G 34420k  181G 0.02 0.99  14         osd.6   
 8 0.17699  1.00000  181G 34336k  181G 0.02 0.99  20         osd.8   
-3 0.70795        -  724G   135M  724G 0.02 1.00   0     host ceph03 
 1 0.17699  1.00000  181G 35568k  181G 0.02 1.03  17         osd.1   
 5 0.17699  1.00000  181G 34432k  181G 0.02 0.99  12         osd.5   
 9 0.17699  1.00000  181G 34272k  181G 0.02 0.99  18         osd.9   
11 0.17699  1.00000  181G 34200k  181G 0.02 0.99  17         osd.11  
-4 0.70795        -  724G   135M  724G 0.02 1.00   0     host ceph02 
 2 0.17699  1.00000  181G 35076k  181G 0.02 1.01  19         osd.2   
 4 0.17699  1.00000  181G 34652k  181G 0.02 1.00  18         osd.4   
 7 0.17699  1.00000  181G 34456k  181G 0.02 0.99  11         osd.7   
10 0.17699  1.00000  181G 34280k  181G 0.02 0.99  16         osd.10  
              TOTAL 2174G   405M 2174G 0.02                          
```


Working with Pools and OSDs
===========================

若要尋找單顆 OSD 的相關資訊：

```bash
# 尋找 OSD physical location
$ ceph osd find 1
{
    "osd": 1,
    "ip": "10.102.41.103:6800\/63011",
    "crush_location": {
        "host": "ceph03",
        "root": "default"
    }
}

# 顯示指定 OSD metadata
$ ceph osd metadata 1
{
    "id": 1,
    "arch": "x86_64",
    "back_addr": "10.102.41.103:6801\/63011",
    "backend_filestore_dev_node": "unknown",
    "backend_filestore_partition_path": "unknown",
    "ceph_version": "ceph version 10.2.5-37.el7cp (033f137cde8573cfc5a4662b4ed6a63b8a8d1464)",
    ......
    "osd_data": "\/var\/lib\/ceph\/osd\/ceph-1",
    "osd_journal": "\/var\/lib\/ceph\/osd\/ceph-1\/journal",
    "osd_objectstore": "filestore"
}
```

建立/移除 pool: 

```bash
# 建立 pool, 名稱為 pve_image, pg 數量為 1024
$ ceph osd pool create pve_images 1024
pool 'pve_images' created

# 顯示目前 pool 詳細狀態
$ ceph osd pool ls detail
pool 0 'rbd' replicated size 3 min_size 2 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 1 flags hashpspool stripe_width 0
pool 1 'pve_images' replicated size 3 min_size 2 crush_ruleset 0 object_hash rjenkins pg_num 1024 pgp_num 1024 last_change 37 flags hashpspool stripe_width 0

# 顯示指定 pool 的詳細資料
$ ceph osd pool get pve_images all
size: 3
min_size: 2
crash_replay_interval: 0
pg_num: 1024
pgp_num: 1024
crush_ruleset: 0
...


# 移除 pool (要重複 pool name 兩次還要加上那有趣的參數)
$ ceph osd pool delete pve_images pve_images --yes-i-really-really-mean-it
pool 'pve_images' removed
```

調整現有 pool 的狀態：

```bash
# 設定 placement groups 的數量
$ ceph osd pool set rbd pg_num 384
set pool 0 pg_num to 384

# pgp => The effective number of placement groups to use when calculating data placement.
$ ceph osd pool set rbd pgp_num 384
set pool 0 pgp_num to 384

# 調整完就會從原本的 warning 狀態變成 health_ok 了!
$ ceph -s
    cluster cd6fcb41-c373-48fb-aab3-f8d330a26ccb
     health HEALTH_OK
     monmap e1: 3 mons at {ceph01=10.102.41.101:6789/0,ceph02=10.102.41.102:6789/0,ceph03=10.102.41.103:6789/0}
            election epoch 8, quorum 0,1,2 ceph01,ceph02,ceph03
     osdmap e47: 12 osds: 12 up, 12 in
            flags sortbitwise,require_jewel_osds
      pgmap v177: 384 pgs, 1 pools, 0 bytes data, 0 objects
            461 MB used, 2174 GB / 2174 GB avail
                 384 active+clean
```


RBD Block Storage
=================

這個部份是用在把 Ceph 作為 block-based storage 時所需要了解的指令：

```bash
# 顯示目前的 RBD volume
$ rbd ls
vm-101-disk-1

# 顯示指定 RBD volume 的狀態
$ rbd status vm-101-disk-1
Watchers:
	watcher=10.102.70.124:0/3361738208 client.143192 cookie=140498849842176

# 顯示指定 RBD volume 的相關資訊
$ rbd info vm-101-disk-1
rbd image 'vm-101-disk-1':
	size 32768 MB in 8192 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.22f52238e1f29
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags: 

# 移除指定的 RBD volume
$ rbd rm vm-101-disk-1

# 建立名稱為 Jenkins_Data，大小為 40GB 的 RBD volume
$ rbd create --pool rbd --image Jenkins_Data --size 40960

# 檢視 RBD volume 資訊
$ rbd --image Jenkins_Data info
rbd image 'Jenkins_Data':
	size 40960 MB in 10240 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.a71d6238e1f29
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags: 

# 變更 RBD volume 大小，縮小要加上 "--allow-shrink" 以確保安全
$ rbd --image Jenkins_Data resize --size 10240 --allow-shrink
Resizing image: 100% complete...done.

# 確認 RBD volume 容量正確變更
$ rbd --image Jenkins_Data info
rbd image 'Jenkins_Data':
	size 10240 MB in 2560 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.a71d6238e1f29
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags: 

# 放大 RBD Volume
$ rbd --image Jenkins_Data resize --size 20480
Resizing image: 100% complete...done.

# 確認 RBD volume 容量正確變更
$ rbd --image Jenkins_Data info
rbd image 'Jenkins_Data':
	size 20480 MB in 5120 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.a71d6238e1f29
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags: 

```



Authentication and Authorization
================================

```bash
# 檢視目前可使用 ceph cluster 的 user list
$ ceph auth list
installed auth entries:

osd.0
	key: ABxxxxxxx
	caps: [mon] allow profile osd
	caps: [osd] allow *
......
client.admin
	key: ABxxxxxxx
	caps: [mds] allow *
	caps: [mon] allow *
	caps: [osd] allow *
client.bootstrap-mds
	key: ABxxxxxxx
	caps: [mon] allow profile bootstrap-mds
client.bootstrap-osd
	key: ABxxxxxxx
	caps: [mon] allow profile bootstrap-osd
client.bootstrap-rgw
	key: ABxxxxxxx
	caps: [mon] allow profile bootstrap-rgw
```





References
==========

- [Ceph From Scratch · GitBook](https://www.gitbook.com/book/tobegit3hub1/ceph_from_scratch/details)

- [Ceph Cheatsheet - sabaini.at](https://sabaini.at/pages/ceph-cheatsheet.html)

- [最新ceph集群常用命令梳理 - 康建华 - 51CTO技术博客](http://michaelkang.blog.51cto.com/1553154/1698287)

- [Pools — Ceph Documentation](http://docs.ceph.com/docs/jewel/rados/operations/pools/)

- [Placement Groups — Ceph Documentation](http://docs.ceph.com/docs/master/rados/operations/placement-groups/)