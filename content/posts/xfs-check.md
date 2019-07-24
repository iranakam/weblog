---
title:  XFSの検査で躓いた
date: 2018-12-06T00:00:00+09:00
tags: ["linux"]
draft: false
---

# 実行環境

CentOS Linux release 7.2.1511 (Core)

# アンマウント

まずは`/dev/sdb1`をオープンしているプロセスを確認します。  

```bash
lsof /dev/sdb1
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF       NODE NAME
smbd     2291 nobody  cwd    DIR   8,17       59 2147483780 /data/hoge
smbd     2291 nobody   31r   DIR   8,17       59 2147483780 /data/hoge
smbd     2291 nobody   41r   DIR   8,17       59 2147483780 /data/hoge
```

smbがオープンしているようなので停止します。  

```bash
systemctl stop smb.service
```

改めて`/dev/sdb1`をオープンしているプロセスを確認します。  

```bash
lsof /dev/sdb1
```

アンマウントします。  

```bash
umount /dev/sdb1 
umount: /data: target is busy.
        (In some cases useful info about processes that use
         the device is found by lsof(8) or fuser(1))
```

ビジーなので、`-l`を付けてアンマウントします。  

```bash
umount -l /data
```

マウントが外れたことを確認します。  

```bash
mount | grep sdb1
```

# 検査

検査を実行します。  

```bash
xfs_repair -v -n /dev/sdb1
xfs_repair: /dev/sdb1 contains a mounted and writable filesystem

fatal error -- couldn\'t initialize XFS library
```

マウントされている旨のエラーが出力されたのでマウント状況を確認します。  

```bash
mount | grep data
```

ここで躓きました。
如何にもアンマウントされているように見えますが...  

[lvm - Why won&#39;t xfs_check run? - Ask Ubuntu](https://askubuntu.com/questions/2054/why-wont-xfs-check-run)

アンマウントはされていますが、nfsがオープンしているようです。  

nfsのサービスを停止します。  

```bash
systemctl stop nfs-server.service
```

改めて検査を実行します。  

```bash
xfs_repair -v -n /dev/sdb1
Phase 1 - find and verify superblock...
...
```

検査が終わったので、マウントします。  

```bash
mount /dev/sdb1 /data
```

停止していたサービスを開始します。  

```bash
systemctl start nfs-server.service
systemctl start smb.service
```

# Lazyなので...

それで、結局は何だったかというと

[Man page of UMOUNT](http://linuxjm.osdn.jp/html/LDP_man-pages/man2/umount.2.html)

> 遅延umountを行う。マウントポイントに対する新規のアクセスは 不可能となり、実際のumountはマウントポイントがビジーでなくなった時点で行う。

[Linuxでlazy umountしたときに厳密にumountされたタイミングを知る方法 - らるるの自宅と職場を往復する人生＠それをすてるなんてとんでもない！](http://www.rarul.com/mt/log/2016/07/linuxlazy-umoun.html)

> そうです、umount -lは成功してもまだ本当にはumountしてません。これをした以後にプロセスが新しく参照することができなくなり、/proc/mountなどからも見えなくなるものの、それ以前に参照していたプロセスは引き続き参照し続けれるんです。で、参照するプロセスがいなくなった時点ではじめて、umountの実処理が行われるわけです。

安易でした…
