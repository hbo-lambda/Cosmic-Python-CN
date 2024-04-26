# 9.使用消息总线大干一场


在本章中，我们将开始使事件成为应用程序内部结构中更为重要的部分。我们将从之前的状态开始：如图1，其中事件是一个可选的副作用……

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0901.png){ align=center }
    <figcaption><font size=2>图1.之前：消息总线是一个可选的附加组件</font></figcaption>
</figure>

…​到现在如图2，一切都通过消息总线进行，我们的应用程序从根本上已经转变为消息处理器。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0902.png){ align=center }
    <figcaption><font size=2>图2.消息总线现在是服务层的主要入口点</font></figcaption>
</figure>

!!! tip
    本章的代码位于[GitHub](https://github.com/cosmicpython/code/tree/chapter_09_all_messagebus)上的chapter_09_all_messagebus 分支中：
    ```
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout chapter_09_all_messagebus
    # or to code along, checkout the previous chapter:
    git checkout chapter_08_events_and_message_bus
    ```

## <font color='red'>新需求引领我们走向新架构</font>

Rich Hickey 谈到了情境化软件，即运行时间较长的软件，用于管理真实世界的流程。例如仓库管理系统、物流调度程序和工资系统。

这种软件编写起来很棘手，因为在现实世界中，总是会发生意想不到的事情，因为现实世界中充满了物理对象和不可靠的人。例如：

1. 在盘点过程中，我们发现三栋`SPRINGY-MATTRESS`房屋因屋顶漏水而遭受了水损。
2. 一批`RELIABLE-FORK`货物缺少所需文件，被海关扣留数周。随后有三批`RELIABLE-FORK`货物未能通过安全测试并被销毁。
3. 全球片材的短缺意味着我们无法生产下一批`SPARKLY-BOOKCASE`。

在这种情况下，我们了解到需要更改系统中已有的批次数量。也许有人在清单上写错了数字，或者一些沙发从卡车上掉了下来。在与企业交谈后，我们将这种情况建模为图3的样子

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0903.png){ align=center }
    <figcaption><font size=2>图3.批次数量改变意味着取消分配和重新分配</font></figcaption>
</figure>

是的，我们将调用的称为`BatchQuantityChanged`的事件应该会引导我们更改批次的数量，但同时也要应用业务规则：如果新数量下降到低于已分配的总量，我们需要 从该批次中取消分配 这些订单。然后每个订单都需要新的分配，我们可以将其捕获为名为`AllocationRequired`的事件。

也许您已经预料到我们的内部消息总线和事件可以帮助实现这一要求。我们可以定义一个名为`change_batch_quantity`的服务，它知道如何调整批次数量以及如何释放多余的订单行，然后每个释放都可以发出一个`AllocationRequired`事件，该事件可以在单独的事务中转发到现有`allocate`服务。再一次，我们的消息总线帮助我们实施单一责任原则，并允许我们在事务和数据完整性之间做出选择。

### <font color='red'>想象一个架构变革：一切都将成为事件处理程序</font>

但在开始之前，请先想一想我们要去哪里。我们的系统中有两种流程：

- 由服务层函数处理的 API 调用
- 内部事件（可能作为服务层函数的副作用而引发）及其处理程序（进而调用服务层函数）

如果一切都是事件处理程序，那不是更简单吗？如果我们将 API 调用重新视为捕获事件，那么服务层函数也可以是事件处理程序，我们不再需要区分内部和外部事件处理程序：

- `services.allocate()`可以作为`AllocationRequired`事件的处理程序 并可以发出`Allocated`事件作为其输出。
- `services.add_batch()`可以是`BatchCreated`事件的处理程序

我们的新需求将符合相同的模式：

- 名为`BatchQuantityChanged`的事件可以调用名为`change_batch_quantity()`的处理程序。
- 并且它可能引发的新`AllocationRequired`事件也可以被传递到`services.allocate()`，因此来自 API 的全新分配和由释放分配内部触发的重新分配之间没有概念上的区别。

听起来有点多？让我们逐步实现这一切。我们将遵循[准备重构](https://martinfowler.com/articles/preparatory-refactoring-example.html)工作流程，即“让更改变得简单；然后进行简单的更改”：

1. 我们将服务层重构为事件处理程序。我们可以习惯于将事件作为我们描述系统输入的方式。具体来说，现有函数`services.allocate()`将成为名为`AllocationRequired`的事件的处理程序。
2. 我们构建一个端到端测试，将`BatchQuantityChanged`事件放入系统并查找`Allocated`出现的事件。
3. 我们的实现在概念上非常简单：一个新的`BatchQuantityChanged`事件处理程序，它将实现发出`AllocationRequired`事件，而这些事件又将由 API 使用的完全相同的分配处理程序来处理。

在此过程中，我们将对消息总线和 UoW 进行小幅调整，将将新事件放到消息总线上的责任转移到消息总线本身。


## <font color='red'>将服务函数重构为消息处理程序</font>

我们首先定义捕获当前 API 输入的两个事件 - `AllocationRequired`和`BatchCreated`：

```py title='BatchCreated 和 AllocationRequired 事件（src/allocation/domain/events.py）'
@dataclass
class BatchCreated(Event):
    ref: str
    sku: str
    qty: int
    eta: Optional[date] = None

...

@dataclass
class AllocationRequired(Event):
    orderid: str
    sku: str
    qty: int
```

然后我们将services.py重命名为handlers.py；我们为`send_out_of_stock_notification`添加现有的消息处理程序；最重要的是，我们更改所有处理程序，使它们具有相同的输入、事件和 UoW：

```py title='处理程序和服务是同一件事（src/allocation/service_layer/handlers.py）'
def add_batch(
    event: events.BatchCreated,
    uow: unit_of_work.AbstractUnitOfWork,
):
    with uow:
        product = uow.products.get(sku=event.sku)
        ...


def allocate(
    event: events.AllocationRequired,
    uow: unit_of_work.AbstractUnitOfWork,
) -> str:
    line = OrderLine(event.orderid, event.sku, event.qty)
    ...


def send_out_of_stock_notification(
    event: events.OutOfStock,
    uow: unit_of_work.AbstractUnitOfWork,
):
    email.send(
        "stock@made.com",
        f"Out of stock for {event.sku}",
    )
```

变化可能更明显一点，如下所示：

``` py title='从服务更改为处理程序（src/allocation/service_layer/handlers.py）'
 def add_batch(
-    ref: str, sku: str, qty: int, eta: Optional[date],
+    event: events.BatchCreated,
     uow: unit_of_work.AbstractUnitOfWork,
 ):
     with uow:
-        product = uow.products.get(sku=sku)
+        product = uow.products.get(sku=event.sku)
     ...


 def allocate(
-    orderid: str, sku: str, qty: int,
+    event: events.AllocationRequired,
     uow: unit_of_work.AbstractUnitOfWork,
 ) -> str:
-    line = OrderLine(orderid, sku, qty)
+    line = OrderLine(event.orderid, event.sku, event.qty)
     ...

+
+def send_out_of_stock_notification(
+    event: events.OutOfStock,
+    uow: unit_of_work.AbstractUnitOfWork,
+):
+    email.send(
     ...
```

在此过程中，我们让服务层的 API 更加结构化和一致。它原本是一堆散乱的原语，现在则使用定义明确的对象。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>从领域对象，到起初的痴迷，再到事件作为接口</strong></p>
<p>你们中的一些人可能还记得[将服务层测试与域完全分离]，其中我们将服务层 API 从域对象更改为基元。现在我们要回到原来的状态，但要更改为不同的对象？发生了什么？</p>
<p>在面向对象圈子里，人们将对原始类型的痴迷视为一种反模式：他们会说，避免在公共 API 中使用原始类型，而是用自定义值类来包装它们。在 Python 世界中，很多人会对此持怀疑态度，认为这是一条经验法则。如果盲目地应用，这肯定会带来不必要的复杂性。所以这不是我们本身正在做的事情。</p>

<p>从领域对象到原语的转变给我们带来了很好的解耦：我们的客户端代码不再直接与领域耦合，因此即使我们决定更改模型，服务层也可以提供保持不变的 API，反之亦然。</p>
<p>那么我们是否倒退了呢？好吧，我们的核心领域模型对象仍然可以自由变化，但相反，我们将外部世界与我们的事件类耦合在一起。它们也是领域的一部分，但我们希望它们变化较少，因此将它们耦合在一起是一种合理的产物。</p>

<p>我们自己买了什么？现在，在我们的应用程序中调用用例时，我们不再需要记住特定的基元组合，而只需要记住一个代表应用程序输入的事件类。这在概念上非常好。最重要的是，正如您将在[附录E 验证]中看到的那样，这些事件类可以成为进行一些输入验证的好地方。</p>
</div>


### <font color='red'>消息总线现在从 UoW 收集事件</font>

我们的事件处理程序现在需要一个 UoW。此外，随着我​​们的消息总线成为我们应用程序的核心，明确让它负责收集和处理新事件是有意义的。到目前为止，UoW 和消息总线之间存在一些循环依赖，因此这将使其成为单向的。我们将让消息总线从 UoW 中拉取事件，而不是让 UoW 将事件推送到消息总线上。

```py title='Handle 采用 UoW 并管理队列 (src/allocation/service_layer/messagebus.py）'
def handle(
    event: events.Event,
    uow: unit_of_work.AbstractUnitOfWork,  #(1)
):
    queue = [event]  #(2)
    while queue:
        event = queue.pop(0)  #(3)
        for handler in HANDLERS[type(event)]:  #(3)
            handler(event, uow=uow)  #(4)
            queue.extend(uow.collect_new_events())  #(5)
```

1. 现在，消息总线每次启动时都会传递 UoW。
2. 当我们开始处理第一个事件时，我们启动一个队列。
3. 我们从队列前面弹出事件并调用它们的处理程序（字典`HANDLERS`没有改变；它仍然将事件类型映射到处理程序函数）。
4. 消息总线将 UoW 传递给每个处理程序。
5. 每个处理程序完成后，我们会收集所有已生成的新事件并将其添加到队列中。

在unit_of_work.py中，`publish_events()`变成不太活跃的方法，`collect_new_events()`：

```py title='UoW 不再将事件直接放在总线上 (src/allocation/service_layer/unit_of_work.py）'
-from . import messagebus  #(1)


 class AbstractUnitOfWork(abc.ABC):
@@ -22,13 +21,11 @@ class AbstractUnitOfWork(abc.ABC):

     def commit(self):
         self._commit()
-        self.publish_events()  #(2)

-    def publish_events(self):
+    def collect_new_events(self):
         for product in self.products.seen:
             while product.events:
-                event = product.events.pop(0)
-                messagebus.handle(event)
+                yield product.events.pop(0)  #(3)
```

1. 该`unit_of_work`模块现在不再依赖于`messagebus`。
2. 我们不再`commit`时自动`publish_events`。取而代之的是，消息总线会跟踪事件队列。
3. 并且 UoW 不再主动将事件放在消息总线上；它只是使它们可用。

### <font color='red'>我们的测试也都是根据事件编写的</font>

我们的测试现在通过创建事件并将其放在消息总线上来运行，而不是直接调用服务层函数：

```py title='处理程序测试使用事件（tests/unit/test_handlers.py）'
class TestAddBatch:
     def test_for_new_product(self):
         uow = FakeUnitOfWork()
-        services.add_batch("b1", "CRUNCHY-ARMCHAIR", 100, None, uow)
+        messagebus.handle(
+            events.BatchCreated("b1", "CRUNCHY-ARMCHAIR", 100, None), uow
+        )
         assert uow.products.get("CRUNCHY-ARMCHAIR") is not None
         assert uow.committed

...

 class TestAllocate:
     def test_returns_allocation(self):
         uow = FakeUnitOfWork()
-        services.add_batch("batch1", "COMPLICATED-LAMP", 100, None, uow)
-        result = services.allocate("o1", "COMPLICATED-LAMP", 10, uow)
+        messagebus.handle(
+            events.BatchCreated("batch1", "COMPLICATED-LAMP", 100, None), uow
+        )
+        result = messagebus.handle(
+            events.AllocationRequired("o1", "COMPLICATED-LAMP", 10), uow
+        )
         assert result == "batch1"
```

### <font color='red'>暂时的丑陋黑客：消息总线必须返回结果</font>

我们的 API 和服务层目前希望在调用我们的`allocate()`处理程序时知道分配的批处理引用。这意味着我们需要对消息总线进行临时修改，以使其返回事件：

```py title='消息总线返回结果（src/allocation/service_layer/messagebus.py）'
 def handle(
     event: events.Event,
     uow: unit_of_work.AbstractUnitOfWork,
 ):
+    results = []
     queue = [event]
     while queue:
         event = queue.pop(0)
         for handler in HANDLERS[type(event)]:
-            handler(event, uow=uow)
+            results.append(handler(event, uow=uow))
             queue.extend(uow.collect_new_events())
+    return results
```

这是因为我们在系统中混合了读写职责。我们将在[12.CQRS](./p.CQRS.md)中解决这个问题。

### <font color='red'>修改 API 以处理事件</font>

```py title='Flask 更改为消息总线作为差异（src/allocation/entrypoints/flask_app.py）'
 @app.route("/allocate", methods=["POST"])
 def allocate_endpoint():
     try:
-        batchref = services.allocate(
-            request.json["orderid"],  #(1)
-            request.json["sku"],
-            request.json["qty"],
-            unit_of_work.SqlAlchemyUnitOfWork(),
+        event = events.AllocationRequired(  #(2)
+            request.json["orderid"], request.json["sku"], request.json["qty"]
         )
+        results = messagebus.handle(event, unit_of_work.SqlAlchemyUnitOfWork())  #(3)
+        batchref = results.pop(0)
     except InvalidSku as e:
```

1. 而不是使用从请求JSON中提取的一堆原语来调用服务层……​
2. 我们实例化一个事件。
3. 然后我们将它传递给消息总线。

我们应该回到一个功能齐全的应用程序，但现在它是完全由事件驱动的：

过去的服务层功能现在是事件处理程序。

这使得它们与我们调用来处理域模型引发的内部事件的函数相同。

我们使用事件作为数据结构来捕获系统输入以及交接内部工作包。

现在，整个应用程序最好被描述为一个消息处理器，或者如果你愿意的话，也可以称为一个事件处理器。我们将在[下一章](./n.Commands%20and%20Command%20Handler.md)中讨论两者的区别 。


## <font color='red'>实现我们的新需求</font>

我们已经完成了重构阶段。让我们看看我们是否真的“让改变变得简单”。让我们实现我们的新需求，如图4.重新分配流程的序列图所示：我们将接收一些新`BatchQuantityChanged`事件作为输入，并将它们传递给处理程序，处理程序又可能会发出一些`AllocationRequired`事件，而这些事件又将返回到我们现有的处理程序进行重新分配。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0904.png){ align=center }
    <figcaption><font size=2>图4.重新分配流程的序列图</font></figcaption>
</figure>

!!! warning
    当您将类似这样的事务拆分到两个工作单元时，您现在有两个数据库事务，因此您将面临完整性问题：可能会发生一些事情，导致第一个事务完成，但第二个事务没有完成。您需要考虑这是否可以接受，以及是否需要注意何时发生并采取措施。

### <font color='red'>我们的新事件</font>

告诉我们批次数量已发生变化的事件很简单；它只需要一个批次参考和一个新的数量：

```py title='新事件（src/allocation/domain/events.py）'
@dataclass
class BatchQuantityChanged(Event):
    ref: str
    qty: int
```

## <font color='red'>测试新的处理程序</font>

根据在[4.服务层](./g.Flask%20API%20and%20Service%20Layer.md)中学到的经验教训，我们可以“全速前进”，在事件方面以尽可能高的抽象级别编写单元测试。它们可能如下所示：

```py title='处理程序对 change_batch_quantity 的测试（tests/unit/test_handlers.py）'
class TestChangeBatchQuantity:
    def test_changes_available_quantity(self):
        uow = FakeUnitOfWork()
        messagebus.handle(
            events.BatchCreated("batch1", "ADORABLE-SETTEE", 100, None), uow
        )
        [batch] = uow.products.get(sku="ADORABLE-SETTEE").batches
        assert batch.available_quantity == 100  #(1)

        messagebus.handle(events.BatchQuantityChanged("batch1", 50), uow)

        assert batch.available_quantity == 50  #(1)

    def test_reallocates_if_necessary(self):
        uow = FakeUnitOfWork()
        event_history = [
            events.BatchCreated("batch1", "INDIFFERENT-TABLE", 50, None),
            events.BatchCreated("batch2", "INDIFFERENT-TABLE", 50, date.today()),
            events.AllocationRequired("order1", "INDIFFERENT-TABLE", 20),
            events.AllocationRequired("order2", "INDIFFERENT-TABLE", 20),
        ]
        for e in event_history:
            messagebus.handle(e, uow)
        [batch1, batch2] = uow.products.get(sku="INDIFFERENT-TABLE").batches
        assert batch1.available_quantity == 10
        assert batch2.available_quantity == 50

        messagebus.handle(events.BatchQuantityChanged("batch1", 25), uow)

        # order1 or order2 will be deallocated, so we'll have 25 - 20
        assert batch1.available_quantity == 5  #(2)
        # and 20 will be reallocated to the next batch
        assert batch2.available_quantity == 30  #(2)
```

1. 这个简单的情况很容易实现；我们只需修改一个数量。
2. 但是，如果我们尝试将数量更改为少于已分配的数量，我们将需要取消分配至少一个订单，并且我们希望将其重新分配给新的批次。

### <font color='red'>实现</font>

```py title='处理程序委托给模型层（src/allocation/service_layer/handlers.py）'
def change_batch_quantity(
    event: events.BatchQuantityChanged,
    uow: unit_of_work.AbstractUnitOfWork,
):
    with uow:
        product = uow.products.get_by_batchref(batchref=event.ref)
        product.change_batch_quantity(ref=event.ref, qty=event.qty)
        uow.commit()
```

我们意识到我们的存储库需要一种新的查询类型：

```py title='我们的存储库中的一种新查询类型（src/allocation/adapters/repository.py）'
class AbstractRepository(abc.ABC):
    ...

    def get(self, sku) -> model.Product:
        ...

    def get_by_batchref(self, batchref) -> model.Product:
        product = self._get_by_batchref(batchref)
        if product:
            self.seen.add(product)
        return product

    @abc.abstractmethod
    def _add(self, product: model.Product):
        raise NotImplementedError

    @abc.abstractmethod
    def _get(self, sku) -> model.Product:
        raise NotImplementedError

    @abc.abstractmethod
    def _get_by_batchref(self, batchref) -> model.Product:
        raise NotImplementedError
    ...

class SqlAlchemyRepository(AbstractRepository):
    ...

    def _get(self, sku):
        return self.session.query(model.Product).filter_by(sku=sku).first()

    def _get_by_batchref(self, batchref):
        return (
            self.session.query(model.Product)
            .join(model.Batch)
            .filter(orm.batches.c.reference == batchref)
            .first()
        )
```

在`FakeRepository`也是：

```py title='更新虚假 repo (tests/unit/test_handlers.py）'
class FakeRepository(repository.AbstractRepository):
    ...

    def _get(self, sku):
        return next((p for p in self._products if p.sku == sku), None)

    def _get_by_batchref(self, batchref):
        return next(
            (p for p in self._products for b in p.batches if b.reference == batchref),
            None,
        )
```

!!! note
    我们正在向存储库添加查询，以使此用例更易于实现。只要我们的查询返回单个聚合，我们就不会违反任何规则。如果您发现自己在存储库上编写了复杂的查询，则可能需要考虑不同的设计。特别是像`get_most_popular_products`或`find_products_by_order_id`这样的方法肯定会触发我们的第六感。
    [11.事件驱动架构：使用事件集成微服务]和[结语]有一些关于管理复杂查询的提示。


### <font color='red'>一种新的领域模型方法</font>

我们向模型添加了新方法，该方法以内联方式执行数量更改和释放并发布新事件。我们还修改了现有的分配函数以发布事件：

```py title='我们的模型不断发展以满足新的需求（src/allocation/domain/model.py）'
class Product:
    ...

    def change_batch_quantity(self, ref: str, qty: int):
        batch = next(b for b in self.batches if b.reference == ref)
        batch._purchased_quantity = qty
        while batch.available_quantity < 0:
            line = batch.deallocate_one()
            self.events.append(
                events.AllocationRequired(line.orderid, line.sku, line.qty)
            )
...

class Batch:
    ...

    def deallocate_one(self) -> OrderLine:
        return self._allocations.pop()
```

我们连接新的处理程序：

```py title='消息总线增长（src/allocation/service_layer/messagebus.py）'
HANDLERS = {
    events.BatchCreated: [handlers.add_batch],
    events.BatchQuantityChanged: [handlers.change_batch_quantity],
    events.AllocationRequired: [handlers.allocate],
    events.OutOfStock: [handlers.send_out_of_stock_notification],
}  # type: Dict[Type[events.Event], List[Callable]]
```

我们的新需求已全面实现。


## <font color='red'>可选：使用虚假消息总线对隔离事件处理程序进行单元测试</font>

我们对重新分配工作流的主要测试是端到端的（请参阅测试驱动新处理程序 中的示例代码）。它使用真实的消息总线，并测试整个流程，其中`BatchQuantityChanged`事件处理程序触发释放，并发出新`AllocationRequired`事件，而这些事件又由它们自己的处理程序处理。一个测试涵盖多个事件和处理程序的链。

根据事件链的复杂程度，您可能决定要单独测试一些处理程序。您可以使用“假”消息总线来执行此操作。

在我们的例子中，我们实际上是通过修改`publish_events()`方法`FakeUnitOfWork`并将其与真实消息总线分离来进行干预，而是让它记录它所看到的事件：

```py title='在 UoW 中实现的虚假消息总线 (tests/unit/test_handlers.py）'
class FakeUnitOfWorkWithFakeMessageBus(FakeUnitOfWork):
    def __init__(self):
        super().__init__()
        self.events_published = []  # type: List[events.Event]

    def publish_events(self):
        for product in self.products.seen:
            while product.events:
                self.events_published.append(product.events.pop(0))
```

现在，当我们使用`FakeUnitOfWorkWithFakeMessageBus`调用`messagebus.handle()`时，它只会运行该事件的处理程序。因此，我们可以编写一个更独立的单元测试：我们不必检查所有副作用，只需检查如果数量低于已分配的总数，`BatchQuantityChanged`是否会导致`AllocationRequired`：

```py title='单独测试重新分配（tests/unit/test_handlers.py）'
def test_reallocates_if_necessary_isolated():
    uow = FakeUnitOfWorkWithFakeMessageBus()

    # test setup as before
    event_history = [
        events.BatchCreated("batch1", "INDIFFERENT-TABLE", 50, None),
        events.BatchCreated("batch2", "INDIFFERENT-TABLE", 50, date.today()),
        events.AllocationRequired("order1", "INDIFFERENT-TABLE", 20),
        events.AllocationRequired("order2", "INDIFFERENT-TABLE", 20),
    ]
    for e in event_history:
        messagebus.handle(e, uow)
    [batch1, batch2] = uow.products.get(sku="INDIFFERENT-TABLE").batches
    assert batch1.available_quantity == 10
    assert batch2.available_quantity == 50

    messagebus.handle(events.BatchQuantityChanged("batch1", 25), uow)

    # assert on new events emitted rather than downstream side-effects
    [reallocation_event] = uow.events_published
    assert isinstance(reallocation_event, events.AllocationRequired)
    assert reallocation_event.orderid in {"order1", "order2"}
    assert reallocation_event.sku == "INDIFFERENT-TABLE"
```

是否要这样做取决于事件链的复杂性。我们建议从边到边测试开始，仅在必要时才诉诸此方法。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>读者练习</strong></p>
<p>强迫自己真正理解一些代码的一个好方法是重构它。在讨论单独测试处理程序时，我们使用了一种称为FakeUnitOfWorkWithFakeMessageBus的方法，它不必要地复杂，并且违反了 SRP。</p>
<p>如果我们将消息总线改为一个类，那么构建FakeMessageBus就更加简单了：</p>
<pre>
    <code style='border: 1px solid;'>
# 抽象消息总线及其真实版本和虚假版本
class AbstractMessageBus:
    HANDLERS: Dict[Type[events.Event], List[Callable]]

    def handle(self, event: events.Event):
        for handler in self.HANDLERS[type(event)]:
            handler(event)


class MessageBus(AbstractMessageBus):
    HANDLERS = {
        events.OutOfStock: [send_out_of_stock_notification],

    }


class FakeMessageBus(messagebus.AbstractMessageBus):
    def __init__(self):
        self.events_published = []  # type: List[events.Event]
        self.HANDLERS = {
            events.OutOfStock: [lambda e: self.events_published.append(e)]
        }
    </code>
</pre>
因此跳转到<a src='https://github.com/cosmicpython/code/tree/chapter_09_all_messagebus'>GitHub</a>上的代码 ，看看是否可以获得一个基于类的版本，然后编写一个<code>test_reallocates_if_necessary_isolated()</code>早期版本。

如果您需要更多灵感，我们在[13.依赖注入]中使用了基于类的消息总线。
</div>

## <font color='red'>总结</font>
让我们回顾一下我们所取得的成就，并思考我们为什么能够做到这一点。

### <font color='red'>我们取得了什么成就？</font>
事件是简单的数据类，用于定义系统中输入和内部消息的数据结构。从 DDD 的角度来看，这非常强大，因为事件通常可以很好地转换为业务语言（如果您还没有了解过，请查阅事件风暴）。

处理程序是我们对事件做出反应的方式。它们可以向下调用我们的模型或向外调用外部服务。如果愿意，我们可以为单个事件定义多个处理程序。处理程序还可以引发其他事件。这使我们能够非常细致地了解处理程序的作用并真正遵守 SRP。

### <font color='red'>我们为什么能取得这些成就？</font>
我们使用这些架构模式的持续目标是尝试让应用程序的复杂性增长速度低于其大小。当我们全力投入消息总线时，我们总是会付出架构复杂性的代价（请参阅表1），但我们获得的模式可以处理几乎任意复杂的需求，而无需对我们的工作方式进行任何概念或架构更改。

这里我们添加了一个相当复杂的用例（更改数量、取消分配、启动新交易、重新分配、发布外部通知），但从架构上看，复杂性并没有增加。我们添加了新事件、新处理程序和新外部适配器（用于电子邮件），所有这些都是我们架构中现有的事物类别，我们理解并知道如何推理，并且很容易向新手解释。我们的移动部件每个都有一项工作，它们以明确定义的方式相互连接，并且没有意外的副作用。

<font color='#186d7a'>表1.整个应用程序是一条消息总线：权衡</font>

|优点	|缺点|
|------|----|
|处理程序和服务是同一件事，因此更简单。|从 Web 的角度来看，消息总线仍然是一种有点难以预测的做事方式。你无法提前知道事情何时会结束。|
|我们有一个很好的数据结构来输入系统。|模型对象和事件之间会有字段和结构的重复，这会产生维护成本。向其中一个对象添加字段通常意味着向其他对象中的至少一个对象添加字段。|

现在，您可能想知道，这些`BatchQuantityChanged`事件将从何而来？答案将在几章后揭晓。但首先，让我们讨论一下[事件与命令](./n.Commands%20and%20Command%20Handler.md)。