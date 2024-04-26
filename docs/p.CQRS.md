# 12.命令查询职责分离


在本章中，我们将从一个相当无争议的见解开始：读取（查询）和写入（命令）是不同的，因此应该区别对待它们（或者将它们的职责分开，如果你愿意的话）。然后我们将尽可能地推进这一见解。

如果你和哈利一样，这一切一开始会显得很极端，但希望我们可以证明这并非完全不合理。

图1.将读取与写入分开可以显示出我们最终的结果。

!!! tip
    本章的代码位于[GitHub](https://github.com/cosmicpython/code/tree/chapter_12_cqrs)上的chapter_12_cqrs 分支中。
    ```
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout chapter_12_cqrs
    # or to code along, checkout the previous chapter:
    git checkout chapter_11_external_events
    ```

首先，为什么要这么麻烦呢？

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_1201.png){ align=center }
    <figcaption><font size=2>图1.读取与写入分离</font></figcaption>
</figure>


## <font color='red'>领域模型是用来写入的</font>

我们在本书中花了很多时间讨论如何构建执行我们领域规则的软件。这些规则或约束对于每个应用程序来说都是不同的，它们构成了我们系统的核心。

在本书中，我们设置了明确的约束，例如“您不能分配超过可用库存的数量”，以及隐含的约束，例如“每个订单行分配给单个批次”。

我们在书的开头将这些规则作为单元测试写下来：

```py title='我们的基本领域测试（tests/unit/test_batches.py）'
def test_allocating_to_a_batch_reduces_the_available_quantity():
    batch = Batch("batch-001", "SMALL-TABLE", qty=20, eta=date.today())
    line = OrderLine("order-ref", "SMALL-TABLE", 2)

    batch.allocate(line)

    assert batch.available_quantity == 18

...

def test_cannot_allocate_if_available_smaller_than_required():
    small_batch, large_line = make_batch_and_line("ELEGANT-LAMP", 2, 20)
    assert small_batch.can_allocate(large_line) is False
```

为了正确应用这些规则，我们需要确保操作的一致性，因此我们引入了工作单元和聚合等模式 来帮助我们提交小块工作。

为了传达这些小块之间的变化，我们引入了域事件模式，以便我们可以编写规则，例如“当库存损坏或丢失时，调整批次上的可用数量，并在必要时重新分配订单”。

所有这些复杂性的存在，都是为了让我们能够在改变系统状态时强制执行规则。我们构建了一套灵活的工具来写入数据。

那么读取呢？


## <font color='red'>大多数用户不会购买你的家具</font>

在 MADE.com，我们有一个非常类似于分配服务的系统。在繁忙的一天，我们可能在一小时内处理一百个订单，并且我们有一个庞大而复杂的系统来为这些订单分配库存。

然而，在同样繁忙的日子里，我们每秒可能会有 100 次产品浏览。每次有人访问产品页面或产品列表页面时，我们都需要确定该产品是否仍有库存以及需要多长时间才能发货。

领域是相同的——我们关注的是库存批次、到货日期以及剩余库存量——但访问模式却大不相同。例如，如果查询晚了几秒钟，我们的客户不会注意到，但如果我们的分配服务不一致，我们就会搞乱他们的订单。我们可以利用这种差异，使我们的读取最终保持一致，从而提高性能。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>读一致性真的可以实现吗？</strong></p>
<p>这种用一致性和性能进行权衡的想法一开始让很多开发人员感到紧张，所以我们来快速讨论一下这个问题。</p>

<p>假设当 Bob 访问`ASYMMETRICAL-DRESSER`页面时，我们的“获取可用库存”查询已过期 30 秒。但与此同时，Harry 已经购买了最后一件商品。当我们尝试分配 Bob 的订单时，我们会失败，我们需要取消他的订单或购买更多库存并延迟交货。</p>
<p>只使用关系数据存储的人会对这个问题感到非常紧张，但值得考虑另外两种情况以获得一些看法。</p>
<p>首先，让我们假设 Bob 和 Harry 同时访问该页面。Harry出去煮咖啡，等他回来时，Bob 已经买下了最后一个梳妆台。当 Harry 下订单时，我们会将其发送给分配服务，由于库存不足，我们不得不退还他的付款或购买更多库存并延迟他的交货。</p>
<p>一旦我们呈现产品页面，数据就已经过时了。这一见解是理解为什么读取可能会出现安全不一致的关键：我们在分配时始终需要检查系统的当前状态，因为所有分布式系统都是不一致的。只要您有 Web 服务器和两个客户，就有可能出现过时的数据。</p>
<p>好吧，让我们假设我们以某种方式解决了这个问题：我们奇迹般地构建了一个完全一致的 Web 应用程序，其中没有人会看到过时的数据。这次 Harry 先进入页面并购买了他的梳妆台。</p>
<p>不幸的是，当仓库工作人员试图处理他的家具时，他的家具从叉车上掉下来，摔成了无数碎片。现在该怎么办？</p>
<p>唯一的选择是要么打电话给哈利并退还订单，要么购买更多库存并延迟交货。</p>
<p>无论我们做什么，我们总会发现我们的软件系统与现实不一致，因此我们总是需要业务流程来应对这些极端情况。在读取方面，牺牲性能换取一致性是可以的，因为陈旧数据本质上是不可避免的。</p>
</div>

我们可以将这些要求视为系统的两个半部分：读取端和写入端，如“表1.读取与写入”中所示。

对于写入方面，我们花哨的领域架构模式有助于我们随着时间的推移发展我们的系统，但迄今为止我们构建的复杂性对于读取数据没有任何帮助。服务层、工作单元和巧妙的领域模型只是臃肿而已。

<font color='#186d7a'>表1.读取与写入</font>

||读端|写端|
|-|-|-|
|行为|简单读|复杂的业务逻辑|
|可缓存性|高度可缓存|不可缓存|
|一致性|可能有旧数据|必须事务一致|


## <font color='red'>Post/Redirect/Get 和 CQS</font>

如果您从事 Web 开发，您可能对 `Post/Redirect/Get` 模式很熟悉。在这种技术中，Web 端点接受 HTTP POST 并以重定向进行响应以查看结果。例如，我们可能 POST 到/batches以创建新批次，并将用户重定向到 /batches/123 以查看他们新创建的批次。

这种方法解决了用户在浏览器中刷新结果页面或尝试将结果页面加入书签时出现的问题。如果是刷新，则可能导致我们的用户重复提交数据，从而购买两张沙发（其实他们只需要一张）。如果是书签，我们倒霉的客户在尝试 GET 一个 POST 端点时最终会得到一个损坏的页面。

这两个问题都是因为我们在响应写入操作时返回数据。`Post/Redirect/Get` 通过分离操作的读取和写入阶段来避免这个问题。

此技术是命令查询分离 (CQS) 的一个简单示例。我们遵循一条简单规则：函数应该修改状态或回答问题，但不能同时修改和回答问题。这使得软件更容易推理：我们应该始终能够问“灯亮了吗？”，而无需轻按电灯开关。

!!! note
    在构建 API 时，我们可以应用相同的设计技术，返回 201 Created 或 202 Accepted，其中 Location 标头包含新资源的 URI。这里重要的不是我们使用的状态代码，而是将工作逻辑地分为写入阶段和查询阶段。

如您所见，我们可以使用 CQS 原则使我们的系统更快、更具可扩展性，但首先，让我们修复现有代码中的 CQS 违规。很久以前，我们引入了一个`allocate`端点，它接受订单并调用我们的服务层来分配一些库存。在调用结束时，我们返回 200 OK 和批次 ID。这导致了一些丑陋的设计缺陷，以便我们可以获取所需的数据。让我们将其更改为返回一个简单的 OK 消息，并提供一个新的只读端点来检索分配状态：

```py title='API 测试在 POST 之后执行 GET（tests/e2e/test_api.py）'
@pytest.mark.usefixtures("postgres_db")
@pytest.mark.usefixtures("restart_api")
def test_happy_path_returns_202_and_batch_is_allocated():
    orderid = random_orderid()
    sku, othersku = random_sku(), random_sku("other")
    earlybatch = random_batchref(1)
    laterbatch = random_batchref(2)
    otherbatch = random_batchref(3)
    api_client.post_to_add_batch(laterbatch, sku, 100, "2011-01-02")
    api_client.post_to_add_batch(earlybatch, sku, 100, "2011-01-01")
    api_client.post_to_add_batch(otherbatch, othersku, 100, None)

    r = api_client.post_to_allocate(orderid, sku, qty=3)
    assert r.status_code == 202

    r = api_client.get_allocation(orderid)
    assert r.ok
    assert r.json() == [
        {"sku": sku, "batchref": earlybatch},
    ]


@pytest.mark.usefixtures("postgres_db")
@pytest.mark.usefixtures("restart_api")
def test_unhappy_path_returns_400_and_error_message():
    unknown_sku, orderid = random_sku(), random_orderid()
    r = api_client.post_to_allocate(
        orderid, unknown_sku, qty=20, expect_success=False
    )
    assert r.status_code == 400
    assert r.json()["message"] == f"Invalid sku {unknown_sku}"

    r = api_client.get_allocation(orderid)
    assert r.status_code == 404
```

好的，Flask 应用程序看起来是什么样子的？

```py title='查看分配的端点（src/allocation/entrypoints/flask_app.py）'
from allocation import views
...

@app.route("/allocations/<orderid>", methods=["GET"])
def allocations_view_endpoint(orderid):
    uow = unit_of_work.SqlAlchemyUnitOfWork()
    result = views.allocations(orderid, uow)  #(1)
    if not result:
        return "not found", 404
    return jsonify(result), 200
```

1. 好吧，一个views.py，足够公平；我们可以在里面保存只读的内容，它将是一个真正的views.py，不像 Django 的，它知道如何构建数据的只读视图……


## <font color='red'>留着你们的午餐吧</font>

嗯，所以我们可能只需向我们现有的存储库对象添加一个列表方法：

```py title='视图是否是原始 SQL？（src/allocation/views.py）'
from allocation.service_layer import unit_of_work


def allocations(orderid: str, uow: unit_of_work.SqlAlchemyUnitOfWork):
    with uow:
        results = uow.session.execute(
            """
            SELECT ol.sku, b.reference
            FROM allocations AS a
            JOIN batches AS b ON a.batch_id = b.id
            JOIN order_lines AS ol ON a.orderline_id = ol.id
            WHERE ol.orderid = :orderid
            """,
            dict(orderid=orderid),
        )
    return [{"sku": sku, "batchref": batchref} for sku, batchref in results]
```

打扰一下？原始 SQL？

如果你和 Harry 一样第一次遇到这种模式，你会想知道 Bob 到底在干什么。我们现在要手动编写自己的 SQL，并将数据库行直接转换为字典？在我们付出所有努力构建一个好的领域模型之后？那么 Repository 模式呢？这难道不是我们对数据库的抽象吗？我们为什么不重用它呢？

好吧，让我们首先探索一下这个看似更简单的替代方案，看看它在实践中是什么样子。

我们仍将视图保留在单独的views.py模块中；在应用程序中强制明确区分读取和写入仍然是一个好主意。我们应用命令查询分离，很容易看出哪些代码修改状态（事件处理程序），哪些代码仅检索只读状态（视图）。

!!! tip
    即使您不想使用全面的 CQRS，将只读视图从状态修改命令和事件处理程序中分离出来可能也是一个好主意。


## <font color='red'>测试 CQRS 视图</font>

在我们开始探索各种选项之前，让我们先谈谈测试。无论您决定采用哪种方法，您都可能需要至少一次集成测试。如下所示：

```py title='视图的集成测试（tests/integration/test_views.py）'
def test_allocations_view(sqlite_session_factory):
    uow = unit_of_work.SqlAlchemyUnitOfWork(sqlite_session_factory)
    messagebus.handle(commands.CreateBatch("sku1batch", "sku1", 50, None), uow)  #(1)
    messagebus.handle(commands.CreateBatch("sku2batch", "sku2", 50, today), uow)
    messagebus.handle(commands.Allocate("order1", "sku1", 20), uow)
    messagebus.handle(commands.Allocate("order1", "sku2", 20), uow)
    # add a spurious batch and order to make sure we're getting the right ones
    messagebus.handle(commands.CreateBatch("sku1batch-later", "sku1", 50, today), uow)
    messagebus.handle(commands.Allocate("otherorder", "sku1", 30), uow)
    messagebus.handle(commands.Allocate("otherorder", "sku2", 10), uow)

    assert views.allocations("order1", uow) == [
        {"sku": "sku1", "batchref": "sku1batch"},
        {"sku": "sku2", "batchref": "sku2batch"},
    ]
```

1. 我们使用应用程序的公共入口点（消息总线）进行集成测试的设置。这使我们的测试与有关如何存储内容的任何实现/基础架构细节脱钩。


## <font color='red'>“显而易见”的替代方案 1：使用现有存储库</font>

如何向我们的`products`存储库添加一个辅助方法？

```py title='使用存储库的简单视图（src/allocation/views.py）'
from allocation import unit_of_work

def allocations(orderid: str, uow: unit_of_work.AbstractUnitOfWork):
    with uow:
        products = uow.products.for_order(orderid=orderid)  #(1)
        batches = [b for p in products for b in p.batches]  #(2)
        return [
            {'sku': b.sku, 'batchref': b.reference}
            for b in batches
            if orderid in b.orderids  #(3)
        ]
```

1. 我们的存储库返回`Product`对象，我们需要按给定的顺序找到 相关SKU 的所有产品，因此我们将在存储库上构建一个新调用的辅助方法`.for_order()`。
2. 现在我们有产品，但我们实际上想要批次引用，因此我们通过列表推导获得所有可能的批次。
3. 我们再次进行过滤，只获取特定订单的批次。这又依赖于我们的Batch对象能够告诉我们它分配了哪些订单 ID。

我们最后使用一个`.orderid`属性来实现：

```py title='我们的模型上可能存在不必要的属性（src/allocation/domain/model.py）'
class Batch:
    ...

    @property
    def orderids(self):
        return {l.orderid for l in self._allocations}
```

您可以开始看到，重用我们现有的存储库和域模型类并不像您想象的那么简单。我们必须为两者添加新的辅助方法，并且我们正在用 Python 进行大量循环和过滤，而这些工作如果由数据库来做会更有效率。

所以是的，从好的方面来说我们重用了现有的抽象，但从坏的方面来说，这一切都感觉很笨重。


## <font color='red'>你的域模型没有针对读取操作进行优化</font>

我们在这里看到的是拥有主要为写操作设计的领域模型的效果，而我们对读取的要求在概念上往往完全不同。

这就是架构师对 CQRS 的辩解。正如我们之前所说，领域模型不是数据模型——我们试图捕捉业务的工作方式：工作流、围绕状态变化的规则、交换的消息；对系统如何响应外部事件和用户输入的关注。 这些东西中的大多数与只读操作完全无关。

!!! tip
    CQRS 的这种理由与域模型模式的理由相关。如果您正在构建一个简单的 CRUD 应用程序，读取和写入将紧密相关，因此您不需要域模型或 CQRS。但是您的域越复杂，您就越有可能需要两者。

简单来说，您的域类将具有多种修改状态的方法，并且您不需要任何一种方法来进行只读操作。

随着领域模型的复杂性不断增长，您会发现自己在如何构建该模型方面做出越来越多的选择，这使得其在读取操作中的使用变得越来越尴尬。


## <font color='red'>“显而易见”的替代方案 2：使用 ORM</font>

你可能会想，好吧，如果我们的存储库很笨重，使用`Products`也很笨重，那么我至少可以使用我的 ORM 并使用Batches。这就是它的用途！

```py title='使用 ORM 的简单视图 (src/allocation/views.py）'
from allocation import unit_of_work, model

def allocations(orderid: str, uow: unit_of_work.AbstractUnitOfWork):
    with uow:
        batches = uow.session.query(model.Batch).join(
            model.OrderLine, model.Batch._allocations
        ).filter(
            model.OrderLine.orderid == orderid
        )
        return [
            {"sku": b.sku, "batchref": b.batchref}
            for b in batches
        ]
```

但是，这真的比 留着你们的午餐吧 中代码示例中的原始 SQL 版本更容易编写或理解吗？它看起来可能还不错，但我们可以告诉你，它经过了多次尝试，并且大量挖掘了 SQLAlchemy 文档。SQL 就是 SQL。

但是 ORM 也可能使我们面临性能问题。


## <font color='red'>SELECT N+1 和其他性能注意事项</font>

所谓的`SELECT N+1`问题是 ORM 的一个常见性能问题：检索对象列表时，ORM 通常会执行初始查询，例如获取所需对象的所有 ID，然后对每个对象发出单独的查询以检索其属性。如果对象之间存在任何外键关系，则尤其可能出现这种情况。

!!! note
    平心而论，我们应该说 SQLAlchemy 在避免这个`SELECT N+1`问题方面做得相当好。它没有在上例中显示它，并且您可以 在处理连接对象时明确 请求预先加载以避免它。

除`SELECT N+1`之外，您可能还有其他原因想要将状态更改的持久化方式与检索当前状态的方式分离开来。一组完全规范化的关系表是确保写入操作不会导致数据损坏的好方法。但使用大量连接检索数据可能会很慢。在这种情况下，通常会添加一些非规范化视图、构建只读副本，甚至添加缓存层。


## <font color='red'>是时候拉回来了</font>

就此而言：我们是否已经说服您，我们的原始 SQL 版本并不像乍一看那么奇怪？也许我们是为了效果而夸大其词？请您稍等。

所以，不管合理与否，那个硬编码的 SQL 查询非常丑陋，对吧？如果我们让它变得更美观会怎样？

```py title='一个更好的查询（src/allocation/views.py）'
def allocations(orderid: str, uow: unit_of_work.SqlAlchemyUnitOfWork):
    with uow:
        results = uow.session.execute(
            """
            SELECT sku, batchref FROM allocations_view WHERE orderid = :orderid
            """,
            dict(orderid=orderid),
        )
        ...
```

...通过为我们的视图模型保留一个完全独立的、非规范化的数据存储？

```py title='嘻嘻嘻，没有外键，只有字符串，YOLO（src/allocation/adapters/orm.py）'
allocations_view = Table(
    "allocations_view",
    metadata,
    Column("orderid", String(255)),
    Column("sku", String(255)),
    Column("batchref", String(255)),
)
```

好的，更好看的 SQL 查询实际上并不能证明任何事情，但是一旦达到索引所能执行的操作的极限，构建针对读取操作进行优化的非规范化数据副本并不罕见。

即使索引经过精心调优，关系数据库也会使用大量 CPU 来执行连接。最快的查询永远是`SELECT * from mytable WHERE key = :value`

不过，这种方法不仅能提高速度，还能让我们获得规模。当我们将数据写入关系数据库时，我们需要确保锁定正在更改的行，这样我们就不会遇到一致性问题。

如果多个客户端同时更改数据，就会出现奇怪的竞争条件。但是，当我们读取数据时，可以同时执行的客户端数量没有限制。因此，只读存储可以水平扩展。

!!! tip
    由于读取副本可能不一致，因此我们可以拥有的数量没有限制。如果您难以扩展具有复杂数据存储的系统，请询问是否可以构建更简单的读取模型。

保持读取模型最新是一项挑战！数据库视图（物化或其他）和触发器是一种常见的解决方案，但这会将您限制在数据库中。我们想向您展示如何重用我们的事件驱动架构。

### <font color='red'>使用事件处理程序更新读取模型表</font>

我们为该`Allocated`事件添加第二个处理程序：

```py title='分配的事件获得一个新的处理程序（src/allocation/service_layer/messagebus.py）'
EVENT_HANDLERS = {
    events.Allocated: [
        handlers.publish_allocated_event,
        handlers.add_allocation_to_read_model,
    ],
```

我们的更新视图模型代码如下：

```py title='分配更新（src/allocation/service_layer/handlers.py）'

def add_allocation_to_read_model(
    event: events.Allocated,
    uow: unit_of_work.SqlAlchemyUnitOfWork,
):
    with uow:
        uow.session.execute(
            """
            INSERT INTO allocations_view (orderid, sku, batchref)
            VALUES (:orderid, :sku, :batchref)
            """,
            dict(orderid=event.orderid, sku=event.sku, batchref=event.batchref),
        )
        uow.commit()
```

不管你信不信，这确实有效！ 而且它与我们其他选项一样，可以进行完全相同的集成测试。

好的，您还需要处理`Deallocated`：

```py title='用于读取模型更新的第二个监听器'
events.Deallocated: [
    handlers.remove_allocation_from_read_model,
    handlers.reallocate
],

...

def remove_allocation_from_read_model(
    event: events.Deallocated,
    uow: unit_of_work.SqlAlchemyUnitOfWork,
):
    with uow:
        uow.session.execute(
            """
            DELETE FROM allocations_view
            WHERE orderid = :orderid AND sku = :sku
            ...
```

图2.读取模型的序列图 显示了两个请求之间的流程。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_1202.png){ align=center }
    <figcaption><font size=2>图2.读取模型的序列图</font></figcaption>
</figure>

在读取模型的序列图中，您可以看到 POST/写入操作中的两个事务，一个用于更新写入模型，一个用于更新读取模型，GET/读取操作可以使用它们。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>从头开始重建</strong></p>
<p>“当它坏了会发生什么？”应该是我们作为工程师问的第一个问题。</p>
<p>我们该如何处理由于错误或临时中断而未更新的视图模型？好吧，这只是事件和命令可能独立失败的另一种情况。</p>
<p>如果我们从未更新视图模型，并且ASYMMETRICAL-DRESSER永远有库存，这会让客户感到烦恼，但allocate服务仍然会失败，我们会采取行动来解决问题。</p>
<p>不过，重建视图模型很容易。由于我们使用服务层来更新视图模型，因此我们可以编写一个执行以下操作的工具：</p>
<ul>
    <li>查询写入端的当前状态，以确定当前分配的内容</li>
    <li>为每个分配的项目调用<code>add_allocation_to_read_model</code>处理程序</li>
</ul>
<p>我们可以使用这种技术从历史数据中创建全新的读取模型。</p>
</div>


## <font color='red'>更改我们的读取模型实现很容易</font>

让我们看看事件驱动模型在实际应用中给我们带来的灵活性，看看如果我们决定使用完全独立的存储引擎 Redis 来实现读取模型会发生什么。

只是看：

```py title='处理程序更新 Redis 读取模型 (src/allocation/service_layer/handlers.py）'
def add_allocation_to_read_model(event: events.Allocated, _):
    redis_eventpublisher.update_readmodel(event.orderid, event.sku, event.batchref)


def remove_allocation_from_read_model(event: events.Deallocated, _):
    redis_eventpublisher.update_readmodel(event.orderid, event.sku, None)
```

我们的 Redis 模块中的助手是一行代码：

```py title='Redis 读取模型读取和更新（src/allocation/adapters/redis_eventpublisher.py）'
def update_readmodel(orderid, sku, batchref):
    r.hset(orderid, sku, batchref)


def get_readmodel(orderid):
    return r.hgetall(orderid)
```

（也许redis_eventpublisher.py这个名字现在用词不当，但你明白我的意思。）

并且视图本身也发生了轻微的变化以适应新的后端：

```py title='适用于 Redis 的视图 (src/allocation/views.py）'
def allocations(orderid: str):
    batches = redis_eventpublisher.get_readmodel(orderid)
    return [
        {"batchref": b.decode(), "sku": s.decode()}
        for s, b in batches.items()
    ]
```

并且我们之前进行的完全相同的集成测试仍然可以通过，因为它们是在与实现分离的抽象级别上编写的：设置将消息放在消息总线上，并且断言与我们的观点相反。

!!! tip
    如果您决定需要更新读取模型，事件处理程序是管理更新的绝佳方式。它们还让您以后可以轻松更改该读取模型的实现。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>读者练习</strong></p>
<p>实现另一个视图，这次显示单个订单行的分配。</p>
<p>在这里，使用硬编码 SQL 与通过存储库之间的权衡应该更加模糊。尝试几个版本（可能包括使用 Redis），看看您更喜欢哪个。</p>
</div>


## 总结

表2.各种视图模型选项的权衡 表明每种选项都有其优点和缺点。

事实上，MADE.com 的分配服务确实使用了“全面”的 CQRS，其读取模型存储在 Redis 中，甚至还有 Varnish 提供的第二层缓存。但它的用例与我们在此处展示的有很大不同。对于我们正在构建的分配服务类型，您似乎不太可能需要使用单独的读取模型和事件处理程序来更新它。

但是随着您的领域模型变得越来越丰富和复杂，简化的读取模型变得越来越引人注目。


<font color='#186d7a'>表2.各种视图模型选项的权衡</font>

|选项|优点|缺点|
|只需使用存储库|简单、一致的方法。|预计复杂查询模式会出现性能问题。|
|在 ORM 中使用自定义查询|允许重用数据库配置和模型定义。|添加另一种具有其自身特点和语法的查询语言。|
|使用手动 SQL 查询普通模型表|通过标准查询语法提供对性能的精细控制。|必须对手动查询和ORM 定义进行 DB 架构更改。高度规范化的架构可能仍存在性能限制。|
|将一些额外的（非规范化）表作为读取模型添加到数据库中|非规范化表的查询速度要快得多。如果我们在同一个事务中更新规范化和非规范化的表，我们仍然可以很好地保证数据一致性|这会稍微减慢写入速度|
|使用事件创建单独的读取存储|只读副本易于扩展。当数据发生变化时可以构建视图，以便查询尽可能简单。|复杂的技术。哈利会永远怀疑你的品味和动机。|

通常，您的读取操作将作用于与您的写入模型相同的概念对象，因此使用 ORM、向您的存储库添加一些读取方法以及使用域模型类进行读取操作就可以了。

在我们的书中示例中，读取操作作用于与我们的域模型截然不同的概念实体。分配服务考虑的是单个 SKU`Batches`，但用户关心的是整个订单的分配，有多个 SKU，因此使用 ORM 最终会有点尴尬。我们很想采用我们在本章开头展示的原始 SQL 视图。

就此而言，让我们进入最后一章。
