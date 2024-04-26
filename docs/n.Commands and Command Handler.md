# 10.命令和命令处理程序


在上一章中，我们讨论了使用事件作为表示系统输入的一种方式，并将应用程序变成了消息处理机器。

为了实现这一点，我们将所有用例函数转换为事件处理程序。当 API 收到创建新批次的 POST 时，它会构建一个新`BatchCreated`事件并将其处理为内部事件。这可能感觉违反直觉。毕竟，批次尚未创建；这就是我们调用 API 的原因。我们将通过引入命令并展示如何通过相同的消息总线处理它们但规则略有不同来解决这个概念上的缺陷。

!!! tip
    本章的代码位于[GitHub](https://github.com/cosmicpython/code/tree/chapter_10_commands)上的chapter_10_commands 分支中：
    ```
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout chapter_10_commands
    # or to code along, checkout the previous chapter:
    git checkout chapter_09_all_messagebus
    ```

## <font color='red'>命令和事件</font>

与事件一样，命令也是一种消息，即系统的一部分向另一部分发送的指令。我们通常用哑数据结构来表示命令，并且可以像处理事件一样处理它们。

然而，命令和事件之间的差异很重要。

命令由一个参与者发送给另一个特定的参与者，期望结果会发生特定的事情。当我们将表单发布到 API 处理程序时，我们正在发送命令。我们用命令式动词短语命名命令，例如“分配库存”或“延迟发货”。

命令捕获意图。它们表达了我们希望系统做某事的愿望。因此，当它们失败时，发送者需要接收错误信息。

事件由参与者向所有感兴趣的听众广播。当我们发布时 BatchQuantityChanged，我们不知道谁会接收它。我们用过去时动词短语命名事件，如“订单分配到库存”或“发货延迟”。

我们经常使用事件来传播有关成功命令的知识。

事件记录过去发生的事情。由于我们不知道谁在处理事件，因此发送者不应该关心接收者是成功还是失败。表1重述了两者的区别。

<font color='#186d7a'>表1.事件与命令</font>

||事件|命令|
|-|-|-|
|命名|过去式|命令式|
|错误处理|独立|吵闹|
|发送给|所有监听者|接收者|

我们的系统中现在有哪些类型的命令？

```py title='提取一些命令（src/allocation/domain/commands.py）'
class Command:
    pass


@dataclass
class Allocate(Command):  #(1)
    orderid: str
    sku: str
    qty: int


@dataclass
class CreateBatch(Command):  #(2)
    ref: str
    sku: str
    qty: int
    eta: Optional[date] = None


@dataclass
class ChangeBatchQuantity(Command):  #(3)
    ref: str
    qty: int
```

1. `commands.Allocate`将取代`events.AllocationRequired`。
2. `commands.CreateBatch`将取代`events.BatchCreated`。
3. `commands.ChangeBatchQuantity`将取代`events.BatchQuantityChanged`。


## <font color='red'>异常处理的差异</font>

仅更改名称和动词就很好了，但这不会改变我们系统的行为。我们希望以类似的方式处理事件和命令，但不完全相同。让我们看看我们的消息总线如何变化：

```py title='以不同方式调度事件和命令 (src/allocation/service_layer/messagebus.py）'
Message = Union[commands.Command, events.Event]


def handle(  #(1)
    message: Message,
    uow: unit_of_work.AbstractUnitOfWork,
):
    results = []
    queue = [message]
    while queue:
        message = queue.pop(0)
        if isinstance(message, events.Event):
            handle_event(message, queue, uow)  #(2)
        elif isinstance(message, commands.Command):
            cmd_result = handle_command(message, queue, uow)  #(2)
            results.append(cmd_result)
        else:
            raise Exception(f"{message} was not an Event or Command")
    return results
```

1. 它仍然有一个主handle()入口点，它接受一个message，可能是命令或事件。
2. 我们将事件和命令发送给两个不同的辅助函数，如下所示。

我们处理事件的方式如下：

```py title='事件不能中断流程（src/allocation/service_layer/messagebus.py）'
def handle_event(
    event: events.Event,
    queue: List[Message],
    uow: unit_of_work.AbstractUnitOfWork,
):
    for handler in EVENT_HANDLERS[type(event)]:  #(1)
        try:
            logger.debug("handling event %s with handler %s", event, handler)
            handler(event, uow=uow)
            queue.extend(uow.collect_new_events())
        except Exception:
            logger.exception("Exception handling event %s", event)
            continue  #(2)
```

1. 事件发送到调度程序，该调度程序可以将每个事件委托给多个处理程序。
2. 它捕获并记录错误，但不会让它们中断消息处理。

以下是我们执行命令的方式：

```py title='命令重新引发异常（src/allocation/service_layer/messagebus.py）'
def handle_command(
    command: commands.Command,
    queue: List[Message],
    uow: unit_of_work.AbstractUnitOfWork,
):
    logger.debug("handling command %s", command)
    try:
        handler = COMMAND_HANDLERS[type(command)]  #(1)
        result = handler(command, uow=uow)
        queue.extend(uow.collect_new_events())
        return result  #(3)
    except Exception:
        logger.exception("Exception handling command %s", command)
        raise  #(2)
```

1. 命令调度程序期望每个命令只有一个处理程序。
2. 如果出现任何错误，它们会快速失败并且会冒泡。

`return result`只是暂时的；这是一个临时的`hack`，允许消息总线返回批处理引用以供 API 使用。我们将在[12.CQRS](./p.CQRS.md)中修复此问题。

我们还将单个`HANDLERS`字典更改为用于命令和事件的不同字典。根据我们的惯例，命令只能有一个处理程序：

```py title='新的处理程序字典（src/allocation/service_layer/messagebus.py）'
EVENT_HANDLERS = {
    events.OutOfStock: [handlers.send_out_of_stock_notification],
}  # type: Dict[Type[events.Event], List[Callable]]

COMMAND_HANDLERS = {
    commands.Allocate: handlers.allocate,
    commands.CreateBatch: handlers.add_batch,
    commands.ChangeBatchQuantity: handlers.change_batch_quantity,
}  # type: Dict[Type[commands.Command], Callable]
```


## <font color='red'>讨论：事件、命令和错误处理</font>

许多开发人员在这一点上感到不安，并询问：“如果事件无法处理会发生什么？我该如何确保系统处于一致状态？”如果我们在内存不足的错误终止进程之前，`messagebus.handle`处理了一半的事件，我们如何减轻丢失消息造成的问题？

让我们从最坏的情况开始：我们无法处理事件，系统处于不一致状态。什么样的错误会导致这种情况？在我们的系统中，通常当只有一半操作完成时，我们就会处于不一致状态。

例如，我们可以将三个单位的 `DESIRABLE_BEANBAG` 分配给客户的订单，但不知何故未能减少剩余库存量。这会导致不一致的状态：这三单位的库存既已分配又可用，具体取决于您如何看待它。稍后，我们可能会将这些订单分配给另一个客户，这给客户支持带来了麻烦。

不过，在我们的分配服务中，我们已经采取措施防止这种情况发生。我们仔细识别了充当一致性边界的聚合，并引入了UoW来管理聚合原子化更新的成功或失败。

例如，当我们将库存分配给订单时，我们的一致性边界是`Product`聚合。这意味着我们不能意外地过度分配：要么将特定订单行分配给产品，要么不分配给产品——不存在不一致状态的空间。

根据定义，我们不要求两个聚合立即保持一致，因此如果我们无法处理事件并仅更新单个聚合，我们的系统仍然可以实现最终一致性。我们不应该违反系统的任何约束。

记住这个例子，我们可以更好地理解将消息拆分为命令和事件的原因。当用户想要让系统执行某项操作时，我们将他们的请求表示为命令。该命令应修改单个聚合，并且整体成功或失败。我们需要做的任何其他记账、清理和通知都可以通过事件进行。我们不需要事件处理程序成功才能使命令成功。

让我们看看另一个例子（来自一个不同的、假想的项目）来看看为什么不行。

假设我们正在建立一个销售昂贵奢侈品的电子商务网站。我们的营销部门希望奖励重复访问的客户。我们将在客户进行第三次购买后将他们标记为 VIP，这将使他们有权享受优先待遇和特殊优惠。我们对这个故事的接受标准如下：

```
Given a customer with two orders in their history,
When the customer places a third order,
Then they should be flagged as a VIP.

When a customer first becomes a VIP
Then we should send them an email to congratulate them
```

使用本书中已经讨论过的技术，我们决定构建一个新的History聚合，用于记录订单，并在满足规则时引发域事件。我们将像这样构建代码：

```py title='VIP 客户（不同项目的示例代码）'
class History:  # Aggregate

    def __init__(self, customer_id: int):
        self.orders = set()  # Set[HistoryEntry]
        self.customer_id = customer_id

    def record_order(self, order_id: str, order_amount: int): #(1)
        entry = HistoryEntry(order_id, order_amount)

        if entry in self.orders:
            return

        self.orders.add(entry)

        if len(self.orders) == 3:
            self.events.append(
                CustomerBecameVIP(self.customer_id)
            )


def create_order_from_basket(uow, cmd: CreateOrder): #(2)
    with uow:
        order = Order.from_basket(cmd.customer_id, cmd.basket_items)
        uow.orders.add(order)
        uow.commit()  # raises OrderCreated


def update_customer_history(uow, event: OrderCreated): #(3)
    with uow:
        history = uow.order_history.get(event.customer_id)
        history.record_order(event.order_id, event.order_amount)
        uow.commit()  # raises CustomerBecameVIP


def congratulate_vip_customer(uow, event: CustomerBecameVip): #(4)
    with uow:
        customer = uow.customers.get(event.customer_id)
        email.send(
            customer.email_address,
            f'Congratulations {customer.first_name}!'
        )
```

1. `History`聚合捕获指示客户何时成为 VIP 的规则。当规则在未来变得更加复杂时，这使我们能够很好地处理变化。
2. 我们的第一个处理程序为客户创建订单并引发域事件`OrderCreated`。
3. 我们的第二个处理程序更新`History`对象以记录订单已创建。
4. 最后，当客户成为 VIP 时，我们会向客户发送一封电子邮件。

使用此代码，我们可以对事件驱动系统中的错误处理获得一些直觉。

在我们当前的实现中，我们在将状态持久化到数据库后 引发有关聚合的事件。如果我们在持久化之前引发这些事件并同时提交所有更改会怎么样？这样，我们就可以确保所有工作都已完成。这不是更安全吗？

但是，如果电子邮件服务器稍微超载，会发生什么情况？如果所有工作必须同时完成，繁忙的电子邮件服务器可能会阻止我们收取订单款项。

如果`History`聚合实现中出现错误，会发生什么情况？我们是否应该因为无法识别您是 VIP 而无法收款？

通过分离这些问题，我们使得事情可以单独失败，从而提高系统的整体可靠性。此代码中唯一需要完成的部分是创建订单的命令处理程序。这是客户唯一关心的部分，也是我们的业务利益相关者应该优先考虑的部分。

请注意，我们有意将事务边界与业务流程的开始和结束对齐。我们在代码中使用的名称与业务利益相关者使用的术语相匹配，我们编写的处理程序与自然语言验收标准的步骤相匹配。名称和结构的一致性有助于我们在系统变得越来越大、越来越复杂时对其进行推理。


## <font color='red'>同步恢复错误</font>

希望我们已经说服了您，事件失败与引发它们的命令无关，这是可以接受的。那么，当错误不可避免地发生时，我们应该怎么做才能确保能够从错误中恢复？

我们首先需要知道错误何时发生，为此我们通常依赖日志。

让我们再次看一下消息总线中的`handle_event`方法：

```py title='当前处理函数（src/allocation/service_layer/messagebus.py）'
def handle_event(
    event: events.Event,
    queue: List[Message],
    uow: unit_of_work.AbstractUnitOfWork,
):
    for handler in EVENT_HANDLERS[type(event)]:
        try:
            logger.debug("handling event %s with handler %s", event, handler)
            handler(event, uow=uow)
            queue.extend(uow.collect_new_events())
        except Exception:
            logger.exception("Exception handling event %s", event)
            continue
```

当我们在系统中处理消息时，我们要做的第一件事就是写一行日志来记录我们要做的事情。对于我们的CustomerBecameVIP用例，日志可能如下所示：

```
Handling event CustomerBecameVIP(customer_id=12345)
with handler <function congratulate_vip_customer at 0x10ebc9a60>
```

因为我们选择使用数据类作为消息类型，所以我们整齐地打印了传入数据的摘要，我们可以将其复制并粘贴到 Python shell 中以重新创建对象。

当发生错误时，我们可以使用记录的数据在单元测试中重现问题或将消息重播到系统中。

手动重放非常适合需要先修复错误才能重新处理事件的情况，但我们的系统总会遇到某种程度的瞬时故障。这包括网络故障、表死锁和部署导致的短暂停机等。

对于大多数情况，我们可以通过重试优雅地恢复。正如谚语所说，“如果一开始你没有成功，请以指数增加的退避期重试该操作。”

```py title='处理重试（src/allocation/service_layer/messagebus.py）'
from tenacity import Retrying, RetryError, stop_after_attempt, wait_exponential #(1)

...

def handle_event(
    event: events.Event,
    queue: List[Message],
    uow: unit_of_work.AbstractUnitOfWork,
):
    for handler in EVENT_HANDLERS[type(event)]:
        try:
            for attempt in Retrying(  #(2)
                stop=stop_after_attempt(3),
                wait=wait_exponential()
            ):

                with attempt:
                    logger.debug("handling event %s with handler %s", event, handler)
                    handler(event, uow=uow)
                    queue.extend(uow.collect_new_events())
        except RetryError as retry_failure:
            logger.error(
                "Failed to handle event %s times, giving up!",
                retry_failure.last_attempt.attempt_number
            )
            continue
```

1. `Tenacity` 是一个实现重试常见模式的 Python 库。
2. 在这里，我们将消息总线配置为最多重试三次操作，并且尝试之间的等待时间呈指数增加。

重试可能失败的操作可能是提高软件弹性的唯一最佳方法。同样，工作单元和命令处理程序模式意味着每次尝试都从一致的状态开始，不会让事情半途而废。

!!! warning
    在某个时候，不考虑tenacity，我们都必须放弃尝试处理该消息。构建具有分布式消息的可靠系统很难，我们必须略过一些棘手的部分。在结语中提供了指向更多参考资料的指针。


## <font color='red'>总结</font>

在本书中，我们决定先介绍事件的概念，然后再介绍命令的概念，但其他指南通常反其道而行之。通过为请求命名并赋予其自己的数据结构，明确我们的系统可以响应的请求，这是相当基本的做法。有时您会看到人们使用命令处理程序模式这一名称来描述我们对事件、命令和消息总线的操作。

表1讨论了您在加入之前应该权衡考虑的一些事情。

<font color='#186d7a'>表1.拆分命令和事件：权衡</font>

|优点|缺点|
|---|---|
|以不同的方式处理命令和事件有助于我们了解哪些事情必须成功以及哪些事情我们可以稍后整理。|命令和事件之间的语义差异可能很微妙。预计这些差异会引起争论。|
|`CreateBatch`肯定比`BatchCreated`更不容易混淆。我们明确表达了用户的意图，明确总比隐含好，对吧？|我们明确地欢迎失败。我们知道有时事情会出错，我们选择通过使故障更小、更孤立来处理这个问题。这会使系统更难推理，需要更好的监控。|

在[11.事件驱动架构：使用事件集成微服务](./o.Event-Driven%20Architecture.md)中我们将讨论使用事件作为集成模式。
