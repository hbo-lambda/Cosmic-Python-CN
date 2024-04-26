# 2.存储库模式


是时候兑现我们的承诺，使用依赖倒置原则将我们的核心逻辑与基础设施问题分离。。

我们将介绍存储库模式，这是一种简化数据存储的抽象，它允许我们将模型层与数据层分离。我们将提供一个具体示例，说明这种简化抽象如何通过隐藏数据库的复杂性来使我们的系统更易于测试。

图1 展示了一些我们将要构建的内容：一个`Repository`位于我们的领域模型和数据库之间的对象。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0201.png){ align=center }
    <figcaption><font size=2>图1.存储库模式的前后对比</font></figcaption>
</figure>

!!! tip
    本章的代码位于 [Github](https://github.com/cosmicpython/code/tree/chapter_02_repository) 上的 Chapter_02_repository 分支中
    ```shell
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout chapter_02_repository
    # or to code along, checkout the previous chapter:
    git checkout chapter_01_domain_model
    ```


## <font color='red'>持久化我们的领域模型</font>

在 [1.领域模型](./d.Domain%20Modeling.md)中，我们建立了一个简单的领域模型，可以将订单分配给库存批次。我们很容易编写针对这段代码的测试，因为没有任何依赖或基础设施需要设置。如果我们需要运行数据库或API并创建测试数据，那么我们的测试将更难编写和维护。

不幸的是，我们总会在某个时候将我们完美的小模型交给用户，并与现实中的电子表格、Web浏览器、竞态条件进行竞争。在接下来的几章中，我们将探讨如何将我们理想化的领域模型与外部状态进行连接。

我们期望以敏捷方式工作，因此我们的优先任务是尽快实现最小化的可行产品。在我们的案例中，这将是一个Web API。实际项目中，您可能会直接进行一些端到端测试，然后插入Web框架，从外到内进行测试。

但我们知道，无论如何，我们都需要某种形式的持久存储，这是一本教科书，所以我们可以允许自己进行一点点自下而上的开发，并开始考虑存储和数据库。


## <font color='red'>一些伪代码：我们需要什么？</font>

当我们构建第一个 API 端点时，我们知道我们会有一些或多或少类似于以下内容的代码。

```py title='我们的第一个API端点是什么样子的'
@flask.route.gubbins
def allocate_endpoint():
    # extract order line from request
    line = OrderLine(request.params, ...)
    # load all batches from the DB
    batches = ...
    # call our domain service
    allocate(line, batches)
    # then save the allocation back to the database somehow
    return 201
```

!!! note
    我们使用Flask是因为它是轻量级的，但是你不需要成为Flask用户来理解本书。事实上，我们将向您展示如何使框架选型变得微不足道

我们需要从数据库中检索批次信息并实例化我们的领域模型对象，我们还需要一种方法将它们保存回数据库。

*什么？哦，"gubbins" 是一个英国词，意思是"东西"。你可以忽略那个词。这只是伪代码，好吗？*


## <font color='red'>将DIP应用于数据访问</font>

如[介绍](./b.Introduction.md)中提到的，分层架构是构建具有 UI、一些逻辑和数据库的系统的常用方法（参见分层架构）。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0202.png){ align=center }
    <figcaption><font size=2>图2.分层架构</font></figcaption>
</figure>

Django的模型-视图-模板结构与模型-视图-控制器（MVC）密切相关。无论如何，目标是保持各层分离（这是一件好事），并且使每个层仅依赖于其下方的层。

但是，我们希望我们的领域模型完全没有任何依赖关系。我们不希望基础设施问题渗入到我们的领域模型中，从而减慢我们的单元测试或我们进行更改的能力。

相反，如介绍中所讨论的那样，我们将把我们的模型看作是“内部的”，并且依赖项向内流向它；这就是人们有时称之为洋葱架构的东西（参见图3.洋葱架构）。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0203.png){ align=center }
    <figcaption><font size=2>图3.洋葱架构</font></figcaption>
</figure>

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p><strong style='font-size:1.6em;color:#186d7a'>这是端口和适配器吗？</strong></p>
<p>如果您一直在阅读有关架构模式的文章，可能会问自己这样的问题：</p>
<em>这是端口和适配器吗？还是六边形架构？和洋葱架构是一样的吗？那干净的架构呢？什么是端口，什么是适配器？为什么你们对同一个东西有这么多称呼？</em><p>尽管有些人喜欢挑剔其中的差异，但所有这些基本上都是同一事物的名称，它们的核心都是依赖倒置原则：高层模块（领域）不应该依赖低层模块（基础架构）。我们将在书的后面讨论一些关于“依赖于抽象”的细节，以及是否有 Python 中的接口等价物。</p></div>


## <font color='red'>回顾：我们的模型</font>

让我们回顾一下我们的领域模型：分配是将`OrderLine`链接到`Batch`的概念。我们将分配集合存储在`Batch`对象上。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0103.png){ align=center }
    <figcaption><font size=2>图4.我们的模型</font></figcaption>
</figure>

让我们看看如何将其转换为关系数据库。


### <font color='red'>"通常"的ORM方法：模型依赖于ORM</font>

如今，您的团队成员不太可能手动编写自己的SQL查询。相反，您几乎肯定会使用某种框架根据您的模型对象为您生成SQL。

这些框架被称为对象关系映射器（ORM），因为它们存在的目的是弥合对象和领域建模世界与数据库和关系代数世界之间的概念鸿沟。

ORM最重要的作用是持久性无知：我们的领域模型不需要知道任何有关如何加载或持久化数据的信息。这有助于让我们的领域摆脱对特定数据库技术的直接依赖。

但是，如果你按照典型的 SQLAlchemy 教程，最终会得到类似下面的东西：

```py title='SQLAlchemy“声明式”语法，模型依赖于ORM (orm.py）'
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship

Base = declarative_base()

class Order(Base):
    id = Column(Integer, primary_key=True)

class OrderLine(Base):
    id = Column(Integer, primary_key=True)
    sku = Column(String(250))
    qty = Integer(String(250))
    order_id = Column(Integer, ForeignKey('order.id'))
    order = relationship(Order)

class Allocation(Base):
    ...
```

您无需了解 SQLAlchemy 就能看到，我们原始的模型现在充满了对 ORM 的依赖，而且看起来也非常丑陋。我们真的可以说这个模型不知道数据库吗？当我们的模型属性直接与数据库列耦合时，它如何与存储问题分开？

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>Django的ORM基本相同，但限制更多</strong></p>
如果你更习惯于Django，上述的"声明式" SQLAlchemy 片段可以转换为以下代码：

<p align='right'><font size=2>Django ORM 示例</font></p>
<pre>
<code style='border: 1px solid'>
class Order(models.Model):
    pass

class OrderLine(models.Model):
    sku = models.CharField(max_length=255)
    qty = models.IntegerField()
    order = models.ForeignKey(Order)

class Allocation(models.Model):
    ...
</code>
</pre>
<p>要点是一样的——我们的模型类直接继承自 ORM 类，所以我们的模型依赖于 ORM。我们希望反过来。</p>
<p>Django 没有提供与 SQLAlchemy 的传统映射器等效的映射器，但请参见[附录_Django]，了解如何将依赖反转和存储库模式应用于Django的示例。</p>
</div>


### <font color='red'>反转依赖关系：ORM依赖于模型</font>

幸运的是，这不是使用SQLAlchemy的唯一方法。另一种方法是单独定义您的架构，并定义一个显式映射器来确定如何在架构和我们的领域模型之间进行转换，SQLAlchemy 称之为 经典映射

```py title='使用SQLAlchemy的 Table 对象进行显式ORM映射（orm.py）'
from sqlalchemy.orm import mapper, relationship

import model  #(1)


metadata = MetaData()

order_lines = Table(  #(2)
    "order_lines",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("sku", String(255)),
    Column("qty", Integer, nullable=False),
    Column("orderid", String(255)),
)

...

def start_mappers():
    lines_mapper = mapper(model.OrderLine, order_lines)  #(3)
```

1. ORM导入（或“依赖”或“了解”）领域模型，而不是相反。
2. 我们使用SQLAlchemy的抽象来定义我们的数据库表和列。
3. 当我们调用`mapper`函数时，SQLAlchemy会通过其魔法将我们的领域模型类绑定到我们定义的各种表上。

最终结果是，如果我们调用`start_mappers`，我们将能够轻松地从数据库加载和保存领域模型实例。但是，如果我们从不调用该函数，我们的领域模型类将完全不知道数据库的存在。

这为我们提供了 SQLAlchemy 的所有好处，包括用于 alembic迁移的能力，以及使用域类透明地查询的能力，正如我们将看到的。

当您第一次尝试构建 ORM 配置时，为其编写测试会很有用，如下例所示：

```py title='直接测试ORM（一次性测试）（test_orm.py）'
def test_orderline_mapper_can_load_lines(session):  #(1)
    session.execute(
        "INSERT INTO order_lines (orderid, sku, qty) VALUES "
        '("order1", "RED-CHAIR", 12),'
        '("order1", "RED-TABLE", 13),'
        '("order2", "BLUE-LIPSTICK", 14）'
    )
    expected = [
        model.OrderLine("order1", "RED-CHAIR", 12),
        model.OrderLine("order1", "RED-TABLE", 13),
        model.OrderLine("order2", "BLUE-LIPSTICK", 14),
    ]
    assert session.query(model.OrderLine).all() == expected


def test_orderline_mapper_can_save_lines(session):
    new_line = model.OrderLine("order1", "DECORATIVE-WIDGET", 12)
    session.add(new_line)
    session.commit()

    rows = list(session.execute('SELECT orderid, sku, qty FROM "order_lines"'))
    assert rows == [("order1", "DECORATIVE-WIDGET", 12)]

```

1. 如果您没有使用过pytest，那么这个测试中的session参数需要解释一下。就本书的目的而言，您不需要担心pytest或其fixture的细节，但简短的解释是，您可以将测试的常见依赖项定义为"fixtures"，pytest会通过查看其函数参数将它们注入到需要它们的测试中。在这种情况下，它是一个SQLAlchemy数据库会话。

您可能不会保留这些测试——很快您将看到，一旦您采取了反转 ORM 和领域模型的依赖关系的步骤，只需再迈出一小步即可实现另一个称为“存储库模式”的抽象，这将更容易编写测试，并将提供一个简单的界面，用于以后的测试。

但我们已经实现了颠覆传统依赖关系的目标：领域模型保持“纯粹”，不受基础架构影响。我们可以抛弃 SQLAlchemy，使用不同的 ORM 或完全不同的持久性系统，而领域模型根本不需要改变。

根据您在领域模型中正在做的事情，特别是如果您偏离了 OO 范式，您可能会发现很难让ORM产生您所需的确切行为，并且可能需要修改您的领域模型。正如架构决策中经常发生的那样，您需要考虑一个权衡。正如Python之禅所说："实用性胜过纯粹性！"

不过，此时，我们的API端点可能看起来像下面这样，我们可以让它正常工作：

```py title='在我们的API端点直接使用SQLAlchemy'
@flask.route.gubbins
def allocate_endpoint():
    session = start_session()

    # extract order line from request
    line = OrderLine(
        request.json['orderid'],
        request.json['sku'],
        request.json['qty'],
    )

    # load all batches from the DB
    batches = session.query(Batch).all()

    # call our domain service
    allocate(line, batches)

    # save the allocation back to the database
    session.commit()

    return 201
```


## <font color='red'>存储库模式简介</font>

存储库 模式是对持久性存储的抽象。它假装我们的所有数据都在内存中，从而隐藏了数据访问的无聊细节。

如果我们的笔记本有无限的内存，那么我们就不需要笨重的数据库了。相反，我们可以随时使用我们的对象。那会是什么样子呢？

```py title='你必须从某处获取数据'
import all_my_data

def create_a_batch():
    batch = Batch(...)
    all_my_data.batches.add(batch)

def modify_a_batch(batch_id, new_quantity):
    batch = all_my_data.batches.get(batch_id)
    batch.change_initial_quantity(new_quantity)
```

即使我们的对象在内存中，我们仍然需要将它们放在某个地方，以便我们能够再次找到它们。我们的内存数据让我们可以添加新对象，就像列表或集合一样。由于对象在内存中，我们永远不需要调用`.save()`方法；我们只需获取我们关心的对象并在内存中修改它。


### <font color='red'>抽象的存储库模式</font>

最简单的存储库只有两种方法：`add()` 将新项目放入存储库，`get()` 返回先前添加的项目。我们严格遵守使用这些方法来访问领域和服务层中的数据。这种自我强加的简单性使我们无需将领域模型与数据库耦合。

下面是我们存储库的抽象基类（ABC）的示例：

```py title='最简单的存储库 (repository.py）'
class AbstractRepository(abc.ABC):
    @abc.abstractmethod  #(1)
    def add(self, batch: model.Batch):
        raise NotImplementedError  #(2)

    @abc.abstractmethod
    def get(self, reference) -> model.Batch:
        raise NotImplementedError
```

1. Python提示：`@abc.abstractmethod`这是使 ABC 在 Python 中真正“起作用”的唯一因素之一。Python将拒绝让您实例化一个未实现其父类中所有抽象方法的类。
2. `raise NotImplementedError`很好用，但既不是必需的也不是充分的。实际上，如果您真的想要，您的抽象方法可以具有子类可以调用的真实行为。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>抽象基类、鸭子类型和协议</strong></p>
<p>在本书中，我们出于教学目的使用抽象基类：我们希望它们能帮助解释存储库抽象的接口是什么。</p>
<p>在现实生活中，我们有时会发现自己从生产代码中删除了 ABC，因为 Python 很容易忽略它们，最终导致它们无人维护，最糟糕的情况是产生误导。在实践中，我们通常只依赖 Python 的鸭子类型来实现抽象。</p>
<p>另一个要考虑的替代方案是 <a src='https://peps.python.org/pep-0544/'>PEP 544 协议</a>。这些协议为您提供了类型提示，而不会出现继承的可能性，这将特别受到“优先使用组合而不是继承”的粉丝的喜爱。</p></div>


### <font color='red'>权衡是什么？</font>

!!! quote
    你知道人们常说经济学家知道一切东西的价格，却不知道任何东西的价值吗？那么，程序员知道一切东西的好处，却不知道任何东西的利弊
    <p align='right'>— Rich Hickey</p>

每当我们在本书中引入一个架构模式时，我们总会问：“我们从中得到了什么？而我们又为此付出了什么代价？”

通常，我们至少会引入一个额外的抽象层，尽管我们可能希望它能够从整体上降低复杂性，但它确实会在局部增加复杂性，并且在移动部件的原始数量和持续维护方面会产生成本

不过，如果您已经走上了 DDD 和依赖反转路线，那么`Repository`模式可能是本书中最简单的选择之一。就我们的代码而言，我们实际上只是将SQLAlchemy抽象（`session.query(Batch)`）替换为我们设计的session.query(Batch)另一个抽象（`batches_repo.get`）。

每次添加要检索的新域对象时，我们都必须在我们的存储库类中编写几行代码，但作为回报，我们会获得一个对我们控制的存储层的简单抽象。存储库模式可以轻松对我们存储事物的方式进行根本性的改变（参见 [附录C csvs](./t.Appendix%20B.md)），而且正如我们将看到的，它很容易被单元测试欺骗。

此外，存储库模式在 DDD 世界中非常常见，如果您与从 Java 和 C# 领域转到 Python 的程序员合作，他们很可能会认出它。存储库模式说明了该模式。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0205.png){ align=center }
    <figcaption><font size=2>图5.存储库模式</font></figcaption>
</figure>


像往常一样，我们从测试开始。这个测试可能被归为集成测试，因为我们正在检查我们的代码（存储库）是否与数据库正确集成；因此，测试倾向于将原始 SQL 与我们自己的代码上的调用和断言混合在一起。

!!! note
    与之前的 ORM 测试不同，这些测试适合长期保留为您的代码库的一部分，特别是当您的领域模型的任何部分意味着对象关系图并不简单时。    

```py title='用于保存对象的存储库测试（test_repository.py）'
def test_repository_can_save_a_batch(session):
    batch = model.Batch("batch1", "RUSTY-SOAPDISH", 100, eta=None)

    repo = repository.SqlAlchemyRepository(session)
    repo.add(batch)  #(1)
    session.commit()  #(2)

    rows = session.execute(  #(3)
        'SELECT reference, sku, _purchased_quantity, eta FROM "batches"'
    )
    assert list(rows) == [("batch1", "RUSTY-SOAPDISH", 100, None)]
```

1. `repo.add()`是这里测试的方法。
2. 我们暴露存储库的`.commit()`，并将其作为调用者的责任。这样做有利有弊；当我们到达 [6.工作单元模式](./i.Unit%20of%20Work%20Pattern.md) 时，我们的一些理由会变得更加清晰。
3. 我们使用原始 SQL 来验证正确的数据是否已保存。

接下来的测试涉及检索批次和分配，所以它更加复杂：

```py title='用于检索复杂对象的存储库测试（test_repository.py）'
def insert_order_line(session):
    session.execute(  #(1)
        "INSERT INTO order_lines (orderid, sku, qty)"
        ' VALUES ("order1", "GENERIC-SOFA", 12）'
    )
    [[orderline_id]] = session.execute(
        "SELECT id FROM order_lines WHERE orderid=:orderid AND sku=:sku",
        dict(orderid="order1", sku="GENERIC-SOFA"),
    )
    return orderline_id


def insert_batch(session, batch_id):  #(2)
    ...

def test_repository_can_retrieve_a_batch_with_allocations(session):
    orderline_id = insert_order_line(session)
    batch1_id = insert_batch(session, "batch1")
    insert_batch(session, "batch2")
    insert_allocation(session, orderline_id, batch1_id)  #(2)

    repo = repository.SqlAlchemyRepository(session)
    retrieved = repo.get("batch1")

    expected = model.Batch("batch1", "GENERIC-SOFA", 100, eta=None)
    assert retrieved == expected  # Batch.__eq__ only compares reference  #(3)
    assert retrieved.sku == expected.sku  #(4)
    assert retrieved._purchased_quantity == expected._purchased_quantity
    assert retrieved._allocations == {  #(4)
        model.OrderLine("order1", "GENERIC-SOFA", 12),
    }
```

1. 这个测试是针对读取方面的，因此原始 SQL 正在准备数据以供`repo.get()`读取。
2. 我们将为您省略` insert_batch` 和`insert_allocation` 的细节；重点是创建几个批次，并且对于我们感兴趣的批次，要有一个现有的订单行分配给它。
3. 这就是我们在这里验证的内容。第一个`assert ==`检查类型是否匹配，并且引用是否相同（因为，你记得的，Batch 是一个实体，我们为它定义了自定义的` __eq__`）。
4. 因此，我们还明确检查了它的主要属性，包括`._allocations`，它是一个包含`OrderLine`值对象的 Python 集合。

是否要为每个模型精心编写测试是一个判断问题。一旦您已经为一个类测试了创建/修改/保存，您可能会愿意继续并对其他类进行最少的往返测试，甚至什么都不做，如果它们都遵循相似的模式。在我们的情况下，设置`._allocations`集合的 ORM 配置有点复杂，因此它值得进行特定的测试。

最终你会得到如下结果：

```py title='典型的存储库（repository.py）'
class SqlAlchemyRepository(AbstractRepository):
    def __init__(self, session):
        self.session = session

    def add(self, batch):
        self.session.add(batch)

    def get(self, reference):
        return self.session.query(model.Batch).filter_by(reference=reference).one()

    def list(self):
        return self.session.query(model.Batch).all()

```

现在我们的 Flask 端点可能看起来像这样：

```py title='直接在我们的 API 端点中使用我们的存储：'
@flask.route.gubbins
def allocate_endpoint():
    batches = SqlAlchemyRepository.list()
    lines = [
        OrderLine(l['orderid'], l['sku'], l['qty'])
         for l in request.params...
    ]
    allocate(lines, batches)
    session.commit()
    return 201
```

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>读者练习</strong></p>
<p>前几天，我们在 DDD 会议上遇到了一位朋友，他说：“我已经 10 年没有使用过 ORM 了。”存储库模式和 ORM 都充当原始 SQL 前面的抽象，因此没有必要前后使用一个。为什么不尝试在不使用 ORM 的情况下实现我们的存储库呢？您可以 <a src='https://github.com/cosmicpython/code/tree/chapter_02_repository_exercise'>Github</a>上找到代码。</p>
<p>我们已经离开了存储库测试，但弄清楚要编写什么 SQL 取决于您。也许这会比您想象的更难；也许会更容易。但好消息是，您的应用程序的其余部分并不关心。</p>
</div>


## <font color='red'>构建一个用于测试的虚假存储库现在变得非常简单</font>

这是存储库模式的最大优点之一：

```py title='使用集合的简单伪存储库（repository.py）'
class FakeRepository(AbstractRepository):

    def __init__(self, batches):
        self._batches = set(batches)

    def add(self, batch):
        self._batches.add(batch)

    def get(self, reference):
        return next(b for b in self._batches if b.reference == reference)

    def list(self):
        return list(self._batches)
```

由于它是一个简单的集合包装器，所有方法都是一行代码。

测试中使用假存储库非常简单，我们有一个简单的抽象，易于使用和理解：

```py title='虚假存储库的使用示例'
fake_repo = FakeRepository([batch1, batch2, batch3])
```

你将在下一章节中看到这个假存储库实际应用。

!!! tip
    为您的抽象构建假象是获得设计反馈的绝佳方式：如果很难构建，那么抽象可能太复杂了。


## <font color='red'>Python中，什么是端口，什么是适配器？ </font>

我们不想在这里过多地讨论术语，因为我们想要重点关注的是依赖倒置，而你使用的技术细节并不是太重要。此外，我们意识到不同的人可能使用稍微不同的定义。

端口和适配器起源于OO，我们坚持的定义是，端口是我们的应用程序与我们希望抽象掉的任何东西之间的接口，适配器是该接口或抽象背后的实现。

现在Python本身没有接口，所以虽然通常很容易识别出适配器，但定义端口可能更难。如果您使用的是抽象基类，则它就是端口。如果不是，端口就是您的适配器遵循的、核心应用程序所期望的鸭子类型——在用的函数和方法名称、参数名称和类型。

具体地，在本章中，`AbstractRepository`是端口，`SqlAlchemyRepository`和`FakeRepository`是适配器。


## <font color='red'>总结</font>

牢记 Rich Hickey 的名言，我们在每一章中总结了我们介绍的每种架构模式的成本和收益。我们想明确一点，我们并不是说每个应用程序都需要以这种方式构建；只有有时应用程序和领域的复杂性才值得投入时间和精力来添加这些额外的间接层。

考虑到这一点，表1展示了存储库模式和我们的持久性无知模型的一些优缺点。

<font color='#186d7a'>表1.存储库模式和持久性无知：权衡</font>

| 优点                                                         | 缺点                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 我们的持久性存储和领域模型之间有一个简单的接口。             | ORM已经帮助您带来一些解耦。更改外键可能很难，但如果你需要，切换到MySQL和Postgres应该相当容易 |
| 由于我们已经将模型与基础设施问题完全分离，因此很容易制作用于单元测试的存储库的虚假版本，或者交换不同的存储解决方案。 | 手动维护ORM映射需要额外的工作和额外的代码。                  |
| 在考虑持久性之前编写领域模型有助于我们专注于手头的业务问题。如果我们想彻底改变我们的方法，我们可以在我们的模型中做到这一点，而不必担心外键或迁移，直到以后再考虑。 | 任何额外的间接层都会增加维护成本，并且对于以前从未见过 Repository 模式的 Python 程序员来说会增加“WTF 因素”。 |
| 我们的数据库模式非常简单，因为我们可以完全控制如何将对象映射到表。 |                                                              |

图6展示了基本论点：是的，对于简单情况，解耦的领域模型比简单的 ActiveRecord/ORM 模式更难。


!!! tip
    如果您的应用程序只是关于数据库简单的CRUD（创建-读取-更新-删除）封装，那么您就不需要一个领域模型或一个存储库。

但是，领域越复杂，从摆脱基础设施问题的角度看，投资越多，在更改的难易程度方面就会越有回报。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0206.png){ align=center }
    <figcaption><font size=2>图6.领域模型权衡</font></figcaption>
</figure>


我们的示例代码并不复杂到足以展示图表右侧的内容，但是其中确实有一些提示。例如，想象一下，如果有一天我们决定希望将分配更改为存在于订单行（OrderLine）上而不是批次（Batch）对象上：如果我们正在使用Django，那么我们必须在运行任何测试之前定义并考虑数据库迁移。然而，由于我们的模型只是普通的Python对象，我们可以将一个 set() 更改为一个新属性，而无需考虑数据库直到以后。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>存储库模式总结</strong></p>
<p><strong>将依赖倒置应用于您的ORM</strong></p>
<p>我们的领域模型应该摆脱基础设施问题的影响，因此您的ORM应该导入您的模型，而不是相反。</p>
<p><strong>存储库模式是围绕永久存储的简单抽象</strong></p>
<p>该存储库让您感觉像是内存中对象的集合。它让您可以轻松创建一个FakeRepository用于测试的存储库，并交换基础架构的基本细节，而不会破坏您的核心应用程序。请参阅 [附录C csvs](./u.Appendix%20C.md)了解示例。</p>
</div>

您可能想知道，我们如何实例化这些存储库（无论是假的还是真的）？我们的Flask应用程序实际上会是什么样子？您将会在下一篇激动人心的[服务层模式](./g.Flask%20API%20and%20Service%20Layer.md)中找到答案。

但首先，让我们稍作偏离一下话题。