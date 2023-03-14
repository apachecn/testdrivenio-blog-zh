# Python 中的干净代码

> 原文：<https://testdriven.io/blog/clean-code-python/>

在本文中，我们将讨论干净的代码——它的好处，不同的代码标准和原则，以及如何编写干净代码的一般准则。

## 什么是干净代码？

干净的代码是一组规则和原则，有助于保持代码的可读性、可维护性和可扩展性。这是编写高质量软件的最重要的方面之一。我们(开发人员)花在阅读代码上的时间比实际写代码的时间要多得多，这就是为什么我们写好代码很重要。

编写代码很容易，但是编写好的、干净的代码却很难。

我们写的代码应该简单，有表现力，并且没有很多重复。表达性代码意味着，即使我们只是向计算机提供指令，当人类阅读时，它仍然应该是可读的，并清楚地传达其意图。

### 干净代码的重要性

编写干净的代码有很多好处。例如，干净的代码是:

*   容易理解
*   更有效率
*   更易于维护、扩展、调试和重构

它也倾向于需要更少的文档。

## 代码标准

代码标准是编码规则、指南和最佳实践的集合。每种编程语言都有自己的编码标准，为了编写更简洁的代码，应该遵循这些标准。他们通常会处理:

*   文件组织
*   编程-实践和原则
*   代码格式(缩进、声明、语句)
*   命名规格
*   评论

### PEP 8 (Python 增强提案)

PEP 8 是一个描述 Python 编码标准的风格指南。这是 Python 社区中最受欢迎的指南。最重要的规则如下:

PEP 8 命名约定:

*   类名应该是驼色(`MyClass`)
*   变量名应该是 snake_case 并且全部小写(`first_name`)
*   函数名应该是 snake_case 并且全部小写(`quick_sort()`)
*   常量应该是 snake_case 并且全部大写(`PI = 3.14159`)
*   模块应该有简短的蛇名和小写字母(`numpy`)
*   单引号和双引号被同等对待(只需选择一个并保持一致)

PEP 8 线条格式:

*   使用 4 个空格缩进(空格优先于制表符)
*   行数不应超过 79 个字符
*   避免在同一行出现多个语句
*   顶级函数和类定义用两个空行包围
*   类中的方法定义由一个空行包围
*   导入应该在单独的行上

PEP 8 空白:

*   避免括号或大括号内有多余的空格
*   避免尾随空白
*   始终用一个空格将二元运算符括起来
*   如果使用具有不同优先级的操作符，可以考虑在优先级最低的操作符周围添加空格
*   当用于表示关键字参数时，不要在=符号周围使用空格

PEP 8 评论:

*   注释不应与代码相矛盾
*   评论应该是完整的句子
*   注释的#号后面应该有一个空格，第一个单词大写
*   函数中使用的多行注释(文档字符串)应该有一个简短的单行描述，后面跟着更多的文本

如果你想了解更多，请阅读官方 PEP 8 参考资料。

### Pythonic 代码

Python 代码是 Python 社区采用的一组习惯用法。这仅仅意味着你很好地使用了 Python 的习惯用法和范例，以使你的代码更加清晰、易读和高性能。

Pythonic 代码包括:

*   可变戏法
*   列表操作(初始化、切片)
*   处理函数
*   显式代码

写 Python 代码和写 Python 代码有很大的区别。要编写 Python 代码，你不能只是习惯性地将另一种语言(如 Java 或 C++)翻译成 Python 你需要用 Python 来思考。

让我们看一个例子。我们必须像这样把前 10 个数字加在一起。

非 Pythonic 解决方案应该是这样的:

```
`n = 10
sum_all = 0

for i in range(1, n + 1):
    sum_all = sum_all + i

print(sum_all)  # 55` 
```

一个更 Pythonic 化的解决方案可能是这样的:

```
`n = 10
sum_all = sum(range(1, n + 1))

print(sum_all)  # 55` 
```

第二个例子对于有经验的 Python 开发人员来说更容易理解，但是它需要对 Python 的内置函数和语法有更深的理解。编写 Python 代码最简单的方法是在编写代码时牢记 Python 的[禅，并逐步学习 Python 的](https://www.python.org/dev/peps/pep-0020/)[标准库](https://docs.python.org/3/library/)。

### Python 的禅

Python 的禅是用 Python 写计算机程序的 19 个“指导原则”的集合。这本集子是软件工程师蒂姆·彼得斯在 1999 年写的。它作为复活节彩蛋包含在 Python 解释器中。

您可以通过执行以下命令来查看它:

```
`>>> import this

The Zen of Python, by Tim Peters

Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Although that way may not be obvious at first unless you're Dutch.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!` 
```

如果你对这首“诗”的含义感到好奇，请查看[Python 的禅宗，解释](https://inventwithpython.com/blog/2018/08/17/the-zen-of-python-explained/)，它提供了逐行的解释。

## 代码原则

为了编写更好的代码，您可以遵循许多编码原则，每种原则都有自己的优缺点和权衡。这篇文章涵盖了四个比较流行的原则:干，吻，SoC，和固体。

### 干(不重复)

> 每一项知识都必须在系统中有一个单一的、明确的、权威的表示。

这是最简单的编码原则之一。它唯一的规则是代码不能重复。不要复制行，要找到一种使用迭代的算法。干代码易于维护。您可以在模型/数据抽象中进一步应用这一原则。

DRY 原则的缺点是，你最终会有太多的抽象、外部依赖和复杂的代码。如果你试图改变你的代码库的一大块，DRY 也会导致复杂。这就是为什么你应该避免过早地干燥你的代码。拥有一些重复的代码段总比错误的抽象要好。

### 亲吻(保持简单，笨蛋)

> 大多数系统如果保持简单，而不是变得复杂，就会工作得最好。

KISS 原则指出，如果系统保持简单而不是变得复杂，大多数系统会工作得最好。简单应该是设计中的一个关键目标，应该避免不必要的复杂。

### 关注点分离

> SoC 是一种设计原则，用于将计算机程序分成不同的部分，以便每个部分解决一个单独的问题。关注点是影响计算机程序代码的一组信息。

SoC 的一个很好的例子就是 [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) (模型-视图-控制器)。

如果你决定采用这种方法，注意不要把你的应用分成太多的模块。您应该只在有意义的时候创建新模块。更多的模块意味着更多的问题。

### 固体

> SOLID 是五个设计原则的首字母缩写词，旨在使软件设计更易于理解、灵活和维护。

SOLID 在编写 OOP 代码时极其有用。它谈到了将你的类分成多个子类、继承、抽象、接口等等。

它由以下五个概念组成:

## 代码格式化程序

代码格式化程序通过自动格式化来加强编码风格，并帮助实现和维护干净的代码。它们中的大多数允许您创建一个可以与同事共享的样式配置文件。

最流行的 Python 代码格式化程序有:

大多数现代 ide 还包括 linters，它在您键入时在后台运行，帮助您识别小的编码错误、错误和危险的代码模式，并保持您的代码格式化。有两种类型的短绒:逻辑和风格。

最受欢迎的蟒蛇皮有:

> 关于林挺和代码格式化的更多信息，请查看 [Python 代码质量](/blog/python-code-quality/)。

## 命名规格

编写干净代码的一个最重要的方面是命名约定。你应该总是使用有意义的和揭示意图的名字。使用长的、描述性的名字总比带注释的短名字好。

```
`# This is bad
# represents the number of active users
au = 55

# This is good
active_user_amount = 55` 
```

在接下来的两节中，我们将会看到更多的例子。

## 变量

### 1.使用名词作为变量名

### 2.使用描述性/揭示意图的名称

其他开发人员应该能够通过读取变量的名称来计算出变量存储的内容。

```
`# This is bad
c = 5
d = 12

# This is good
city_counter = 5
elapsed_time_in_days = 12` 
```

### 3.使用易发音的名字

你应该总是使用容易发音的名字；否则，你将很难大声解释你的算法。

```
`from datetime import datetime

# This is bad
genyyyymmddhhmmss = datetime.strptime('04/27/95 07:14:22', '%m/%d/%y %H:%M:%S')

# This is good
generation_datetime = datetime.strptime('04/27/95 07:14:22', '%m/%d/%y %H:%M:%S')` 
```

### 4.避免使用模糊的缩写

不要试图想出自己的缩写。变量最好有一个更长的名字，而不是一个容易混淆的名字。

```
`# This is bad
fna = 'Bob'
cre_tmstp = 1621535852

# This is good
first_name = 'Bob'
creation_timestamp = 1621535852` 
```

### 5.总是使用相同的词汇

命名变量时避免使用同义词。

```
`# This is bad
client_first_name = 'Bob'
customer_last_name = 'Smith'

# This is good
client_first_name = 'Bob'
client_last_name = 'Smith'` 
```

### 6.不要使用“神奇的数字”

幻数是代码中出现的奇怪数字，没有明确的含义。让我们来看一个例子:

```
`import random

# This is bad
def roll():
    return random.randint(0, 36)  # what is 36 supposed to represent?

# This is good
ROULETTE_POCKET_COUNT = 36

def roll():
    return random.randint(0, ROULETTE_POCKET_COUNT)` 
```

我们可以不使用幻数，而是将它们提取到一个有意义的变量中。

### 7.使用解决方案域名

如果你在你的算法或者类中使用了很多不同的数据类型，并且你不能从变量名本身中找出它们，不要害怕给你的变量名加上数据类型后缀。例如:

```
`# This is good
score_list = [12, 33, 14, 24]
word_dict = {
    'a': 'apple',
    'b': 'banana',
    'c': 'cherry',
}` 
```

这里有一个不好的例子(因为你不能从变量名中判断出数据类型):

```
`# This is bad
names = ["Nick", "Mike", "John"]` 
```

### 8.不要添加多余的上下文

不要在变量名中添加不必要的数据，尤其是在处理类的时候。

```
`# This is bad
class Person:
    def __init__(self, person_first_name, person_last_name, person_age):
        self.person_first_name = person_first_name
        self.person_last_name = person_last_name
        self.person_age = person_age

# This is good
class Person:
    def __init__(self, first_name, last_name, age):
        self.first_name = first_name
        self.last_name = last_name
        self.age = age` 
```

我们已经在`Person`类中，所以没有必要给每个类变量添加前缀`person_`。

## 功能

### 1.函数名使用动词

### 2.不要用不同的词来表达同一个概念

为每个概念选择一个词，并坚持下去。对同一个概念使用不同的词会引起混淆。

```
`# This is bad
def get_name(): pass
def fetch_age(): pass

# This is good
def get_name(): pass
def get_age(): pass` 
```

### 3.编写简短的函数

### 4.函数应该只执行一项任务

如果你的函数包含关键字“and ”,你可以把它分成两个函数。让我们看一个例子:

```
`# This is bad
def fetch_and_display_personnel():
    data = # ...

    for person in data:
        print(person)

# This is good
def fetch_personnel():
    return # ...

def display_personnel(data):
    for person in data:
        print(person)` 
```

函数应该做一件事，作为读者，它们做你期望它们做的事。

> 一个好的经验法则是，任何给定的函数都不应该花超过几分钟的时间来理解。回顾一下你几个月前写的一些旧代码。你可能应该重构任何需要超过五分钟才能理解的函数。这毕竟是你的代码。想想另一个开发者要多久才能理解。

### 5.将你的论点保持在最低限度

函数中的参数应该保持最小。理想情况下，您的函数应该只有一到两个参数。如果您需要为函数提供更多的参数，您可以创建一个 config 对象，将它传递给函数或将它分成多个函数。

示例:

```
`# This is bad
def render_blog_post(title, author, created_timestamp, updated_timestamp, content):
    # ...

render_blog_post("Clean code", "Nik Tomazic", 1622148362, 1622148362, "...")

# This is good
class BlogPost:
    def __init__(self, title, author, created_timestamp, updated_timestamp, content):
        self.title = title
        self.author = author
        self.created_timestamp = created_timestamp
        self.updated_timestamp = updated_timestamp
        self.content = content

blog_post1 = BlogPost("Clean code", "Nik Tomazic", 1622148362, 1622148362, "...")

def render_blog_post(blog_post):
    # ...

render_blog_post(blog_post1)` 
```

### 6.不要在函数中使用标志

标志是传递给函数的变量(通常是布尔值)，函数用它来决定自己的行为。它们被认为是糟糕的设计，因为函数应该只执行一个任务。避免标记的最简单的方法是把你的函数分成更小的函数。

```
`text = "This is a cool blog post."

# This is bad
def transform(text, uppercase):
    if uppercase:
        return text.upper()
    else:
        return text.lower()

uppercase_text = transform(text, True)
lowercase_text = transform(text, False)

# This is good
def uppercase(text):
    return text.upper()

def lowercase(text):
    return text.lower()

uppercase_text = uppercase(text)
lowercase_text = lowercase(text)` 
```

### 7.避免副作用

如果一个函数除了接受一个值并返回另一个或多个值之外，还做其他任何事情，它就会产生副作用。例如，副作用可能是写入文件或修改全局变量。

无论我们如何努力去写干净的代码，你的程序中仍然会有一些部分需要额外的解释。注释允许我们快速地告诉其他开发人员(以及我们未来的自己)为什么我们以这样的方式编写它。请记住，添加太多的注释会使你的代码比没有注释时更混乱。

代码注释和文档有什么区别？

| 类型 | 答案 | 利益相关者 |
| --- | --- | --- |
| 证明文件 | 何时和如何 | 用户 |
| 代码注释 | 为什么 | 开发商 |
| 干净的代码 | 什么 | 开发商 |

> 要了解代码注释和文档之间的更多区别，请查看[记录 Python 代码和项目](/blog/documenting-python/)一文。

注释糟糕的代码——也就是`# TODO: RE-WRITE THIS TO BE BETTER`——只能在短期内帮助你。迟早你的一个同事将不得不处理你的代码，在花了几个小时试图弄清楚它做了什么之后，他们最终会重写它。

如果你的代码足够易读，你就不需要注释。添加无用的注释只会降低代码的可读性。这里有一个不好的例子:

```
`# This checks if the user with the given ID doesn't exist.
if not User.objects.filter(id=user_id).exists():
    return Response({
        'detail': 'The user with this ID does not exist.',
    })` 
```

一般来说，如果你需要添加评论，它们应该解释你为什么做某事，而不是正在发生什么。

不要添加对代码没有任何价值的注释。这很糟糕:

```
`numbers = [1, 2, 3, 4, 5]

# This variable stores the average of list of numbers.
average = sum(numbers) / len(numbers)
print(average)` 
```

这也不好:

![Cat comment meme](img/1efa355dbd18f1ca1fd857352e7eaf77.png)

大多数编程语言都有不同的注释类型。了解它们的区别，并相应地使用它们。您还应该学习注释文档语法。一个很好的例子:

```
`def model_to_dict(instance, fields=None, exclude=None):
    """
 Returns a dict containing the data in ``instance`` suitable for passing as
 a Form's ``initial`` keyword argument.
 ``fields`` is an optional list of field names. If provided, return only the
 named.
 ``exclude`` is an optional list of field names. If provided, exclude the
 named from the returned dict, even if they are listed in the ``fields``
 argument.
 """
    opts = instance._meta
    data = {}
    for f in chain(opts.concrete_fields, opts.private_fields, opts.many_to_many):
        if not getattr(f, 'editable', False):
            continue
        if fields is not None and f.name not in fields:
            continue
        if exclude and f.name in exclude:
            continue
        data[f.name] = f.value_from_object(instance)
    return data` 
```

你能做的最糟糕的事情就是在你的程序中把代码注释掉。在进入版本控制系统之前，所有的调试代码或调试消息都应该被删除，否则，你的同事会害怕删除它，而你的注释代码会永远留在那里。

## 装饰器、上下文管理器、迭代器和生成器

在这一节中，我们将了解一些 Python 概念和技巧，我们可以用它们来编写更好的代码。

### 装修工

装饰器是 Python 中一个非常强大的工具，它允许我们为函数添加一些自定义功能。本质上，它们只是被称为内部函数的函数。通过使用它们，我们利用了 SoC(关注点分离)原则，并使我们的代码更加模块化。学习它们，你将踏上通往 Pythonic 代码的道路！

假设我们有一台受密码保护的服务器。我们可以在每个服务器方法中询问密码，或者创建一个装饰器来保护我们的服务器方法，如下所示:

```
`def ask_for_passcode(func):
    def inner():
        print('What is the passcode?')
        passcode = input()

        if passcode != '1234':
            print('Wrong passcode.')
        else:
            print('Access granted.')
            func()

    return inner

@ask_for_passcode
def start():
    print("Server has been started.")

@ask_for_passcode
def end():
    print("Server has been stopped.")

start()  # decorator will ask for password
end()  # decorator will ask for password` 
```

我们的服务器现在会在每次调用`start()`或`end()`时询问密码。

### 上下文管理器

上下文管理器简化了我们与外部资源(如文件和数据库)的交互方式。最常见的用法是`with`语句。它们的好处是可以自动释放块外的内存。

让我们看一个例子:

```
`with open('wisdom.txt', 'w') as opened_file:
    opened_file.write('Python is cool.')

# opened_file has been closed.` 
```

如果没有上下文管理器，我们的代码将如下所示:

```
`file = open('wisdom.txt', 'w')
try:
    file.write('Python is cool.')
finally:
    file.close()` 
```

### 迭代程序

迭代器是包含可数个值的对象。迭代器允许对象被迭代，这意味着您可以遍历所有的值。

假设我们有一个名字列表，我们想遍历它。我们可以使用`next(names)`循环遍历它:

```
`names = ["Mike", "John", "Steve"]
names_iterator = iter(names)

for i in range(len(names)):
    print(next(names_iterator))` 
```

或者使用增强循环:

```
`names = ["Mike", "John", "Steve"]

for name in names:
    print(name)` 
```

> 在增强循环中，避免使用像`item`或`value`这样的变量名，因为这样会很难判断一个变量存储了什么，尤其是在嵌套的增强循环中。

### 发电机

生成器是 Python 中的一个函数，它返回一个迭代器对象，而不是一个单一的值。普通函数和生成器的主要区别在于，生成器使用`yield`关键字，而不是`return`。迭代器中的每个下一个值都是使用`next(generator)`获取的。

假设我们想要生成第一个`x`的倍数`n`。我们的生成器看起来会像这样:

```
`def multiple_generator(x, n):
    for i in range(1, n + 1):
        yield x * i

multiples_of_5 = multiple_generator(5, 3)
print(next(multiples_of_5))  # 5
print(next(multiples_of_5))  # 10
print(next(multiples_of_5))  # 15` 
```

## 模块化和类

为了让你的代码尽可能的有条理，你应该把它分成多个文件，然后这些文件被分成不同的目录。如果你用面向对象的语言编写代码，你也应该遵循基本的面向对象的原则，比如封装、抽象、继承和多态。

将代码分成多个类将使你的代码更容易理解和维护。一个文件或一个类应该有多长并没有固定的规则，但是尽可能保持它们很小(最好在 200 行以下)。

Django 的默认项目结构是一个很好的例子，说明了你的代码应该如何构建:

```
`awesomeproject/
├── main/
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── blog/
│   ├── migrations/
│   │   └── __init__.py
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── models.py
│   ├── tests.py
│   ├── urls.py
│   └── views.py
└── templates` 
```

Django 是一个 MTV(模型-模板-视图)框架，类似于我们之前讨论的 MVC 框架。这种模式将程序逻辑分成三个相互联系的部分。你可以看到每个应用程序都在一个单独的目录中，每个文件都服务于一个特定的事物。如果你的项目被分成多个应用程序，你应该确保这些应用程序不会过于依赖彼此。

## 测试

没有测试就没有高质量的软件。测试软件可以让我们在软件部署之前发现软件中的错误。测试和生产代码一样重要，你应该花相当多的时间在它们上面。

有关测试干净代码和编写干净测试代码的更多信息，请查看以下文章:

1.  [Python 中的测试](/blog/testing-python/)
2.  [Python 中的现代测试驱动开发](/blog/modern-tdd/)

## 结论

编写干净的代码很难。没有一个简单的方法可以让你写出好的、干净的代码。需要时间和经验去掌握。我们已经了解了一些编码标准和通用指南，它们可以帮助您编写更好的代码。我能给你的最好的建议之一是保持一致性，尽量写易于测试的简单代码。如果你发现你的代码很难测试，它可能很难使用。

如果您想了解更多，请查看[完整的 Python 开发指南](/guides/complete-python/)，在那里您将从实践中学习如何编写干净的代码。