# 6.工作单元模式


在本章中，我们将介绍将存储库和服务层模式结合在一起的最后一块拼图：工作单元模式。

如果存储库模式是我们对持久存储概念的抽象，那么工作单元 (UoW) 模式就是我们对原子操作概念的抽象。它将使我们能够最终完全将服务层与数据层分离。

图1表明，目前，大量的通信发生在我们基础设施的各个层之间：API直接与数据库层对话以开启`session`，与存储库层对话以实例话SQLAlchemyRepository，并与服务层对话以要求其进行分配。

!!! tip
    本章的代码位于[GitHub](https://github.com/cosmicpython/code/tree/chapter_06_uow)的chapter_06_uow分支中：
    ```
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout chapter_06_uow
    # or to code along, checkout Chapter 4:
    git checkout chapter_04_service_layer
    ```

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0601.png){ align=center }
    <figcaption><font size=2>图1.没有UoW：API直接与三层对话</font></figcaption>
</figure>

图2展示我们的目标状态。Flask API现在只做两件事：初始化工作单元，并调用服务。服务与UoW协作（我们喜欢将UoW视为服务层的一部分），但服务函数本身和Flask现在都不需要直接与数据库对话。

我们将使用Python语法中的一个可爱的部分——上下文管理器来完成所有操作。

<figure markdown='span'>
    ![](https://www.cosmicpython.com/book/images/apwp_0602.png){ align=center }
    <figcaption><font size=2>图2.使用UoW：UoW现在管理数据库状态</font></figcaption>
</figure>


## <font color='red'>工作单元与存储库协作</font>

让我们看看工作单元的实际作用。完成后，服务层将如下所示

```py title='工作单元运行预览 (src/allocation/service_layer/services.py）'
def allocate(
    orderid: str, sku: str, qty: int,
    uow: unit_of_work.AbstractUnitOfWork,
) -> str:
    line = OrderLine(orderid, sku, qty)
    with uow:  #(1)
        batches = uow.batches.list()  #(2)
        ...
        batchref = model.allocate(line, batches)
        uow.commit()  #(3)
```

1. 我们将启动 UoW 作为上下文管理器。
2. `uow.batches`是批次repo，因此UoW为我们提供对永久存储的访问权限。
3. 完成后，我们使用 UoW 提交或回滚我们的工作。

UoW 充当我们持久存储的单一入口点，并跟踪已加载的对象及其最新状态。[ 1 ]

这给了我们三个有用的东西：

- 可以使用的数据库的稳定快照，以便我们使用的对象不会在操作过程中发生变化
- 一种一次性保存所有更改的方法，这样如果出现问题，我们不会陷入不一致的状态
- 针对持久性问题的简单 API 以及获取存储库的便捷位置


## <font color='red'>使用集成测试来测试UoW</font>

以下是我们针对 UOW 的集成测试：

```py title='UoW 的基本“往返”测试 (tests/integration/test_uow.py）'
def test_uow_can_retrieve_a_batch_and_allocate_to_it(session_factory):
    session = session_factory()
    insert_batch(session, "batch1", "HIPSTER-WORKBENCH", 100, None)
    session.commit()

    uow = unit_of_work.SqlAlchemyUnitOfWork(session_factory)  #(1)
    with uow:
        batch = uow.batches.get(reference="batch1")  #(2)
        line = model.OrderLine("o1", "HIPSTER-WORKBENCH", 10)
        batch.allocate(line)
        uow.commit()  #(3)

    batchref = get_allocated_batch_ref(session, "o1", "HIPSTER-WORKBENCH")
    assert batchref == "batch1"
```

1. 我们使用自定义会话工厂初始化 UoW，在`with`代码块中取回一个`uow`对象以供使用。
2. UoW 让我们可以通过`uow.batches`访问批次存储库。
3. 当完成后，我们会调用`commit()`

对于那些好奇的人来说，`insert_batch`和`get_allocated_batch_ref`辅助函数看起来像这样：

```py title='执行 SQL 操作的助手 (tests/integration/test_uow.py）'
def insert_batch(session, ref, sku, qty, eta):
    session.execute(
        "INSERT INTO batches (reference, sku, _purchased_quantity, eta)"
        " VALUES (:ref, :sku, :qty, :eta)",
        dict(ref=ref, sku=sku, qty=qty, eta=eta),
    )


def get_allocated_batch_ref(session, orderid, sku):
    [[orderlineid]] = session.execute(  #(1)
        "SELECT id FROM order_lines WHERE orderid=:orderid AND sku=:sku",
        dict(orderid=orderid, sku=sku),
    )
    [[batchref]] = session.execute(  #(1)
        "SELECT b.reference FROM allocations JOIN batches AS b ON batch_id = b.id"
        " WHERE orderline_id=:orderlineid",
        dict(orderlineid=orderlineid),
    )
    return batchref
```

1. 语法`=`有点太聪明了，抱歉。发生的事情是`session.execute`返回一个行列表，其中每行都是列值的元组；在我们的特定情况下，它是一个一行的列表，它是一个包含一个列值的元组。左侧的双方括号正在执行（双重）赋值解包，以从这两个嵌套序列中获取单个值。一旦你用过几次，它就变得可读了！


## <font color='red'>工作单元及其上下文管理器</font>

在我们的测试中，我们隐式地定义了 UoW 需要执行的操作的接口。让我们使用抽象基类来明确这一点：

```py title='抽象 UoW 上下文管理器 (src/allocation/service_layer/unit_of_work.py）'
class AbstractUnitOfWork(abc.ABC):
    batches: repository.AbstractRepository  #(1)

    def __exit__(self, *args):  #(2)
        self.rollback()  #(4)

    @abc.abstractmethod
    def commit(self):  #(3)
        raise NotImplementedError

    @abc.abstractmethod
    def rollback(self):  #(4)
        raise NotImplementedError
```

1. UoW 提供了一个名为`.batches`的属性，它将使我们能够访问批次存储库。
2. 如果您从未见过上下文管理器，那么`__enter__`和`__exit__`分别是我们进入`with`块和退出`with`块时执行的两个魔术方法。它们是我们的启动和拆卸阶段。
3. 当我们准备好时，我们将调用此方法来明确提交我们的工作。
4. 如果我们不提交，或者我们通过引发错误退出上下文管理器，我们将执行`rollback`。（如果`commit()`已调用，则`rollback`无效。请继续阅读更多多关于此内容的讨论。）


### <font color='red'>真正的工作单元使用 SQLAlchemy 会话</font>

我们的具体实现主要添加的是数据库会话：

```py title='真正的 SQLAlchemy UoW (src/allocation/service_layer/unit_of_work.py）'
DEFAULT_SESSION_FACTORY = sessionmaker(  #(1)
    bind=create_engine(
        config.get_postgres_uri(),
    )
)


class SqlAlchemyUnitOfWork(AbstractUnitOfWork):
    def __init__(self, session_factory=DEFAULT_SESSION_FACTORY):
        self.session_factory = session_factory  #(1)

    def __enter__(self):
        self.session = self.session_factory()  # type: Session  #(2)
        self.batches = repository.SqlAlchemyRepository(self.session)  #(2)
        return super().__enter__()

    def __exit__(self, *args):
        super().__exit__(*args)
        self.session.close()  #(3)

    def commit(self):  #(4)
        self.session.commit()

    def rollback(self):  #(4)
        self.session.rollback()
```

1. 该模块定义了一个将连接到 Postgres 的默认会话工厂，但我们允许在集成测试中覆盖它，以便我们可以改用 SQLite。
2. 该`__enter__`方法负责启动数据库会话并实例化可以使用该会话的真实存储库。
3. 我们在退出时关闭会话。
4. 最后，我们提供使用数据库会话的`commit()`和`rollback()`方法。


### <font color='red'>用于测试的伪工作单元</font>

以下是我们在服务层测试中使用伪 UoW 的方法：

```py title='伪 UoW (tests/unit/test_services.py）'
class FakeUnitOfWork(unit_of_work.AbstractUnitOfWork):
    def __init__(self):
        self.batches = FakeRepository([])  #(1)
        self.committed = False  #(2)

    def commit(self):
        self.committed = True  #(2)

    def rollback(self):
        pass


def test_add_batch():
    uow = FakeUnitOfWork()  #(3)
    services.add_batch("b1", "CRUNCHY-ARMCHAIR", 100, None, uow)  #(3)
    assert uow.batches.get("b1") is not None
    assert uow.committed


def test_allocate_returns_allocation():
    uow = FakeUnitOfWork()  #(3)
    services.add_batch("batch1", "COMPLICATED-LAMP", 100, None, uow)  #(3)
    result = services.allocate("o1", "COMPLICATED-LAMP", 10, uow)  #(3)
    assert result == "batch1"
...
```

1. `FakeUnitOfWork`和`FakeRepository`紧密耦合，就像真实的`UnitofWork`和`Repository`类一样。这很好，因为我们认识到这些对象是合作者。
2. 注意`FakeSession`伪造的`commit()`函数的相似性（我们现在可以将其删除）。但这是一个实质性的改进，因为我们现在伪造的是自己编写的代码，而不是第三方代码。有人说，“不要模拟你不拥有的东西”。
3. 在我们的测试中，我们可以实例化 UoW 并将其传递给我们的服务层，而不是传递存储库和会话。这样就不那么麻烦了。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>不要嘲笑你不拥有的东西</strong></p>
<p>为什么我们觉得模拟 UoW 比模拟会话更舒服？我们的两个伪造都实现了相同的目的：它们为我们提供了一种交换持久层的方法，这样我们就可以在内存中运行测试，而不需要与真实的数据库通信。不同之处在于最终的设计。</p>
<p>如果我们只关心编写快速运行的测试，我们可以创建模拟来替代SQLAlchemy 并在整个代码库中使用它们。问题是，Session是一个复杂的对象，暴露了许多与持久性相关的功能。用Session对数据库进行任意查询是很容易得，但这很快会导致数据访问代码散布在整个代码库中。为了避免这种情况，我们希望限制对持久层的访问，以便每个组件都只拥有它所需要的内容，仅此而已。</p>
<p>通过耦合到Session接口，您正在选择的是耦合到 SQLAlchemy 的所有复杂性上。相反，我们希望选择一个更简单的抽象，并使用它来明确划分职责。我们的 UoW 比会话简单得多，我们对服务层能够启动和停止工作单元感到满意。</p>
<p>“不要模拟你不拥有的东西”是一条经验法则，它迫使我们在混乱的子系统上构建这些简单的抽象。这与模拟 SQLAlchemy 会话具有相同的性能优势，但鼓励我们仔细思考我们的设计。</p>
</div>


## <font color='red'>在服务层中使用 UoW</font>

以下是我们的新服务层的样子：

```py title='使用 UoW 的服务层 (src/allocation/service_layer/services.py）'
def add_batch(
    ref: str, sku: str, qty: int, eta: Optional[date],
    uow: unit_of_work.AbstractUnitOfWork,  #(1)
):
    with uow:
        uow.batches.add(model.Batch(ref, sku, qty, eta))
        uow.commit()


def allocate(
    orderid: str, sku: str, qty: int,
    uow: unit_of_work.AbstractUnitOfWork,  #(1)
) -> str:
    line = OrderLine(orderid, sku, qty)
    with uow:
        batches = uow.batches.list()
        if not is_valid_sku(line.sku, batches):
            raise InvalidSku(f"Invalid sku {line.sku}")
        batchref = model.allocate(line, batches)
        uow.commit()
    return batchref
```

1. 我们的服务层现在只有一个依赖项，再次依赖于抽象的UoW。


## <font color='red'>提交/回滚行为的显式测试</font>

为了确保提交/回滚行为有效，我们编写了几个测试：
```py title='回滚行为的集成测试（tests/integration/test_uow.py）'
def test_rolls_back_uncommitted_work_by_default(session_factory):
    uow = unit_of_work.SqlAlchemyUnitOfWork(session_factory)
    with uow:
        insert_batch(uow.session, "batch1", "MEDIUM-PLINTH", 100, None)

    new_session = session_factory()
    rows = list(new_session.execute('SELECT * FROM "batches"'))
    assert rows == []


def test_rolls_back_on_error(session_factory):
    class MyException(Exception):
        pass

    uow = unit_of_work.SqlAlchemyUnitOfWork(session_factory)
    with pytest.raises(MyException):
        with uow:
            insert_batch(uow.session, "batch1", "LARGE-FORK", 100, None)
            raise MyException()

    new_session = session_factory()
    rows = list(new_session.execute('SELECT * FROM "batches"'))
    assert rows == []
```

!!! tip
    我们在这里没有展示这一点，但值得针对“真实”数据库（即同一引擎）测试一些更“模糊”的数据库行为（如事务）。目前，我们使用 SQLite 而不是 Postgres，但在[7.聚合](./j.aggregates%20and%20Consistency%20Boundaries.md)中，我们将切换一些测试以使用真实数据库。我们的 UoW 类让这一切变得简单，非常方便！


## <font color='red'>显式提交与隐式提交</font>

现在我们简单讨论一下实现 UoW 模式的不同方法。

我们可以想象一个略有不同的 UoW 版本，它默认提交，只有发现异常时才回滚：

```py title='具有隐式提交的 UoW...（src/allocation/unit_of_work.py）'
class AbstractUnitOfWork(abc.ABC):

    def __enter__(self):
        return self

    def __exit__(self, exn_type, exn_value, traceback):
        if exn_type is None:
            self.commit()  #(1)
        else:
            self.rollback()  #(2)
```

1. 在正常情况下用一个隐含的提交？
2. 只在有异常的情况下回滚？

它可以让我们保存一行代码并从我们的客户端代码中删除显式提交：

```py title='...可以为我们节省一行代码（src/allocation/service_layer/services.py）'
def add_batch(ref: str, sku: str, qty: int, eta: Optional[date], uow):
    with uow:
        uow.batches.add(model.Batch(ref, sku, qty, eta))
        # uow.commit()
```

这是一个判断，但我们倾向于要求明确的提交，以便我们必须选择何时刷新状态。

尽管我们使用了一行额外的代码，但这默认使软件安全。默认行为是不更改任何内容。反过来，这使得我们的代码更容易推理，因为只有一条代码路径会导致系统发生变化：完全成功和显式提交。任何其他代码路径、任何异常、任何提前退出 UoW 范围都会导致安全状态。

类似地，我们更喜欢默认回滚，因为这样更容易理解；这会回滚到最后一次提交，因此要么用户进行了一次提交，要么我们将其更改删除。虽然苛刻，但很简单。


## <font color='red'>示例：使用 UoW 将多个操作分组为一个原子单元</font>

以下几个示例展示了工作单元模式的使用。您可以看到它如何简单地推断哪些代码块是一起发生的。

### <font color='red'>示例1：重新分配</font>

假设我们希望能够取消分配然后重新分配订单：

```py title='重新分配服务功能'
def reallocate(
    line: OrderLine,
    uow: AbstractUnitOfWork,
) -> str:
    with uow:
        batch = uow.batches.get(sku=line.sku)
        if batch is None:
            raise InvalidSku(f'Invalid sku {line.sku}')
        batch.deallocate(line)  #(1)
        allocate(line)  #(2)
        uow.commit()
```

1. 如果`deallocate()`失败了，显然我们不想调用`allocate()`。
2. 如果`allocate()`失败了，我们可能也不想真正地提交`deallocate()`。

### <font color='red'>示例2：更改批次数量</font>
我们的船运公司给我们打电话说，其中一个集装箱门打开了，我们的一半沙发都掉进了印度洋。哎呀！

```py title='更改数量'
def change_batch_quantity(
    batchref: str, new_qty: int,
    uow: AbstractUnitOfWork,
):
    with uow:
        batch = uow.batches.get(reference=batchref)
        batch.change_purchased_quantity(new_qty)
        while batch.available_quantity < 0:
            line = batch.deallocate_one()  #(1)
        uow.commit()
```

1. 这里我们可能需要释放任意数量的行。如果我们在任何阶段失败，我们可能不想提交任何更改。


## <font color='red'>整理集成测试</font>

我们现在有三组测试，基本上都指向数据库： test_orm.py、test_repository.py和test_uow.py。我们应该丢弃一些吗？

```
└── tests
    ├── conftest.py
    ├── e2e
    │   └── test_api.py
    ├── integration
    │   ├── test_orm.py
    │   ├── test_repository.py
    │   └── test_uow.py
    ├── pytest.ini
    └── unit
        ├── test_allocate.py
        ├── test_batches.py
        └── test_services.py
```

如果您认为测试不会在长期内增加价值，那么您可以随时将其丢弃。我们认为test_orm.py主要是一个帮助我们学习 SQLAlchemy 的工具，因此我们不需要长期使用它，尤其是如果它的主要功能在test_repository.py中介绍过的话。您可能会保留最后一个测试，但我们肯定可以看到一个论点，即仅将所有内容保持在尽可能高的抽象级别（就像我们对单元测试所做的那样）。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>读者练习</strong></p>
<p>对于本章，最好的尝试可能是从头开始实现 UoW。代码一如既往地位于<a src='https://github.com/cosmicpython/code/tree/chapter_06_uow_exercise'>GitHub</a> 上。您可以严格遵循我们的模型，也可以尝试将 UoW（其职责是commit()、，rollback()和提供.batches存储库）与上下文管理器分离，上下文管理器的工作是初始化事物，然后在退出时执行提交或回滚。如果您想使用全功能，而不是摆弄所有这些类，您可以使用contextlib库的@contextmanager装饰器</p>

<p>我们删除了实际的 UoW 和伪造的 UoW，并削减了抽象的 UoW。如果您想出了一些让您特别自豪的东西，为什么不向我们发送您的仓库链接呢？</p>
</div>

!!! tip
    这是[5.TDD高速和低速](./h.TDD%20in%20High%20Gear%20and%20Low%20Gear.md) 课程的另一个例子：当我们构建更好的抽象时，我们可以移动我们的测试来针对它们运行，这使我们可以自由地更改底层细节。


## 总结

希望我们已经说服您，工作单元模式是有用的，并且上下文管理器是一种非常好的 Python 方式，可以直观地将代码分组为我们希望原子化的块。

这种模式非常有用，事实上，SQLAlchemy 已经在`Session`对象形式中使用了UoW。SQLAlchemy中的`Session`对象是您的应用程序从数据库加载数据的方式。

每次从数据库加载新实体时，`session`都会开始跟踪实体的更改，并且当`session`刷新时，所有更改都会一起保存。如果 SQLAlchemy session已经实现了我们想要的模式，为什么还要费力抽象它呢？

表1 讨论一些权衡的内容

<font color='#186d7a'>表1.工作单元模式：权衡</font>

|优点|缺点|
|---|----|
|我们对原子操作的概念有一个很好的抽象，并且上下文管理器可以很容易地直观地看到哪些代码块是原子组合在一起的。|您的 ORM 可能已经对原子性进行了一些非常好的抽象。SQLAlchemy 甚至有上下文管理器。您只需传递一个会话就可以走很远。|
|我们可以明确控制事务的开始和结束时间，并且我们的应用程序默认会以安全的方式失败。我们永远不必担心操作只提交了一部分。|我们让它看起来很容易，但你必须仔细考虑诸如回滚、多线程和嵌套事务之类的事情。也许只要坚持使用 Django 或 Flask-SQLAlchemy 提供的功能，你的生活就会更简单。|
|它是放置所有存储库的好地方，以便客户端代码可以访问它们。||
|正如您将在后面的章节中看到的，原子性不仅仅与事务有关；它可以帮助我们处理事件和消息总线。||

首先，Session API 非常丰富，支持我们领域中不想要或不需要的操作。我们UnitOfWork将会话简化为其基本核心：可以启动、提交或丢弃。

另一方面，我们使用UnitOfWork来访问我们的Repository对象。这为开发人员提供了便利，而使用普通的 SQLAlchemy Session则无法做到这一点。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>工作单元模式回顾</strong></p>
<p><strong>工作单元模式是围绕数据完整性的抽象</strong></p>
<p>它让我们在操作结束时执行单个刷新操作，有助于加强我们领域模型的一致性，并提高性能。</p>
<p><strong>它与存储库和服务层模式紧密配合</strong></p>
<p>工作单元模式通过表示原子更新完善了我们对数据访问的抽象。我们的每个服务层用例都在单个工作单元中运行，该工作单元以块的形式成功或失败。</p>
<p><strong>对于上下文管理器来说，这是一个很好的例子</strong></p>
<p>上下文管理器是 Python 中定义范围的惯用方法。我们可以使用上下文管理器在请求结束时自动回滚我们的工作，这意味着系统默认是安全的。</p>
<p><strong>SQLAlchemy 已经实现了这个模式</strong></p>
<p>我们在 SQLAlchemySession对象上引入了一个更简单的抽象，以便“缩小” ORM 和我们的代码之间的接口。这有助于保持松散耦合。</p>
</div>

最后，我们再次受到依赖倒置原则的激励：我们的服务层依赖于一个薄的抽象，我们在系统的外部边缘附加一个具体的实现。这与 SQLAlchemy 的建议非常吻合 ：

!!! quote
    将会话（通常还有事务）的生命周期保持独立和外部。最全面的方法（建议用于更实质性的应用程序）将尝试将会话、事务和异常管理的细节与执行其工作的程序的细节保持尽可能远的距离。
    <p align='right'>— SQLALchemy“会话基础”文档</p>







