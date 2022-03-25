# 命令行工具运行 SQL 查询 JSON, CSV, Excel, Parquet 等。

这是一个 CLI 伴侣 [DataStation](https://github.com/multiprocessio/datastation)(一个 GUI)，
用于对数据文件运行 SQL 查询。因此，如果您想要这个的 GUI 版本，请查看DataStation。



## 安装

Binaries for amd64 (x86_64) are provided for each release.

### macOS Homebrew

`dsq` is available on macOS Homebrew:

```bash
$ brew install dsq
```

### Binaries on macOS, Linux, WSL

On macOS, Linux, and WSL you can run the following:

```bash
$ curl -LO "https://github.com/multiprocessio/dsq/releases/download/0.8.1/dsq-$(uname -s | awk '{ print tolower($0) }')-x64-0.8.1.zip"
$ unzip dsq-*-0.8.1.zip
$ sudo mv dsq /usr/local/bin/dsq
```

Or install manually from the [releases
page](https://github.com/multiprocessio/dsq/releases), unzip and add
`dsq` to your `$PATH`.

### Binaries on Windows (not WSL)

Download the [latest Windows
release](https://github.com/multiprocessio/dsq/releases), unzip it,
and add `dsq` to your `$PATH`.

### Build and install from source

If you are on another platform or architecture or want to grab the
latest release, you can do so with Go 1.18+:

```bash
$ go install github.com/multiprocessio/dsq@latest
```

`dsq` will likely work on other platforms that Go is ported to such as
AARCH64 and OpenBSD, but tests and builds are only run against x86_64
Windows/Linux/macOS.

## 用法

你可以将数据传输到 `dsq`，也可以将文件名传递给它。注意:管道数据不能在 Windows 上工作。

如果您要传递一个文件，它必须具有其内容类型的常用扩展名。

例如:

```bash
$ dsq testdata.json "SELECT * FROM {} WHERE x > 10"
```

或者:

```bash
$ dsq testdata.ndjson "SELECT name, AVG(time) FROM {} GROUP BY name ORDER BY AVG(time) DESC"
```

### Pretty print

默认情况下，`dsq` 输出丑陋的 JSON。这是最有效的方式。

```bash
$ dsq testdata/userdata.parquet 'select count(*) from {}'
[{"count(*)":1000}
]
```

如果你想要更漂亮的 JSON，你可以管道 `dsq` 到 `jq`。

```bash
$ dsq testdata/userdata.parquet 'select count(*) from {}' | jq
[
  {
    "count(*)": 1000
  }
]
```

或者你可以在 `dsq` 中使用 `-p` 或 `--pretty` 来启用 pretty 打印，这将以 ASCII 表的形式显示你的结果。

```bash
$ dsq --pretty testdata/userdata.parquet 'select count(*) from {}'
+----------+
| count(*) |
+----------+
|     1000 |
+----------+
```

### 管道数据到dsq

当管道数据到 `dsq` 时，你需要设置 `-s` 标志并指定文件扩展名或 MIME 类型。

例如:

```bash
$ cat testdata.csv | dsq -s csv "SELECT * FROM {} LIMIT 1"
```

或者:

```bash
$ cat testdata.parquet | dsq -s parquet "SELECT COUNT(1) FROM {}"
```

### 多个文件和 Join

您可以将多个文件传递给 DSQ。
只要它们是受支持的、格式合法的数据文件，就可以将所有文件作为表运行 SQL。
每个表都可以通过字符串 `{N}` 访问，其中 `N` 是在命令行上传递的文件列表中基于 0 的文件索引。

For example this joins two datasets of differing origin types (CSV and JSON).
例如，它连接两个不同来源类型的数据集(CSV和JSON)。

```bash
$ dsq testdata/join/users.csv testdata/join/ages.json \
  "select {0}.name, {1}.age from {0} join {1} on {0}.id = {1}.id"
[{"age":88,"name":"Ted"},
{"age":56,"name":"Marjory"},
{"age":33,"name":"Micah"}]
```

你也可以给文件表名别名，因为 `dsq` 使用标准 SQL:

```bash
$ dsq testdata/join/users.csv testdata/join/ages.json \
  "select u.name, a.age from {0} u join {1} a on u.id = a.id"
[{"age":88,"name":"Ted"},
{"age":56,"name":"Marjory"},
{"age":33,"name":"Micah"}]
```

### 将数据转换为JSON而不需要查询

作为 `dsq testdata.csv "SELECT * FROM {}"` 的缩写，将支持的文件类型转换为 JSON，您可以跳过查询，转换后的 JSON 将转储到标准输出。

例如：

```bash
$ dsq testdata.csv
[{...some csv data...},{...some csv data...},...]
```

### 嵌套在对象内的对象数组

DataStation 和 dsq 的 SQL 集成操作一个对象数组。
如果你的对象数组恰好在顶层，你不需要做任何事情。
但是如果你的数组数据嵌套在一个对象中，你可以添加一个 `path` 参数到表引用。

例如你有这样的数据：

```bash
$ cat api-results.json
{
  "data": {
    "data": [
      {"id": 1, "name": "Corah"},
      {"id": 3, "name": "Minh"}
    ]
  },
  "total": 2
}
```

你需要告诉 `dsq` 数组数据的路径是 `data.data`:

```bash
$ dsq --pretty api-results.json 'SELECT * FROM {0, "data.data"} ORDER BY id DESC'
+----+-------+
| id | name  |
+----+-------+
|  3 | Minh  |
|  1 | Corah |
+----+-------+
```

如果你只有一个表，你也可以使用简写 `{"path"}` 或者 `{'path'}`:

```bash
$ dsq --pretty api-results.json 'SELECT * FROM {"data.data"} ORDER BY id DESC'
+----+-------+
| id | name  |
+----+-------+
|  3 | Minh  |
|  1 | Corah |
+----+-------+
```

可以对路径使用单引号或双引号。

#### 多个 Excel sheet

具有多个工作表的 Excel 文件被存储为一个对象，键是工作表名称，值是工作表数据作为一个对象数组。

如果你有一个 Excel 文件，里面有两个表格，分别叫做 Sheet1 和 Sheet2，你可以通过指定表格名称作为路径，在第二个表格上运行 `dsq`:

```bash
$ dsq data.xlsx 'SELECT COUNT(1) FROM {"Sheet2"}'
```

#### 限制:嵌套的数组
    

不能指定通过数组的路径，只能指定对象。

### 嵌套对象值

展示一个例子是最简单的。假设你有以下 JSON 文件名为 `user_addresses.json`:

```bash
$ cat user_addresses.json
[
  {"name": "Agarrah", "location": {"city": "Toronto", "address": { "number": 1002 }}},
  {"name": "Minoara", "location": {"city": "Mexico City", "address": { "number": 19 }}},
  {"name": "Fontoon", "location": {"city": "New London", "address": { "number": 12 }}}
]
```

你可以像这样查询嵌套的字段:

```sql
$ dsq user_addresses.json 'SELECT name, "location.city" FROM {}'
```

如果你需要消除 table 的歧义:

```sql
$ dsq user_addresses.json 'SELECT name, {}."location.city" FROM {}'
```

#### Caveat: PowerShell, CMD.exe

On PowerShell and CMD.exe you must escape inner double quotes with backslashes:

```powershell
> dsq user_addresses.json 'select name, \"location.city\" from {}'
[{"location.city":"Toronto","name":"Agarrah"},
{"location.city":"Mexico City","name":"Minoara"},
{"location.city":"New London","name":"Fontoon"}]
```

#### 嵌套对象解释

嵌套的对象被折叠，它们的新列名成为 `.` 连接值的 JSON 路径。
路径中的实际点必须用反斜杠进行转义。
因为 `.` 在 SQL 中是一个特殊的字符，您必须引用整个新的列名。

#### 限制:嵌套的数组

数组内嵌套的对象仍然被' dsq '忽略/删除。
如果你有这样的数据:

```json
[
  {"field1": [1]},
  {"field1": [2]},
]
```

您不能访问 `field1` 中的任何数据。
您将需要使用其他工具对数据进行预处理。

#### 限制:整个对象检索

不能查询整个对象，必须请求产生标量值的特定路径。

例如在 `user_addresses.json` 示例上面你不能这样做:

```sql
$ dsq user_addresses.json 'SELECT name, {}."location" FROM {}'
```

因为 `location` 不是标量值。它是一个对象。

### REGEXP

因为 DataStation 和 `dsp` 都构建在 SQLite 之上。
你可以使用 `x REGEXP 'y'` 进行过滤，其中 `x` 是一些列或值，`y` 是一个 REGEXP 字符串。
SQLite 不选择 regexp 实现。
DataStation 和 `dsq` 使用 Go 的 regexp 实现，这比 PCRE2 更受限制，因为 Go 对 PCRE2 的支持还不是很成熟。

```sql
$ dsq user_addresses.json "SELECT * FROM {} WHERE name REGEXP 'A.*'"
[{"location.address.number":1002,"location.city":"Toronto","name":"Agarrah"}]
```

## 支持的数据类型

| Name | File Extension(s) | Mime Type | Notes |
|-----------|-|-|--------------------|
| CSV | `csv` | `text/csv` | |
| TSV | `tsv`, `tab` | `text/tab-separated-values` | |
| JSON | `json` | `application/json` | Must be an array of objects or a [path to an array of objects](https://github.com/multiprocessio/dsq#array-of-objects-nested-within-an-object). |
| Newline-delimited JSON | `ndjson`, `jsonl` | `application/jsonlines` ||
| Concatenated JSON | `cjson` | `application/jsonconcat` ||
| ORC | `orc` | `orc` ||
| Parquet | `parquet` | `parquet` ||
| Excel | `xlsx`, `xls` | `application/vnd.ms-excel` | If you have multiple sheets, you must [specify a sheet path](https://github.com/multiprocessio/dsq#multiple-excel-sheets). |
| ODS | `ods` |`application/vnd.oasis.opendocument.spreadsheet` | If you have multiple sheets, you must [specify a sheet path](https://github.com/multiprocessio/dsq#multiple-excel-sheets). |
| Apache Error Logs | NA | `text/apache2error` | Currently only works if being piped in. |
| Apache Access Logs | NA | `text/apache2access` | Currently only works if being piped in. |
| Nginx Access Logs | NA | `text/nginxaccess` | Currently only works if being piped in. |

## Engine

在底层，dsq 使用[DataStation](https://github.com/multiprocessio/datastation) 作为库，而在底层，
DataStation 使用 SQLite 来支持对任意(结构化)数据的这类 SQL 查询。


## Comparisons

| Name | Link | Supported File Types | Engine |
|----|-|-|------------------------------------------------------------------------|
| q | http://harelba.github.io/q/ | CSV, TSV | SQLite |
| textql | https://github.com/dinedal/textql | CSV, TSV | SQLite |
| octoql | https://github.com/cube2222/octosql | JSON, CSV, Excel, Parquet | Custom engine |
| dsq | Here | CSV, TSV, a few variations of JSON, Parquet, Excel, ODS (OpenOffice Calc), Logs | SQLite |

And many other similar tools:
[sqlite-utils](https://github.com/simonw/sqlite-utils),
[csvq](https://github.com/mithrandie/csvq),
[trdsql](https://github.com/noborus/trdsql).

## Community, support, help out

For support, check out [#dsq on the Multiprocess Discord](https://discord.gg/9BRhAMhDa5).

If you're interested in helping out, check out [#dev on the Multiprocess Discord](https://discord.gg/22ZCpaS9qm).

## Subscribe

If you want to hear about new features and how this works under
the hood, [sign up here](https://forms.gle/wH5fdxrxXwZHoNxk8).

## License

This software is licensed under an Apache 2.0 license.
