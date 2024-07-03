# Linux常用命令


### cut

cut命令用于从文件中的每一行剪切字节、字符和字段并将这些字节、字符和字段输出。

> 基本用法

cut [选项参数] filename

说明：默认分割符是制表符

> 参数说明

| 选项参数 | 功能                       |
| -------- | -------------------------- |
| -f       | 列号，提取第几列           |
| -d       | 分隔符，按指定分隔符分割列 |

> 实践

文件test_cut.txt

```
ni,hao,ya
wo,shi,shei
ni,shi,bu,shi,sha
ni,gan,ma,ma,wo
wo,jiu,ma,ni,zen,mo,l
```

1. 切割cut第一列

`cut -d "," -f 1 test_cut.txt`

```
ni
wo
ni
ni
wo
```

2. 切割第4、5列

`cut -d "," -f 4,5 test_cut.txt`

```


shi,sha
ma,wo
ni,zen
```

注意：前面几个空行是因为前面两行没有第四第五列

3. 在test_cut.txt文件中切割出第一列中的`wo`

`cut -d "," -f 1 test_cut.txt | grep wo`

```
wo
wo
```

4. 在test_cut.txt文件中切割出包含`wo`所在行的第一列

`cat test_cut.txt | grep "wo" | cut -d "," -f 1`

```
wo
ni
wo
```

5. 选取系统变量PATH值，获取第2个`:`开始后的所有路径

`echo $PATH | cut -d ":" -f 2-`

```
/usr/local/bin:/usr/sbin:/usr/bin:/usr/local/go/bin:/root/bin
```


