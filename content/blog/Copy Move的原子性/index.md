---
title: Copy Move的原子性
date: "2019-08-01T14:25:00.000Z"
description: "在如何保证文件完整性上的摸索"
---

最近在使用一个Linux脚本，这个脚本本意就是把文件拷贝并压缩，本来没有什么问题，可是后来发现，无论是拷贝还是压缩，都没有做到原子性，换句话说，如果有有另外一个进程要读拷贝过或者压缩过的文件，都会出现不完整的读。

原本的脚本：

```bash
cp -f "$xmlFile" "${quattroFile}"
gzip "${quattroFile}"
```

更改后:

```bash
cp -f "$xmlFile" "${quattroFile}.~tmp"
mv -f "${quattroFile}.~tmp" "${quattroFile}~"	# keep ~ just for NOT fit file name pattern
gzip "${quattroFile}~"
finalQuattroFile=${quattroFile}.gz
mv -f "${quattroFile}~.gz" $finalQuattroFile
echo "    cp $xmlFile to ${quattroFile}~.gz then to $finalQuattroFile"	
```

第二步中只所以要保留一个特殊字符，是为了不让别的进程读它（别的进程以filename来确定读取与否）。再仔细看下，我们这两个脚本最终都是要得到一个gz压缩文件，第一个拷贝发生在一个文件系统中，所以通过使用mv(rename)来达到这个操作的[原子性](https://rcrowley.org/2010/01/06/things-unix-can-do-atomically.html)，也就是文件肯定是完整的，否则不会重命名成功。第二个也是，给gzip足够的时间来压缩成功，保证最后的mv(rename)也成功。其实主要区别就在多出来的这两个mv(rename),是它保证了文件的完整性。


rsync 是另一个经常使用的命令，某些场景下比cp更合适，可是它也[不是原子性的](https://stackoverflow.com/questions/3769263/are-rsync-operations-atomic-at-file-level)。

**cp or rsync?**

### cp

>Make copy of files and directories. This is a more "higher" level of abstraction, where you can copy directories recursively, without caring about block size, file conversion, etc. It is a better tool to deal with "1-to-many" cases of file copy, ownership, symbolic link follow, recursively copy, and verbosity. However, it has it's limitations like dealing with file changes, remote copy, and those things better handled by rsync.

### rsync

>Can to copy files inside the same computer, but it's features are more useful on remote copy scenarios. Some of the features are ownership handling/manipulation, more easy "exclude" expressions for a better copy, file checksum to see if a file was already copied, delete origin files during or after copu, the use of a "transparent shell" by invoking the protocol wanted using a specific URI(ssh://, rsync://...), pipelining and other stuff that create optimized environment for remote mirroring things.


## 感谢

* https://unix.stackexchange.com/questions/153804/what-is-a-good-strategy-to-generate-and-copy-files-atomically

* https://unix.stackexchange.com/questions/381741/do-cp-and-tar-create-temporary-files-and-then-do-an-atomic-mv

* https://rcrowley.org/2010/01/06/things-unix-can-do-atomically.html
