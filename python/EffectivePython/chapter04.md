# 第四章: 元类及属性 #

## 第29条: 用纯属性取代get和set方法 ##

### 介绍 ###

setter和getter等工具方法, 有助于定义类的接口, 它也使得开发者能够更加方便的封装功能, 验证用法并限定取值范围.
但是对于Python来说, 基本上不需要手工实现setter和getter方法, 应该先从简单的public属性开始使用.

如果想在设置属性的时候实现特殊行为, 可以改用 @property 修改器和setter方法来做.

```
class VoltageResistance(Resistor):
    def __init__(self, ohms):
        super().__init__(ohms)
        self._voltage = 0
    @property
    def voltage(self):
        return self._voltage
    @voltage.setter
    def voltage(self, voltage):
        self._voltage = voltage
        self.current = self._voltage / self.ohms
```

我们也可以用 @property 来防止父类的属性遭到修改:

```
class RixedResistance(Resistor):
    @property
    def ohms(self):
        return slef._ohms
    @ohms.setter
    def onms(self, ohms):
        if hasattr(slef, '_ohms'):
            raise AttributeError("Can't set attribute")
        self._ohms = ohms
```

@property 属性最大的缺点在于: 和属性相关的方法, 只能在子类里面共享, 而与之无关的其他类无法复用同一份实现代码.

### 要点 ###

1. 编写新类时, 应该用简单的public属性来定义其接口, 而不要手工实现set和get方法
2. 如果访问对象的某个属性时, 需要表现出特殊的行为, 那就用 @property 来定义这种行为
3. @property 方法应该遵循最小惊讶原则, 而不应该产生奇怪的副作用
4. @property方法需要执行得迅速一些, 耗时或复杂的工作应该放在普通的方法里面

## 第30条: 考虑用 @property 来替代属性重构 ##

### 介绍 ###

Python内置的 @property 修饰器, 使调用者能够轻松的访问该类的实例属性. 它也可以把简单的数值属性迁移为实时计算的属性.

假设要用纯Python对象实现带有配额的漏桶(漏桶算法), 把当前剩余的配额以及重置配额的周期放在Bucket类里:

```
class Bucket(object):
    def __init__(self, period):
        self.period_delta = timedelta(seconds=period)
        self.reset_time = datetime.now()
        self.max_quota = 0
        self.quota_consumed = 0
        
    @property
    def quota(self):
        return self.max_quota - self.quota_consumed
        
    @quota.setter
    def quota(slef, amount):
        delta = self.max_quota - amount
        if amount == 0:
            self.quota_consumed = 0
            self.max_quota = 0
        elif delta < 0:
            assert self.quota_consumed == 0
            self.max_quota = amount
        else:
            assert self.max_quota >= self.quota_consumed
            self.quota_consumed += delta
```

### 要点 ###

1. @property 可以为现有的实例属性添加新的功能
2. 可以用 @property 来逐步完善数据模型
3. 如果 @property 用的太过频繁, 那就应该考虑彻底重构该类并修改相关的调用代码

## 第31条: 用描述符来改写需要复用的 @property 方法 ##

### 介绍 ###

Python内置的 @property 修饰器有个明显的缺点: 就是不便于复用, 受它修饰的这些方法, 无法为同一个类中的其他属性所复用, 而且与之无关的类也无法复用这些方法.

```
class Homework(object):
    def __init__(self):
        self._grade = 0
    @property
    def grade(self):
        return self._grade
    @grade.setter
    def grade(self, value):
        if not (0 <= value <= 100):
            raise ValueError('Grade must be between 0 and 100')
        self._grade = grade
```

现在假设要把这套验证逻辑放在考试成绩上面, 而考试成绩又是多个科目的小成绩组成, 每一科都要单独计分:

```
class Exam(object):
    def __init__(self):
        self._writing_grade = 0
        self._math_grade = 0
    @staticmethod
    def _check_grade(value):
        if not (0 <= value <= 100):
            raise ValueError('Grade must be between 0 and 100')
    @property
    def writing_grade(self):
        return self._writing_grade
    @writing_grade.setter
    def writing_grade(self, value):
        self._check_grade(value)
        self._writing_grade = value
    @property
    def math_grade(self):
        return self._math_grade
    @math_grade.setter
    def math_grade(self, value):
        self._check_grade(value)
        self._math_grade = value
```
这种写法不够通用, 需要反复编写 @property 代码和 \_check\_grade方法.

还有一种方式能够更好的实现上述要求, 那就是采用Python的描述符. Python会通过描述符协议来对访问操作进行一定的转义.
描述符类可以提供 \_\_get\_\_ 和 \_\_set\_\_ 方法, 使得开发者无需再编写例行代码, 即可复用分数验证功能.

```
class Grade(object):
    def __get__(*args, **kwargs):
        #
    def __set__(*args, **kwargs):
        #
        
class Exam(object):
    math_grade = Grade()
    writing_grade = Grade()
    
exam = Exam()
exam.writing_grade = 40
# 转义之后的代码
# Exam.__dict__['writing_grade'].__set__(exam, 40)
```

之所以会有这样的转义, 关键就在于object类的 \_\_getattribute\_\_ 方法. 简单来说, 如果Exam实例没有名为 writing_grade 的属性,
那么Python就会转向Exam类, 并在该类中寻找同名的属性, 而如果这个类属性实现了 \_\_get\_\_ 和 \_\_set\_\_ 方法的对象, 那么Python就会默认对象遵从描述符协议.

我们可以使用以下的 Grade类来实现Homework类的分数验证逻辑:

```
class Grade(object):
    def __init__(self):
        self._value = 0
    def __get__(self, instance, instance_type):
        return self._value
    def __set__(self, instance, value):
        if not (0 <= value <= 100):
            raise ValueError('Grade must be between 0 and 100')
        self._value = value
```
不幸的是, 上面这种实现方式是错误的, 它会导致不符合预期的行为. 在多个Exam实例上面分别操作某一属性, 就会导致错误的结果. 因为对于 writing_grade 这个类属性来说, 它只会在程序的生命期中构建一次.

为解决这个问题, 我们需要把每个Exam实例所对应的值记录到 Grade中.

```
class Grade(object):
    def __init__(self):
        self._values = {}
    def __get__(self, instance, instance_type):
        if instance is None: return self
        return self._values.get(instance, 0)
    def __set__(self, instance, value):
        if not (0 <= value <= 100):
            raise ValueError('Grade must be between 0 and 100')
        self._values[instance] = value
```
上面这种方式简单而且正确, 但是它会泄漏内存. 因为 values字典会保存指向 Exam实例的引用, 从而导致该实例的引用计数不能降为0, 垃圾回收器无法将其回收.

使用WeakKeyDictionary的特殊字典来解决此问题, 如果运行期系统发现这种字典所持有的引用是整个程序里面的最后一个引用, 那么系统就会自动将该实例从字典的键中移除.

### 要点 ###

1. 如果想复用 @property 方法及其验证机制, 那么可以自己定义描述符类
2. WeakKeyDictionary 可以保证描述符类不会泄漏内存
3. 通过描述符协议来实现属性的获取和设置操作时, 不要纠结与 \_\_getattribute\_\_ 的方法具体运作细节.

## 第32条: 用 \_\_getattr\_\_, \_\_getattribute\_\_ 和 \_\_setattr\_\_ 实现按需生成的属性 ##

### 介绍 ###

如果某个类定义了 \_\_getattr\_\_ , 同时系统在该类对象的实例字典中又找不到待查询的属性, 那么系统就会调用这个方法.

```
class LazyDB(object):
    def __getattr__(self, name):
        value = 'Value for %s' % (name)
        setattr(self, name, value)
        return value
```

然后给LazyDB添加记录功能, 把程序对 \_\_getattr\_\_ 的调用行为记录下来:

```
class LoggingLazyDB(LazyDB):
    def __getattr__(self, name):
        print('Called __getattr__(%s)' % (name))
        return super().__getattr__(name)     # 避免递归调用
```

程序每次访问对象的属性时, Pyhton系统都会调用 \_\_getattribute\_\_ 方法, 即使属性字典里已经有了该属性.

```
class ValidatingDB(object):
    def __getattribute__(self, name):
         print('Called __getattribute__(%s)' % (name))
         try:
             return super().__getattribute__(name)
         except AttributeError:
             value = 'Value for %s' % (name)
             setattr(self, name, value)
             return value
```

我们经常会使用 hasattr函数来判断对象是否已经拥有了相关的属性, 并用内置 getattr 函数来获取属性值. 这些函数会先在实例字典中搜索待查询的属性, 然再调用 \_\_getattr\_\_ .

使用 \_\_getattribute\_\_ 和 \_\_setatttr\_\_ Hook方法时需要注意, 每次访问对象属性时它们都会触发, 而这可能并不是你想要的结果.

```
class BrokenDictionaryDB(object):
    def __getattribute__(self, name):
        return self._data[name]
```
上面这段代码会在访问 self.\_data时再次调用 \_\_getattribute\_\_ , 并导致无限循环.

1. \_\_getattribute\_\_(self, name): 当特性name被访问时使用, 新式类, 同时拦截对\_\_dict\_\_的访问, 在该函数中访问与self相关的属性时, 使用super函数是唯一安全的路径
2. \_\_getattr\_\_(self, name): 当特性name被访问且对象没有相应的特性时使用
3. \_\_setattr\_\_(self, name, value): 当试图给特性name赋值时调用
4. \_\_delattr\_\_(self, name): 当试图删除特性name时调用

### 要点 ###

1. 通过 \_\_getattr\_\_ 和 \_\_setattr\_\_ , 我们可以用惰性的方式来加载并保存对象的属性
2. 要理解 \_\_getattr\_\_ 与 \_\_getattribute\_\_ 的区别: 前者只会在待访问的属性缺失时触发, 而后者会在每次访问属性时触发
3. 如果要在 \_\_getattribute\_\_ 和 \_\_setatttr\_\_ 方法中访问实例属性, 那么应该直接通过 super() 来做, 以避免无限递归

## 第33条: 用元类来验证子类 ##

### 介绍 ###

在构建复杂的类体系时, 我们可能需要确保类的风格协调一致, 确保某些方法得到了覆写, 或是确保类属性之间具备某些严格的关系, 元类提供了一种可靠的验证方式, 当开发者定义新的类时, 都会运行验证代码以确保这个新类符合预订的规范.

定义元类的时候要从type中继承, 而对于使用该元类的其他类来说, Pyhton默认会把那些类的class语句体中所含有的相关内容都发送给元类的 \_\_new\_\_ 方法, 我们就可以在系统构建那种类型之前先修改那个类的信息.

```
class Meta(type):
    def __new__(meta, name, bases, class_dict):
        return type.__new__(meta, name, bases, class_dict)

# Python3
class MyClass(object, metaclass=Meta):
    stuff = 123
    def foo(self):
        pass
        
# Python2
class MyClassInPython2(object):
    __metaclass__ = Meta
```

为了在定义某个类的时候确保该类的所有参数都有效, 我们可以把相关的验证逻辑添加到 Meta.\_\_new\_\_ 方法中.

例如, 定义一个验证多边形的基类与元类.

```
class ValidatePolygon(type):
    def __new__(meta, name, bases, class_dict):
        # 跳过对多边形类的基类的验证
        if bases != (object, ):
            if class_dict['sides'] < 3:
                raise ValueError('Polygons need 3+ sides')
        return type.__new__(meta, name, bases, class_dict)
        
class Polygon(object, metaclass=ValidatePolygon):
    sides = None
    
class Triangle(Polygon):
    sides = 3
```

假如我们尝试定义一种边数小于3的多边形子类, 那么class语句体刚一结束, 元类中的验证代码立刻就会拒绝这个class.

### 要点 ###

1. 通过元类, 我们可以在生成子类对象之前, 先验证子类的定义是否合乎规范
2. Python2和Python3指定元类的语法略有不同
3. Pyhton系统把子类的整个class语句体处理完毕之后, 就会调用其元类的 \_\_new\_\_ 方法

## 第34条: 用元类来注册子类 ##

### 介绍 ###

元类还有一个元类, 就是在程序中自动注册类型.

例如, 需要将Python对象表示为JSON格式的序列化数据:

```
class Serializable(object):
    def __init__(self, *args):
        self.args = args
    def serialize(self):
        return json.dumps({'args': self.args})
        
class Point2D(Serializable):
    def __init__(self, x, y):
        super().__init__(x, y)
        self.x = x
        self.y = y
```

然后构建一个反序列化的基类:

```
class Deserializable(Serializable):
    @classmethod
    def deserialize(cls, json_data):
        params = json.loads(json_data)
        return cls(*params['args'])
```
这种方法的缺点是, 我们必须提前知道序列化的数据是什么类型, 然后才能对其做反序列化操作. 理想的方案是, 有很多类都可以把本类对象转换为JSON格式的序列化字符串, 但是只需要一个公用的反序列化函数, 就可以将任意的JSON字符串还原成相应的Python对象.

```
class BetterSerializable(object):
    def __init__(self, *args):
        self.args = args
    def serialize(self):
        return json.dumps({
            'class': self.__class__.__name__,
            'args': self.args
        })
        
registry = {}

def register_class(target_class):
    registry[target_class.__name__] = target_class

def deserialize(data):
    params = json.loads(data)
    name = params['class']
    target_class = registry[name]
    return target_class(*params['args'])
```
在这种方案中, 我们必须用register\_class 把将来可能要执行反序列化操作的那些类都注册一遍, 但是开发者可能会忘记调用 registry\_class 函数.
我们应该保证程序会自动调用 register\_class 函数将新的子类注册好, 这个功能可以通过元类来实现.

```
class Meta(type):
    def __new__(meta, name, bases, class_dict):
        cls = type.__new__(meta, name, bases, class_dict)
        register_class(cls)
        return cls
```
这个元类可以自动实现类的注册, 以确保每一个子类都不会遗漏.
这种方案, 适用于序列化和反序列化操作, 也适用于数据库的对象关系映射, 插件系统和系统挂钩.

### 要点 ###

1. 在构建模块化的Python程序时, 类的注册是一种很有用的模式
2. 开发者每次从基类中继承子类时, 基类的元素都可以自动运行注册代码
3. 通过元类来实现类的注册, 可以确保所有子类都不会遗漏, 从而避免后续的错误

## 第35条: 用元类来注解类的属性 ##

### 介绍 ###

元类还有一个更有用处的功能, 那就是可以在某个类刚定义好的但是尚未使用的时候提前修改或注解类的属性. 这种写法通常会与描述符搭配起来, 令这些属性可以更加详细的了解自己在外围类中的使用方式.

例如, 要定义新类来表示客户数据库里的某一行, 同时在类的相关属性与数据库表的每一列之间建立对应关系.

```
class Field(object):
    def __init__(self, name):
        self.name = name
        self.internal_name = '_' + name
    def __get__(self, instance, instance_type):
        if instance is None: return self
        return getattr(instance, self.internal_name)
    def __set__(self, instance, instance_type):
        setattr(instance, self.internal_name, value)
        
# 表示数据行的类
class Customer(object):
    first_name = Field('first_name')
    last_name = Field('last_name')
    prefix = Field('prefix')
    suffix = Field('suffix')
```

但是上面的代码有些重复, 因为需要重复把字段名传给Field的构造器. 我们可以使用元类改写:

```
class Meta(type):
    def __new__(meta, name, bases, class_dict):
        for key, value in class_dict.items():
            if isinstance(value, Field):
                value.name = key
                value.internal_name = '_' + key
        cls = type.__new__(meta, name, bases, class_dict)
        return cls
        
# 数据库行的基类
class DatabaseRow(object, metaclass=Meta):
    pass
```

### 要点 ###

1. 借助元类, 我们可以在某个类完全定义好之前, 率先修改该类的属性
2. 描述符与元类能够有效的组合起来, 以便对某种行为做出修饰器, 或在程序运行时探查相关信息
3. 如果把元类与描述符相结合, 那就可以在不使用weakref模块的前提下避免内存泄漏
