# 附录E：验证


每当我们教授和讨论这些技术时，一个反复出现的问题是“我应该在哪里进行验证？这是否属于领域模型中的业务逻辑，或者这是一个基础设施问题？”

与任何建筑问题一样，答案是：视情况而定！

最重要的考虑是，我们希望保持代码的良好分离，以便系统的每个部分都很简单。我们不想让无关的细节弄乱我们的代码。


## <font color='red'>验证到底是什么？</font>

当人们使用“验证”一词时，他们通常指的是测试操作输入以确保其符合某些标准的过程。符合标准的输入被视为有效，不符合标准的输入被视为无效。

如果输入无效，操作将无法继续，而应该以某种错误退出。换句话说，验证就是创建先决条件。我们发现将先决条件分为三个子类型很有用：语法、语义和语用。


## <font color='red'>验证语法</font>

在语言学中，语言的句法是控制语法句子结构的一组规则。例如，在英语中，句子“Allocate three unit of TASTELESS-LAMPto order twenty-seven”在语法上是合理的，而短语“hat hat hat hat hat hat wibble”则不合理。我们可以将语法正确的句子描述为结构良好的。

这如何映射到我们的应用程序中？ 以下是一些句法规则的示例：

- 一个`Allocate`命令必须有一个订单ID、一个SKU和一个数量。
2. 数量是正整数。
3. SKU 是一个字符串。

这些是关于传入数据的形状和结构的规则。`Allocate` 没有 SKU 或订单 ID 的命令不是有效消息。它相当于短语“分配三个”。

我们倾向于在系统边缘验证这些规则。我们的经验法则是，消息处理程序应始终只接收格式正确且包含所有必需信息的消息。

一种选择是将验证逻辑放在消息类型本身上：

```py title='消息类别的验证（src/allocation/commands.py）'
from schema import And, Schema, Use


@dataclass
class Allocate(Command):

    _schema = Schema({  #(1)
        'orderid': str,
        'sku': str,
        'qty': And(Use(int), lambda n: n > 0)
     }, ignore_extra_keys=True)

    orderid: str
    sku: str
    qty: int

    @classmethod
    def from_json(cls, data):  #(2)
        data = json.loads(data)
        return cls(**_schema.validate(data))
```

1. schema库让我们能够以一种良好的声明方式描述消息的结构和验证。
2. 该`from_json`方法将字符串读取为 JSON 并将其转换为我们的消息类型。

但是，这可能会重复，因为我们需要指定两次字段，所以我们可能需要引入一个辅助库来统一我们的消息类型的验证和声明：

```py title='带有模式的命令工厂（src/allocation/commands.py）'
def command(name, **fields):  #(1)
    schema = Schema(And(Use(json.loads), fields), ignore_extra_keys=True)
    cls = make_dataclass(name, fields.keys())  #(2)
    cls.from_json = lambda s: cls(**schema.validate(s))  #(3)
    return cls

def greater_than_zero(x):
    return x > 0

quantity = And(Use(int), greater_than_zero)  #(4)

Allocate = command(  #(5)
    'Allocate',
    orderid=int,
    sku=str,
    qty=quantity
)

AddStock = command(
    'AddStock',
    sku=str,
    qty=quantity
```

1. 该`command`函数接受消息名称，加上消息有效负载字段的 kwargs，其中 kwarg 的名称是字段的名称，值是解析器。
2. 我们使用`make_dataclass`数据类模块中的函数来动态创建我们的消息类型。
3. 我们将该`from_json`方法修补到我们的动态数据类上。
4. 我们可以为数量、SKU 等创建可重复使用的解析器，以保持 DRY 原则。
5. 声明消息类型只需一行。

这是以丢失数据类的类型为代价的，所以请记住这种权衡。


## <font color='red'>Postel 定律和宽容读者模式</font>

Postel 定律，又称稳健性原则，告诉我们：“接受时要宽容，发出时要保守。”我们认为这在与其他系统集成时尤其适用。这里的理念是，我们在向其他系统发送消息时应该严格，但在接收来自其他系统的消息时则应尽可能宽松。

例如，我们的系统可以验证 SKU 的格式。我们一直在使用像`UNFORGIVING-CUSHION`和`MISBEGOTTEN-POUFFE`这样的虚构 SKU。它们遵循一个简单的模式：两个单词，用破折号分隔，其中第二个单词是产品类型，第一个单词是形容词。

开发人员喜欢在消息中验证此类信息，并拒绝任何看起来像无效 SKU 的信息。当一些无政府主义者发布了一款名为`COMFY-CHAISE-LONGUE`的产品，或者当供应商出现问题导致运送`CHEAP-CARPET-2`时，这会导致可怕的问题

实际上，作为分配系统， SKU 的格式与我们无关。我们需要的只是一个标识符，因此我们可以简单地将其描述为字符串。这意味着采购系统可以随时更改格式，我们不会关心。

同样的原则也适用于订单号、客户电话号码等。大多数情况下，我们可以忽略字符串的内部结构。

同样，开发人员喜欢使用 JSON Schema 等工具来验证传入消息，或者构建库来验证传入消息并在系统之间共享它们。这同样不符合稳健性测试。

例如，让我们想象一下，采购系统在 `ChangeBatchQuantity` 消息中添加新字段，记录变更的原因和负责变更的用户的电子邮件。

由于这些字段对分配服务来说无关紧要，我们应该直接忽略它们。我们可以在schema库中通过传递关`ignore_extra_keys=True`来做到这一点。

这种模式，即我们只提取我们关心的字段并对它们进行最少的验证，就是宽容读取器模式。

!!! tip
    尽可能少地进行验证。只读取您需要的字段，不要过度指定其内容。这将有助于您的系统在其他系统随时间变化时保持稳健。抵制在系统之间共享消息定义的诱惑：相反，请轻松定义您所依赖的数据。有关更多信息，请参阅 Martin Fowler 关于 [Tolerant Reader](https://martinfowler.com/bliki/TolerantReader.html) 模式的文章。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px">
    <p align='center'><strong style='font-size:1.6em;color:#186d7a'>Postel总是对的吗？</strong></p>
    <p>提到 Postel 可能会让一些人感到很生气。他们会 告诉你 Postel 就是互联网上一切崩溃、我们无法拥有美好事物的确切原因。有朝一日向 Hynek 询问 SSLv3 的事情。</p>
    <p>我们喜欢在我们控制的服务之间基于事件的集成特定环境中使用 Tolerant Reader 方法，因为它允许这些服务独立发展。</p>
    <p>如果您负责在恶劣的互联网上向公众开放的 API，那么可能有充分的理由对允许的输入采取更加保守的态度。</p>
</div>


## <font color='red'>在边缘验证</font>

之前，我们说过，我们希望避免让无关的细节扰乱我们的代码。具体来说，我们不想在领域模型中防御性地编写代码。相反，我们希望确保在领域模型或用例处理程序看到请求之前，请求是有效的。这有助于我们的代码长期保持干净和可维护。我们有时将此称为在系统边缘进行验证。

除了保持代码清洁并避免无休止的检查和断言之外，还要记住，系统中游走的无效数据是一颗定时炸弹；它藏得越深，造成的损害就越大，而您用来应对它的工具就越少。

回到[8.事件和消息总线](./l.Events%20and%20the%20Message%20Bus.md)，我们说过消息总线是放置横切关注点的绝佳位置，而验证就是一个很好的例子。以下是我们可以如何更改总线以执行验证：

```py title='验证'
class MessageBus:

    def handle_message(self, name: str, body: str):
        try:
            message_type = next(mt for mt in EVENT_HANDLERS if mt.__name__ == name)
            message = message_type.from_json(body)
            self.handle([message])
        except StopIteration:
            raise KeyError(f"Unknown message name {name}")
        except ValidationError as e:
            logging.error(
                f'invalid message of type {name}\n'
                f'{body}\n'
                f'{e}'
            )
            raise e
```

以下是我们如何从 Flask API 端点使用该方法：

```py title='API 冒泡验证错误（src/allocation/flask_app.py）'
@app.route("/change_quantity", methods=['POST'])
def change_batch_quantity():
    try:
        bus.handle_message('ChangeBatchQuantity', request.body)
    except ValidationError as e:
        return bad_request(e)
    except exceptions.InvalidSku as e:
        return jsonify({'message': str(e)}), 400

def bad_request(e: ValidationError):
    return e.code, 400
```

以下是我们如何将其插入到异步消息处理器中：

```py title='处理 Redis 消息时的验证错误（src/allocation/redis_pubsub.py）'
def handle_change_batch_quantity(m, bus: messagebus.MessageBus):
    try:
        bus.handle_message('ChangeBatchQuantity', m)
    except ValidationError:
        print('Skipping invalid message')
    except exceptions.InvalidSku as e:
        print(f'Unable to change stock for missing sku {e}')
```

请注意，我们的入口点仅关注如何从外界获取消息以及如何报告成功或失败。我们的消息总线负责验证我们的请求并将其路由到正确的处理程序，而我们的处理程序则专注于我们的用例的逻辑。

!!! tip
    当您收到无效消息时，通常您除了记录错误并继续之外几乎无能为力。在 MADE，我们使用指标来计算系统收到的消息数量，以及其中有多少被成功处理、跳过或无效。如果我们发现坏消息数量激增，我们的监控工具会提醒我们。


## <font color='red'>验证语义</font>

句法与消息的结构有关，而语义则研究消息中的含义。句子“Undo no dogs from ellipsis four”在句法上是有效的，并且与句子“Allocate one teapot to order five”具有相同的结构，但它毫无意义。

我们可以将此 JSON blob 读取为Allocate命令，但无法成功执行它，因为它毫无意义：

```py title='毫无意义的信息'
{
  "orderid": "superman",
  "sku": "zygote",
  "qty": -1
}
```

我们倾向于使用一种基于契约的编程来验证消息处理程序层的语义问题：

```py title='先决条件（src/allocation/ensure.py）'
"""
This module contains preconditions that we apply to our handlers.
"""

class MessageUnprocessable(Exception):  #(1)

    def __init__(self, message):
        self.message = message

class ProductNotFound(MessageUnprocessable):  #(2)
    """"
    This exception is raised when we try to perform an action on a product
    that doesn't exist in our database.
    """"

    def __init__(self, message):
        super().__init__(message)
        self.sku = message.sku

def product_exists(event, uow):  #(3)
    product = uow.products.get(event.sku)
    if product is None:
        raise ProductNotFound(event)
```

1. 我们使用一个通用基类来表示消息无效的错误。
2. 针对此问题使用特定的错误类型可以更轻松地报告和处理错误。例如，它很容易映射`ProductNotFound`到 Flask 中的 404。
3. `product_exists`是先决条件。如果条件是`False`，我们会引发错误。

这使服务层中逻辑的主要流程保持清晰和声明性：

```py title='确保服务中的调用（src/allocation/services.py）'
# services.py

from allocation import ensure

def allocate(event, uow):
    line = model.OrderLine(event.orderid, event.sku, event.qty)
    with uow:
        ensure.product_exists(event, uow)

        product = uow.products.get(line.sku)
        product.allocate(line)
        uow.commit()
```

我们可以扩展这项技术，以确保我们幂等地应用消息。例如，我们要确保不会多次插入一批库存。

如果我们被要求创建一个已经存在的批次，我们将记录一个警告并继续下一条消息：

```py title='对可忽略事件引发 SkipMessage 异常（src/allocation/services.py）'
class SkipMessage (Exception):
    """"
    This exception is raised when a message can't be processed, but there's no
    incorrect behavior. For example, we might receive the same message multiple
    times, or we might receive a message that is now out of date.
    """"

    def __init__(self, reason):
        self.reason = reason

def batch_is_new(self, event, uow):
    batch = uow.batches.get(event.batchid)
    if batch is not None:
        raise SkipMessage(f"Batch with id {event.batchid} already exists")
```

引入SkipMessage异常让我们能够在消息总线中以通用的方式处理这些情况：

```py title='总线现在知道如何跳过（src/allocation/messagebus.py）'
class MessageBus:

    def handle_message(self, message):
        try:
            ...
        except SkipMessage as e:
            logging.warn(f"Skipping message {message.id} because {e.reason}")
```

这里有几个陷阱需要注意。首先，我们需要确保我们使用的 UoW 与用例的主要逻辑相同。否则，我们就会面临恼人的并发错误。

其次，我们应该尽量避免将所有业务逻辑都放入这些先决条件检查中。根据经验，如果规则可以在我们的领域模型中测试，那么就应该在领域模型中测试它。


## <font color='red'>验证语用学</font>

语用学是研究我们如何在语境中理解语言的学科。在我们解析了一条消息并掌握了它的含义之后，我们仍然需要在语境中处理它。例如，如果你在拉取请求中收到一条评论说“我认为这非常勇敢”，这可能意味着审阅者钦佩你的勇气——除非他们是英国人，在这种情况下，他们试图告诉你，你正在做的事情非常危险，只有傻瓜才会尝试。语境就是一切。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px">
    <p align='center'><strong style='font-size:1.6em;color:#186d7a'>验证回顾</strong></p>
    <p><strong>验证对不同的人来说有不同的含义</strong></p>
    <p>谈论验证时，请确保您清楚自己在验证什么。我们发现考虑语法、语义和语用学很有用：消息的结构、消息的意义以及控制我们对消息的响应的业务逻辑。</p>
    <p><strong>尽可能在边缘进行验证</strong></p>
    <p>验证必填字段和允许的数字范围很无聊，我们希望将其排除在我们干净的代码库之外。处理程序应该始终只接收有效消息。</p>
    <p><strong>仅验证您需要的内容</strong></p>
    <p>使用 Tolerant Reader 模式：只读取应用程序所需的字段，不要过度指定其内部结构。将字段视为不透明字符串可为您带来很大的灵活性。</p>
    <p><strong> 花时间编写验证助手</strong></p>
    <p>拥有一种良好的声明式方法来验证传入的消息并将先决条件应用于处理程序将使您的代码库更加简洁。值得花时间让枯燥的代码易于维护。</p>
    <p><strong>将三种类型的验证分别放在正确的位置</strong></p>
    <p>验证语法可以发生在消息类上，验证语义可以发生在服务层或消息总线上，而验证语用则属于领域模型。</p>
</div>

!!! tip
    在系统边缘验证命令的语法和语义后，域就是进行其余验证的地方。语用验证通常是业务规则的核心部分。

从软件角度来说，操作的实用性通常由领域模型管理。当我们收到“分配三百万单位`SCARCE-CLOCK`以 订购 76543”这样的消息时，该消息在语法和语义上都是有效的，但我们无法执行，因为我们没有可用的库存。