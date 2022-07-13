# sed 流编辑器

sed (stream editor) 即流编辑器，它读取输入流 (文件或来自管道的输入)，逐行处理，依照一系列命令对内容进行编辑修改，最后将结果写入标准输出。

sed 是很老牌的文本处理工具，它逐渐被其它工具所取代，例如 awk。但由于 sed 的几个命令非常简洁易用，比如替换、插入和删除，所以它仍然广受喜爱。

一个简单的文本替换案例：

``` shell-session
$  # 使用 sed
$  echo "A beautiful girl" | sed 's/girl/woman/'
A beautiful woman
$  
$  # 使用 awk
$  echo "A beautiful girl" | awk '{gsub(/girl/,"woman"); print $0}'
A beautiful woman
```

## sed 常用选项

- -E, --extended-regexp, 使用扩展正则表达式 (ERE)
- -e script, --expression=script, 添加 sed 执行的命令
- -i[SUFFIX], --in-place[=SUFFIX], 用于输入流从文件读取时，把修改后的结果输出重定向到文件，如果提供了SUFFIX，原文件将会使用SUFFIX备份

### -i 修改文件

sed 对文本进行修改的命令，默认只会将输出打印出来，加上 `-i` 会把输出从定向到文件（把修改后的结果写入文件），例如：

``` bash
# 将替换后的文本打印出来
sed 's/foo/bar/' file

# 对文件进行文本替换
sed -i 's/foo/bar/' file

# 对文件进行文本替换，将会创建一个备份文件 file.sed
sed -i.sed 's/foo/bar/' file
```

`-i` 与其它选项一起使用时应分开写，例如 `-i -E` 不能写成 `-iE`，请看下面的例子，因为-i后面跟着的会被视为SUFFIX：

``` bash
# 下面两条命令是等价的
sed -Ei '...' file
sed -E -i '...' file

# -iE 等价于 --in-place=E，会创建一个后缀为E的备份文件 fileE
sed -iE '...' file
```

### -E 正则表达式

GNU `sed -E` 使用 ERE 语法，与  [grep ERE 语法](../grep-regexp) 一致。

### -e 添加执行命令

为sed添加处理文本的任务,可以添加多个。
例如，需要替换某个单词，同时在最后添加一行：

``` bash
# 后面会学到添加行的命令
sed -E -i 's/foo/bar/' -e '$a/append' file
```

## 处理命令

sed通过特定的命令语法对文本进行处理，处理流程：逐一获取每一行的文本，然后执行所有命令。

每一个命令的执行流程：先“行匹配”，匹配后才会执行文本处理。

语法：[行匹配]命令符/[文本处理]，当省略行匹配，就代表对所有行进行文本处理。

命令符：
- s 文本替换
- d 行删除
- c 行修改
- i 行前插入
- a 行后插入

## s 文本替换

语法：`s/regexp/replacement/n`

regexp为正则查找需要被替换的文本，replacement用于替换的文本，n为替换行中第n个regexp匹配的文本，默认为第一个（n=1），当n为g时表示替换全部匹配的文本。

当省去n，默认n为1，但是replacement后面的分隔符不能省去。

例如：

``` shell-session
$  seq 31 36 | sed 's/33/replace/'
31
32
replace
34
35
36
$  # 替换行尾的数字 3 和 5
$  seq 31 36 | sed 's/[35]$/replace/'
31
32
3replace
34
3replace
36
$  # 替换第 3 次匹配
$  echo "a dog and a cat" | sed 's/a/A/3'
a dog and A cat
$  
$  # 替换所有匹配
$  echo "a dog and a cat" | sed 's/a/A/g'
A dog And A cAt
```

### 自定义分割符

替换命令中的分割符 `/` 也可以用其他符号代替，例如：

``` shell-session
$  # 常规书写方式，如果regexp或replacement用到了/分隔符，就需要转义，多的时候就显得混乱
$  echo "/bin/sh test.sh" | sed 's/\/bin\/sh/\/bin\/bash/'
/bin/bash test.sh
$  
$  # 自定义分割符，省去了转义，更加的清晰
$  echo "/bin/sh test.sh" | sed 's%/bin/sh%/bin/bash%'
/bin/bash test.sh
$  echo "/bin/sh test.sh" | sed 's!/bin/sh!/bin/bash!'
/bin/bash test.sh
```

上面这个例子需要在正则表达式中多次书写 `/` 符号，如果是 `s/regexp/replacement/` 这样的写法，正则表达式里面好几个 `/` 需要加转义字符，写出来非常难看。改成 `s%regexp%replacement%` 这样的写法会简洁很多。

可以使用任意单个符号替代 `/`，恢复该符号的字面量需要加转义字符。


### 行匹配

这里借助s文本替换命令讲解行匹配，但同样适用于其他命令。

行匹配，可以使用正则，通过文本内容行匹配，也可以直接使用数字来指定某些行。

正则：/regex/命令符/文本处理

数字：n[,m]命令符/文本处理，n和m为数字，当表示从第n行到第m行，行号从1开始，单使用n表示指定第n行，使用$表示最后一行。

注意，使用数字匹配行时，如果在行前行后插入行，或者删除行，不会影响n，因为n是从输入流中读取到的第n行，插入的行和删除的行，只对缓冲中的内容影响，不会影响输入流。

例如：

``` shell-session
# 匹配每一行是url开头的行，对行中的文本进行替换,只替换第一个匹配的文本
sed -E '/^url:/s%(http|ftp):%https:%' file
web:http://www.baidu.com
url:https://www.qq.com,http://www.taobao.com
web:ftp://file.github.com
$
$  # 定位第三行，$结束位置替换foo，也就是在第三行结尾添加foo
$  seq 6 | sed '3s/$/foo/'
1
2
3foo
4
5
6
$  # 从第三行到第五行
$  seq 6 | sed '3,5s/$/foo/'
1
2
3foo
4foo
5foo
6
$  # 从第三行到最后一行,^开始位置替换为foo，也就是在开头插入foo
$  seq 6 | sed '3,$s/^/foo/'
1
2
foo3
foo4
foo5
foo6
```

## 修改行

命令 `c` 用于修改行，c\replacement，直接把该行的文本替换为replacement，例如：

``` shell-session
$  seq 6 | sed '3c\change'
1
2
change
4
5
6
$  echo -e "line 1\nline 2\nend"
line 1
line 2
end
$  echo -e "line 1\nline 2\nend" | sed '/line/c\change'
change
change
end
```

## d 删除行

命令 `d` 用于删除行，例如：

``` shell-session
$  seq 6 | sed '3d'
1
2
4
5
6
```

## i 插入行 和 a 附加行

命令 `i` 在匹配的行前面插入新行。命令 `a` 在匹配的行后面附加新行。请看下面的例子：

``` shell-session
$  seq 6 | sed '6i\insert'
1
2
3
4
5
insert
6
$  seq 6 | sed '6a\append'
1
2
3
4
5
6
append
```

## 验证行号

`-e` 选项可以执行多条命令，例如：

``` shell-session
$  # 删除行，不会影响行号
$  seq 6 | sed  -e '2d' -e '5d'
1
3
4
6
$  # 插入行，也不会影响行号
$  seq 6 | sed  -e '2d' -e '3i\insert' -e '5d'
1
insert
3
4
6
```

## -e 可以通过分隔符添加多个命令
`;` 分割多条命令，例如：

``` shell-session
$  seq 6 | sed  -e '2d; 5d'
1
3
4
6
```

## sed 实践

### 修改 apt 源

``` shell-session hl_lines="13"
$  # 源文件
$  cat /etc/apt/sources.list 
deb http://deb.debian.org/debian buster main
deb-src http://deb.debian.org/debian buster main

deb http://deb.debian.org/debian-security/ buster/updates main
deb-src http://deb.debian.org/debian-security/ buster/updates main

deb http://deb.debian.org/debian buster-updates main
deb-src http://deb.debian.org/debian buster-updates main
$ 
$  # 修改文件
$  sudo sed -E -i.sed '/^deb/s%(https?|ftp)://[^/]+/%http://mirrors.aliyun.com/%' /etc/apt/sources.list
$  
$  # 修改后的文件
$  cat /etc/apt/sources.list
deb http://mirrors.aliyun.com/debian buster main
deb-src http://mirrors.aliyun.com/debian buster main

deb http://mirrors.aliyun.com/debian-security/ buster/updates main
deb-src http://mirrors.aliyun.com/debian-security/ buster/updates main

deb http://mirrors.aliyun.com/debian buster-updates main
deb-src http://mirrors.aliyun.com/debian buster-updates main
```

### 修改 profile 文件

``` shell-session hl_lines="9 13"
$  # 写入样例数据
$  sed -i '$a\export EDITOR=nano' ~/.profile
$  sed -i '$a\export EDITOR=vim' ~/.profile
$  grep 'export EDITOR=' ~/.profile
export EDITOR=nano
export EDITOR=vim
$  
$  # 删除配置项
$  sed -i '/^export EDITOR\s*=/d' ~/.profile
$  grep 'export EDITOR=' ~/.profile
$  
$  # 添加配置项
$  sed -i '$a\export EDITOR=/usr/bin/vim' ~/.profile
$  grep 'export EDITOR=' ~/.profile
export EDITOR=/usr/bin/vim
```

上面这个例子，配置文件有多行重复的配置项，可以先删除再添加。
