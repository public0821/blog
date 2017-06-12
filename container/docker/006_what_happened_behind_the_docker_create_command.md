## layerdb/mounts
root@dev:/var/lib/docker/image/aufs/layerdb/mounts/aafcf521d0a6e28311f5a7daf6fa5f05bd0b588d41ff13a1d53c202acf3b4e8c# ls
init-id  mount-id  parent

root@dev:/var/lib/docker/image/aufs/layerdb/mounts/aafcf521d0a6e28311f5a7daf6fa5f05bd0b588d41ff13a1d53c202acf3b4e8c# cat init-id
26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b-init
root@dev:/var/lib/docker/image/aufs/layerdb/mounts/aafcf521d0a6e28311f5a7daf6fa5f05bd0b588d41ff13a1d53c202acf3b4e8c# cat mount-id
26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b
root@dev:/var/lib/docker/image/aufs/layerdb/mounts/aafcf521d0a6e28311f5a7daf6fa5f05bd0b588d41ff13a1d53c202acf3b4e8c# cat parent
sha256:9840d207a275f956f3e634148044e63dc78df511fd72e22d8cb3dad57dc49bf6

root@dev:/var/lib/docker/image# cat ./aufs/layerdb/sha256/9840d207a275f956f3e634148044e63dc78df511fd72e22d8cb3dad57dc49bf6/diff
sha256:d8b353eb3025c49e029567b2a01e517f7f7d32537ee47e64a7eac19fa68a33f3

>新加的这两层layer只保存在layerdb/mounts下面，在layerdb/sha256目录下没有相关信息


## aufs/layers
root@dev:/var/lib/docker/aufs# cat ./layers/26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b-init
7938f2b32c53a9e0d3974f9579dd9dbb450202e1e11fe514e31556d4ea808c4e
4c10796e21c796a6f3d83eeb3613c566ca9e0fd0a596f4eddf5234b87955b3c8
fd0ba28a44491fd7559c7ffe0597fb1f95b63207a38a3e2680231fb2f6fe92bd
b656bf5f0688069cd90ab230c029fdfeb852afcfd0d1733d087474c86a117da3
1e83d2ea184e08eed978127311cc96498e319426abe2fb5004d4b1454598bd76

root@dev:/var/lib/docker/aufs# cat ./layers/26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b
26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b-init
7938f2b32c53a9e0d3974f9579dd9dbb450202e1e11fe514e31556d4ea808c4e
4c10796e21c796a6f3d83eeb3613c566ca9e0fd0a596f4eddf5234b87955b3c8
fd0ba28a44491fd7559c7ffe0597fb1f95b63207a38a3e2680231fb2f6fe92bd
b656bf5f0688069cd90ab230c029fdfeb852afcfd0d1733d087474c86a117da3
1e83d2ea184e08eed978127311cc96498e319426abe2fb5004d4b1454598bd76

root@dev:~# find /var/lib/docker/image/ -name cache-id|xargs grep 7938f2b32c53a9e0d3974f9579dd9dbb450202e1e11fe514e31556d4ea808c4e
/var/lib/docker/image/aufs/layerdb/sha256/9840d207a275f956f3e634148044e63dc78df511fd72e22d8cb3dad57dc49bf6/cache-id:7938f2b32c53a9e0d3974f9579dd9dbb450202e1e11fe514e31556d4ea808c4e

## diff
root@dev:/var/lib/docker/aufs/diff# tree 26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b
26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b

0 directories, 0 files

root@dev:/var/lib/docker/aufs/diff# tree 26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b-init/
26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b-init/
├── dev
│   └── console
└── etc
    ├── hostname
    ├── hosts
    ├── mtab -> /proc/mounts
    └── resolv.conf

2 directories, 5 files

root@dev:/var/lib/docker/aufs/diff# cd 26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b-init/
root@dev:/var/lib/docker/aufs/diff/26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b-init# ls -l dev/console
-rwxr-xr-x 1 root root 0 Jun 11 20:39 dev/console
root@dev:/var/lib/docker/aufs/diff/26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b-init# ls -l etc/
total 0
-rwxr-xr-x 1 root root  0 Jun 11 20:39 hostname
-rwxr-xr-x 1 root root  0 Jun 11 20:39 hosts
lrwxrwxrwx 1 root root 12 Jun 11 20:39 mtab -> /proc/mounts
-rwxr-xr-x 1 root root  0 Jun 11 20:39 resolv.conf

## mnt
root@dev:/var/lib/docker/aufs/mnt# tree 26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b
26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b

0 directories, 0 files
root@dev:/var/lib/docker/aufs/mnt# tree 26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b-init/
26d9289273f6c426f95a04cf8a873ce2cb599a26683319fcbcf0a7b506735d1b-init/

0 directories, 0 files

## /var/lib/docker/containers/
root@dev:/var/lib/docker/containers/aafcf521d0a6e28311f5a7daf6fa5f05bd0b588d41ff13a1d53c202acf3b4e8c# tree
.
├── checkpoints
├── config.v2.json
└── hostconfig.json

1 directory, 2 files

checkpoints跟容器的checkpoint有关，checkpoint在当前版本还是experimental状态。


1
down vote
Checkpoint/Restore allows to dump the state of all the processes (memory, file descriptors, sockets etc) to the filesystem, so that a running container can be resumed on the same host or a different one (therefore allowing live migration or a running container). Docker export is just about the file system state, and does not allow live migration of containers. The checpkoint/restore feature is available (albeit in experimental state) in docker 1.13.

