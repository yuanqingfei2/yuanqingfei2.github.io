---
title: SQL Server主键最大之后
date: "2019-05-16T15:45:00.000Z"
description: "在SQL Server数据库中如何应对ID最大化"
---
通常数据库的主键都有一个最大值，如果你有一个产品有幸运行足够长的时间，那么你也将“有幸”遇到这个问题

## 错误

```bash
Server: Msg 8115, Level 16, State 1, Line 1 Arithmetic overflow error converting IDENTITY to data type int. Arithmetic overflow occurred.
```

## 预防

```sql
-- define the max value for each data type
CREATE TABLE #DataTypeMaxValue (DataType varchar(50), MaxValue bigint)

INSERT INTO #DataTypeMaxValue VALUES 
   ('tinyint' , 255),
   ('smallint' , 32767),
   ('int' , 2147483647),
   ('bigint' , 9223372036854775807)

-- retrieve identity column information
SELECT 
   distinct OBJECT_NAME (IC.object_id) AS TableName,
   IC.name AS ColumnName,
   TYPE_NAME(IC.system_type_id) AS ColumnDataType,
   DTM.MaxValue AS MaxDataTypeValue,
   IC.seed_value IdentitySeed,
   IC.increment_value AS IdentityIncrement, 
   IC.last_value,
   DBPS.row_count AS NumberOfRows,
   (convert(decimal(18,2),CONVERT(bigint,IC.last_value)*100/DTM.MaxValue)) AS ReachMaxValuePercent 
FROM sys.identity_columns IC
   JOIN sys.tables TN ON IC.object_id = TN.object_id
   JOIN #DataTypeMaxValue DTM ON TYPE_NAME(IC.system_type_id)=DTM.DataType
   JOIN sys.dm_db_partition_stats DBPS ON DBPS.object_id =IC.object_id 
   JOIN sys.indexes as IDX ON DBPS.index_id =IDX.index_id 
WHERE DBPS.row_count >0 
ORDER BY ReachMaxValuePercent desc

DROP TABLE #DataTypeMaxValue
```

## 方法

* 先做备份

```sql
EXEC sp_rename 'XXX','XXX_20190515'
```

* 创建新表，这次注意使用bigint

* 删除掉备份表中的垃圾数据

如果数据量非常大而你的服务器比较弱，大服务器一次删除一百万是没有问题的。那么你可能需要下面的分批删除脚本。

```sql
DECLARE @BATCHSIZE INT, @WAITFORVAL VARCHAR(8), @ITERATION INT, @TOTALROWS INT, @MAXRUNTIME VARCHAR(8), @BSTOPATMAXTIME BIT, @MSG VARCHAR(500)
SET DEADLOCK_PRIORITY LOW;
SET @BATCHSIZE = 4000
SET @WAITFORVAL = '00:00:10'
SET @MAXRUNTIME = '08:00:00' -- 8AM
SET @BSTOPATMAXTIME = 1 -- ENFORCE 8AM STOP TIME
SET @ITERATION = 0 -- LEAVE THIS
SET @TOTALROWS = 0 -- LEAVE THIS

WHILE @BATCHSIZE>0
BEGIN
    -- IF @BSTOPATMAXTIME = 1, THEN WE'LL STOP THE WHOLE JOB AT A SET TIME...
    IF CONVERT(VARCHAR(8),GETDATE(),108) >= @MAXRUNTIME AND @BSTOPATMAXTIME=1
    BEGIN
        RETURN
    END

    DELETE TOP(@BATCHSIZE)
    FROM SOMETABLE
    WHERE 1=2

    SET @BATCHSIZE=@@ROWCOUNT
    SET @ITERATION=@ITERATION+1
    SET @TOTALROWS=@TOTALROWS+@BATCHSIZE
    SET @MSG = 'Iteration: ' + CAST(@ITERATION AS VARCHAR) + ' Total deletes:' + CAST(@TOTALROWS AS VARCHAR)
    RAISERROR (@MSG, 0, 1) WITH NOWAIT
    WAITFOR DELAY @WAITFORVAL 
END
```

* 使用[bcp](https://docs.microsoft.com/en-us/sql/tools/bcp-utility?view=sql-server-2017)导出备份表，然后导入新表

BCP OUT:

```bash
bcp MSADB.dbo.SecurityFirmBranches out %DATADIR%\input\oasys\SecurityFirmBranches.data       -c -I %BINDIR%\sqlini.txt -S %OASYSSERVER% -U %OASYSUSER% -P %OASYSPASS% -t^^
```

BCP IN:

```bash
bcp CVA_Staging.dbo.MCCRiskClass in %DATADIR%\input\mcc\risk_class.list.%bdwkly% -f %CONFIGDIR%\risk_class.fmt -b 10000 -U %ICVAUSER% -P %ICVAPASS% -S %ICVASERVER% -F 2
```

## RESEED

这个方法对于已经到达最大值的情况不适用。
```sql
DECLARE @maxVal INT
SELECT @maxVal = ISNULL(max(ID),0)+1 from mytable
DBCC CHECKIDENT('mytable', RESEED, @maxVal)
```

## 感谢

* https://www.mssqltips.com/sqlservertip/4375/monitor-sql-server-identity-column-values-to-prevent-arithmetic-overflow-errors/

* https://stackoverflow.com/questions/2923855/how-to-automatically-reseed-after-using-identity-insert

* https://stackoverflow.com/questions/24785439/deleting-1-millions-rows-in-sql-server
