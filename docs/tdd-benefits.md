# 用测试驱动开发自信地改进代码

> 原文：<https://testdriven.io/blog/tdd-benefits/>

像许多开发人员一样，当我第一次接触到[测试驱动开发](/test-driven-development/) (TDD)时，我一点也不理解它。我不知道(也没有耐心)如何先开始写测试。因此，在添加测试之前，我没有花太多精力，而是按照正常的流程编写代码。这种情况持续了好几年。

早在 2017 年，我就共同创立了 [typless](https://typless.com) ，目前我在那里领导工程工作。我们必须在开始时快速行动，因此我们积累了相当多的技术债务。当时，这个平台本身是由一个大型的整体 Django 应用程序支持的。调试非常困难。代码很难读懂，更难修改。我们会消灭一只虫子，然后另外三只虫子会取代它。说到这里，是时候给 TDD 一次机会了。我所读到的一切都表明它会有所帮助——事实也的确如此。我终于看到了 TDD 的一个核心副作用:它使得代码修改变得更加容易。

## 软件是有生命的东西

对任何一个软件来说，最重要的质量因素之一就是它有多容易改变。

> “好的设计意味着当我做出改变时，就好像整个程序都是在预料之中的。我可以用几个选择函数调用完美地解决一个任务，不会在代码平静的表面留下丝毫波纹。”[来源](https://twitter.com/grady_booch/status/1252427714851569672)

软件随着业务需求的变化而变化。不管是什么推动了这种变化，昨天行之有效的解决方案今天可能行不通。

更改干净的、模块化的代码要容易得多，由测试覆盖，这正是 TDD 倾向于产生的代码类型。

让我们看一个例子。

## 要求

假设您有一个客户，希望您开发一个基本的电话簿，用于添加和显示(按字母顺序)电话号码。

您是否应该创建一个数字列表以及一些用于追加、排序和打印的辅助函数？还是应该创建一个类？当然，在这一点上，这可能并不重要。您可以开始编写代码来满足当前的需求。感觉这是最自然的事，对吧？但是，如果这些需求发生了变化，您必须包括搜索或删除，该怎么办呢？如果你不决定一个聪明的策略，代码很快就会变得混乱。

所以退一步，先写一些测试。

## 首先编写测试

首先，创建(并激活)一个虚拟环境并安装 [pytest](https://docs.pytest.org/) :

```
`(venv)$ pip install pytest` 
```

创建一个名为 *test_phone_book.py* 的新文件来保存您的测试。

用两个方法开始一个类似乎是合理的，`add`和`all`，对吗？

*   给定一个具有`records`属性的`PhoneBook`类
*   当调用`all`方法时
*   那么所有的数字都应该以升序返回

测试应该是这样的:

```
`class TestPhoneBook:

    def test_all(self):

        phone_book = PhoneBook(
            records=[
                ('John Doe', '03 234 567 890'),
                ('Marry Doe', '01 234 567 890'),
                ('Donald Doe', '02 234 567 890'),
            ]
        )

        previous = ''

        for record in phone_book.all():
            assert record[0] > previous
            previous = record[0]` 
```

这里，我们检查前一个元素在字母顺序上总是小于当前元素。

运行它:

测试当然会失败。

为了实现，添加一个名为 *phone_book.py* 的新文件:

```
`class PhoneBook:

    def __init__(self, records=None):
        self.records = records or []

    def all(self):
        return sorted(self.records)` 
```

将其导入到测试文件中:

```
`from phone_book import PhoneBook

class TestPhoneBook:

    def test_all(self):

        phone_book = PhoneBook(
            records=[
                ('John Doe', '03 234 567 890'),
                ('Marry Doe', '01 234 567 890'),
                ('Donald Doe', '02 234 567 890'),
            ]
        )

        previous = ''

        for record in phone_book.all():
            assert record[0] > previous
            previous = record[0]` 
```

再次运行:

测试现在通过了。你已经满足了第一个要求。

现在为一个`add`方法编写一个测试来检查一个新的数字是否在`records`中。

*   用一个`add`方法给定一个`PhoneBook`
*   当添加一个数字并调用`all`方法时
*   那么新号码是返回号码的一部分

```
`from phone_book import PhoneBook

class TestPhoneBook:

    def test_all(self):

        phone_book = PhoneBook(
            records=[
                ('John Doe', '03 234 567 890'),
                ('Marry Doe', '01 234 567 890'),
                ('Donald Doe', '02 234 567 890'),
            ]
        )

        previous = ''

        for record in phone_book.all():
            assert record[0] > previous
            previous = record[0]

    def test_add(self):

        record = ('John Doe', '01 234 567 890')
        phone_book = PhoneBook(
            records=[
                ('Marry Doe', '01 234 567 890'),
                ('Donald Doe', '02 234 567 890'),
            ]
        )
        phone_book.add(record)

        assert record in phone_book.all()` 
```

测试应该会失败，因为`add`方法还没有实现。

```
`class PhoneBook:

    def __init__(self, records=None):
        self.records = records or []

    def all(self):
        return sorted(self.records)

    def add(self, record):
        self.records.append(record)` 
```

`PhoneBook`类现在满足了上述所有要求。数字可以相加，全部可以按字母顺序排序返回。顾客很高兴。打包并交付代码。

## 新要求

让我们回顾一下第一个实现。

尽管我们使用测试来更好地定义应该做什么，但是没有测试我们也可以轻松地编写代码。事实上，这些测试似乎减缓了这一过程。

几个星期过去了，你还没有客户的消息。他们一定很喜欢添加和查看电话号码。干得好。给自己一个鼓励，给客户发一个关于未付发票的温馨提示。在你点击发送后不到 30 秒，你收到了一封令人沮丧的电子邮件回复，指出检索数字非常慢。

这是怎么回事？嗯，每次调用`all`方法时，你都在对记录进行排序，这将随着时间的推移而变慢。因此，让我们更改代码，在列表初始化和添加新数字时进行排序。

由于我们关注的是*测试接口而不是底层实现*，我们可以在不破坏测试的情况下修改代码。

```
`class PhoneBook:

    def __init__(self, records=None):
        self.records = sorted(records or [])

    def add(self, record):
        self.records.append(record)
        self.records = sorted(self.records)

    def all(self):
        return self.records` 
```

测试应该还能通过。

这很好，但我们实际上可以加快速度，因为数字已经排序了。

```
`class PhoneBook:

    def __init__(self, records=None):
        self.records = sorted(records or [], key=lambda rec: rec[0])

    def add(self, record):

        index = len(self.records)
        for i in range(len(self.records)):
            if record[0] < self.records[i][0]:
                index = i
                break

        self.records.insert(index, record)

    def all(self):
        return self.records` 
```

这里，我们按顺序插入新的数字，并取消排序。

尽管我们已经改变了实现来满足新的需求，但是我们仍然满足最初的需求。我们怎么知道？进行测试。

## 我们能做得更好吗？

我们已经满足了所有的要求。太好了。我们的客户支付发票。一切都好。时光流逝。你忘了这个项目。然后，你会突然在收件箱里看到他们发来的一封电子邮件，抱怨当添加新号码时，应用程序现在很慢。

您打开文本编辑器并开始调查。忘记了项目，您从测试开始，然后钻研代码。查看`add`方法，您会发现您必须在插入之前找到插入数字的确切位置，以保持顺序。这两种方法——插入和搜索插入索引——的时间复杂度都是 O(n)。

那么，如何提高那里的性能呢？

求助谷歌，栈溢出。运用你的信息检索技能。一个小时左右，你发现插入一棵二叉树的时间复杂度是 O(log n)。那更好。除此之外，元素可以通过有序遍历以排序的顺序返回。因此，请继续将您的实现改为使用二叉树而不是列表。

### 二叉树

> 不熟悉二叉树？查看 Python 中的[二叉树:简介和遍历算法](https://youtube.com/watch?v=6oL-0TdVy28)视频以及优秀的[二叉树](https://pypi.org/project/binarytree/)库，

首先，定义一个节点:

```
`class Node:

    def __init__(self, data):

        self.left = None
        self.right = None
        self.data = data` 
```

其次，添加一个插入方法:

```
`class Node:

    def __init__(self, data):

        self.left = None
        self.right = None
        self.data = data

    def insert(self, data):
        # Compare the new value with the parent node
        if self.data:
            if data[0] < self.data[0]:
                if self.left is None:
                    self.left = Node(data)
                else:
                    self.left.insert(data)
            elif data[0] > self.data[0]:
                if self.right is None:
                    self.right = Node(data)
                else:
                    self.right.insert(data)
        else:
            self.data = data` 
```

这里，我们检查当前节点是否有数据集。

如果不是，则设置数据。

如果设置了数据，它会检查第一个元素是大于还是小于我们需要插入的数据。在此基础上，它添加左或右节点。

最后，添加有序遍历方法:

```
`class Node:

    def __init__(self, data):

        self.left = None
        self.right = None
        self.data = data

    def insert(self, data):
        # Compare the new value with the parent node
        if self.data:
            if data[0] < self.data[0]:
                if self.left is None:
                    self.left = Node(data)
                else:
                    self.left.insert(data)
            elif data[0] > self.data[0]:
                if self.right is None:
                    self.right = Node(data)
                else:
                    self.right.insert(data)
        else:
            self.data = data

    def inorder_traversal(self, root):
        res = []
        if root:
            res = self.inorder_traversal(root.left)
            res.append(root.data)
            res = res + self.inorder_traversal(root.right)
        return res` 
```

有了它，我们可以将它实现到我们的`PhoneBook`:

```
`class Node:

    def __init__(self, data):

        self.left = None
        self.right = None
        self.data = data

    def insert(self, data):
        # Compare the new value with the parent node
        if self.data:
            if data[0] < self.data[0]:
                if self.left is None:
                    self.left = Node(data)
                else:
                    self.left.insert(data)
            elif data[0] > self.data[0]:
                if self.right is None:
                    self.right = Node(data)
                else:
                    self.right.insert(data)
        else:
            self.data = data

    def inorder_traversal(self, root):
        res = []
        if root:
            res = self.inorder_traversal(root.left)
            res.append(root.data)
            res = res + self.inorder_traversal(root.right)
        return res

class PhoneBook:

    def __init__(self, records=None):
        records = records or []

        if len(records) == 1:
            self.records = Node(records[0])
        elif len(records) > 1:
            self.records = Node(records[0])
            for elm in records[1:]:
                self.records.insert(elm)
        else:
            self.records = Node(None)

    def add(self, record):
        self.records.insert(record)

    def all(self):
        return self.records.inorder_traversal(self.records)` 
```

进行测试。他们应该通过。

## 结论

首先编写测试有助于很好地定义问题，这有助于编写更好的解决方案。

> 您可以使用测试来帮助澄清问题以及令人困惑的特性范围。

测试然后检查您的解决方案是否解决了问题。

大多数客户不会在乎*你如何*解决问题，只要它有效；因此，我们将测试的重点放在了接口上，而不是实现上。当我们对代码进行修改时，我们不需要修改我们的测试，因为代码解决的问题没有改变。

> 随着实现复杂性的增加，可能也有必要在那个级别添加单元测试。我建议将您的时间和注意力集中在实现级别的集成测试上，并且只有当您发现您的代码在某个特定区域反复中断时才添加单元测试。

测试给了我们一定的自由，因为我们可以改变实现，而不必担心破坏接口。毕竟，它如何工作并不重要，重要的是它能工作。

TDD 可以提供自信地更好地重构代码所需的信心。更快，更干净，更好的结构-这不重要。

编码快乐！