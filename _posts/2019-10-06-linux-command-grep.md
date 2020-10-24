---
layout: post
title: linux命令-grepl
date: 2019-10-08
categories:
    - linux
comments: true
permalink: linux-command-grep.html
---

grep 这个词，我们分成三段来看，是 g|re|p。

- g 就是 global，全局；
- re 就是 regular expression，正则表达式；
- p 就是 pattern，模式。

所以这个指令的作用是通过正则表达式全局搜索一个文件找到匹配的模式

参数

- **-a 或  --text** : 不要忽略二进制的数据。   
- **-A<显示行数> 或   --after-context=<显示行数>** : 除了显示符合范本样式的那一列之外，并显示该行之后的内容。   
- **-b  或 --byte-offset** : 在显示符合样式的那一行之前，标示出该行第一个字符的编号。   
- **-B<显示行数> 或  --before-context=<显示行数>** : 除了显示符合样式的那一行之外，并显示该行之前的内容。   
- **-c  或  --count** : 计算符合样式的列数。   
- **-C<显示行数>  或  --context=<显示行数>或-<显示行数>** : 除了显示符合样式的那一行之外，并显示该行之前后的内容。   
- **-d <动作>  或    --directories=<动作>** : 当指定要查找的是目录而非文件时，必须使用这项参数，否则grep指令将回报信息并停止动作。
- ​    **-e<范本样式> 或 --regexp=<范本样式>** : 指定字符串做为查找文件内容的样式。   
- **-E    或  --extended-regexp** : 将样式为延伸的正则表达式来使用。   
- **-f<规则文件> 或 --file=<规则文件>** : 指定规则文件，其内容含有一个或多个规则样式，让grep查找符合规则条件的文件内容，格式为每行一个规则样式。   
- **-F  或 --fixed-regexp** : 将样式视为固定字符串的列表。   
- **-G  或 --basic-regexp** : 将样式视为普通的表示法来使用。   
- **-h  或 --no-filename** : 在显示符合样式的那一行之前，不标示该行所属的文件名称。   
- **-H  或 --with-filename** : 在显示符合样式的那一行之前，表示该行所属的文件名称。   
- **-i  或  --ignore-case** : 忽略字符大小写的差别。   
- **-l   或 --file-with-matches** : 列出文件内容符合指定的样式的文件名称。   
- **-L 或  --files-without-match** : 列出文件内容不符合指定的样式的文件名称。   
- **-n  或 --line-number** : 在显示符合样式的那一行之前，标示出该行的列数编号。   
- **-o 或 --only-matching** : 只显示匹配PATTERN 部分。 
- **-q  或 --quiet或--silent** : 不显示任何信息。   
- **-r  或 --recursive** : 此参数的效果和指定"-d recurse"参数相同。   
- **-s   或 --no-messages** : 不显示错误信息。   
- **-v   或 --invert-match** : 显示不包含匹配文本的所有行。   
- **-V  或 --version** : 显示版本信息。   
- **-w 或  --word-regexp** : 只显示全字符合的列。   
- **-x    --line-regexp** : 只显示全列符合的列。   
- **-y** : 此参数的效果和指定"-i"参数相同。

示例

- **在文件中查找**

```
grep match_pattern file_name
```

- **在多个文件中查找**

```
grep "match_pattern" file_1 file_2 file_3 ..
```

- **输出除之外的所有行 **

```
grep -v "match_pattern" file_name
```

- **标记匹配颜色**

```
grep "match_pattern" file_name --color=auto
```

- **使用正则表达式**

```
grep -E "[1-9]+"
```

- **统计文件或者文本中包含匹配字符串的行数**

```
grep -c "text" file_name
```

- **输出包含匹配字符串的行数**

```
grep "text" -n file_name
```

- **搜索多个文件并查找匹配文本在哪些文件中**

```
grep -l "text" file1 file2 file3...
```

- **在多级目录中对文本进行递归搜索**

```
grep "text" . -r -n
# .表示当前目录。
```

- **忽略匹配样式中的字符大小写**

```
echo "hello world" | grep -i "HELLO"
```

- **选项 -e多个匹配样式**

```
echo this is a text line | grep -e "is" -e "line" -o

#也可以使用-f选项来匹配多个样式，在样式文件中逐行写出需要匹配的字符。
cat patfile
aaa
bbb

echo aaa bbb ccc ddd eee | grep -f patfile -o
```

- **在grep搜索结果中包括或者排除指定文件**

```
#只在目录中所有的.php和.html文件中递归搜索字符"main()"
grep "main()" . -r --include *.{php,html}

#在搜索结果中排除所有README文件
grep "main()" . -r --exclude "README"

#在搜索结果中排除filelist文件列表里的文件
grep "main()" . -r --exclude-from filelist
```

- **打印出匹配文本之前或者之后的行**

```
#显示匹配某个结果之后的3行，使用 -A 选项：
seq 10 | grep "5" -A 3
5
6
7
8

#显示匹配某个结果之前的3行，使用 -B 选项：
seq 10 | grep "5" -B 3
2
3
4
5

#显示匹配某个结果的前三行和后三行，使用 -C 选项：
seq 10 | grep "5" -C 3
2
3
4
5
6
7
8

#如果匹配结果有多个，会用“--”作为各匹配结果之间的分隔符：
echo -e "a\nb\nc\na\nb\nc" | grep a -A 1
a
b
--
a
b
```

参考资料

https://man.linuxde.net/grep