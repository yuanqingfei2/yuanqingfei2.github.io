一朋友是摄影爱好者，遇到一个问题，在导出图片文件时，文件的创建日期不对，问我能不能用文件名中的日期来更改。询问得之，是windows10系统，而且文件几百上千。

解决方案一就是使用现成的工具[BulkFileChanger](https://www.nirsoft.net/utils/bulk_file_changer.html)，这是款免费软件，可以很容易就更改，可是考虑到文件极多，必须使用编程方式。

经过一点搜索，决定使用Win10自带的PowerShell，程序如下：

```powershell
$Pattern = '(\d{4})-(\d{2})-(\d{2}) (\d{2})-(\d{2})-(\d{2})\..*'
Get-ChildItem | ForEach {
    #Make sure the file matches the pattern
    If ($_.Name -match $Pattern) {
        $Date = $Matches[2],$Matches[3],$Matches[1] -join '/'
        $Time = $Matches[4..6] -join ':'
		$_.creationtime=$(Get-Date "$Date $Time")
    }
}
```

使用方法很简单，打开PowShell，切换到图片文件夹，一次性拷贝上面的全部代码，粘贴在Powshell命令行，然后执行即可。

感谢：

1. https://www.ghacks.net/2017/10/09/how-to-edit-timestamps-with-windows-powershell/

2. https://stackoverflow.com/questions/32401486/extract-timestamp-from-filename-and-sort
