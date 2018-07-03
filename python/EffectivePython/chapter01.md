# 第一章: 用Pythonic方式来思考 #

## 第1条: 确认自己所用的Python版本 ##

### 介绍 ###

1. 使用 **python --version** 命令确认所使用的具体的Python版本

2. 通常可以用 **python3** 来运行Python3

3. 通过sys模块查看:

```
import sys
print(sys.version_info)
print(sys.version)
```

### 要点 ###

1. 有两个版本的Python处于活跃状态: Python2和Python3

2. 有多种Python运行时环境, 包括 CPython, Jython, IronPython以及 PyPy

3. 应该优先考虑 Python3

## 第2条: 遵循PEP8风格指南 ##

### 介绍 ###

<Python Enhancement Proposal #8>(8号Python增强提案)又叫[PEP8](http://www.python.org/dev/peps/pep-0008), 它是针对Python代码格式而编订的风格指南.

下面列取几条绝对应该遵守的规则:

#### 空白 ####

1. 使用SPACE表示缩进, 不要使用TAB
2. 和语法相关的每一层缩进都用4个空格表示
3. 每行的字符数不应超过79
4. 对于占用多行的表达式, 除首行的其余行应该在缩进级别之上再加4个空格
5. 文件中的函数和类应该用两个空行隔开
6. 在同一个类中, 各方法之间用一个空行隔开
7. 在使用下标来获取列表元素, 调用函数或给关键字参数赋值的使用, 不用在两旁添加空格
8. 为变量赋值时, 赋值符号的左侧和右侧各加一个空格

#### 命名 ####

1. 函数, 变量和属性应该用小写字母来拼写, 各单词间用下划线分割
2. 受保护的实例属性, 以单个下划线开头
3. 私有的实例属性, 以两个下划线开头
4. 类与异常, 应该以每个单词首字母均大写的形式来命名
5. 模块级别的常量, 应该全部采用大写字母, 各单词间用下划线分割
6. 类中的实例方法, 应该把首个参数名命名为self
7. 类方法的首个参数, 应该命名为cls

#### 表达式和语句 ####

1. 采用内联形式的否定词, 而不要把否定词放在整个表达式的前面, 例如使用 if a is not b 而不是 if not a is b
2. 不要通过检测长度的办法来判断somelist是否为[]或 ''等值, 应该用 if not somelist
3. 不要编写单行的 if, for, while , except等复合语句
4. import 语句总应该放在文件开头
5. 引入模块的时候, 应该使用绝对名称
6. 如果需要以相对名称来import, 应采用 from . import foo这种明确的写法
7. import语句应该分为三个部分, 标准库模块, 第三方模块以及自用模块

### 要点 ###

1. 当编写Python代码时, 总是应该遵循PEP8风格
2. 可以使用pylint对代码进行静态分析

## 第3条: 了解bytes, str与unicode的区别 ##

### 介绍 ###

Python3有两种表示字符序列的类: bytes和str. 前者包含原始的8位值, 后者包含Unicode字符

Python2也有两种表示字符序列的类: str和unicode. str包含原始的8位值, unicode包含Unicode字符

Python3的str以及Python2的unicode没有和特定的二进制编码形式相关联, 要想把unicode字符转换为二进制数据, 需要使用encode方法.

编写Python程序的时候, 一定要把编码的解码操作放在界面的最外围处理, 程序的内部应该使用 unicode字符类型, 且不要对字符编码做任何假设.

Python3中的转换代码如下:

```
def to_str(bytes_or_str):
    if isinstance(bytes_or_str, bytes):
        value = bytes_or_str.decode('utf-8')
    else:
        value = bytes_or_str
    return value
    
def to_bytes(bytes_or_str):
    if isinstance(bytes_or_str, str):
        value = bytes_or_str.encode('utf-8')
    else:
        value = bytes_or_str
    return value
```

Python2中的转换代码如下:

```
def to_unicode(val):
    if isinstance(val, str):
        value = val.decode('utf-8')
    else:
        value = val
    return value
    
def to_str(val):
    if isinstance(val, unicode):
        value = val.encode('utf-8')
    else:
        value = val
    return value
```

还有以下两个问题需要注意:

1. 在Python2中, 如果str只包含ASCII字符, 则可以近似等价与unicode. 而在Python3中不存在这种等价
2. 在Python3中, 使用open函数获取的文件句柄默认是采用UTF-8编码格式操作, 不同于Python2中采用的二进制形式

### 要点 ###

1. 在Python3中不能混同操作 bytes和str
2. 使用辅助函数进行字符类型的转换
3. 在读取或写入文件二进制数据时, 总是应该采用二进制模式开启文件

## 第4条: 用辅助函数来取代复杂的表达式 ##

### 介绍 ###

表达式如果变得比较复杂, 那就应该考虑将其拆分成小块, 并把这些逻辑移入到辅助函数中, 这会令代码更加易读.

例如以下代码:

```
red = my_values.get('red', [''])[0] or 0
green = my_values.get('green', [''])[0] or 0
cap = my_values.get('cap', [''])[0] or 0
```

使用if/else重构后:

```
red = my_values.get('red', [''])
red = int(red[0]) if red[0] else 0

green = my_values.get('green', [''])
red = int(green[0]) if green[0] else 0

cap = my_values.get('cap', [''])
red = int(cap[0]) if cap[0] else 0
```

使用辅助函数重构后:

```
def get_first_int(values, key, default=0):
    found = values.get(key, [''])
    if found[0]:
        fount = int(found[0])
    else:
        found = default
    return found
    
red = get_first_int(my_values, 'red') 
green = get_first_int(my_values, 'green') 
cap = get_first_int(my_values, 'cap') 
```

### 要点 ###

1. 将复杂表达式移入辅助函数中, 特别是需要反复使用的相同逻辑
2. 使用if/else表达式, 要比用or或and这样的表达式更加清晰

## 第5条: 了解切割序列的方法 ##

### 介绍 ###

Python提供了切片操作, 能轻易的访问由序列中的某些元素所构成的子集. 切割操作还延伸到了实现乐 \_\_getitem\_\_ 和 \_\_setitem\_\_ 这两个特殊方法的Python类上.

切片操作的基本写法为: somelist[start:end]. 如果需要从列表开头获取切片, 则应该把start留空; 如果需要取到列表结尾, 则应该把end留空.

在指定切片起止索引时, 如果需要从后往前算, 则可以使用负值来表示偏移.

在赋值操作时对左侧列表使用切片操作, 会把该列表中处在指定范围内的对象替换为新值. 
如果此时没有指定起止索引, 则系统会把右侧的新值复制一份, 并用这份拷贝来替换左侧列表的全部内容, 而不会重新分配新的列表.

```
a = [1, 2, 3]
b = a
a[:] = [101, 102, 103]
assert a is b
```

### 要点 ###

1. 当start索引为0, 或end索引为序列长度时, 应该将其省略
2. 切片操作不会计较start与end索引是否越界
3. 对list赋值时如果使用切片操作, 就会把原列表中处在相关范围内的值替换为新值, 即使长度不同也依然可以替换

## 第6条: 在单次切片操作中, 不要同时指定 start, end和stride ##

### 介绍 ###

Python提供了 somelist[start:end:stride] 形式的写法, 来实现步进式的切割.

```
a = ['red', 'orange', 'yellow', 'green', 'blue', 'purple']
odds = a[::2]        # ['red', 'yellow', 'blue']
evens = a[1::2]      # ['orange', 'green', 'purple']
```

但是, 采用 stride方式进行切片时, 经常会出现不符合预期的结果. 例如:

```
x = b'mongoose'
y = x[::-1]        # 'esoognom'
```
但是这种技巧对于编码成UTF-8的unicode字符无效.

当切割列表时, 如果指定了stride, 那么代码可能会相当难以阅读, 当stride为负值时尤其如此.

我们不应该把stride和start, end写在一起, 如果必须使用stride, 尽量采用正值, 同时省略start和end. 如果一定要配合start和end, 可以考虑步进式切片:

```
b = a[::2]
c = b[1:-1]
```
上述方法会多产生一份拷贝, 如果程序对执行时间或者内存使用量要求特别严格, 则应该考虑用Python的itertools模块的islice方法.

### 要点 ###

1. 既有start和end, 又有stride的切割操作可能会令人费解
2. 尽量使用stride为正数, 且没有start和end索引的切割操作
3. 如果需要同时指定start, end和stride, 可以拆为两条语句或使用内置的itertools模块的islice函数.

## 第7条: 用列表推导来取代map和filter ##

### 介绍 ###

Python提供了一种精炼的写法, 可以根据一份列表在制作另外一份.

```
a = range(1, 11)
squares = [x**2 for x in a]
```
对于简单的情况来说, 列表推导比内置的map函数更清晰. 而且列表推导可以直接过滤原列表中的元素:

```
even_squares = [x**2 for x in a if x % 2 == 0]
```

字典和列表也有类似的推导机制.

```
chile_ranks = {'ghost': 1, 'habanero': 2, 'cayenne': 3}
rank_dict = {rank: name for name, rank in chile_ranks.items()}
chile_len_set = {len(name) for name in rank_dict.values()}
```

### 要点 ###

1. 列表推导比内置的map和filter函数更清晰, 因为它无需额外的lambda函数
2. 列表推导可以过滤输入列表中的元素
3. 字典与集合也支持列表推导

## 第8条: 不要使用含有两个以上表达式的列表推导 ##

### 介绍 ###

列表推导也支持多重循环.

```
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [x for row in matrix for x in row]       # [1, 2, 3, 4, 5, 6, 7, 8, 9]

squared = [[x**2 for x in row] for row in matrix]     # [[1, 4, 9], [16, 25, 36], [49, 64, 81]]
```

列表推导也支持多个if条件, 处在同一循环级别中的多项条件, 彼此之间默认形成and表达式.

```
filtered = [[x for x in row if x % 3 == 0]
             for row in martix if sum(row) >= 10]
# [[6], [9]]
```

在列表推导中, 最好不要使用两个以上的表达式, 否则应该使用普通的if和for语句, 并编写辅助函数.

### 要点 ###

1. 列表推导支持多级循环, 每一级循环也支持多项条件
2. 超过两个表达式的列表推导是难以理解的, 应该尽量避免

## 第9条: 用生成器表达式来改写数据量较大的列表推导 ##

### 介绍 ###

列表推导有个缺点: 在推导过程中, 对于输入序列中的每个值, 都可能要创建仅含一项元素的全新列表, 当输入数据非常多时, 可能会消耗大量内存, 并导致程序崩溃.

```
value = [len(x) for x in open('/tmp/my_file.txt')]
```

为了解决这个问题, Python引入了生成器表达式, 它是对列表推导和生成器的一种泛化. 生成器表达式在运行时, 会将输出序列估值为迭代器.
把实现列表推导的那种写法放在一对圆括号中, 就构成了生成器表达式.

```
it = (len(x) for x in open('/tmp/my_file.txt'))              # generator object
next(it)
next(it)
```

生成器表达式可以互相组合,

```
roots = ((x, x**0.5) for x in it)
next(roots)
```

操作大量数据时, 最好用生成器表达式来实现. 而且生成器表达式的返回迭代器是有状态的, 用过一轮之后就不能反复使用.

### 要点 ###

1. 当输入数据量较大时, 列表推导可能会使用太多内存
2. 由生成器表达式所返回的迭代器, 可以逐次产生输出值, 避免占用大量内存
3. 生成器表达式也可以互相组合
4. 串在一起的生成器表达式执行很快

## 第10条: 尽量使用enumerate取代range ##

### 介绍 ###

enumerate可以把各种迭代器包装成生成器, 生成器每次产生一对输出值, 其中前者表示循环下标, 后者表示从迭代器中获取到的下一个序列元素.

```
flavor_list = ['vanilla', 'chocolate', 'pecan', 'strawberry']
for i, flavor in enumerate(flavor_list):
    print('%d: %s' % (i + 1, flavor))
```

还可以指定enumerate函数开始计数的值, 

```
for i, flavor in enumerate(flavor_list, 1):
    print('%d: %s' % (i, flavor))
```

### 要点 ###

1. enumerate函数提供了一种简便的写法, 可以在遍历迭代器时获知每个元素的索引
2. 尽量用enumerate来改写那种将range和下标访问结合的序列遍历代码
3. 可以给enumerate提供第二个参数, 以指定开始计数时的起始值

## 第11条: 用zip函数同时遍历两个迭代器 ##

### 介绍 ###

在Python3中的zip函数, 可以把两个或两个以上的迭代器封装为生成器, 以便稍后求值. 该生成器会从每个迭代器中获取下一个值, 并把这些值汇聚成元组返回.

```
for name, count in zip(names, letters):
    if count > max_letters:
        longest_name = name
        max_letters = count
```

内置的zip函数有两个问题:

1. Python2中的zip函数并不是生成器, 而它会将所有迭代器的产生值汇聚成元组返回, 这可能会占用大量的内存. 如果在Python2中需要zip来遍历数据量非常大的迭代器, 应该使用itertools模块中的izip函数
2. 如果迭代器的长度不同, zip会在任意一个迭代器耗尽时结束. 若不能确定zip所封装的列表是否等长, 可以考虑用itertools模块的zip\_longest函数(在Python2中为 izip\_longest).

### 要点 ###

1. zip可以平行的遍历多个迭代器
2. Python3中zip为生成器, Python2中不是
3. 如果迭代器长度不同, zip会在最短的迭代器长度时终止
4. itertools模块的zip_longest函数可以遍历长度不等的迭代器

## 第12条: 不要在for和while循环后面写else块 ##

### 介绍 ###

Python可以在循环内部的语句块后面接else块:

```
for i in range(0, 3):
    print('Loop %d' % (i))
else:
    print('Else block!')
```

这种else块会在循环执行完成之后立刻执行. 

当混淆内部使用break语句提前跳出时, 会导致程序不执行else块. 而且, 如果for循环要遍历的序列是空的, 也会导致else块的执行.

### 要点 ###

1. Python可以在for或while循环内部的语句块后面接else块
2. 只有当循环主体没有执行break语句时, else块才会执行
3. 不要使用这种else块语法, 因为它既不直观, 也容易让人误解

## 第13条: 合理利用try/except/else/finally结构中的每个代码块 ##

### 介绍 ###

Python的异常处理有4种时机, 可以使用 try, except, else和finally块来表述.

1. finally块

如果即要异常向上传播, 又要在异常发生时执行清理动作, 可以使用try/finally块, 例如常见的关闭文件句柄:

```
handle = open('xxx')
try:
    data = handle.read()
finally:
    handle.close()
```

2. else块

try/except/else结构可以清晰的描述哪些异常会自己处理, 哪些异常会传播到上一级. 如果try块没有发生异常, 则会执行else块. 
这种结构可以尽量缩减try块内的代码量, 使其更易读.

```
def load_json_key(data, key):
    try:
        result_dict = json.loads(data)
    except ValueError as ex:
        raise KeyError from ex
    else:
        return result_dict[key]
```

3. 混合使用

可以使用完整的结构获取到完整的机制.

```
UNDEFINED = object()

def divide_json(path):
    handle = open(path, 'r+')
    try:
        data = handle.read()
        op = json.loads(data)
        value = (
            op['numerator'] / 
            op['denominator'])
    except ZeroDivisionError as e:
        return UNDEFINED
    else:
        op['result'] = value
        result = jaon.dumps(op)
        handle.seek(0)
        handle.write(result)
        return value
    finally:
        handle.close()
```

### 要点 ###

1. 无论try块是否发生异常, 都可以使用finally块执行清理动作
2. else块可以缩减try块的代码量, 并把没有异常发生时所需要执行的代码与try/except代码块隔开
3. 顺利运行try块之后, 如有代码需要在finally块之前执行, 可以放在else块中
