# 8.事件和消息总线


到目前为止，我们已经花费了大量的时间和精力来解决一个简单的问题，而使用 Django 可以轻松解决。您可能会问，增加的可测试性和表现力是否真的值得付出所有努力。

然而，在实践中，我们发现，并不是那些显而易见的功能搞乱了我们的代码库：而是边缘上的混乱。这些混乱是报告、权限和工作流，它们触及无数对象。

我们的示例是一个典型的通知要求：当我们因为缺货而无法分配订单时，我们应该通知采购团队。他们会通过购买更多库存来解决问题，一切都会好起来。

对于第一个版本，我们的产品所有者说我们可以通过电子邮件发送警报。

让我们看看，当需要插入一些构成我们系统的大部分普通东西时，我们的架构如何承受。

我们将从最简单、最快捷的事情开始，并讨论为什么正是这种决定导致我们陷入“大泥球”。

然后，我们将展示如何使用领域事件模式将副作用与用例分开，以及如何使用简单的消息总线模式根据这些事件触发行为。我们将展示一些创建这些事件的选项以及如何将它们传递到消息总线，最后我们将展示如何修改工作单元模式以优雅地将两者连接在一起，如图1中所示。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0801.png){ align=center }
    <figcaption><font size=2>图1.流经系统的事件</font></figcaption>
</figure>

!!! tip
    本章的代码位于[GitHub](https://github.com/cosmicpython/code/tree/chapter_08_events_and_message_bus)上的chapter_08_events_and_message_bus 分支中：
    ```
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout chapter_08_events_and_message_bus
    # or to code along, checkout the previous chapter:
    git checkout chapter_07_aggregate
    ```

## <font color='red'>避免弄乱</font>

所以。缺货时会发送电子邮件提醒。当我们有新的要求（比如那些与核心领域实际上无关的要求）时，很容易就开始将这些东西转储到我们的 Web 控制器中。

### <font color='red'>首先，让我们避免弄乱我们的 Web 控制器</font>

作为一次性的黑客攻击，这可能是可以的：

```py title='只需将其敲入端点即可——会出什么问题？（src/allocation/entrypoints/flask_app.py）'
@app.route("/allocate", methods=["POST"])
def allocate_endpoint():
    line = model.OrderLine(
        request.json["orderid"],
        request.json["sku"],
        request.json["qty"],
    )
    try:
        uow = unit_of_work.SqlAlchemyUnitOfWork()
        batchref = services.allocate(line, uow)
    except (model.OutOfStock, services.InvalidSku) as e:
        send_mail(
            "out of stock",
            "stock_admin@made.com",
            f"{line.orderid} - {line.sku}"
        )
        return {"message": str(e)}, 400

    return {"batchref": batchref}, 201
```

…​但很容易看出，通过这样的修补，我们很快就会陷入混乱。发送电子邮件不是我们 HTTP 层的工作，我们希望能够对这个新功能进行单元测试。


### <font color='red'>不要把我们的模型弄乱</font>

假设我们不想将此代码放入我们的 Web 控制器中，因为我们希望它们尽可能精简，我们可以考虑将其放在源头的模型中：

```py title='我们模型中的电子邮件发送代码也不太好（src/allocation/domain/model.py）'
    def allocate(self, line: OrderLine) -> str:
        try:
            batch = next(b for b in sorted(self.batches) if b.can_allocate(line))
            #...
        except StopIteration:
            email.send_mail("stock@made.com", f"Out of stock for {line.sku}")
            raise OutOfStock(f"Out of stock for sku {line.sku}")
```

但那更糟糕！我们不希望我们的模型对诸如此类的基础设施问题有任何依赖, 如`email.send_mail`。

发送电子邮件这件事是令人讨厌的麻烦事，它扰乱了我们系统整洁的流程。我们希望我们的领域模型专注于“您不能分配比实际可用的东西更多的东西”这一规则。


### <font color='red'>或者服务层！</font>

要求“尝试分配一些库存，如果失败则发送电子邮件”是工作流编排的一个例子：它是系统为实现目标必须遵循的一组步骤。

我们已经编写了一个服务层来为我们管理业务流程，但即使在这里，该功能也感觉不合适：

```py title='而在服务层，它的位置不合适（src/allocation/service_layer/services.py）'
def allocate(
    orderid: str, sku: str, qty: int,
    uow: unit_of_work.AbstractUnitOfWork,
) -> str:
    line = OrderLine(orderid, sku, qty)
    with uow:
        product = uow.products.get(sku=line.sku)
        if product is None:
            raise InvalidSku(f"Invalid sku {line.sku}")
        try:
            batchref = product.allocate(line)
            uow.commit()
            return batchref
        except model.OutOfStock:
            email.send_mail("stock@made.com", f"Out of stock for {line.sku}")
            raise
```

捕获异常并重新引发它？这可能会更糟，但这肯定会让我们不高兴。为什么很难找到适合此代码的归宿？


## <font color='red'>单一职责原则</font>

确实，这违反了单一责任原则(SRP)。我们的用例是分配。我们的端点、服务功能和域方法都被称为`allocate`，而不是 `allocate_and_send_mail_if_out_of_stock`。

!!! tip
    经验法则：如果您不能在不使用“然后”或“和”之类的词语的情况下描述您的功能，那么您可能违反了 SRP。

SRP 的一个表述是，每个类应该只有一个改变的原因。当我们从电子邮件切换到短信时，我们不应该更新我们的 allocate()功能，因为这显然是一项单独的责任。

为了解决这个问题，我们将把流程分成几个步骤，这样不同的关注点就不会纠缠在一起。[ 2 ] 领域模型的工作是知道我们缺货了，但发送警报的责任属于其他地方。我们应该能够打开或关闭此功能，或者改用短信通知，而无需更改领域模型的规则。

我们还希望服务层不受实现细节的影响。我们希望将依赖倒置原则应用于通知，以便我们的服务层依赖于抽象，就像我们通过使用工作单元避免依赖数据库一样。

## <font color='red'>全部放到消息总线</font>

我们在这里要介绍的模式是领域事件和消息总线。我们可以通过几种方式实现它们，因此在确定我们最喜欢的一种之前，我们将展示几种。

### <font color='red'>模型记录事件</font>

首先，我们的模型将不再关注电子邮件，而是负责记录事件（已发生的事情的事实）。我们将使用消息总线来响应事件并调用新操作。

### <font color='red'>事件是简单的数据类</font>

事件是一种值对象。事件没有任何行为，因为它们是纯数据结构。我们总是用领域语言来命名事件，并将它们视为领域模型的一部分。

我们可以将它们存储在model.py中，但我们也可以将它们保存在自己的文件中（这可能是考虑重构一个名为 domain 的目录的好时机，以便我们有domain/model.py和domain/events.py）：

```py title='事件类（src/allocation/domain/events.py）'
from dataclasses import dataclass


class Event:  #(1)
    pass


@dataclass
class OutOfStock(Event):  #(2)
    sku: str
```

1. 一旦我们有了一定数量的事件，我们就会发现拥有一个可以存储通用属性的父类很有用。它对于消息总线中的类型提示也很有用，您很快就会看到。

2. `dataclasses`对于领域事件来说也非常有用。

### <font color='red'>模型引发事件</font>

当我们的领域模型记录发生的事实时，我们说它引发了一个事件。

从外部看它是这样的；如果我们要求Product分配但无法分配，它应该引发一个事件：

```py title='测试我们的聚合以引发事件（tests/unit/test_product.py）'
def test_records_out_of_stock_event_if_cannot_allocate():
    batch = Batch("batch1", "SMALL-FORK", 10, eta=today)
    product = Product(sku="SMALL-FORK", batches=[batch])
    product.allocate(OrderLine("order1", "SMALL-FORK", 10))

    allocation = product.allocate(OrderLine("order2", "SMALL-FORK", 1))
    assert product.events[-1] == events.OutOfStock(sku="SMALL-FORK")  #(1)
    assert allocation is None
```

1. 我们的聚合将公开一个`.events`新属性，该属性将以`Event`对象的形式包含已发生事件的事实列表。

模型的内部结构如下：

```py title='该模型引发域事件（src/allocation/domain/model.py）'
class Product:
    def __init__(self, sku: str, batches: List[Batch], version_number: int = 0):
        self.sku = sku
        self.batches = batches
        self.version_number = version_number
        self.events = []  # type: List[events.Event]  #(1)

    def allocate(self, line: OrderLine) -> str:
        try:
            #...
        except StopIteration:
            self.events.append(events.OutOfStock(line.sku))  #(2)
            # raise OutOfStock(f"Out of stock for sku {line.sku}")  #(3)
            return None
```

1. `.events`这是我们正在使用的新属性。
2. 我们不是直接调用某些电子邮件发送代码，而是仅使用域的语言在事件发生的地点记录这些事件。
3. 我们还将停止针对缺货情况引发异常。事件将完成异常所执行的工作。

!!! note
    我们实际上正在解决迄今为止存在的代码异味，即我们使用[异常来表示控制流](https://softwareengineering.stackexchange.com/questions/189222/are-exceptions-as-control-flow-considered-a-serious-antipattern-if-so-why)。通常，如果您正在实现域事件，请不要引发异常来描述相同的域概念。正如您稍后在工作单元模式中处理事件时所看到的那样，必须同时推理事件和异常会令人困惑。


### <font color='red'>消息总线将事件映射到处理程序</font>

消息总线基本上说，“当我看到此事件时，我应该调用以下处理程序函数。”换句话说，这是一个简单的发布-订阅系统。处理程序被订阅以接收我们发布到总线的事件。这听起来比实际要难，我们通常用一个字典来实现它：

```py title='简单消息总线（src/allocation/service_layer/messagebus.py）'
def handle(event: events.Event):
    for handler in HANDLERS[type(event)]:
        handler(event)


def send_out_of_stock_notification(event: events.OutOfStock):
    email.send_mail(
        "stock@made.com",
        f"Out of stock for {event.sku}",
    )


HANDLERS = {
    events.OutOfStock: [send_out_of_stock_notification],
}  # type: Dict[Type[events.Event], List[Callable]]
```

!!! note
    请注意，所实现的消息总线无法提供并发性，因为一次只能运行一个处理程序。我们的目标不是支持并行线程，而是从概念上分离任务，并尽可能保持每个 UoW 较小。这有助于我们理解代码库，因为如何运行每个用例的“配方”都写在一个地方。请参阅以下边栏。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>这像Celery吗？</strong></p>
<p>Celery是 Python 世界中流行的工具，用于将独立的工作块延迟到异步任务队列。我们在这里介绍的消息总线非常不同，因此对上述问题的简短回答是否定的；我们的消息总线与 Express.js 应用程序、UI 事件循环或参与者框架有更多共同之处。</p>

<p>如果您确实需要将工作移出主线程，您仍然可以使用我们基于事件的隐喻，但我们建议您为此 使用外部事件。 [11.外部事件权衡]中有更多讨论，但本质上，如果您实现一种将事件持久化到集中存储的方法，则可以订阅其他容器或其他微服务。 然后，使用事件在单个流程/服务内的工作单元之间分离职责的相同概念可以扩展到多个流程——可能是同一服务内的不同容器，也可能是完全不同的微服务。</p>

<p>如果您按照我们的方法操作，那么您用于分发任务的 API 就是您的事件类，或者它们的 JSON 表示形式。这让您在分发任务给谁方面拥有很大的灵活性；它们不一定是 Python 服务。Celery 用于分发任务的 API 本质上是“函数名称加参数”，这更加严格，并且仅适用于 Python。</p>
</div>


## <font color='red'>选项1：服务层从模型中获取事件并将其放在消息总线上</font>

我们的领域模型会引发事件，而我们的消息总线会在事件发生时调用正确的处理程序。现在我们需要做的就是将两者连接起来。我们需要一些东西来从模型中捕获事件并将它们传递给消息总线——发布步骤。

最简单的方法是在我们的服务层添加一些代码：

```py title='具有显式消息总线的服务层（src/allocation/service_layer/services.py）'
from . import messagebus
...

def allocate(
    orderid: str, sku: str, qty: int,
    uow: unit_of_work.AbstractUnitOfWork,
) -> str:
    line = OrderLine(orderid, sku, qty)
    with uow:
        product = uow.products.get(sku=line.sku)
        if product is None:
            raise InvalidSku(f"Invalid sku {line.sku}")
        try:  #(1)
            batchref = product.allocate(line)
            uow.commit()
            return batchref
        finally:  #(1)
            messagebus.handle(product.events)  #(2)
```

1. 我们保留了`try/finally`早期丑陋的实现（我们还没有摆脱所有异常，只是`OutOfStock`）。
2. 但是现在，服务层不再直接依赖电子邮件基础设施，而是负责将事件从模型传递到消息总线。

这已经避免了我们在简单实现中遇到的一些丑陋现象，并且我们有几个像这样工作的系统，其中服务层明确地从聚合中收集事件并将它们传递给消息总线。    


## <font color='red'>选项2：服务层引发自己的事件</font>

我们使用的另一种变体是让服务层负责直接创建和引发事件，而不是由域模型引发事件：

```py title='服务层直接调用messagebus.handle（src/allocation/service_layer/services.py）'
def allocate(
    orderid: str, sku: str, qty: int,
    uow: unit_of_work.AbstractUnitOfWork,
) -> str:
    line = OrderLine(orderid, sku, qty)
    with uow:
        product = uow.products.get(sku=line.sku)
        if product is None:
            raise InvalidSku(f"Invalid sku {line.sku}")
        batchref = product.allocate(line)
        uow.commit() #(1)

        if batchref is None:
            messagebus.handle(events.OutOfStock(line.sku))
        return batchref
```
1. 和以前一样，即使分配失败我们也会提交，因为这样代码更简单，也更容易推理：除非出现问题，否则我们总是提交。当我们没有更改任何内容时提交是安全的，并且可以保持代码整洁。

再次强调，我们的生产应用程序以这种方式实现该模式。哪种方法适合您将取决于您面临的权衡，但我们想向您展示我们认为最优雅的解决方案，其中我们让工作单元负责收集和引发事件。


## <font color='red'>选项3：UoW 将事件发布到消息总线</font>

UoW 已经有一个`try/finally`，并且它知道当前正在运行的所有聚合，因为它提供对存储库的访问权限。因此，它是发现事件并将其传递到消息总线的好地方：

```py title='UoW 与消息总线相遇（src/allocation/service_layer/unit_of_work.py）'
class AbstractUnitOfWork(abc.ABC):
    ...

    def commit(self):
        self._commit()  #(1)
        self.publish_events()  #(2)

    def publish_events(self):  #(2)
        for product in self.products.seen:  #(3)
            while product.events:
                event = product.events.pop(0)
                messagebus.handle(event)

    @abc.abstractmethod
    def _commit(self):
        raise NotImplementedError

...

class SqlAlchemyUnitOfWork(AbstractUnitOfWork):
    ...

    def _commit(self):  #(1)
        self.session.commit()
```

1. 我们将改变提交方法，以要求子类使用`._commit() `私有方法。
2. 提交后，我们将遍历存储库中看到的所有对象，并将其事件传递给消息总线。
3. 这依赖于存储库使用`.seen`新属性跟踪已加载的聚合，正如您将在下一个清单中看到的那样。

!!! note
    您是否想知道如果其中一个处理程序失败会发生什么？我们将在[10.命令](./n.Commands%20and%20Command%20Handler.md) 中详细讨论错误处理。

```py title='存储库跟踪通过它的传递的聚合（src/allocation/adapters/repository.py'
class AbstractRepository(abc.ABC):
    def __init__(self):
        self.seen = set()  # type: Set[model.Product]  #(1)

    def add(self, product: model.Product):  #(2)
        self._add(product)
        self.seen.add(product)

    def get(self, sku) -> model.Product:  #(3)
        product = self._get(sku)
        if product:
            self.seen.add(product)
        return product

    @abc.abstractmethod
    def _add(self, product: model.Product):  #(2)
        raise NotImplementedError

    @abc.abstractmethod  #(3)
    def _get(self, sku) -> model.Product:
        raise NotImplementedError


class SqlAlchemyRepository(AbstractRepository):
    def __init__(self, session):
        super().__init__()
        self.session = session

    def _add(self, product):  #(2)
        self.session.add(product)

    def _get(self, sku):  #(3)
        return self.session.query(model.Product).filter_by(sku=sku).first()
```

1. 为了使 UoW 能够发布新事件，它需要能够向存储库询问`Product`在此会话期间使用了哪些对象。我们在`.seen`使用一个set来存储它们。这意味着我们的实现需要调用`super().__init__()` 。
2. 父类`add()`方法调用`.seen`添加内容，现在需要子类来实现`._add()`。
3. 类似地，`.get()`委托给子类来实现一个`._get()`函数，以便捕获所看到的对象。

!!! note
    使用<code>._underscorey()</code>方法和子类化绝对不是实现这些模式的唯一方法。请尝试 本章中的 “读者练习” ，并尝试一些替代方法。

当 UoW 和存储库以这种方式协作以自动跟踪活动对象并处理其事件后，服务层就可以完全摆脱事件处理问题：

```py title='服务层再次清理（src/allocation/service_layer/services.py）'
def allocate(
    orderid: str, sku: str, qty: int,
    uow: unit_of_work.AbstractUnitOfWork,
) -> str:
    line = OrderLine(orderid, sku, qty)
    with uow:
        product = uow.products.get(sku=line.sku)
        if product is None:
            raise InvalidSku(f"Invalid sku {line.sku}")
        batchref = product.allocate(line)
        uow.commit()
        return batchref
```

我们还必须记住更改服务层中的伪造并让它们super()在正确的位置调用，并实现下划线方法，但变化很小：

``` py title='服务层伪造需要调整（tests/unit/test_services.py）'
class FakeRepository(repository.AbstractRepository):
    def __init__(self, products):
        super().__init__()
        self._products = set(products)

    def _add(self, product):
        self._products.add(product)

    def _get(self, sku):
        return next((p for p in self._products if p.sku == sku), None)

...

class FakeUnitOfWork(unit_of_work.AbstractUnitOfWork):
    ...

    def _commit(self):
        self.committed = True
```

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>读者练习</strong></p>
<p>用我们敬爱的技术评论员 Hynek 的话来说，您是否觉得所有这些._add()方法._commit()都“非常恶心”？它是否“让您想用毛绒蛇敲打哈利的头”？嘿，我们的代码清单仅供参考，而不是完美的解决方案！为什么不去看看您是否可以做得更好？</p>

<p>一种组合优于继承的方法是实现一个包装类：</p>
<pre>
<code style='border: 1px solid;'>
# 包装器添加功能然后委托（src/adapters/repository.py）
class TrackingRepository:
    seen: Set[model.Product]

    def __init__(self, repo: AbstractRepository):
        self.seen = set()  # type: Set[model.Product]
        self._repo = repo

    def add(self, product: model.Product):  #(1)
        self._repo.add(product)  #(1)
        self.seen.add(product)

    def get(self, sku) -> model.Product:
        product = self._repo.get(sku)
        if product:
            self.seen.add(product)
        return product
</code>
</pre>
<ol>
    <li>通过包装存储库，我们可以调用实际的.add() 和.get()方法，避免奇怪的下划线方法。</li>
</ol>
看看是否可以对我们的UoW类应用类似的模式，以摆脱那些Java-y _commit()方法。你可以在<a src='https://github.com/cosmicpython/code/tree/chapter_08_events_and_message_bus_exercise'>GitHub</a>上找到代码。

将所有 ABC 转换为typing.Protocol是强迫自己避免使用继承的好方法。如果您想出好办法，请告诉我们！

</div>


## <font color='red'>总结</font>

领域事件为我们提供了一种处理系统中工作流的方法。我们经常发现，听取领域专家的意见，他们以因果或时间的方式表达需求 - 例如，“当我们尝试分配库存但没有可用库存时，我们应该向采购团队发送电子邮件。”

“当 X 发生时，则 Y”这个神奇的词经常会告诉我们一个可以在系统中具体化的事件。在我们的模型中将事件视为一等事物有助于我们使代码更易于测试和观察，并有助于隔离问题。

表1显示了我们所看到的权衡。

<font color='#186d7a'>表1.领域事件：权衡</font>

|优点|缺点|
|---|---|
|当我们必须采取多项措施来响应请求时，消息总线为我们提供了一种很好的责任分离方法。|消息总线是需要您思考的另外一件事；工作单元为我们引发事件的实现既简洁又神奇。当我们调用commit它时，我们也会向人们发送电子邮件，这一点并不明显。|
|事件处理程序与“核心”应用程序逻辑很好地分离，使得以后可以轻松更改其实现。|此外，隐藏的事件处理代码是同步执行的，这意味着您的服务层函数只有在所有事件的处理程序完成之后才会完成。这可能会导致您的 Web 端点出现意外的性能问题（添加异步处理是可能的，但这会使事情变得更加混乱）。|
|领域事件是模拟现实世界的绝佳方式，我们可以在与利益相关者一起建模时将其用作业务语言的一部分。|更一般地说，事件驱动的工作流可能会令人困惑，因为在事物被分割到多个处理程序链之后，系统中没有一个地方可以让您了解如何满足请求。|
||您还会面临事件处理程序之间出现循环依赖以及无限循环的可能性。|

不过，事件的作用远不止发送电子邮件。在[7.聚合](./j.aggregates%20and%20Consistency%20Boundaries.md)中，我们花了很多时间说服您定义聚合，即保证一致性的边界。人们经常问：“如果我需要更改请求中的多个聚合，我该怎么办？”现在我们有了回答这个问题所需的工具。

如果我们有两个可以事务隔离的东西（例如，订单和产品 ），那么我们可以通过使用事件使它们最终保持一致。当订单被取消时，我们应该找到分配给该订单的产品并删除分配。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>领域事件和消息总线回顾</strong></p>
<p><strong>事件有助于实现单一责任原则</strong></p>
<p>当我们将多个关注点混合在一个地方时，代码就会变得混乱。事件可以通过将主要用例与次要用例分开来帮助我们保持整洁。我们还使用事件在聚合之间进行通信，这样我们就不需要运行锁定多个表的长期事务。</p>

<p><strong>消息总线将消息路由到处理程序</strong></p>
<p>您可以将消息总线视为从事件映射到其消费者的字典。它不“知道”事件的任何含义；它只是用于在系统内传递消息的愚蠢基础设施。</p>

<p><strong>选项1：服务层引发事件并将其传递给消息总线</strong></p>
<p>在系统中开始使用事件的最简单方法是，在bus.handle(some_new_event)提交工作单元后通过调用处理程序来引发它们。</p>

<p><strong>选项2：领域模型引发事件，服务层将其传递给消息总线</strong></p>
<p>关于何时引发事件的逻辑实际上应该与模型共存，因此我们可以通过从域模型引发事件来改善系统的设计和可测试性。我们的处理程序可以轻松地从模型对象中收集事件commit并将其传递给总线。</p>

<p><strong>选项3：UoW 从聚合中收集事件并将其传递到消息总线</strong></p>
<p>向每个处理程序添加内容bus.handle(aggregate.events)很烦人，因此我们可以让工作单元负责引发由加载对象引发的事件，从而简化流程。这是最复杂的设计，可能依赖于 ORM 魔法，但一旦设置好，它就很简洁且易于使用。</p>
</div>

在[9.使用消息总线大干一场](./m.Going%20to%20Town%20on%20the%20Message%20Bus.md)中，我们将使用新的消息总线构建更复杂的工作流程，更详细地研究这个想法。
