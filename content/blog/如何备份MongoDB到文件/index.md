---
title: 如何备份MongoDB到文件
date: "2021-09-10T14:11:00.000Z"
description: "记录一下自己完成的经历"
---

大家都知道用mongodump或者mongoexport就可以了，可是真正执行起来，还是遇到不少问题。

## 如何传bash变量到query中去

在MongoDB的[官方文档](https://docs.mongodb.com/v4.2/reference/program/mongoexport/)中，明确提到了在写query的时候需要使用单引号，目的就是不要与bash的变量搞混了。

>You must enclose the query document in single quotes ('{ ... }') to ensure that it does not interact with your shell environment.

可是我的需求明确要用到环境变量，比如说我需要把时间大于180天的记录都备份到文件中去，这个过程，你需要保证你备份的和最后删除这两步用到的是一个时间变量。如果是单引号，是没有办法做的。但如果是双引号呢，如果不按照官方推荐来做，行不行呢？ 答案是可以的，而且必须这样做.

```bash
tss=`date --date="180 days ago" +%s`000
archiveFileName=`date -d @$( echo "($tss + 500) / 1000" | bc) +%Y%m%d`.json
/usr/local/bin/mongoexport --host $MONGOHOST --port $MONGOPORT --username $USER --password $PWD --authenticationDatabase admin --authenticationMechanism SCRAM-SHA-256 --db xxxArchive --ssl --sslAllowInvalidCertificates --collection=TradeArchive --query="{\"endDate\": {\"\$lt\": {\"\$date\": $tss}}}" --out=$QUATTRO_DATA/archive/tradeJsonData/$archiveFileName
```

上面这个脚本中，query这个参数使用的是双引号，这样`$tss`才能被识别。但是其他参数的的双引号就必须要转义符号了（经过测试，参数必须要有双引号），另外`$`也必须要有转义符号。

## 如何在query中使用日期

在query参数中，是不可以直接使用`new Date`或者`ISODate()`的，它只能识别relaxed模式的[extended-json-date](https://docs.mongodb.com/v4.2/reference/mongodb-extended-json/#extended-json-date)，也就是这个格式`{\"\$date\": $tss}}}`

## 如何删除

删除很简单，使用的是 **eval** 参数，和query参数类似，因为也要用环境变量，所以也要用双引号，但是它的日期就不是query要求的那么严格，可以使用正常的日期，所以就如下所示:

```bash
/usr/local/bin/mongo --host $MONGOHOST --port $MONGOPORT --username $USER --password $PWD --authenticationDatabase admin --authenticationMechanism SCRAM-SHA-256 --ssl --sslAllowInvalidCertificates cvaArchive  --eval "db.XXXArchive.deleteMany({\"endDate\": {\"\$lt\": new Date($tss)}})"
```

## 如何保证备份出错的时候终止程序

因为archive本质上就是先备份，再删除原有的过程，如果第一步出错了，就不要删除了。这里我采用的是`set -e`，具体可以参考感谢中的链接。

## 如何把备份的数据恢复到数据库

当你查看数据的时候，你发现，mongoexport 保存的是oid，这个可以使用`mongoimport`来恢复。但是如果直接用的话，不可以有人写了个程序可以完成`oid`到`ObjectId()`的转换。

```javascript
foreach ( $rows as $row ) {
    foreach ( array_keys($row) as $key) {
        if ( $key == "_id" ) {
            $row[$key] = new \MongoId( $row[$key]['$oid'] );
        }
    }
    $this->db->Users->insert( $row );
}
```

## 最后两个注意点

* 上面命令中都要有`--ssl` 否则就会出错`network error while attempting to run command 'isMaster' on host xxx`
* 上面命令中都要有`--sslAllowInvalidCertificates`，否则就会出错`Last error: connection() : x509: certificate signed by unknown authority }, ]`

## 感谢

* https://stackoverflow.com/questions/6996999/how-can-i-use-mongodump-to-dump-out-records-matching-a-specific-date-range

* https://stackoverflow.com/questions/6753330/specify-which-database-to-use-in-mongodb-js-script

* https://stackoverflow.com/questions/12362562/convert-milliseconds-timestamp-to-date-from-unix-command-line/12364979

* https://stackoverflow.com/questions/51042616/error-network-error-while-attempting-to-run-command-ismaster-on-host-127-0-0

* https://stackoverflow.com/questions/24797366/php-mongodb-oid-is-not-valid-for-storage-error-when-inserting-from-json-expor

* https://stackoverflow.com/questions/2870992/automatic-exit-from-bash-shell-script-on-error
