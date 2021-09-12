---
title: JS写CSV
date: "2019-07-12T17:27:00.000Z"
description: "自己在写CSV过程中的一点小曲折"
---

在JS框架盛行的现在，临时输出一个CSV文件让用户下载，是个很平常的需求。我遇到了一点问题。

## 问题

你有一个json对象的数组，需要输出作为csv供用户下载。

`[{...}, {...}, {...}]`

## 通常模式

```javascript
const replacer = (key, value) => value === null ? '' : value; // specify how you want to handle null values here
const header = Object.keys(data[0]);
var csv = data.map(row => header.map(fieldName => JSON.stringify(row[fieldName], replacer)).join(','));
csv.unshift(header.join(','));
csv = csv.join('\r\n');
$scope.downloadCsv = 'data:text/csv;charset=utf-8,' + encodeURI(csv);
```

在加上如下的html即可。

```html
	<div ng-if="xxx">
		Please download <a href={{$scope.downloadCsv}} download="haicurt.csv" target="_blank">here</a> if you want all.
	</div>
```

## 高级模式

可是，如果你的csv文件内容太大，Chrome浏览器就会说`Network Error`导致你没有办法下载。解决办法是使用`BLOB`.

```javascript
const replacer = (key, value) => value === null ? '' : value; // specify how you want to handle null values here
const header = Object.keys(data[0]);
var csv = data.map(row => header.map(fieldName => JSON.stringify(row[fieldName], replacer)).join(','));
csv.unshift(header.join(','));
csv = csv.join('\r\n');
var csvData = new Blob([csv], {type: 'text/csv'});
ctrl.downloadCsv = URL.createObjectURL(csvData);
```


## 感谢

* https://stackoverflow.com/questions/8847766/how-to-convert-json-to-csv-format-and-store-in-a-variable

* https://stackoverflow.com/questions/23301467/javascript-exporting-large-text-csv-file-crashes-google-chrome



