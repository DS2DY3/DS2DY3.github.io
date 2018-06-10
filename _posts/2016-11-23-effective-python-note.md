---
published: true
title: 《Effective Python》学习笔记
layout: post
description: Effective Python，学习笔记
category: Learning-Note
---
* 目录
{:toc}

## 1.Pythonic Thinking

### Item 1: Know Which Version of Python You’re Using

- 使用python -version查看当前Python版本
- Python的运行时版本：CPython，JyPython，IronPython和PyPy等


### Item 2: Follow the PEP 8 Style Guide

可以在线查看：http://www.python.org/dev/peps/pep-0008/

- Whitespace:
    - 使用四个空格缩进，使用四个空格对长表达式换行缩进
    - class和funciton之间用两空行，class的method之间用一个空行
    - list索引和函数调用，关键字参数赋值不用空格，变量赋值前后都只用一个空格
- Naming:
    - protected attribute用_leading_underscore格式，private attribute用__double_leading_underscore格式
- Expressions and Statements:
    - 不要使用肯定表达式的负：if not a is b
    - 不要判断空值（[]和''）的长度，if 空值就是False
    - import模块顺序：标准模块，第三方库，自己的模块，并且进行字母排序

### Item 3: Know the Differences Between bytes, str, and unicode

- Python3两种字符串类型：bytes和str，bytes表示8-bit的二进制值，str表示unicode字符
- Python2两种字符串类型：str和unicode，str表示8-bit的二进制值，unicode表示unicode字符
- 二进制值和unicode字符需要经过encode和decode转换，Python2的unicode和Python3的str没有关联二进制编码，通常使用UTF-8
- Python2转换函数：
    - to_unicode

      ~~~ python
      # Python 2
      def to_unicode(unicode_or_str):
          if isinstance(unicode_or_str, str):
              value = unicode_or_str.decode(‘utf-8’)
          else:
              value = unicode_or_str
          return value # Instance of unicode
      ~~~

    - to_str

      ~~~ python
      # Python 2
      def to_str(unicode_or_str):
          if isinstance(unicode_or_str, unicode):
              value = unicode_or_str.encode(‘utf-8’)
          else:
              value = unicode_or_str
          return value # Instance of str
      ~~~

- Python2，如果str只包含7-bit的ascii字符，unicode和str是一样的类型，所以：
    - 使用+连接：str + unicode
    - 可以对str和unicode进行比较
    - unicode可以使用格式字符串，'%s'
    - 上面的规则Python3不能用
- 使用open返回的文件操作，在Python3是默认进行UTF-8编码，但在Pyhton2是二进制编码

  ~~~ python
  with open(‘/tmp/random.bin’, ‘w’) as f:
      f.write(os.urandom(10))
  # >>>
  #TypeError: must be str, not bytes
  ~~~

### Item 4: Write Helper Functions Instead of Complex Expressions

### Item 5: Know How to Slice Sequences

- list，str，bytes和实现__getitem__和__setitem__的类都支持slice操作
- somelist[start:end]，不包括end，-1表示最后一个
- slice list是shadow copy，somelist[-0:]会复制原list
- slice赋值会修改slice list，即使长度不一致（增删改）

### Item 6: Avoid Using start, end, and stride in a Single Slice

- 避免同时使用start，end和stride进行一次slice操作，可以拆分成两次slice操作

### Item 7: Use List Comprehensions Instead of map and filter

- map和filter需要lambda函数，使得代码更不可读

### Item 8: Avoid More Than Two Expressions in List Comprehensions

- 使用list comprehensions目的就是扁平化和可读性，避免使用多于两个表达式，if条件后置
- prefer

  ~~~ python
  matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
  flat = [x for row in matrix for x in row]
  ~~~

- not

  ~~~ python
  squared = [[x**2 for x in row] for row in matrix]
  ~~~

### Item 9: Consider Generator Expressions for Large Comprehensions

- list comprehension有个问题：会一次生成全部元素。使用generator表达式避免大量内存的分配
- generator表达式还可以组合使用，而且执行非常快：

  ~~~ python
  it = [100, 57, 15, 1, 12, 75, 5, 86, 89, 11]
  roots = ((x, x**0.5) for x in it)
  print(next(roots))
  # (15, 3.872983346207417)
  ~~~

- The only gotcha is that the iterators returned by generator expressions are stateful, so you must be careful not to use them more than once

### Item 10: Prefer enumerate Over range

- enumerate wraps any iterator with a lazy generator.
- Prefer

  ~~~ python
  for i, flavor in enumerate(flavor_list):
      print(‘%d: %s’ % (i + 1, flavor))
  ~~~

- not

  ~~~ python
  for i in range(len(flavor_list)):
      flavor = flavor_list[i]
          print(‘%d: %s’ % (i + 1, flavor))
  ~~~

- enumerate不但可以获得索引，还能指定开始索引

### Item 11: Use zip to Process Iterators in Parallel

- Prefer

  ~~~ python
  names = [‘Cecilia’, ‘Lise’, ‘Marie’]
  letters = [len(n) for n in names]
  for name, count in zip(names, letters):
      if count > max_letters:
          longest_name = name
          max_letters = count
  ~~~

- not

  ~~~ python
  for i, name in enumerate(names):
  count = letters[i]
      if count > max_letters:
          longest_name = name
          max_letters = count
  ~~~

- In Python 3, zip wraps two or more iterators with a lazy generator. The zip generator yields tuples containing the next value from each iterator.

### Item 12: Avoid else Blocks After for and while Loops

- 循环后面的else块的行为会造成困扰

### Item 13: Take Advantage of Each Block in try/except/else/finally

~~~ python
UNDEFINED = object()
def divide_json(path):
    handle = open(path, ‘r+’) # May raise IOError
    try:
        data = handle.read() # May raise UnicodeDecodeError
        op = json.loads(data) # May raise ValueError
        value = (
        op[‘numerator’] /
        op[‘denominator’]) # May raise ZeroDivisionError
    except ZeroDivisionError as e:
        return UNDEFINED
    else:
        op[‘result’] = value
        result = json.dumps(op)
        handle.seek(0)
        handle.write(result) # May raise IOError
        return value
    finally:
        handle.close() # Always runs
~~~

## 2.Functions

### Item 14: Prefer Exceptions to Returning None

- 不能用None代替Exception返回

### Item 15: Know How Closures Interact with Variable Scope

- Python编译器变量查找域的顺序：
    1. The current function’s scope
    2. Any enclosing scopes (like other containing functions)
    3. The scope of the module that contains the code (also called the global scope)
    4. The built-in scope (that contains functions like len and str)
- 如果变量有赋值，这个变量就是当前作用域定义的新变量（覆盖上一层），注意UnboundLocalError
- Python3 nonlocal, Python2使用可变变量（list等）绕过

### Item 16: Consider Generators Instead of Returning Lists

### Item 17: Be Defensive When Iterating Over Arguments

- generator不能重用：

  ~~~ python
    it = read_visits(‘/tmp/my_numbers.txt’)
    print(list(it))
    print(list(it)) # Already exhausted
    # >>>
    [15, 35, 80]
    []
  ~~~

- for,list等Python标准库会捕获StopIteration异常
- list复制generator iterator就可以多次遍历

  ~~~ python
  def normalize_copy(numbers):
      numbers = list(numbers) # Copy the iterator
      total = sum(numbers)
      result = []
      for value in numbers:
      percent = 100 * value / total
      result.append(percent)
      return result
  ~~~

- 每次调用都创建iterator避免上面list分配内存

  ~~~ python
    def normalize_func(get_iter):
        total = sum(get_iter()) # New iterator
        result = []
        for value in get_iter(): # New iterator
        percent = 100 * value / total
        result.append(percent)
        return result
  ~~~
- for循环会调用内置iter函数，进而调用对象的__iter__方法，\__iter__会返回iterator对象（实现__next__方法）
- 用iter函数检测iterator：

  ~~~ python
    def normalize_defensive(numbers):
        if iter(numbers) is iter(numbers): # An iterator — bad!
            raise TypeError(‘Must supply a container’)
        total = sum(numbers)
        result = []
        for value in numbers:
            percent = 100 * value / total
            result.append(percent)
        return result

        visits = [15, 35, 80]
        normalize_defensive(visits) # No error

        it = iter(visits)
        normalize_defensive(it)
        # >>>
        TypeError: Must supply a container
  ~~~

### Item 18: Reduce Visual Noise with Variable Positional Arguments

- def function(\*args)需要注意两点：
    - 如果*args传入generator会生成所有元素，造成内存消耗
    - 给函数新增*args参数，调用没有及时跟进，会出现很难发现的bug

### Item 19: Provide Optional Behavior with Keyword Arguments

- 默认参数传值最好加上关键字，防止多个默认参数干扰

### Item 20: Use None and Docstrings to Specify Dynamic Default Arguments

- 默认参数的值只会在函数模块加载时候生成，对于可变类型会产生奇怪的行为
- 所以，使用None作为默认参数
    - prefer

      ~~~ python
      def log(message, when=None):
          when = datetime.now() if when is None else when
          print(‘%s: %s’ % (when, message))

      log(‘Hi there!’)
      sleep(0.1)
      log(‘Hi again!’)

      # >>>
      # 2014-11-15 21:10:10.472303: Hi there!
      # 2014-11-15 21:10:10.573395: Hi again!
      ~~~
    - not

      ~~~ python
      def log(message, when=datetime.now()):
          print(‘%s: %s’ % (when, message))
      log(‘Hi there!’)
      sleep(0.1)
      log(‘Hi again!’)
      # >>>
      # 2014-11-15 21:10:10.371432: Hi there!
      #2014-11-15 21:10:10.371432: Hi again!
      ~~~

### Item 21: Enforce Clarity with Keyword-Only Arguments

## 3. Classes and Inheritance

### Item 22: Prefer Helper Classes Over Bookkeeping with Dictionaries and Tuples

- 避免dict嵌套dict或大tuple
- 轻量不可变容器可以用namedtuple
- 拆分多个层辅助类代替复杂dict

### Item 23: Accept Functions for Simple Interfaces Instead of Classes

- 实例对象也可以当做函数调用，并且会执行__call__函数
- 如果需要维护函数的状态，可以定义一个有__call__的类代替闭包：
    - prefer

      ~~~ python
      class BetterCountMissing(object):
              def __init__(self):
                  self.added = 0
              def __call__(self):
                  self.added += 1
                  return 0
              counter = BetterCountMissing()
              counter()
      ~~~

    - not

      ~~~ python
      def increment_with_report(current, increments):
          added_count = 0
          def missing():
              nonlocal added_count # Stateful closure
              added_count += 1
              return 0
          result = defaultdict(missing, current)
          for key, amount in increments:
          result[key] += amount
          return result, added_count
      ~~~  

### Item 24: Use @classmethod Polymorphism to Construct Objects Generically

- Python只允许一个__init__方法，使用@classmethod定义构造函数实现多态

### Item 25: Initialize Parent Classes with super

- 理解MRO规则：深度优先，从左到右，如果有重复保留后面的，C3 linearization
- 始终使用super初始化父类

### Item 26: Use Multiple Inheritance Only for Mix-in Utility Classes

- 只有Mix-in类使用多重继承
    http://blog.csdn.net/gzlaiyonghao/article/details/1656969

### Item 27: Prefer Public Attributes Over Private Ones
- \__private_field可以通过_classname__private_filed在外部访问，尽可能不用

### Item 28: Inherit from collections.abc for Custom Container Types

- 自定义容器类继承collections.abc抽象类确保实现所有接口和行为

## 4. Metaclasses and Attributes

### Item 29: Use Plain Attributes Instead of Get and Set Methods

- 使用public属性避免set和get方法，@property定义一些特别的行为
- 确保@property方法是快速的，如果是慢或者复杂的工作用正常的方法

### Item 30: Consider @property Instead of Refactoring Attributes

- 使用@property给已有属性扩展新需求，当@property太复杂了才考虑重构

### Item 31: Use Descriptors for Reusable @property Methods

- descriptor:
    - def \__get__(*args,**kwargs)
    - def \__set__(*args,**kwargs)
- 需要大量@property方法的类可以使用descriptor实现
- 用WeakKeyDictionary确保descriptor类不会引起内存泄露

### Item 32: Use \__getattr__, \__getattribute__, and \__setattr__ for Lazy Attributes

- obj.name，getattr和hasattr都会调用__getattribute__方法，如果name不在obj.\__dict__里面，还会调用__getattr__方法，如果没有自定义__getattr__方法会AttributeError异常
- 只要有赋值操作（=，setattr）都会调用__setattr__方法（包括a = A()）

### Item 33: Validate Subclasses with Metaclasses

- 使用元类对类型对象进行验证
- 元类的__new__会在类语句全部执行完后调用

### Item 34: Register Class Existence with Metaclasses

- 使用元类进行类信息注册，序列化，orm

### Item 35: Annotate Class Attributes with Metaclasses

- 利用元类修改类属性，元类和descriptor对类属性进行伪装匿名

## 5. Concurrency and Parallelism

### Item 36: Use subprocess to Manage Child Processes

- 使用subprocess模块运行子进程管理自己的输入和输出流
- subprocess可以并行执行最大化CPU的使用
- communicate的timeout参数避免死锁和被挂起的子进程

### Item 37: Use Threads for Blocking I/O, Avoid for Parallelism

- 因为GIL，Python thread并不能并行运行多段代码
- Python保留thread的两个原因：1.可以模拟多线程，2.多线程可以处理I/O阻塞的情况
- Python thread可以并行执行多个系统调用，这个可以用来做并行计算

### Item 38: Use Lock to Prevent Data Races in Threads

- 虽然Python thread不能同时执行，但是Python解释器还是会打断操作数据的两个字节码指令，所以还是需要锁
- thread模块的Lock类是Python的互斥锁实现

### Item 39: Use Queue to Coordinate Work Between Threads

- Queue类具备构建健壮并发管道的特性：阻塞操作，缓存大小和连接

### Item 40: Consider Coroutines to Run Many Functions Concurrently

- 线程有三个大问题：
    - 需要特定工具去确定安全性
    - 单个线程需要8M的内存
    - 线程启动消耗
- coroutine只有1kb的内存消耗
- generator可以通过send方法把值传递给yield

  ~~~ python
  def my_coroutine():
      while True:
          received = yield
          print("Received:", received)
  it = my_coroutine()
  next(it)
  it.send("First")
  ('Received:', 'First')
  ~~~

- Python2不支持直接yield generator，可以使用for循环yield

### Item 41: Consider concurrent.futures for True Parallelism

- CPU瓶颈模块使用C扩展
- concurrent.futures的multiprocessing可以并行处理一些任务，Python2没有这个模块

## 6. Built-in Modules

### Item 42: Define Function Decorators with functools.wraps

- 装饰器可以对函数进行封装，但是会改变函数信息
- 使用functools的warps可以解决这个问题

  ~~~ python
  def trace(func):
      @wraps(func)
      def wrapper(*args, **kwargs):
          # …
      return wrapper
  @trace
  def fibonacci(n):
      # …
  ~~~

### Item 43: Consider contextlib and with Statements for Reusable try/finally Behavior

- 使用with语句代替try/finally，增加代码可读性
- 使用contextlib提供的contextmanager装饰函数就可以被with使用
- with和yield返回值使用

### Item 44: Make pickle Reliable with copyreg

- pickle模块只能序列化和反序列化确认没有问题的对象
- copyreg的pickle支持属性丢失，版本和导入类表信息

### Item 45: Use datetime Instead of time for Local Clocks

- 不要使用time模块在转换不同时区的时间
- 而用datetime配合pytz转换，总数保持UTC时间，最后面在输出本地时间

### Item 46: Use Built-in Algorithms and Data Structures

- 使用内置的算法和数据结构：
    - collections.deque
    - collections.OrderedDict
    - collection.defaultdict
    - heapq模块操作list（优先队列）：heappush，heappop和nsmallest

      ~~~ python
      a = []
      heappush(a, 5)
      heappush(a, 3)
      heappush(a, 7)
      heappush(a, 4)
      print(heappop(a), heappop(a), heappop(a), heappop(a))
      # >>>
      # 3 4 5 7
      ~~~

    - bisect模块：bisect_left可以对有序列表进行高效二分查找
    - itertools模块（Python2不一定支持）：
        - 连接迭代器：chain，cycle，tee和zip_longest
        - 过滤：islice，takewhile，dropwhile，filterfalse
        - 组合不同迭代器：product，permutations和combination

### Item 47: Use decimal When Precision Is Paramount

- 高精度要求的使用Decimal处理

### Item 48: Know Where to Find Community-Built Modules

- 在 https://pypi.python.org 查找通用模块，并且用pip安装

## 7. Collaboration

### Item 49: Write Docstrings for Every Function, Class, and Module

### Item 50: Use Packages to Organize Modules and Provide Stable APIs

### Item 51: Define a Root Exception to Insulate Callers from APIs

### Item 52: Know How to Break Circular Dependencies

### Item 53: Use Virtual Environments for Isolated and Reproducible Dependencies

## 8. Production

### Item 54: Consider Module-Scoped Code to Configure Deployment Environments

### Item 55: Use repr Strings for Debugging Output

- repr作用于内置类型会产生可打印的字符串，eval可以获得这个字符串的原始值
- \__repr__自定义上面输出的字符串

### Item 56: Test Everything with unittest

- 使用unittest编写测试用例，不光是单元测试，集成测试也很重要
- 继承TestCase，并且每个方法名都以test开始

### Item 57: Consider Interactive Debugging with pdb

- 启用pdb，然后在配合shell命令调试
    import pdb;
    pdb.set_trace();

### Item 58: Profile Before Optimizing

- cProfile比profile更精准
    - ncalls:调用次数
    - tottime:函数自身耗时，不包括调用函数的耗时
    - cumtime:包括调用的函数耗时

### Item 59: Use tracemalloc to Understand Memory Usage and Leaks

- gc模块可以知道有哪些对象存在，但是不知道怎么分配的
- tracemalloc可以得到内存的使用情况，但是只在Python3.4及其以上版本提供
