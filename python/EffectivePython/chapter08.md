# 第八章: 部署 #

## 第54条: 考虑用模块级别的代码来配置不同的部署环境 ##

### 介绍 ###

所谓部署环境就是程序在运行的时候所用的一套配置. 如果要在程序运行的时候支持不同的配置, 最佳的方案是在程序启动的时候覆写其中的某些部分, 以便根据部署环境来提供不同的功能.

```
# dev_main.py
TESTING = True
import db_connection
db = db_connection.Databse()

# prod_main.py
TESTING = False
import db_connection
db = db_connection.Databse()

# db_connection.py
import __main__

class TestingDatabase(object):
    pass
    
class RealDatabase(object):
    pass
    
if __main__.TESTING:
    Database = TestingDatabase
else:
    Database = TestingDatabase
```

### 要点 ###

1. 程序通常需要在不同的部署环境之中运行, 而这些环境所需的先决条件及配置信息也都互不相同
2. 我们可以在模块的范围内编写普通的Python语句, 以便根据不同的环境来定制本模块的内容
3. 我们可以根据外部条件来决定模块的内容

## 第55条: 通过repr字符串来输出调试信息 ##

### 介绍 ###

在调试某个对象时应该打印repr版本的字符串, 这种形式是最为清晰且又利于理解的一种字符串表示形式.

### 要点 ###

1. 针对内置的Python类型来调用print函数会根据该值打印出一条易于阅读的字符串, 这个字符串隐藏了类型信息
2. 针对内置的Python类型来调用repr函数, 会根据该值返回一条可供打印的字符串. 把这个repr字符串传给内置的eval函数就可以将其还原为初始值
3. 在格式化字符串中使用 %s 可以产生与str函数相仿的易读字符串. 使用 %r 则能够产生与repr函数的返回值相仿的可打印字符串
4. 可以在类中编写 \_\_repr\_\_ 方法来提供一种可供打印的字符串
5. 可以在任意对象上查询 \_\_dict\_\_ 属性以观察其内部信息

## 第56条: 用unittest来测试全部代码 ##

### 介绍 ###

要编写测试, 最简单的方式就是使用内置的unittest模块. 测试是以TestCase类的形式来组织, 每个以test开头的方法都是一项测试. 还有一种常用的做法是通过mock函数或mock类来替换受测程序的某些行为.


### 要点 ###

1. 要想确信Python程序能够正常工作, 唯一的办法就是编写测试
2. 内置的unittest模块提供了测试者所需的很多功能, 可以借助这些机制编写出良好的测试
3. 可以在TestCase子类中为每一个需要测试的行为定义对应的测试方法
4. 必须同时编写单元测试和集成测试, 前者用来独立检验程序中的每个功能, 后者用来检验模块之间的交互行为

## 第57条: 考虑用pdb实现交互调试 ##

### 介绍 ###

我们只需要引入内置的pdb模块, 并运行其 set\_trace函数即可触发调试器.

```
import pdb; pdb.set_trace()
```

只要运行到那个语句程序就会暂停, 然后转入交互式的Python提示符界面.
调试器提供了以下的命令来方便查看正在调试的程序.

1. bt: 打印调用栈回溯信息
2. up: 把调试范围沿着函数调用栈上移一层, 回到当前函数的调用者那里. 该命令可以使我们检视调用栈上层的局部变量
3. down: 把调试范围沿着函数调用栈下移一层
4. step: 单步执行当前行代码, 可以步进函数调用
5. next: 执行当前代码, 可以跳过函数调用
6. return: 直达当前函数的return语句开头
7. continue: 继续运行程序直至下一个断点或下一个 set\_trace 调用点

### 要点 ###

1. 可以利用 pdb.set\_trace 触发互动调试器
2. Python调试器也是一个完整的Python提示符界面, 可以检视并修改受调用程序的状态
3. 我们可以在pdb提示符中输入命令, 以便精确的控制程序的执行流程

## 第58条: 先分析性能, 然后再优化 ##

### 介绍 ###

Python提供了内置的性能分析工具可以计算出程序中某个部分的执行时间, 在总体执行时间中所占的比率, 通过这些数据可以找到最为显著的性能瓶颈并把注意力放在优化这部分代码上面.

```
def insert_sort(data):
    #
    insert_value(result, value)
    # 
    
def insert_value(array, value):
    pass
    
from random import randint

data = [randing(0, max_size) for _ in range(max_size)]
test = lambda: insert_sote(data)

# 利用cProfile模块的Profile对象运行函数
profile = Profile()
profile.runcall(test)

# 输出性能统计数据
stats = Stats(profile)
stats.strip_dirs()
stats.sort_stats('cumulative')
stats.print_stats()
```

上面的代码会输出一张表格, 其中的信息是按照函数来分组的. 性能统计中的每一列的意义是:

| 列名称 | 含义 |
|:--:|:--|
| ncalls | 该函数在性能分析期间的调用次数 |
| tottime | 执行该函数所花的总秒数, 本函数调用其他函数所耗费的时间不计入 |
| tottime percall | 每次调用该函数所花的平均秒数, 调用其他函数所耗费的时间不计入, 值相当于 tottime/ncalls |
| cumtime | 执行该函数及其中全部函数调用操作所花的总秒数 |
| cumtime percall | 每次调用该函数及集中全部函数调用操作所花的平均秒数, 值相当于 cumtime/ncalls |

有些时候一个函数会被多个地方调用, 我们可以通过 stats.print\_callers() 函数来打印出来调用者.

### 要点 ###

1. 优化程序之前一定要先进行性能分析
2. 做性能分析时应该使用cProfile而不是profile麻婆块, 前者能够给出更为精确的性能分析数据
3. 可以通过Profile对象的runcall方法来分析程序的性能
4. 可以用Stats对象来筛选性能分析数据

## 第59条: 用tracemalloc来掌握内存的使用及泄漏情况 ##

### 介绍 ###

CPython中内存管理是通过引用计数来处理的, 另外还内置了循环检测器使得垃圾回收机制能够把那些自我引用的对象清理掉.

调试内存状态的第一种办法是使用内置的gc模块.

```
import gc
found_objects = gc.get_objects()
for obj in found_objects[:3]:
    print(repr(obj))
```
gc模块不能告诉我们对象到底是如何分配出来的. Python3.4 推出了一个名为tracemalloc的内置模块, 可以解决这个问题.

```
import tracemalloc

# 保存10个栈帧
tracemalloc.start(10)

time1 = tracemalloc.take_snapshot()
import waste_memory
x = waste_memory.run()
time2 = tracemalloc.task_snapshot()

stats = time2.compare_to(time1, 'lineno')
for stat in stats[:3]:
    print(stat)
    
# 打印分配内存操作时的完整堆栈信息
stats = time2.compare_to(time1, 'traceback')
top = stats[0]
print('\n'.join(top.traceback.format()))
```

### 要点 ###

1. Python程序的内存使用情况和内存泄漏情况是很难判断的
2. 可以通过gc模块来了解程序中的对象, 但是并不能看出这些对象究竟是如何分配出来的
3. 内置的tracemalloc模块提供了许多强大的工具, 使得我们可以找出导致内存使用量增大的根源
4. 只有Python3.4及后续版本才支持tracemalloc模块
