---
title: SSH Key的那些事
date: "2020-10-14T11:20:00.000Z"
description: "SSH Key的那些事"
---

最近公司服务器升级，各个系统都从RH5升级到RH7，所有的SSH key都要重新装一遍，否则之前那些scp,sftp都不工作了。中间遇到了各种事情，这里略微记录一下。

基本上，如果你需要拷贝文件到一个远程的服务器(uploading)或者从一个远程服务器上拷贝文件到本地(downloading)，这个过程你不想每次都输入密码，那么你需要把你这个服务器上的public key安装到远程服务器上。这个就是一个典型的public/private key的应用，具体可以参考[维基百科](https://en.wikipedia.org/wiki/Public-key_cryptography)

* cipher

当远程服务器已经从RH5升级到RH7，而本地服务器还是RH5的时候，就需要这样的参数了。 如下面所示：

```bash
/usr/bin/scp -c aes128-ctr ${remoteServer}/${remoteFile}.csv ${localFolder}/${localNamePart1}.${localNamePart2}.csv
```

* StrictHostKeyChecking

当远程服务器的名字是一个集群名字而非某一个具体机器的时候，需要这个参数，如下所示：

```bash
/usr/bin/scp -o "StrictHostKeyChecking=no" ${localFolder}/${localNamePart0}.${localNamePart2}.csv ${remoteServer}/${remoteFile}.csv
```

另外一种方式就是在那个用户的.ssh目录下面创建一个config文件，文件内容如下：

```txt
Host *
   StrictHostKeyChecking no
   UserKnownHostsFile=/dev/null
```

这样所有的命令都不需要再额外加参数了。

* convert

如果要安装远程服务器的key但是提供的格式是SSH2的，而你需要的是OPENSSH的格式，可以使用下面的命令进行转换:

```bash
ssh-keygen -i -f ./xxx@xxx_prod.pub > ./xxx@xxx_prod_openssh.pub
```

具体请参考[这里](https://tutorialinux.com/convert-ssh2-openssh/)

* sftp replace scp

即便有了如上措施，还有可能遇到那个账户只能使用sftp权限而不能使用scp的情况，这个时候你必须使用sftp来一步完成任务，而不是像通常那样进行来回交互。 具体分uploading和downloading两种情况:

downloading:

```bash
sftp {user}@{host}:{remoteFileName} {localFileName}
```

uploading:

```bash
sftp ${REMOTE_SERVER}${REMOTE_DIR} <<< $"put ${LOCAL_HOME}/data/input/${LOCAL_FILE}" 
```

要注意上面这个命令，后面只能用双引号，不能用单引号，因为引号里面有环境变量，双引号才会解释出来。

更多信息请阅读[这里](https://stackoverflow.com/questions/16721891/single-line-sftp-from-terminal)