# 11.事件驱动架构：使用事件集成微服务


在前面的章节中，我们从未真正谈论过如何接收“批量数量改变”事件，或者如何通知外界有关重新分配的信息。

我们有一个带有 Web API 的微服务，但是还有其他方式可以与其他系统通信吗？我们如何知道货物是否延误或数量是否修改？我们如何告诉仓库系统订单已分配并需要发送给客户？

在本章中，我们将展示如何扩展事件隐喻以涵盖我们处理来自系统的传入和传出消息的方式。在内部，我们的应用程序的核心现在是一个消息处理器。让我们继续进行下去，以便它在外部也成为一个消息处理器。如图1所示，我们的应用程序将通过外部消息总线从外部源接收事件（我们将使用 Redis 发布/订阅队列作为示例），并以事件的形式将其输出发布回那里。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_1101.png){ align=center }
    <figcaption><font size=2>图1.我们的应用程序是一个消息处理器</font></figcaption>
</figure>

!!! tip
    本章的代码位于[GitHub](https://github.com/cosmicpython/code/tree/chapter_11_external_events)上的chapter_11_external_events 分支中：
    ```
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout chapter_11_external_events
    # or to code along, checkout the previous chapter:
    git checkout chapter_10_commands
    ```

## <font color='red'>分布式泥球和名词思维</font>

在讨论这个问题之前，我们先来谈谈替代方案。我们经常与那些试图构建微服务架构的工程师交谈。他们通常从现有应用程序迁移，他们的第一反应是将系统拆分为名词。

到目前为止，我们在系统中引入了哪些名词？好吧，我们有库存、订单、产品和客户批次。因此，一种简单的拆分系统的方法可能看起来像图2（请注意，我们以名词`Batches`而不是`Allocation`来命名我们的系统）。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_1102.png){ align=center }
    <figcaption><font size=2>图2.基于名词的服务的上下文图</font></figcaption>
</figure>


我们系统中的每个“事物”都有一个相关的服务，它公开一个 HTTP API。

让我们通过图3的命令流示例来看一下：我们的用户访问一个网站，可以从有库存的产品中进行选择。当他们将商品添加到购物车时，我们会为他们保留一些库存。订单完成后，我们会确认保留，这会导致我们向仓库发送调度指令。假设这是客户的第三个订单，我们希望更新客户记录以将其标记为 VIP。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_1103.png){ align=center }
    <figcaption><font size=2>图3.命令流1</font></figcaption>
</figure>

我们可以将每个步骤视为系统中的一个命令：`ReserveStock`，`ConfirmReservation`，`DispatchGoods`，`MakeCustomerVIP` 等等。

这种架构风格，即我们为每个数据库表创建一个微服务，并将我们的 HTTP API 视为贫血模型的 CRUD 接口，是人们采用面向服务设计的最常见的初始方式。

这对于非常简单的系统来说很有效，但它很快就会退化为分布式的泥球。

为了了解原因，我们来考虑另一种情况。有时，当库存到达仓库时，我们发现物品在运输过程中被水损坏了。我们无法出售被水损坏的沙发，因此我们不得不将其扔掉并向合作伙伴请求更多库存。我们还需要更新库存模型，这可能意味着我们需要重新分配客户的订单。

这个逻辑到哪里去了？

好吧，仓库系统知道库存已经损坏，所以也许它应该拥有这个过程，如图4.命令流2所示。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_1104.png){ align=center }
    <figcaption><font size=2>图4.命令流2</font></figcaption>
</figure>

这种方式也行得通，但现在我们的依赖关系图很乱。为了分配库存，订单服务驱动批次系统，批次系统驱动仓库；但为了处理仓库的问题，我们的仓库系统驱动批次，批次系统驱动订单。

将此数字乘以我们需要提供的所有其他工作流程，您就会发现服务很快就会变得混乱不堪。


## <font color='red'>分布式系统中的错误处理</font>

“事情总会出错”是软件工程的普遍规律。当我们的某个请求失败时，我们的系统会发生什么？假设在我们接受用户 3 个`MISBEGOTTEN-RUG`订单后立即发生网络错误，如图5.带有错误的命令流所示 。

这里我们有两个选择：我们可以继续下订单，不分配，或者我们可以拒绝接受订单，因为无法保证分配。我们的批次服务的故障状态已经出现，并且正在影响我们订单服务的可靠性。

当两件事必须同时改变时，我们称它们为耦合。我们可以将这种故障级联视为一种时间耦合：系统的每个部分必须同时工作，系统的任何部分才能工作。随着系统变得越来越大，某个部分退化的概率呈指数级增长。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_1105.png){ align=center }
    <figcaption><font size=2>图5.带错误的命令流</font></figcaption>
</figure>

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>Connascence（共生性）</strong></p>
<p>我们在这里使用术语“耦合”，但还有另一种方法来描述我们系统之间的关系。Connascence是一些作者用来描述不同类型的耦合的术语。</p>
<p>Connascence并不坏，但有些类型的Connascence比其他类型的更强。当两个类紧密相关时，我们希望在局部有强Connascence，但关系紧密则希望有弱Connascence。</p>
<p>在我们的第一个分布式泥球例子中，我们看到了执行的Connascence：多个组件需要知道正确的工作顺序才能使操作成功。</p>
<p>当我们在这里考虑错误条件时，我们谈论的是时间的Connascence：为了使操作成功，必须接连发生多件事。</p>
<p>当我们用事件替换 RPC 风格的系统时，我们会用一种较弱的类型替换这两种类型的Connascence。这就是Connascence这个名称的由来：多个组件只需要就事件的名称及其携带的字段名称达成一致。</p>
<p>我们永远无法完全避免耦合，除非让我们的软件不与任何其他软件通信。我们想要的是避免不适当的耦合。Connascence提供了一个思维模型，用于理解不同架构风格固有的耦合强度和类型。在<a src='https://connascence.io/'>connascence.io</a>上阅读有关它的所有信息。</p>
</div>

## <font color='red'>替代方案：使用异步消息传递实现时间解耦</font>

我们如何获得适当的耦合？我们已经看到了部分答案，那就是我们应该用动词而不是名词来思考。我们的领域模型是关于业务流程建模的。它不是关于事物的静态数据模型；它是动词的模型。

因此，我们不会考虑订单系统和批次系统，而是考虑订购系统和分配系统等等。

当我们以这种方式分离事物时，更容易看出哪个系统应该负责什么。在考虑订购时，我们真的想确保当我们下订单时，订单就被下了。其他一切都可以稍后发生，只要它发生了。

!!! note
    如果这听起来很熟悉，那就对了！划分职责与我们在设计聚合和命令时经历的过程相同。

与聚合一样，微服务也应该是一致性边界。在两个服务之间，我们可以接受最终一致性，这意味着我们不需要依赖同步调用。每个服务都接受来自外部世界的命令并引发事件来记录结果。其他服务可以监听这些事件以触发工作流中的下一步。

为了避免“分布式泥球”反模式，我们希望使用异步消息传递来集成我们的系统，而不是时间耦合的 HTTP API 调用。我们希望我们的`BatchQuantityChanged`消息作为来自上游系统的外部消息传入，我们希望我们的系统发布`Allocated`事件以供下游系统监听。

为什么这样更好？首先，因为事情可以独立失败，所以处理降级行为更容易：如果分配系统出现问题，我们仍然可以接受订单。

其次，我们正在降低系统之间的耦合强度。如果我们需要更改操作顺序或在流程中引入新步骤，我们可以在本地进行。


## <font color='red'>使用 Redis 发布/订阅通道进行集成</font>

让我们看看它具体是如何工作的。我们需要某种方式将事件从一个系统转移到另一个系统，就像我们的消息总线一样，但对于服务来说。这部分基础设施通常称为消息代理。消息代理的作用是从发布者那里获取消息并将其传递给订阅者。

在 MADE.com，我们使用[Event Store](https://www.eventstore.com/)；`Kafka` 或 `RabbitMQ` 是有效的替代方案。基于 Redis [发布/订阅](https://redis.io/docs/latest/develop/interact/pubsub/)渠道的轻量级解决方案也可以很好地工作，而且由于 Redis 更为人们所熟悉，因此我们认为在本书中使用它。

!!! note
    我们忽略了选择合适的消息传递平台所涉及的复杂性。消息排序、故障处理和幂等性等问题都需要考虑。

我们的新流程将看起来像图6.重新分配流程的序列图：Redis 提供`BatchQuantityChanged`启动整个过程的事件，`Allocated`最后我们的事件再次发布回 Redis。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_1106.png){ align=center }
    <figcaption><font size=2>图6.重新分配流程的序列图</font></figcaption>
</figure>


## <font color='red'>使用端到端测试来测试一切</font>

以下是我们开始端到端测试的方法。我们可以使用现有的 API 创建批次，然后测试入站和出站消息：

```py title='我们的发布/订阅模型的端到端测试（tests/e2e/test_external_events.py）'
def test_change_batch_quantity_leading_to_reallocation():
    # start with two batches and an order allocated to one of them  #(1)
    orderid, sku = random_orderid(), random_sku()
    earlier_batch, later_batch = random_batchref("old"), random_batchref("newer")
    api_client.post_to_add_batch(earlier_batch, sku, qty=10, eta="2011-01-01")  #(2)
    api_client.post_to_add_batch(later_batch, sku, qty=10, eta="2011-01-02")
    response = api_client.post_to_allocate(orderid, sku, 10)  #(2)
    assert response.json()["batchref"] == earlier_batch

    subscription = redis_client.subscribe_to("line_allocated")  #(3)

    # change quantity on allocated batch so it's less than our order  #(1)
    redis_client.publish_message(  #(3)
        "change_batch_quantity",
        {"batchref": earlier_batch, "qty": 5},
    )

    # wait until we see a message saying the order has been reallocated  #(1)
    messages = []
    for attempt in Retrying(stop=stop_after_delay(3), reraise=True):  #(4)
        with attempt:
            message = subscription.get_message(timeout=1)
            if message:
                messages.append(message)
                print(messages)
            data = json.loads(messages[-1]["data"])
            assert data["orderid"] == orderid
            assert data["batchref"] == later_batch
```

1. 您可以从评论中了解此测试中发生的事情：我们想要向系统发送一个事件，导致订单行被重新分配，并且我们看到重新分配也作为`Redis` 中的事件出现。
2. `api_client`是我们重构后的在两种测试类型之间共享的小助手；它包装了我们对`requests.post`的调用。
3. `redis_client`是另一个小测试助手，其细节并不重要；它的作用是能够从各种 Redis 通道发送和接收消息。我们将使用一个名为`change_batch_quantity`的通道，发送我们的请求以更改批次的数量，我们将监听另一个名为`line_allocated`的通道以查找预期的重新分配。
4. 由于被测系统的异步特性，我们需要再次使用`tenacity`库来添加重试循环 - 首先，因为我们的`line_allocated`新消息可能需要一些时间才能到达，而且因为它不是该通道上的唯一消息。

### <font color='red'>Redis 是我们消息总线的另一个薄适配器</font>

我们的 Redis 发布/订阅监听器（我们称之为事件消费者）非常类似于 Flask：它将外部世界转化为我们的事件：

```py title='简单的 Redis 消息监听器 (src/allocation/entrypoints/redis_eventconsumer.py）'
r = redis.Redis(**config.get_redis_host_and_port())


def main():
    orm.start_mappers()
    pubsub = r.pubsub(ignore_subscribe_messages=True)
    pubsub.subscribe("change_batch_quantity")  #(1)

    for m in pubsub.listen():
        handle_change_batch_quantity(m)


def handle_change_batch_quantity(m):
    logging.debug("handling %s", m)
    data = json.loads(m["data"])  #(2)
    cmd = commands.ChangeBatchQuantity(ref=data["batchref"], qty=data["qty"])  #(2)
    messagebus.handle(cmd, uow=unit_of_work.SqlAlchemyUnitOfWork())
```

1. `main()`在运行时时订阅`change_batch_quantity`通道。。
2. 作为系统入口点，我们的主要工作是反序列化`JSON`，将其转换为`Command`，然后将其传递给服务层——就像 Flask 适配器所做的那样。

我们还构建了一个新的下游适配器来完成相反的工作——将域事件转换为公共事件：

```py title='简单的 Redis 消息发布者（src/allocation/adapters/redis_eventpublisher.py）'
r = redis.Redis(**config.get_redis_host_and_port())


def publish(channel, event: events.Event):  #(1)
    logging.debug("publishing: channel=%s, event=%s", channel, event)
    r.publish(channel, json.dumps(asdict(event)))
```

1. 我们在这里采用硬编码通道，但您也可以存储事件类/名称和相应通道之间的映射，从而允许一种或多种消息类型进入不同的通道。

#### <font color='red'>我们的新外出事件</font>

`Allocated`事件内容如下：

```py title='新事件（src/allocation/domain/events.py）'
@dataclass
class Allocated(Event):
    orderid: str
    sku: str
    qty: int
    batchref: str
```

它捕获了我们需要了解的有关分配的所有信息：订单行的详细信息，以及分配给哪个批次。

我们将其添加到我们的模型的`allocate()`方法中（自然，首先添加了一个测试）：

```py title='Product.allocate() 发出新事件来记录发生的事情（src/allocation/domain/model.py）'
class Product:
    ...
    def allocate(self, line: OrderLine) -> str:
        ...

            batch.allocate(line)
            self.version_number += 1
            self.events.append(
                events.Allocated(
                    orderid=line.orderid,
                    sku=line.sku,
                    qty=line.qty,
                    batchref=batch.reference,
                )
            )
            return batch.reference
```

`ChangeBatchQuantity`处理程序已经存在，因此我们需要添加的只是一个发布外出事件的处理程序：

```py title='消息总线增长（src/allocation/service_layer/messagebus.py）'
HANDLERS = {
    events.Allocated: [handlers.publish_allocated_event],
    events.OutOfStock: [handlers.send_out_of_stock_notification],
}  # type: Dict[Type[events.Event], List[Callable]]
发布事件使用 Redis 包装器的辅助函数：

发布到 Redis（src/allocation/service_layer/handlers.py）
def publish_allocated_event(
    event: events.Allocated,
    uow: unit_of_work.AbstractUnitOfWork,
):
    redis_eventpublisher.publish("line_allocated", event)
```


## <font color='red'>内部事件与外部事件</font>

最好将内部事件和外部事件区分开来。有些事件可能来自外部，有些事件可能会升级并发布到外部，但并非所有事件都会升级和发布到外部。如果你涉足[事件溯源](https://io.made.com/blog/2018-04-28-eventsourcing-101.html)（不过，这很可能是另一本书的主题），这一点尤其重要。

!!! tip
    出站事件是应用验证的重要场所之一。请参阅[附录E 验证](./w.Appendix%20E.md)了解一些验证理念和示例。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>读者练习</strong></p>
<p>本章的一个简单示例：使主要allocate()用例也可以通过 Redis 通道上的事件调用，以及（或代替）通过 API 调用。</p>
<p>您可能想要添加新的 E2E 测试并将一些更改反馈到 redis_eventconsumer.py中。</p>
</div>

## <font color='red'>总结</font>

事件可以来自外部，但也可以发布到外部——我们的publish处理程序将事件转换为 Redis 通道上的消息。我们使用事件与外界交流。这种时间解耦为我们的应用程序集成带来了很大的灵活性，但一如既往，这是有代价的。

!!! quote
    事件通知很不错，因为它意味着低耦合度，并且设置起来非常简单。但是，如果确实存在一个贯穿各种事件通知的逻辑流程，那么它可能会成为问题……很难看到这样的流程，因为它在任何程序文本中都不明确……这会使调试和修改变得困难。

    Martin Fowler，[“‘事件驱动’是什么意思”](https://martinfowler.com/articles/201701-event-driven.html)

表1显示了一些需要权衡的考量。

<font color='#186d7a'>表1.基于事件的微服务集成：权衡</font>

|优点|缺点|
|---|---|
|避免分散的大泥球。|整体的信息流动更加难以看清。|
|服务是解耦的：更改单个服务和添加新服务更加容易。|最终一致性是一个需要处理的新概念。|
|需要仔细考虑消息可靠性以及至少一次与最多一次传递的选择。|更一般地，如果你从同步消息模型转向异步消息模型，你还会遇到一系列与消息可靠性和最终一致性有关的问题。请继续阅读[footguns]。|