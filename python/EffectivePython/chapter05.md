# 第五章: 并发及并行 #

## 第36条: 用subprocess模块来管理子进程 ##

### 介绍 ###

用subprocess模块运行子进程, 读取子进程的输出信息并等待其终止:

```
proc = subprocess.Popen(
           ['echo', 'Hello from the child!'],
           stdout=subprocess.PIPE)
out, err = proc.communicate()
```

开发者也可以从Python程序向子进程输送数据, 然后获取子进程的输出数据.

```
def run_openssl(data):
    env = os.environ.copy()
    env['password'] = b'\xe24U\n\xd0Q13S\x11'
    proc = subprocess.Popen(
               ['openssl', 'enc', '-des3', '-pass', 'env:password'],
               env=env,
               stdin=subprocess.PIPE,
               stdout=subprocess.PIPE)
    proc.stdin.write(data)
    proc.stdin.flush()
    return proc
    
# 把一些随机生成的字节数据传给加密函数
procs = []
for _ in range(3):
    data = os.urandom(10)
    proc = run_openssl(data)
    procs.append(proc)
    
# 等待输出
for proc in procs:
    out, err = proc.communicate()
    print(out[-10:0])
```

我们也可以把子进程的输出作为其余子进程的输入, 例如将ssl的输出作为md5的输入进行HASH:

```
def run_md5(input_stdin):
    proc = subprocess.Popen(
               ['md5'],
               stdin=input_stdin,
               stdout=subprocess.PIPE)
    return proc
```

如果担心子进程一直不终止, 或担心它的输出管道及输出管道由于某些原因发生了阻塞, 那么可以给communicate函数传入timeout参数, 以便没有给出响应时抛出异常。

不过, timeout参数仅在 Python3.3 及后续版本中有效, 对于之前的Python版本来说, 我们需要使用内置的select模块来处理, 以确保I/O操作的超时机制能够生效.

### 要点 ###

1. 可以用subprocess模块运行子进程, 并管理其输入流与输出流
2. Python解释器能够平行地运行多条子进程, 这使得开发者可以充分利用CPU的处理能力
3. 可以给communicate函数传入timeout参数, 以避免进程死锁或失去响应

## 第37条: 可以用线程来执行阻塞式I/O, 但不要用它做平行计算 ##

### 介绍 ###

标准CPython解释器中的多线程程序会受到GIL的影响, 并不能利用多线程的优势, 这样Python为何还要支持多线程呢?

1. 多线程使得程序看上去好像能够在同一时间做许多事情
2. 处理阻塞式的IO操作, 开发者可以借助线程把这些耗时的IO操作隔离开

尽管受制于GIL, 但是用多个Python线程来执行系统调用的时候可以平行的执行, 线程在执行系统调用的时候会释放GIL, 并且一直等到执行完成才会重新获取GIL.

### 要点 ###

1. 因为受到GIL的限制, 所以多条Python线程不能在多个CPU核心上面平行的执行字节码
2. Python多线程可以轻松地模拟出同一时刻执行多项任务的效果
3. 多个Python线程, 可以平行的执行多个系统调用, 使得程序在执行阻塞的IO操作的同时执行一些运算操作

## 第38条: 在线程中使用Lock来防止数据竞争 ##

### 介绍 ###

假设使用Counter类来进行计数:

```
class Counter(object):
    def __init__(self):
        self.count = 0
    def increment(self, offset):
        self.count += offset
        
# 工作线程
def worker(counter):
    counter.increment(1)
```
在多线程情况下count的计数会出现数据竞争而导致错误, 为了防止此类的数据竞争行为, Python在内置的threading模块里提供了一套健壮的工具, 例如Lock类来进行数据保护.

```
class Counter(object):
    def __init__(self):
        self.lock = Lock()
        self.count = 0
    def increment(self, offset):
        with self.lock()
            self.count += offset
```

### 要点 ###

1. 虽然Python有全局解释器锁, 但是在编写自己的程序时依然要设法防止多个线程争用同一份数据
2. 如果在不加锁的情况下允许多条线程修改同一个对象, 那么程序的数据结构可能会遭到破坏
3. 在Python内置的threading模块中有个类名叫 Lock, 它用标准的方式实现了互斥锁

## 第39条: 用Queue来协调各线程之间的工作 ##

### 介绍 ###

我们可以使用线程安全的方式来对生产者-消费者队列进行建模:

```
class MyQueue(object):
    def __init__(self):
        self.items = deque()
        self.lock = Lock()
    def put(self, item):
        with self.lock():
            self.items.append(item)
    def get(self):
        with self.lock:
            return self.item.popleft()
            
# worker线程
class Worker(Thread):
    def __init__(self, func, in_queue, out_queue):
        super().__init__()
        self.func = func
        self.in_queue = in_queue
        self.out_queue = out_queue
        self.polled_count = 0
        self.work_done = 0
    def run(self):
        while True:
            self.polled_count += 1
            try:
                item = self.in_queue.get()
            except IndexError:
                sleep(0.01)
            else:
                retult = self.func(item)
                self.out_queue.put(result)
                self.work_done += 1
                
# 启动队列与工作线程
download_queue = MyQueue()
resize_queue = MyQueue()
upload_queue = MyQueue()
done_queue = MyQueue()
threads = [
    Worker(download, download_queue, resize_queue),
    Worker(resize, resize_queue, upload_queue),
    Worker(upload, upload_queue, done_queue)
    ]
```
这个范例程序可以正常运行, 但是有几个问题:

1. 在run中需要捕获IndexError异常, 在线程的工作进度不匹配时会多次出现
2. 为了判断所有的任务是否都彻底处理完毕, 我们必须再编写一个循环来判断done_queue中任务的数量
3. 没有办法通知worker线程的run函数退出
4. 如果任务中的某个阶段发生迟滞, 则可能导致程序崩溃

我们可以使用Queue类来弥补自编队列的缺陷, Queue类的get方法会持续阻塞, 直到有新的数据加入; 我们也可以使用Queue类来限定队列中待处理的最大任务数据; 还可以通过task_done方法来追踪工作的进度.

```
class ClosableQueue(Queue):
    SENTINEL = object()
    def close(self):
        self.put(self.SENTINEL)
    def __iter__(self):
        while True:
            item = slef.get()
            try:
                if item is self.SENTINEL:
                    return
                yield item
            finally:
                slef.task_done()
                
class StoppableWorker(Thread):
    def __init__(self, func, in_queue, out_queue):
        #
    def run(self):
        for item in self.in_queue:
            result = self.func(item)
            self.out_queue.put(result)
```

### 要点 ###

1. 管线是一种优秀的处理方式, 它可以把处理流程划分为若干阶段, 并使用多条Python线程来同时执行这些任务
2. 构建并发式的管线时, 要注意许多问题: 比如如何防止持续等待, 如何停止工作线程, 如何防止内存膨胀等
3. 可以使用Queue类来构建优秀的管线

## 第40条: 考虑用协程来并发地运行多个函数 ##

### 介绍 ###

Python可以使用线程来运行多个函数, 但是线程有以下的缺点:

1. 为了确保数据安全, 必须用特殊的工具来协调线程, 会令程序变得难于扩展和维护
2. 线程需要占用大量的内存, 大约8MB
3. 线程的启动开销大

Python的协程可以避免上述问题, 协程的实现方式实际上是对生成器的扩展. 协程的工作原理是这样的: 每当生成器函数执行到yield表达式的时候, 消耗生成器的那段代码就通过send函数给生成器回传一个值, 而生成器在收到了经由send函数所传进来的这个值之后会将其视为yield表达式的执行结果.

```
def my_coroutine():
    while True:
        received = yield
        print('Received:', received)
        
it = my_coroutine()
# 先调用next函数以便将生成器推进到yield表达式处
next(it)
it.send('First')
it.send('Second')
```
使用生成器可以模拟线程的行为.

下面使用一个生命游戏的例子来演示协程的协同运作效果. 游戏很简单, 在一个二维表格中每个细胞都处于生存或空白的状态.

```
# 状态定义
ALIVE = '*'
EMPTY = '_'

# 查询对象
Query = namedtuple('Query', ('y', 'x'))

# 查询周边细胞的状态生成器
def count_neighbors(y, x):
    n_ = yield Query(y + 1, x + 0)
    ne = yield Query(y + 1, x + 1)
    # e_, se, s_, sw, w_, nw
    neighbors_states = [n_, ne, e_, se, s_, sw, w_, nw]
    count = 0
    for state in neighbors_states:
        if state == ALIVE:
            count += 1
    return count

Transition = namedtuple('Transition', ('y', 'x', 'state'))

# 状态迁移生成器
def stop_cell(y, x):
    state = yield Query(y, x)
    # 组合生成器协程
    neighbors = yield from count_neighbors(y, x)
    next_state = game_logic(state, neighbors)
    yield Transition(y, x, next_state)
    
# 游戏逻辑函数
def game_logic(state, neighbors):
    if state == ALIVE:
        # 若本细胞存活且周围存活数不等于3则本细胞下一轮转换为死亡
        if neighbors < 2:
            return EMPTY
        elif neighbors > 3:
            return EMPTY
    else:
        # 若本细胞死亡且周围存活数刚好等于3, 则本细胞下一轮转换为存活
        if neighbors == 3:
            return ALIVE
    return state

TICK = object()

# 推进所有细胞状态转换
def simulate(height, width):
    while True:
        for y in range(height):
            for x in range(width):
                yield from step_cell(y, x)
        yiled TICK
        
# 定义整个网络
class Grid(object):
    def __init__(self, height, width):
        self.height = height
        self.width = width
        self.rows = []
        for _ in range(self.height):
            self.rows.append([EMPTY] * self.width)
            
    def query(self, y, x):
        return self.rows[y % self.height][x % self.width]
    def assign(self, y, x, state):
        self.rows[y % self.height][x % self.width] = state
        
def live_a_generation(grid, sim):
    progeny = Grid(grid.height, grid.width)
    item = next(sim)
    while item is not TICK:
        if isinstance(item, Query):
            state = grid.query(item.y, item.x)
            item = sim.send(state)
        else:
            progeny.assign(item.y, item.x, item.state)
            item = next(sim)
    return progeny
```

#### Python2中的协程 ####

Python2中没有 yield from 表达式, 这代表需要把两个生成器协程组合起来, 那就需要再委派给另一个协程:

```
def delegated():
    yield 1
    yield 2
    
def composed():
    yield 'A'
    for value in delegated():
        yield value
    yield 'B'
```
Python2中也不支持在生成器中编写带返回值的return语句, 为了通过try/except/finally代码块正确的实现出与Python3相同的行为, 我们需要定义自己的异常类型, 并在需要返回某个值的时候抛出该异常.

```
class MyReturn(Exception):
    def __init__(self, value):
        self.value = value
        super(MyReturn, self).__init__()
        
def delegated():
    yield 1
    raise MyReturn(2)
    yield 'Not reached'
    
def composed():
    try:
        for value in delegated():
            yield value
    except MyReturn as e:
        output = e.value
    yield output * 4
```

### 要点 ###

1. 协程提供了一种有效的方式, 令程序看上去好像能够同时运行大量函数
2. 对于生成器内的yield表达式来说, 外部代码通过send方法传给生成器的那个值就是该表达式所要具备的值
3. 协程是一种强大的工具, 它可以把程序的核心逻辑同程序外部环境交互时所使用的代码相分离
4. Python2不支持yield from 表达式, 也不支持从生成器内通过return语句向外部返回某个值

## 第41条: 考虑用concurrent.futures来实现真正的并行计算 ##

### 介绍 ###

我们可以通过内置的concurrent.futures模块来利用另一个名叫multiprocessing的内置模块, 以实现可以利用多个CPU核心的并行计算, 解决性能问题. 这种做法会以子进程的形式平行的运行多个Python解释器, 从而令Python程序能够利用多核心CPU来提升执行速度.

先编写一个查找最大公约数的方法:

```
def gcd(pair):
    a, b = pair
    low = min(a, b)
    for i in range(low, 0, -1):
        if a % i == 0 and b % i == 0:
            return i
```

假设需要求取各组数据的最大公约数, 我们可以采用多线程方式:

```
numbers = []  # 数据
pool = ThreadPoolExecutor(max_workers=2)
results = list(pool.map(gcd, numbers))
```
上面这个版本的程序可能会比单线程的程序运行得还要慢, 因为线程的启动以及与线程池的通行都会有开销.

利用concurrent.futures可以提升整体的速度:

```
numbers = []  # 数据
pool = ProcessPoolExecutor(max_workers=2)
results = list(pool.map(gcd, numbers))
```

ProcessPoolExecutor会逐步完成以下操作:

1. 把numbers列表中的每一项输入数据都传给map
2. 用pickle模块对数据进行序列化
3. 通过本地套接字将序列化之后的数据从主解释器所在的进程发送到子解释器所在的进程
4. 在子进程中用pickle对数据进行反序列化操作, 将其还原为Python对象
5. 引入包含gcd函数的那个Python模块
6. 各子进程平行的对各自的数据数据运行gcd函数
7. 对运行结果进行序列化操作
8. 将这些数据通过socket复制到主进程中
9. 主进程对数据进行反序列化操作, 将其还原为Python对象
10. 把每个子进程得出的结果合并到一份列表, 并返回给调用者

使用这套方案一般需要满足两个条件: 一是运行的函数不需要与程序中的其他部分共享状态, 二是在进程中传递的数据量小.

### 要点 ###

1. 把引发CPU性能瓶颈的那部分代码用C语言扩展模块来改写, 即可在尽量发挥Python特性的前提下有效提升程序的执行速度. 但是这样做的工作量比较大, 而且可能会引入BUG.
2. multiprocsssing模块提供了一些强大的工具, 对于某些类型的任务来说开发者只需要编写少量代码, 即可实现平行计算
3. 若想利用强大的multiprocessing模块, 最恰当的方式是通过concurrent.futures模块以及ProcessPoolExecutor类来使用它
4. multiprocsssing模块所提供的那些高级功能都特别复杂, 所以开发者尽量不要直接使用它们
