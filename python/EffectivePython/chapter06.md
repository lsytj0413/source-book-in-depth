# 第六章: 内置模块 #

## 第42条: 用functools.wraps定义函数修饰器 ##

### 介绍 ###

Python使用修饰器来修饰函数, 能够在那个函数执行之前以及执行完毕之后分别运行一些附加代码.

假设要打印某个函数在受到调用时所接收的参数以及该函数的返回值:

```
def trace(func):
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        print('%s(%r, %r) -> %r' %
              (func.__name__, args, kwargs, result))
        return result
    return wrapper
    
@trace
def fibonacci(n):
    if n in (0, 1):
        return n
    return (fibonacci(n - 1) + fibonacci(n - 2))
```
上面的修饰器虽然可以正常运作, 但是修饰器所返回的那个值其名称会和原来的函数不同, 而且help函数也会失效.

这个问题可以使用内置模块functools中名为wraps的辅助函数来解决, 将wraps修饰器运用到wrapper函数之后, 它就会把内部函数相关的重要元数据全部复制到外围函数.

### 要点 ###

1. Python为修饰器提供了专门的语法, 使得程序在运行的时候能够用一个函数来修饰另一个函数
2. 对于调试器这种依靠内省机制的工具, 直接编写修饰器会引发奇怪的行为
3. 内置的functools模块提供了名为wraps的修饰器, 开发者在定义自己的修饰器时应该用wraps对其做一些处理, 以避免一些问题

## 第43条: 考虑以contextlib和with语句来改写可复用的try/finally代码 ##

### 介绍 ###

开发者可以用内置的contextlib模块来处理自己所编写的对象和函数, 使它们能够支持with语句. 该模块提供了名为contextmanager的修饰器, 一个简单的函数只需经过contextmanager修饰即可用在with语句中. 如果按标准方式来做就需要定义新的类, 并提供名为 \_\_enter\_\_ 和 \_\_ext\_\_ 的特殊方法.

```
def my_function():
    logging.debug('Some debug data')
    logging.error('Error log here')
    logging.debug('More debug data')
    
# 定义一个临时修改日志级别的修饰器
@contextmanager
def debug_logging(level):
    logger = logging.getLogger()
    old_level = logger.getEffectiveLevel()
    logger.setLevel(level)
    try:
        yield
    finally:
        logger.setLevel(old_level)
```
yield表达式所在的地方, 就是with块中的语句所要展开执行的地方. with块所抛出的任何异常都会由yield表达式重新抛出.

```
with debug_logging(logging.DEBUG):
    my_function()
```

传给with语句的那个情境管理器本身也可以返回一个对象, 开发者可以通过with复合语句中的as关键字来指定一个局部变量, Python会把那个对象赋值给这个局部变量.

我们只需要在情境管理器中通过yield语句返回一个值即可.

### 要点 ###

1. 可以用with语句来改写try/finally块中的逻辑, 以便提升复用度并使代码更加整洁
2. 内置的contextlib模块提供了名为contextmanager的修饰器, 开发者只需要用它来修饰自己的函数即可令该函数支持with语句
3. 情境管理器可以通过yield语句向with语句返回一个值, 这个值会赋值给as关键字所指定的变量

## 第44条: 用copyreg实现可靠的pickle操作 ##

### 介绍 ###

内置的pickle模块能够将Python对象序列化为字节流, 也能把这些字节反序列化为Python的内置对象. pickle的设计目标是提供一种二进制流, 使开发者能够在自己所控制的各程序之间传递Python对象.

pickle序列化的结果如果在不同版本间传递, 比如如果新版本增加了新的属性, 老的版本的字节反序列化的结果却不会出现新加的属性. 要解决这个问题, 开发者可以使用 copyreg模块注册一些函数来负责Python对象的序列化操作.

1. 为缺失的属性提供默认值

```
class GameState(object):
    def __init__(self, level=0, lives=4, points=6):
        self.level = level
        self.lives = lives
        self.points = points
        
def pickle_game_state(game_state):
    kwargs = game_state.__dict__
    return unpickle_game_state, (kwargs, )

def unpickle_game_state(kwargs):
    return GameState(**kwargs)
    
copyreg.pickle(GameState, pickle_game_state)
```

2. 用版本号来管理类

通过上面的实现方式, 可以解决增加字段的问题, 但是无法解决移除字段的问题. 因为移除过字段之后, 对旧版本的反序列化操作会传入无效的关键字参数.

解决办法是, 在pickle时向数据中增加一个代表版本号的参数, 反序列时通过版本号来选择不同的操作.

3. 固定的引入路径

在使用pickle模块时, 当类的名称改变之后原有的数据会无法进行反序列化操作, 因为序列化之后的数据会把该对象的引入路径写入数据中.

我们可以给函数指定一个固定的标识符, 令它采用这个标识符来对数据进行unpickle操作, 使得我们在反序列化操作的时候能把原来的数据迁移到名称不同的其他类上面.

```
copyreg.pickle(BetterGameState, pickle_game_state)
```

使用这种copyreg之后, 序列化之后的数据不再包含对象所属类的引入路径, 而是包含unpickle\_game\_state 函数的路径. 所以在这种情况下不能修改这个函数的引入路径.

### 要点 ###

1. 内置的pickle模块只适合在彼此信任的程序之间传递数据
2. 如果用法比较复杂, 那么pickle模块的功能也许就会出现问题
3. 我们可以把内置的copyreg模块和pickle结合使用, 以便为旧的数据添加缺失的属性值, 进行类的版本管理, 并给序列化之后的数据提供固定的引入路径

## 第45条: 应该用datetime模块来处理本地时间, 而不是用time模块 ##

### 介绍 ###

协调世界时(UTC) 是一种标准的时间表述方式, 它与时区无关, 有些计算机用某一时刻与UNIX时间原点之间相差的秒数来表示那个时刻所对应的时间. 但这中方式对用户不太友好, 需要寻找一种在UTC与当地时间进行转换的方式.

Python提供了两种时间转换方式, 旧的方式是使用内置的time模块, 比较容易出错; 新的方式是使用内置的datetime模块, 效果非常好.

#### time模块 ####

在内置time模块中有个localtime函数可以把UNIX时间戳转换为与宿主计算机的时区相符的当地时间.

```
from time import localtime, strftime

now = 1407694710
local_tuple = localtime(now)
time_format = '%Y-%m-%d %H:%M:%S'
time_str = strftime(time_format, local_tuple)
```

也可以将当地时间转换为UNIX时间戳.

```
from time import mktime, strptime

time_tuple = strptime(time_str, time_format)
utc_now = mktime(time_tuple)
```

如何把某个时区的当地时间转换为另一个时区的当地时间, 但是直接通过time, localtime和strptime等函数的返回值进行转换不是一个好办法. 许多操作系统都提供了时区配置文件, 如果时区信息发生变化, 它们就会自动更新.

但是time模块依赖操作系统而运作, 它的实际行为取决于底层C函数如何与宿主操作系统相交互.

#### datetime模块 ####

datetime可以把UTC格式的当前时间转换为本地时间。

```
from datetime import datetime, timezone

now = datetime(2014, 8, 10, 18, 18, 30)
now_utc = now.replace(tzinfo=timezone.utc)
now_local = now_utc.astimezone()
```

也可以把本地时间转换为UTC格式的UNIX时间戳.

```
time_str = '2014-08-10 11:18:30'
now = datetime.strptime(time_str, time_format)
time_tuple = now.timetuple()
utc_now = mktime(time_tuple)
```

与time模块不同的是, datetime模块提供了一套机制, 能够把某一种当地时间可靠的转换为另外一种当地时间. 然而在默认情况下, 只能通过datetime中的tzinfo类及相关方法来使用这套时区操作机制, 因为它并没有提供UTC之外的时区定义.

我们可以使用pytz模块填补这一个空缺. 为了有效的使用pytz模块, 应该把当地时间转换为UTC, 然后针对UTC值进行datetime操作, 然后再把UTC转换回当地时间.

```
# 获取UTC时间
arrival_nyc = '2014-05-01 21:33:24'
nyc_dt_naive = datetime.strptime(arrival_nyc, time_format)
eastern = pytz.timezone('US/Eastern')
nyc_dt = eastern.localize(nyc_dt_naive)
utc_dt = pytz.utc.normalize(nyc_dt.astimezone(pytz.utc))

# 转换回旧金山当地时间
pacific = pytz.timezone('US/Pacific')
sf_dt = pacific.normalize(utc_dt.astimezone(pacific))
```

### 要点 ###

1. 不要用time模块在不同时区之间进行转换
2. 如果要在不同时区之间进行可靠的转换操作, 就应该把内置的datetime模块和开发者社区提供的pytz模块搭配起来使用
3. 开发者总是应该把时间表示成UTC格式然后对其进行转换操作, 然后再转换回本地时间

## 第46条: 使用内置算法与数据结构 ##

### 介绍 ###

Python的标准程序库里面, 内置了各种算法与数据结构.

#### 双向队列 ####

collections模块中的deque类是一种双向队列, 从该队列的头部或尾部插入或移除一个元素, 只需要消耗常数级别的时间, 适合用来表示先进先出的队列.

```
fifo = deque()
fifo.append(1)
x = fifo.popleft()
```

#### 有序字典 ####

标准的字典是无序的, 这是因为其快速哈希表的实现方式导致的.

collections模块中的OrderedDict类是一种特殊的字典, 它能够按照键的插入顺序来保留键值对在字典中的次序.

```
a = OrderedDict()
a['foo'] = 1
a['bar'] = 2
```

#### 带默认值的字典 ####

collections模块中的defaultdict类是一种带默认值的字典, 如果字典里面没有这个键, 那么它就会把某个默认值与这个键自动关联起来.

```
stats = defaultdict(int)
stats['my_counter'] += 1
```

#### 堆队列 ####

堆是一种数据结构, 很适合用来实现优先级队列. heapq模块提供了heappush, heappop和nsmallest等一些函数, 能够在标准的list类型之中创建堆结构.

```
a = []
heappush(a, 5)
heappush(a, 3)
heappush(a, 7)
heappush(a, 4)

# 按照优先级从高到低弹出(数值较小的优先级较高)
print(heappop(a))
print(heappop(a))
print(heappop(a))
print(heappop(a))
```

#### 二分查找 ####

bisect模块中的bisect_left等函数提供了高效的二分折半搜索算法, 能够在一系列排序好的元素之间搜寻某个值.

#### 与迭代器有关的工具 ####

内置的itertools模块中, 包含大量的函数, 可以用来组合并操控迭代器. 这个函数分为三大类:

- 连接迭代器的函数
    - chain: 将多个迭代器按顺序连成一个迭代器
    - cycle: 无限的重复某个迭代器中的各个元素
    - tee: 把一个迭代器拆分成多个平行的迭代器
    - zip_longest: 与内置的zip函数相似, 但是可以应对长度不同的迭代器
- 从迭代器中过滤元素的函数
    - islice: 在不进行复制的前提下, 根据索引值来切割迭代器
    - takewhile: 在判定函数为True的时候从迭代器中逐个返回元素
    - dropwhile: 从判定函数初次为False的地方开始, 逐个返回迭代器中的元素
    - filterfalse: 从迭代器中逐个返回能令判断函数为False的所有元素, 效果与filter相反
- 把迭代器中的元素组合起来的函数
    - product: 根据迭代器中的元素计算迪卡尔积, 并将其返回
    - permutations: 用迭代器中的元素构建长度为N的各种有序排列, 并将所有排列形式返回给调用者
    - combination: 用迭代器中的元素构建长度为N的各种无序组合, 并将所有排列形式返回给调用者

### 要点 ###

1. 应该用Python内置的模块来描述各种算法和数据结构
2. 开发者不应该自己去重新实现那些功能

## 第47条: 在重视精确度的场合, 应该使用decimal ##

### 介绍 ###

Decimal类解决了IEEE754浮点数产生的精度问题, 而且开发者还可以更加精确的控制该类的舍入行为. Decimal类提供了一个内置的函数可以按照开发者所要求的精度和舍入方式来准确的调整数值.

```
rate = Decimal('1.45')
seconds = Decimal('222')
cost = rate * seconds / Decimal('60')

rounded = cost.quantize(Decimal('0.01'), rounding=ROUND_UP)
```

Decimal在精度方面仍有局限, 如果需要精度不受限制的方式来表达有理数, 那么可以考虑用内置的fractions模块的Fraction类.

### 要点 ###

1. 对于编程中可能用到的每一种数值, 我们都可以拿对应的Python内置类型或者内置模块中的类来表示
2. Decimal类非常适合用在那种对精度要求很高, 且对舍入行为要求很严的场合

## 第48条: 学会安装由Python开发者社区所构建的模块 ##

### 介绍 ###

Python有个中央仓库, 里面存放这由Python社区构建并维护的各种模块. 我们可以用pip命令行工具来安装这些模块.

```
pip install pytz
```

### 要点 ###

1. PyPI包含了许多常用的软件包
2. pip是个命令行工具, 可以从PyPI中安装软件包
3. Python3.4及后续版本默认装有pip
4. 大部分PyPI模块, 都是自由软件或开源软件
