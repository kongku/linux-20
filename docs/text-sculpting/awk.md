# gawk 基础用法

awk 是一款用于处理文本的编程语言工具。它提供了比较强大的功能：可以进行正则表达式的匹配，流控制、数学运算符、进程控制语句还有内置的变量和函数。

gawk 即 GNU awk，是许多 Linux 发行版默认的 awk 程序，`gawk` 相当于`awk -E` ，使用 ERE 语法，与  [grep ERE 语法](../grep-regexp) 基本一致，。

## gawk 正则表达式

gawk 有两点需要注意：

- awk 正则表达式写在 2 个 `/` 中间，书写普通斜杠符号需要加转义字符：`\/`。
- 因为 `\b` 在 awk 语言里被定义为 backspace，所以 gawk 使用 `\y` 代替 `\b` 作为锚点符号，匹配单词的边缘。

## 安装

``` bash
sudo apt install gawk
```

##  awk 语法

awk跟grep一样，都推荐使用ERE语法，所以下面统一使用gawk。

- awk [选项参数] 'program' var=value file[,file2,...]

或
- awk [选项参数] -f programfile var=value file[,file2,...]

运行 gawk：

``` bash
# 命令行运行，对文本的处理命令写在program中
gawk 'program' input-file1 input-file2

# 从程序源文件运行（从文件读取文本处理命令），使用 -f 选项
gawk -f program-file input-file1 input-file2
```

### 常用选项
- -F fs or --field-separator fs

  指定输入文件折分隔符，fs是一个字符串或者是一个正则表达式，如`-F :`，使用分号来拆分 

- -v var=value or --asign var=value

  赋值一个用户定义变量

### program 命令格式

```
'pattern { action }'
```
- program 整个命令使用`''`包括，action部分使用`{}`包裹。

- pattern为行匹配，当匹配成功，对行文本执行action的文本处理，由于action中可以写复杂的语句，可以使用`{}`，所以program中第一个`{`和最后一个`}`之间的内容为action。

- action最后必须有输出（例如,使用print输出，printf格式化输出），不然即便行匹配了，对文本进行了处理，也不会有任何的输出。

- 如果省略pattern，则表示对所有行都执行action；

- 如果省略{ action }，则默认为{print $0}，输出pattern匹配的行；

- 不能同时省略，就是不能是空命令，不然输出gawk的帮助文档。


## 简单例子， gawk 代替 grep 和 sed

gawk 可以用于文本搜索和文本替换，一定程度上可代替 grep 和 sed，请看下面的例子。

用于文本搜索：

``` bash
# 查看发行版，只有pattern部分，而且使用正则来匹配行，输出匹配的行
cat /etc/os-release | gawk '/PRETTY/'
cat /etc/os-release | grep 'PRETTY'

# 过滤注释行或空白行
cat ~/.profile | gawk '!/^\s*(#|$)/'
cat ~/.profile | egrep -v '^\s*(#|$)'
```

用于文本替换：

``` shell-session
$  # 只有action，对所有行都进行文本处理，调用gsub函数替换文本，$0变量是当前行的文本，把它输出
$  echo "A beautiful girl" | gawk '{gsub(/girl/,"woman"); print $0}'
A beautiful woman
$  
$  echo "A beautiful girl" | sed 's/girl/woman/'
A beautiful woman
```

gawk 是比较全能的文本处理工具，但是在一些基本的用途中，其简洁性不如 grep 和 sed，如何选择则根据个人习惯。

## 字段分割

awk 根据字段分割符划分文本行，字段分隔符默认为空格或制表符tab，并将如下变量分配给数据字段：

- `$0` 代表整行文本
- `$1` 代表行中的第 1 个数据字段
- `$2` 代表行中的第 2 个数据字段
- `$n` 代表行中的第 n 个数据字段
- `$NF` 代表最后一个字段，因为NF是awk的内建变量，表示当前行的字段数目

默认的分割符是空格或者制表符，可以使用 `-F` 选项指定分割符，请看下面的例子：

``` bash
# 使用默认分割符，打印第二个字段
cat /etc/apt/sources.list | gawk '{print $2}'

# 指定分割符，打印第一个字段
cat /etc/passwd | gawk -F : '{print $1}'
```

## pattern 行匹配

对行进行筛选，匹配的行则执行action，pattern支持以下语法：
- 使用整行来匹配，使用正则匹配
- 使用指定字段进行匹配，可以使用多重匹配方式
 - 正则表达式
 - 数学表达式
 - 布尔表达式

### 整行匹配
``` bash
# 搜索包含 bin 的行，使用整行来匹配/bin/
cat /etc/passwd | gawk -F : '/bin/'
```

### 指定字段匹配

awk 可以指定数据字段进行文本匹配，而不是整行匹配：

#### 正则表达式

`指定字段 ~ 正则表达式`或`指定字段 !~ 正则表达式`，正则表达式使用//包裹，例如：

``` bash
# 搜索第一个字段包含 bin 的行，即用户名包含 bin
cat /etc/passwd | gawk -F : '$1 ~ /bin/'

# 不匹配用户id是三位数及以下的行
cat /etc/passwd | gawk -F : '$3 !~ /^[0-9]{1,3}$/'
```

#### 数学表达式

数学表达式支持 `==` `>` `>=` `<` `<=` 这些数学符号，例如：

``` bash
# 找出 uid 大于等于 1000 的用户
cat /etc/passwd | gawk -F : '$3 >= 1000'
```

`==` 也可以用于字符串精确匹配，例如：

``` bash
# 使用数学表达式找出 bin 用户
cat /etc/passwd | gawk -F : '$1 == "bin"'

# 使用正则表达式找出 bin 用户
cat /etc/passwd | gawk -F : '$1 ~ /^bin$/'
```

#### 布尔表达式

布尔操作符 `||` (or), `&&` (and), `!` (not), 例如：

``` bash
# 大于等于 1000 或等于 0
cat /etc/passwd | gawk -F : '$3 >= 1000 || $3 == 0'

# 大于等于 1000 或且小于 2000
cat /etc/passwd | gawk -F : '$3 >= 1000 && $3 < 2000'

# 不小于 1000
cat /etc/passwd | gawk -F : '!($3 < 1000)'
```

## printf 格式化打印

printf 支持更灵活的打印输出，例如：

``` bash
# 按指定宽度打印用户名和 uid，实现等宽排列
cat /etc/passwd | gawk -F : '{printf "%-20s %5s\n", $1, $3}'
```

将得到用户名和 uid 整齐的输出格式，如下所示：

```
root                     0
daemon                   1
nobody               65534
systemd-timesync       100
systemd-network        101
```

gawk 中 printf 的用法跟C语言一样:

```
printf "%-20s %5s\n", $1, $3
```

这个例子中 2 个 `%s` 是字符串指示符，输出时被后面对应位置的变量替换，中间的数字 `20` 和 `5` 代表输出字段的最小宽度，默认右对其，数字前加 `-` 代表左对其。

下面再看一个例子：

``` bash
cat /etc/group | gawk -F : '{printf "%-20s %-10s Members: %s\n", $1, $3, $4}'
```

将得到组名，gid，组成员整齐的输出格式，如下所示：

```
root                 0          Members: 
ssl-cert             109        Members: postgres
scanner              119        Members: saned,jack
docker               130        Members: jack
```


案例：

``` bash
# 以下几种写法效果一样
awk -F : '$3 >= 1000 {print}' /etc/passwd
cat /etc/passwd | gawk -F : '$3 >= 1000 {print}'
cat /etc/passwd | gawk -F : '$3 >= 1000 {print $0}'
# {print} 等同于 {print $0}，可省略不写
cat /etc/passwd | gawk -F : '$3 >= 1000'
```

## awk 其他进阶用法

``` bash
# if 语句
cat /etc/passwd | gawk -F : '{if ($1 == "bin") print $0}'
# 代码块{}，语句使用;分隔，使用空格连接字符串
cat /etc/passwd | gawk -F : '{if ($1 == "bin") {id="id:" $3;print id}}'

# 调用 shell 环境变量
cat /etc/passwd | gawk -F : '{if ($1 == ENVIRON["USER"]) print $0}'

# 字符函数，转换为小写字母
cat /etc/os-release | gawk 'tolower($0) ~ /pretty/'
env | gawk '{print tolower($0)}'
```

## 内建变量

| 变量 | 描述 |
| ---- | ---- |
| $n | 当前记录的第n个字段，字段间由FS分隔 |
| $0 | 完整的输入记录 |
| ARGC | 命令行参数的数目 |
| ARGIND | 命令行中当前文件的位置(从0开始算) |
| ARGV | 包含命令行参数的数组 |
| CONVFMT | 数字转换格式(默认值为%.6g)ENVIRON环境变量关联数组 |
| ERRNO | 最后一个系统错误的描述 |
| FIELDWIDTHS | 字段宽度列表(用空格键分隔) |
| FILENAME | 当前文件名 |
| FNR | 各文件分别计数的行号 |
| FS | 字段分隔符(默认是任何空格) |
| IGNORECASE | 如果为真，则进行忽略大小写的匹配 |
| NF | 一条记录的字段的数目 |
| NR | 已经读出的记录数，就是行号，从1开始 |
| OFMT | 数字的输出格式(默认值是%.6g) |
| OFS | 输出字段分隔符，默认值与输入字段分隔符一致。 |
| ORS | 输出记录分隔符(默认值是一个换行符) |
| RLENGTH | 由match函数所匹配的字符串的长度 |
| RS | 记录分隔符(默认是一个换行符) |
| RSTART | 由match函数所匹配的字符串的第一个位置 |
| SUBSEP | 数组下标分隔符(默认值是/034) |
