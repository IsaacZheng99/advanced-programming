# 装饰器

## 1. 写在前面

- 这里的装饰器是指的广义的装饰器，并不单指python中的装饰器，但是目前只介绍了python中的装饰器。
- 整理了一下咨询 ChatGPT 有关python装饰器的问题：
  - 装饰器的主要作用是在不修改原函数代码的情况下，动态地添加功能或修改函数的行为。
  - 在 Python 中，装饰器是一种特殊的语法，用于在函数定义前添加 @ 符号和装饰器函数。
  - 装饰器是一种通用的编程概念，不仅限于 Python，其他编程语言也可以实现类似的功能。例如，在 Java 中，可以使用注解（Annotation）来实现类似于 Python 装饰器的功能。在 C++ 中，可以使用模板元编程（Template Metaprogramming）来实现类似的功能。
  - 因此，装饰器不是 Python 独有的概念，它是一种通用的编程概念，可以在许多编程语言中使用。
- 笔者个人认为 ChatGPT 的说法是没什么问题的，虽然笔者没有查到有关的其他说法，但是从个人理解上来看是ok的，个人认为理解其思想是最重要的。当然，如果有不准确的地方，欢迎指正。

## 2. 示例

### 2.1 python @decorator_function

#### 2.1.1 使用场景

- 用于装饰模式的实现，这也是最直观的做法。

- 用于注册，常用于策略模式的字典映射的注册，或者一批需要按顺序但是同时执行的操作。（核心还是注册，具体如何使用根据需求来即可）代码示例如下：

  ```python
  # 这里给出一批需要按顺序但是同时执行的操作
  
  # 存储对应操作的函数
  g_Triggers: List = []
  
  
  # 操作类型
  class Type:
      A = 1
      B = 2
      C = 3
  
  
  # 装饰器（注意这里是有参装饰器）
  def register(type: int):
      def decorator(func):
          if type >= len(g_Triggers):
              g_Triggers.extend([None for _ in range(type + 1 - len(g_Triggers))])
          if g_Triggers[type]:
              return func
          g_Triggers[type] = func
          return func
      return decorator
  
  
  @register(Type.A)
  def a(i):
      print(i)
  @register(Type.B)
  def b(i):
      print(i)
  @register(Type.C)
  def c(i):
      print(i)
  
  
  def execute(i):
      for func in g_Triggers:
          if func:
              func(i)
  ```

#### 2.1.2 分类

- 无参装饰器。

  - 代码示例如下：

    ```python
    def my_decorator(func):
        def wrapper():
            print("Before the function is called.")
            func()
            print("After the function is called.")
        return wrapper
    
    @my_decorator
    def say_hello():
        print("Hello!")
    
    say_hello()
    ```

  - wrapper里面的逻辑会在被装饰函数执行时被执行是因为装饰器本质上是一个高阶函数，它的输入参数是一个函数对象，而输出参数是一个新的函数对象。当我们使用装饰器修饰一个函数时，实际上是将这个函数对象作为参数传递给装饰器函数，然后用装饰器函数的返回值来替换原来的函数对象。

- 有参装饰器。

  - 代码示例如下：

    ```python
    def repeat(num):
        def my_decorator(func):
            def wrapper():
                for i in range(num):
                    func()
            return wrapper
        return my_decorator
    
    @repeat(num=3)
    def say_hello():
        print("Hello!")
    
    say_hello()
    ```

  - 对于有参装饰器，实际的装饰器函数是第一层和第二层函数的结合（repeat和my_decorator），第一层函数用于接收装饰器参数，第二层函数用于接收被装饰函数，最终返回一个闭包函数（wrapper）。

#### 2.1.3 总结思考

- 有参装饰器和无参装饰器的本质区别在于它们生成闭包函数的方式不同。无参装饰器生成闭包函数的方式是通过一个单层函数，而有参装饰器生成闭包函数的方式是通过一个嵌套的函数。
- 我们可以把装饰器函数的三层分别称为传参层、语法糖层、核心逻辑层。
- 对于两种装饰器，最终都是把被装饰函数替换成了wrapper()函数。
- 对于有参装饰器，并不是必须要三层，例如在2.1.1中笔者举的使用装饰器进行注册的例子，这里装饰器只用来注册，不需要添加额外功能，所以可以不用第三层的wrapper()。



