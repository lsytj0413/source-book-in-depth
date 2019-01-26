# 第三十四章: 字符串和数字 #

## 34.1 参数扩展 ##

### 34.1.1 基本参数 ###

参数扩展的最简单形式体现在平常对变量的使用中, 例如 $a 扩展后称为变量a所包含的内容, 简单参数也可以被大括号包围, 例如 ${a}.

### 34.1.2 空变量扩展的管理 ###

有的参数扩展用于处理不存在的变量和空变量, 形式如下:

```
${parameter:-word}
```

如果parameter未被设定或者是空参数, 则其扩展为word的值; 如果parameter非空, 则扩展为parameter的值.

另外一种扩展形式如下:

```
${parameter:=word}
```

如果parameter未被设定或者是空参数, 则其扩展为word的值, 同时将值赋值给parameter; 如果parameter非空, 则扩展为parameter的值.

需要注意的是, 位置参数和其他特殊参数不能使用这种方式赋值.

第三种扩展形式如下:

```
${parameter:?word}
```

如果parameter未被设定或者是空参数, 则扩展会导致脚本出错而退出, 同时word的内容会输出到标准错误; 如果parameter非空, 则扩展为parameter的值.

第四种扩展形式如下:

```
${parameter:+word}
```

若parameter未设定或为空, 将不产生任何扩展; 若parameter非空, word的值将取代parameter的值, 但是parameter的值不发生变化.

### 34.1.3 返回变量名的扩展 ###

shell具有返回变量名的功能, 形式如下:

```
${!prefix*}
${!prefix@}
```

该扩展返回当前以prefix开头的变量名, 根据bash文档, 这两种扩展形式执行效果相同.

### 34.1.4 字符串操作 ###

对字符串的操作存在着大量的扩展集合, 其中一些扩展尤其适用于对路径名的操作, 形式如下:

```
${#parameter}
```

扩展为parameter内包含的字符串的长度, 一般来说参数parameter是个字符串, 如果parameter是 @ 或 *, 那么扩展结果就是位置参数的个数.

提取字符串的扩展形式如下:

```
${parameter:offset}
${parameter:offset:length}
```

这个扩展可以提取一部分包含在参数parameter中的字符串, 扩展以offset字符开始, 直到字符串末尾, 除非length特别指定.

如果offset的值为负, 则表示从字符串末尾开始, 需要注意的是, 负值前必须有一个空格, 如果有length的话, length不能小于0.

如果参数是@的话, 扩展的结果则是从offset开始, length为位置参数.

去除字符串内容的扩展形式如下:

```
${parameter#pattern}
${parameter##pattern}
${parameter%pattern}
${parameter%%pattern}
```

根据pattern的定义, 这些扩展去除了包含在pattern中的字符串的主要部分, pattern是一个通配符模式. # 形式去除最短匹配, ## 形式去除最常匹配, %和%%形式则是从字符串末尾去除文本.

对字符串进行替换的扩展如下:

```
${parameter/pattern/string}
${parameter//pattern/string}
${parameter/#pattern/string}
${parameter/%pattern/string}
```

这种形式的展开对 parameter 的内容执行查找和替换操作. 如果找到了匹配通配符 pattern 的文本, 则用 string 的内容替换它.
在正常形式下, 只有第一个匹配项会被替换掉, 在 // 形式下, 所有的匹配项都会被替换掉.  /# 要求匹配项出现在字符串的开头, 而 /% 要求匹配项出现在字符串的末尾. /string 可以省略掉, 这样会导致删除匹配的文本.

参数扩展是一个重要的功能, 进行字符串操作的扩展可以替代其他常用的命令, 例如set和cut命令. 扩展通过取代外部程序, 也改善了脚本的执行效率.
使用参数扩展修改 longest-word程序如下:

```
#!/bin/bash
# longest-word3 : find longest string in a file
for i; do
    if [[ -r $i ]]; then
        max_word=
        max_len=
        for j in $(strings $i); do
            len=${#j}
            if (( len > max_len )); then
                max_len=$len
                max_word=$j
            fi
        done
        echo "$i: '$max_word' ($max_len characters)"
    fi
    shift
done
```

### 34.1.5 大小写转换 ###

最新的 bash 版本已经支持字符串的大小写转换了, bash 有四个参数展开和 declare 命令的两个选项来支持大小写转换.

这个 declare 命令可以用来把字符串规范成大写或小写字符. 使用 declare 命令, 我们能强制一个变量总是包含所需的格式, 无论如何赋值给它:

```
#!/bin/bash
# ul-declare: demonstrate case conversion via declare
declare -u upper
declare -l lower
if [[ $1 ]]; then
    upper="$1"
    lower="$1"
    echo $upper
    echo $lower
fi
```

在上面的脚本中, 我们使用 declare 命令来创建两个变量upper 和 lower. 并设定了它们能够包含的格式.

四个可以执行大小写转换操作的参数展开如下表：

| 格式 | 结果 |
|:--|:--|
| ${parameter,,} | 把 parameter 的值全部展开成小写字母 |
| ${parameter,} | 仅仅把 parameter 的第一个字符展开成小写字母 |
| ${parameter^^} | 把 parameter 的值全部转换成大写字母 |
| ${parameter^} | 仅仅把 parameter 的第一个字符转换成大写字母 |

下面的脚本演示了这些展开格式:

```
#!/bin/bash
# ul-param - demonstrate case conversion via parameter expansion
if [[ $1 ]]; then
    echo ${1,,}
    echo ${1,}
    echo ${1^^}
    echo ${1^}
fi
```

## 34.2 算术计算和扩展 ##

算术扩展的基本形式如下:

```
$((expression))
```

### 34.2.1 数字进制 ###

shell支持任何进制表示的整数, 如下表:

| 符号 | 描述 |
|:--|:--|
| Number | 十进制, 默认情况 |
| 0Number | 八进制, 以0开始的数字 |
| 0xNumber | 十六进制, 以0x开始 |
| base#Number | base进制的数字 |

例如:

```
echo $((0xff))
echo $((2#11111111))
```

### 34.2.2 一元运算符 ###

有两种一元运算符: + 和 -, 分别用来表示数字的正负.

### 34.2.3 简单算术 ###

shell支持的普通算术运算符如下表:

| 操作符 | 描述 |
|:--|:--|
| + | 加法 |
| - | 减法 |
| * | 乘法 |
| / | 除法 |
| ** | 求幂 |
| % | 取模, 即求余数 |

由于shell的算术运算符仅适用于整数, 则除法的结果永远是完整的数字. 一个shell算术运算的实例如下:

```
#!/bin/bash
# modulo : demonstrate the modulo operator
for ((i = 0; i <= 20; i = i + 1)); do
    remainder=$((i % 5))
    if (( remainder == 0 )); then
        printf "<%d> " $i
    else
        printf "%d " $i
    fi
done
printf "\n"
```

以上代码突出显示5的倍数.

### 34.2.4 赋值 ###

每当赋值给一个变量一个值时, 就是赋值操作.

```
foo=5
if (( foo=5 )); then echo "It is true."; fi
```

shell支持的一些其他赋值语句如下:

| 运算符 | 描述 |
|:--|:--|
| parameter=value | 简单赋值 |
| parameter+=value | 加法, 等价于 parameter=parameter+value |
| parameter-=value | 减法, 等价于 parameter=parameter-value |
| parameter*=value | 乘法, 等价于 parameter=parameter*value |
| parameter/=value | 除法, 等价于 parameter=parameter/value |
| parameter%=value | 取模, 等价于 parameter=parameter%value |
| parameter++ | 变量使用后自增, 等价于 parameter=parameter+1 |
| parameter-- | 变量使用后自减, 等价于 parameter=parameter-1 |
| ++parameter | 变量使用前自增, 等价于 parameter=parameter+1 |
| --parameter | 变量使用前自减, 等价于 parameter=parameter-1 |

使用++操作符修改modulo脚本如下:

```
#!/bin/bash
# modulo2 : demonstrate the modulo operator
for ((i = 0; i <= 20; ++i )); do
    if (((i % 5) == 0 )); then
        printf "<%d> " $i
    else
        printf "%d " $i
    fi
done
printf "\n"
```

### 34.2.5 位操作 ###

shell支持的位操作符如下表:

| 操作符 | 描述 |
|:--|:--|
| ~ | 按位取反 |
| << | 按位左移 |
| >> | 按位右移 |
| & | 按位与 |
| \| | 按位或 |
| ^ | 按位异或 |

除按位取反之外, 对于位操作符也存在对应的赋值操作, 例如 <<= .

以下为一个产生2的次方的数字的代码:

```
for ((i=0;i<8;++i)); do echo $((1<<i)); done
```

### 34.2.6 逻辑操作 ###

(()) 命令支持的用于逻辑判断的操作符如下表:

| 操作符 | 描述 |
|:--|:--|
| <= | 小于或等于 |
| >= | 大于或等于 |
| < | 小于 |
| > | 大于 |
| == | 等于 |
| != | 不等于 |
| && | 逻辑与 |
| \|\| | 逻辑或 |
| expr1?expr2:expr3 | 三元比较操作, 如果expr1为true, 那么执行expr2, 否则执行expr3 |

当使用逻辑操作时, 表达式遵循算术逻辑的规则, 即值为0表示false, 值为非0表示true.

注意在表达式内的赋值操作不能简单使用, 需要用括号来包围赋值表达式, 例如:

```
((a<1?(a+=1):(a-=1)))
```

一个输出数字表的脚本如下:

```
#!/bin/bash
# arith-loop: script to demonstrate arithmetic operators
finished=0
a=0
printf "a\ta**2\ta**3\n"
printf "=\t====\t====\n"
until ((finished)); do
    b=$((a**2))
    c=$((a**3))
    printf "%d\t%d\t%d\n" $a $b $c
    ((a<10?++a:(finished=1)))
done
```

## 34.3 bc: 一种任意精度计算语言 ##

bc程序读取一个使用类C语言编写的程序文件, 并执行它. bc脚本可以是一个单独的文件, 也可以从标准输入中读取. bc语言支持很多功能, 包括变量, 循环以及程序员自定义的函数.

一个 2+2 的bc脚本如下:

```
/* A very simple bc script */
2 + 2
```

脚本的第一行是注释, bc使用和C语言相同的注释语法.

### 34.3.1 bc的使用 ###

如果将上述脚本保存为 foo.bc , 那么可以使用以下命令运行它:

```
bc foo.bc
```

默认情况下bc会输出版权信息, 可以使用 -q 选项禁止显示. bc也可以交互的使用, 这时只需要简单的输入值, 计算结果会立刻被显示, 使用quit命令可以结束交互会话.

通过标准输入传递一个脚本到bc是可行的:

```
bc < foo.bc
```

也可以使用嵌入文档, 嵌入字符串和管道传递文本:

```
bc <<< "2+2"
```

### 34.3.2 脚本例子 ###

一个简单的计算按月偿还贷款的脚本如下:

```
#!/bin/bash
# loan-calc : script to calculate monthly loan payments
PROGNAME=$(basename $0)
usage () {
    cat <<- EOF
    Usage: $PROGNAME PRINCIPAL INTEREST MONTHS
    Where:
    PRINCIPAL is the amount of the loan.
    INTEREST is the APR as a number (7% = 0.07).
    MONTHS is the length of the loan's term.
    EOF
}
if (($# != 3)); then
    usage
    exit 1
fi
principal=$1
interest=$2
months=$3
bc <<- EOF
    scale = 10
    i = $interest / 12
    p = $principal
    n = $months
    a = p * ((i * ((1 + i) ^ n)) / (((1 + i) ^ n) - 1))
    print a, "\n"
EOF
```

以上代码使用嵌入文档传递脚本到bc. bc脚本中的scale命令决定了精度.

## 34.4 本章结尾语 ##

## 34.5 附加项 ##

- 《ash Hackers Wiki》对参数展开有一个很好的论述:

[http://wiki.bash-hackers.org/syntax/pe](http://wiki.bash-hackers.org/syntax/pe)

- 《Bash 参考手册》也介绍了这个:

[http://www.gnu.org/software/bash/manual/bashref.html#Shell-Parameter-Expansion](http://www.gnu.org/software/bash/manual/bashref.html#Shell-Parameter-Expansion)

- Wikipedia 上面有一篇很好的文章描述了位运算:

[http://en.wikipedia.org/wiki/Bit_operation](http://en.wikipedia.org/wiki/Bit_operation)

- 一篇关于三元运算的文章:

[http://en.wikipedia.org/wiki/Ternary_operation](http://en.wikipedia.org/wiki/Ternary_operation)

- 还有一个对计算还贷金额公式的描述, 我们的 loan-calc 脚本中用到了这个公式:

[http://en.wikipedia.org/wiki/Amortization_calculator](http://en.wikipedia.org/wiki/Amortization_calculator)
