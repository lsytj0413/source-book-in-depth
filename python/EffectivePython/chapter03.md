# 第三章: 类与继承 #

## 第22条: 尽量用辅助类来维护程序的状态, 而不要用字典和元组 ##

### 介绍 ###

假设需要记录许多学生的成绩:

```
class SimpleGradeBook(object):
    def __init__(self):
        self._grades = {}
    def add_student(self, name):
        self._grades[name] = []
    def report_grade(self, name, score):
        self._grades[name].append(score)
    def average_grade(self, name):
        grades = self._grades[name]
        return sum(grades) / len(grades)
```
由于字典用起来很方便, 所以有可能因为功能过分膨胀而导致代码出问题.
假设需要扩充以上的SimpleGradeBook类使它按照科目来保存成绩:

```
class BySubjectGradeBook(object):
    def __init__(self):
        self._grades = {}
    def add_student(self, name):
        self._grades[name] = {}
    def report_grade(self, name, subject, score):
        by_subject = self._grades[name]
        grade_list = by_subject.setdefault(subject, [])
        grade_list.append(score)
    def average_grade(self, name):
        by_subject = self._grades[name]
        total, count = 0, 0
        for grades in by_subject.values():
            total += sum(grades)
            count += len(grades)
        return total / count
```
可以看到实现上已经稍显复杂, 现在假设需求上还需要记录科目成绩占科目总成绩的权重, 实现方式之一是修改内部的字典, 改用一系列元组作为值, 每个元组都具备 (score, weight) 的形式.

新修改的类用起来已经比较麻烦, 此时我们就应该从字典和元组迁移到类体系.

用来保存程序状态的数据结构一旦变得过于复杂, 就应该将其拆解为类, 以便提供更为明确的接口, 并更好的封装数据. 这样做也能够在接口与具体实现之间创建抽象层.

1. 科目考试成绩的重构

由于分数和权重都不再变化, 而且信息也比较简单, 所以可以使用元组来记录科目的历次考试成绩.

```
grades = []
grades.append((95, 0.45))
```
问题在于普通的元组只是按照位置来排布其中的各项数值. 如果我们要给每次成绩附加老师的评语, 则需要修改原来使用二元组的代码. 元组的长度逐步扩张也意味着代码渐趋复杂. 元组里的数据一旦超过两项, 就应该考虑用其他办法来实现.

collections模块中的namedtuple(具名元组) 类型非常适合这种需求, 很容易就能定义出精简而又不可变的数据类.

```
import collections
Grade = collections.namedtuple('Grade', ('score', 'weight'))
```
构建具名元组时, 即可以按照位置访问, 也可以采用关键字指定.

namedtuple也有局限:

- 无法指定各参数的默认值
- 属性依然可以通过下标及迭代访问, 这可能导致其他人以不符合设计者意图的方式使用这些元组

2. 科目类重构

```
class Subject(object):
    def __init__(self):
        self._grades = []
    def report_grade(self, score, weight):
        self._grades.append(Grade(score, weight))
    def average_grade(self):
        total, total_weight = 0, 0
        for grade in self._grades:
            total += grade.score * grade.weight
            total_weight += grade.weight
        return total / total_weight
```

3. 学生类重构

```
class Student(object):
    def __init__(self):
        self._subjects = {}
    def subject(self, name):
        if name not in self._subjects:
            self._subjects[name] = Subject()
        return self._subjects[name]
    def average_grade(self):
        total, count = 0, 0
        for subject in self._subjects.values():
            total += subject.average_grade()
            count += 1
        return total / count
```

4. 容器类重构

```
class Gradebook(object):
    def __init__(self):
        self._students = {}
    def student(self, name):
        if name not in self._students:
            self._students[name] = Student()
        return self._students[name]
```

### 要点 ###

1. 不要使用包含其他字典的字典, 也不要使用过长的元组
2. 如果容器中包含简单而又不可变的数据, 那么可以先使用 namedtuple 来表示, 待稍后有需要时再修改为完整的类
3. 保存内部状态的字典如果变得比较复杂, 就应该把这些拆解为多个辅助类

## 第23条: 简单的接口应该接受函数, 而不是类的实例 ##

### 介绍 ###

Python有许多内置的API都允许调用者传入函数(Hook)以定制其行为. 例如list类型的sort方法接受可选的key参数, 用以指定每个索引位置间应该如何排序.

```
names = ['Socrates', 'Archimedes', 'POlato']
names.sort(key=lambda x: len(x))
```
用函数作为Hook是比较合适的, 因为它们很容易就能描述这个挂钩的功能, 而且比定义一个类要简单.

例如要定制defaultdict类的行为, 在字典里找不到待查询的键时打印一条信息, 并返回0, 以作为该键所对应的值:

```
def log_missing():
    print('Key added')
    return 0
    
current = {'green': 12}
result = defaultdict(log_missing, current)
```
提供log_missing这样的函数可以使API更容易构建, 也更易测试, 因为它能够把附带的效果与确定的行为分离开.

例如要统计字典中遇到多少个缺失的键:

```
def increment_with_report(current, increments):
    added_count = 0
    
    def missing():
        nonlocal added_count
        added_count += 1
        return 0
        
    result = defaultdict(missing, current)
    for key, amount in increments:
        result[key] += amount
    return result, added_count
```
上面的代码使用了闭包函数来隐藏状态, 这样的缺点就是读起来要比无状态的函数难懂, 也可以定义一个小型的类来封装状态.

### 要点 ###

1. 对于连接各种Python组件的简单接口, 通常应该给其传入函数而不是某个类的实例
2. Python中的函数和方法都可以像一级类那样引用
3. 通过名为 \_\_call\_\_的特殊方法, 可以使类的实例像普通的Pyhton函数那样得到调用
4. 如果要用函数来保存状态, 就应该定义新的类并实现 \_\_call\_\_方法, 而不要定义带状态的闭包

## 第24条: 以 @classmethod形式的多态去通用地构建对象 ##

### 介绍 ###

在Python中类同对象一样也是支持多态的, 多态使得继承体系中的各个类都能以各自所独有的方式来实现某个方法.

例如为实现一套 MapReduce 流程, 先定义公共基类来表示输入的数据:

```
class InputData(object):
    def read(self):
        raise NotImplementedError
```
现在编写子类以便从磁盘文件里读取数据:

```
class PathInputData(InputData):
    def __init__(self, path):
        super().__init__()
        self.path = path
    def read(self):
        return open(self.path).read()
```

然后为MapReduce工作线程定义一套类似的抽象接口, 以便用标准的方式来处理输入的数据:

```
class Worker(object):
    def __init__(self, input_data):
        self.input_data = input_data
        self.result = None
    def map(self):
        raise NotImplementedError
    def reduce(self, other):
        raise NotImplementedError
```

接下来定义具体的Worker子类:

```
class LineCountWorker(Worker):
    def map(self):
        data = self.input_data.read()
        self.result = data.count('\n')
    def reduce(self, other):
        self.result += other.result
```
以上的MapReduce实现方式有一个问题, 就是如何把这些组件拼接起来.
例如可以手工构建相关对象并把它们联系起来:

```
def generate_inputs(data_dir):
    for name in os.listdir(data_dir):
        yield PathInputData(os.path.join(data_dir, name))
        
def create_workers(input_list):
    workers = []
    for input_data in input_list:
        workers.append(LineCountWorker(input_data))
    return workers
    
def execute(workers):
    threads = [Thread(target=w.map) for w in workers]
    for thread in threads: thread.start()
    for thread in threads: thread.join()
    
    first, rest = workers[0], workers[1:]
    for worker in rest:
        first.reduce(worker)
    return first.result
    
def mapreduce(data_dir):
    inputs = generate_inputs(data_dir)
    workers = create_workers(inputs)
    return execute(workers)
```

但是这种写法有个很严重的问题, 那就是MapReduce函数不够通用. 如果要编写新的InputData或Worker子类, 那就得重写generate\_inputs, create\_workers和mapreduce函数以便与之匹配.

解决这个问题的最佳方案, 是使用 @classmethod形式的多态.

首先修改InputData类, 添加通用的generate_inputs类方法:

```
class GenericInputData(object):
    def read(self):
        raise NotImplementedError
    @classmethod
    def generate_inputs(cls, config):
        raise NotImplementedError
        
class PathInputData(GenericInputData):
    def read(self):
        return open(self.path).read()
    @classmethod
    def generate_inputs(cls, config):
        data_dir = config['data_dir']
        for name in os.listdir(data_dir):
            yield cls(os.path.join(data_dir, name))
```

然后实现 GenericWorker类:

```
class GenericWorker(object):
    def map(self):
        raise NotImplementedError
    def reduce(self, other):
        raise NotImplementedError
    @classmethod
    def create_workers(cls, input_class, config):
        workers = []
        for input_data in input_class.generate_inputs(config):
            workers.append(cls(input_data))
        return workers
```

最后重写 mapreduce函数, 令其变得完全通用:

```
def mapreduce(worker_class, input_class, config):
    workers = worker_class.create_workers(input_class, config)
    return execute(workers)
```

### 要点 ###

1. 在Python程序中, 每个类只能有一个构造器, 即 \_\_init\_\_方法
2. 通过 @classmethod机制, 可以使用一种与构造器相仿的方式来构造类的对象
3. 通过类方法多态机制, 我们能够以更加通用的方式来构建并拼接具体的子类

## 第25条: 用super初始化父类 ##

### 介绍 ###

在Python中子类可以直接调用父类的 \_\_init\_\_方法来初始化父类, 但是这种写法有以下的问题:

1. 调用顺序不固定
2. 在钻石型继承之中, 会使顶部的公共基类多次执行 \_\_init\_\_方法

在Python2中增加了内置 super函数, 并且重新定义了方法解析顺序(MRO) 以解决这一问题, MRO以标准的流程来安排超类之间的初始化顺序(C3算法, 拓扑排序).

内置的super函数可以正常运作, 但是在Python2中有两个问题值得注意:

1. super语句有点麻烦, 必须指定当前类和self对象, 并且还要指定方法名称和方法参数
2. 调用super时必须写出当前类的名称, 在修改类名称时必须修改这一条语句

Python3中的super函数调用方式没有这些问题.

```
class Explicit(MyBaseClass):
    def __init__(self, value):
        super(__class__, self).__init__(value * 2)

class Implicit(MyBaseClass):
    def __init__(self, value):
        super().__init__(value * 2)
```

### 要点 ###

1. Python采用标准的方法解析顺序来解决超类初始化次序以及钻石继承问题
2. 总是应该使用内置的super函数来初始化父类

## 第26条: 只在使用Mix-in组件制作工具类时进行多重继承 ##

### 介绍 ###

我们应该尽量避开多重继承, 若一定要利用多重继承所带来的便利性以及封装性, 那就考虑编写 Mix-in类.
Mix-in是一种小型的类, 只定义了其他类可能需要的一套附加方法, 而不定义自己的实例属性, 此外也不要求调用自己的 \_\_init\_\_构造器.

例如要把内存中的Python对象转换为字典形式, 以便将其序列化:

```
class ToDictMixin(object):
    def to_dict(self):
        return self._traverse_dict(self.__dict__)
    def _traverse_dict(self, instance_dict):
        output = {}
        for key, value in instance_dict.items():
            output[key] = self._traverse(key, value)
        return output
    def _traverse(self, key, value):
        if isinstance(value, ToDictMixin):
            return value.to_dict()
        elif isinstance(value, dict):
            return self._traverse_dict(value)
        elif isinstance(value, list):
            return [self._traverse(key, i) for i in value]
        elif hasattr(value, '__dict__'):
            return self._traverse_dict(value.__dict__)
        else:
            return value
```
mix-in的最大优势在于, 使用者可以随时安插这些通用的功能, 并在必要的时候覆写它们.

### 要点 ###

1. 能用mix-in组件实现的效果, 就不要用多重继承来做
2. 将各功能实现为可插拔的mix-in组件, 然后令相关的类继承自己需要的那些组件即可定制该类实例所应具备的行为
3. 把简单的行为封装到mix-in组件里, 然后就可以用多个mix-in组合出复杂的行为

## 第27条: 多用public属性, 少用private属性 ##

### 介绍 ###

对于Python类来说, 其属性的可见度只有两种, 即public(公开的)和private(私密的).

以两个下划线开头的属性是private字段, 本类的方法可以直接访问它们, 在类的外面直接访问private字段会引发异常. 方法也同理.
Python会对私有属性的名称做一些简单的变换, 当编译器会把 \_\_private_field 字段的名称修改为 \_ClassName\_\_private\_field.

Python之所以不从语法上严格保证private字段的私密性:

1. 大家认为开放性比封闭好
2. Python语言本身就已经提供了一些属性挂钩, 使得开发者能够按照自己的需要来操作对象内部的数据

为了减少无意间访问内部属性所带来的意外, Python程序员会遵照《Python风格指南》的建议, 采用一种习惯性的命名方式来表示这种字段. 也就是说,
以单个下划线开头的字段, 应该视为protected字段.

一般不应该使用两个下划线开头的来表示不应该由子类或外部来访问的API或属性, 因为这在继承时可能会导致访问失效. 更合适的做法是让子类更多的去访问超类的protected属性, 并在文档中说明哪些字段是可供子类访问的属性.

只有一种情况应该使用private属性, 即避免子类的属性名和超类相冲突.

### 要点 ###

1. Python编译器无法严格保证private字段的私密性
2. 不要盲目的将属性设置为private, 并允许子类更多的访问超类的内部API
3. 应该更多的protected属性, 并在文档中描述. 不要试图用private属性来限制子类访问这些字段
4. 只有当子类不受自己控制时, 才应该使用private属性来避免名称冲突

## 第28条: 继承collections.abc 以实现自定义的容器类型 ##

### 介绍 ###

假设要创建一种自定义的列表类型, 并提供统计各元素出现频率的方法:

```
class FrequencyList(list):
    def __init__(self, members):
        super().__init__(members)
    def frequency(self):
        counts = {}
        for item in self:
            counts.setdefault(item, 0)
            counts[item] += 1
        return counts
```
上面的类获得了由list所提供的全部标准功能.

现在假设要编写这么一种对象, 它本身不属于list子类, 但用起来却和list一样, 也可能通过下标访问其中的元素. 假如要令下面这个表示二叉树节点的类, 也能够像list或tuple等序列那样来访问:

```
class BinaryNode(object):
    def __init__(self, value, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right
```

Python会用一些名称比较特殊的实例方法, 来实现与容器有关的行为. 用下标访问序列中元素时Python会转换为对 \_\_getitem\_\_ 方法的调用.

```
class IndexableNode(BinaryNode):
    def _search(self, count, index):
        # return (found, count)
    def __getitem__(self, index):
        found, _ = self._search(0, index)
        if not found:
            raise IndexError('Index out of range')
        return found.value
```

然而只实现 \_\_getitem\_\_ 方法是不够的, 它并不能使该类型支持我们想要的每一种序列操作. 例如想要内置的len函数正常运作, 就必须实现 \_\_len\_\_ 方法. 还有很多其他的方法.

为了在编写Python程序时方便的实现容器, 可以使用内置的 collections.abc 模块, 该模块定义了一系列抽象基类, 如果忘记实现某个方法, 该模块便会指出这个错误.

如果子类已经实现了抽象基类所要求的每个方法, 那么基类就会自动提供剩下的方法.

### 要点 ###

1. 如果要定义的子类比较简单, 就可以直接从Python的容器类型继承
2. 想正确实现自定义的容器类型, 可能需要编写大量的特殊方法
3. 编写自制的容器类型时, 可以从collections.abc 模块的抽象基类中继承, 那些基类能够确保我们的子类具备适当的接口及行为
