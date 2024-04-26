# 13.依赖注入（和引导）


在 Python 世界中，依赖注入 (DI) 受到质疑。到目前为止，在本书的示例代码中，我们没有使用依赖注入，也运行得很好！

在本章中，我们将探讨代码中的一些痛点，这些痛点导致我们考虑使用 DI，并且我们将提供一些执行选项，让您选择您认为最 Pythonic 的选项。

我们还将向我们的架构添加一个名为bootstrap.py的新组件；它将负责依赖项注入，以及我们经常需要的一些其他初始化内容。我们将解释为什么这种东西在 OO 语言中被称为组合根，以及为什么bootstrap 脚本非常适合我们的目的。

图1.没有引导程序：入口点做了很多事情，展示了我们的应用程序在没有引导程序的情况下是什么样子：入口点做了很多初始化工作，并传递了我们的主要依赖项 UoW。

!!! tip
    如果您还没有读过，那么在继续阅读本章之前，值得先阅读[3.耦合和抽象](./f.Coupling%20and%20Abstractions.md) ，特别是关于函数式与面向对象依赖管理的讨论。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_1301.png){ align=center }
    <figcaption><font size=2>图1.没有引导程序：入口点做了很多事情</font></figcaption>
</figure>

!!! tip
    本章的代码位于[GitHub](https://github.com/cosmicpython/code/tree/chapter_13_dependency_injection)上的chapter_13_dependency_injection 分支中：

    ```
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout chapter_13_dependency_injection
    # or to code along, checkout the previous chapter:
    git checkout chapter_12_cqrs
    ```

图2.Bootstrap 将所有工作集中在一个地方，表明我们的引导程序接管了这些职责。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_1302.png){ align=center }
    <figcaption><font size=2>图2.Bootstrap 集中处理所有这些</font></figcaption>
</figure>


## <font color='red'>隐式依赖与显式依赖</font>

根据您的大脑类型，此时您可能会在内心深处感到一丝不安。让我们来谈谈这个问题。我们向您展示了两种管理依赖关系和测试依赖关系的方法。

对于我们的数据库依赖关系，我们构建了一个精心设计的显式依赖关系框架，并提供了在测试中覆盖它们的简单选项。我们的主要处理程序函数声明了对 UoW 的显式依赖关系：

```py title='我们的处理程序对 UoW 有明确的依赖性（src/allocation/service_layer/handlers.py）'
def allocate(
    cmd: commands.Allocate,
    uow: unit_of_work.AbstractUnitOfWork,
):
```

这使得在我们的服务层测试中替换伪 UoW 变得很容易：

```py title='针对伪 UoW 进行服务层测试：（tests/unit/test_services.py）'
    uow = FakeUnitOfWork()
    messagebus.handle([...], uow)

UoW 本身声明了对会话工厂的明确依赖：

```py title='UoW 依赖于会话工厂 (src/allocation/service_layer/unit_of_work.py）'
class SqlAlchemyUnitOfWork(AbstractUnitOfWork):
    def __init__(self, session_factory=DEFAULT_SESSION_FACTORY):
        self.session_factory = session_factory
        ...
```

我们在集成测试中利用它，有时可以使用 SQLite 而不是 Postgres：

```py title='针对不同数据库的集成测试（tests/integration/test_uow.py）'
def test_rolls_back_uncommitted_work_by_default(sqlite_session_factory):
    uow = unit_of_work.SqlAlchemyUnitOfWork(sqlite_session_factory)  #(1)
```

1. 集成测试将默认的 Postgres 替换session_factory为 SQLite。


## <font color='red'>显式依赖关系不是很奇怪而且很像 Java 吗？</font>

如果你习惯了 Python 中通常发生的事情，你会认为这一切有点奇怪。标准做法是通过简单地导入来隐式声明我们的依赖项，然后如果我们需要为测试更改它，我们可以使用 `monkeypatch`，就像动态语言中的 Right 和 True 一样：

```py title='电子邮件发送作为正常的基于导入的依赖项（src/allocation/service_layer/handlers.py）'
from allocation.adapters import email, redis_eventpublisher  #(1)
...

def send_out_of_stock_notification(
    event: events.OutOfStock,
    uow: unit_of_work.AbstractUnitOfWork,
):
    email.send(  #(2)
        "stock@made.com",
        f"Out of stock for {event.sku}",
    )
```

1. 硬编码导入
2. 直接致电特定电子邮件发件人

为什么只是为了测试而用不必要的参数污染我们的应用程序代码？`mock.patch`使`monkeypatching`变得简单而方便：

```py title='模拟点补丁，感谢 Michael Foord (tests/unit/test_handlers.py）'
    with mock.patch("allocation.adapters.email.send") as mock_send_mail:
        ...
```

问题是，我们让它看起来很容易，因为我们的玩具示例不会发送真正的电子邮件（`email.send_mail`只是`print`），但在现实生活中，您最终必须为每个可能导致缺货通知的测试调用`mock.patch`。如果您使用大量模拟来防止不良副作用的代码库，您就会知道模拟样板有多烦人。

你会知道，模拟将我们与实现紧密联系在一起。通过选择 monkeypatch `email.send_mail`，我们被束缚着去做`import email`，如果我们想做`from email import send_mail`一个简单的重构，我们就必须改变我们所有的模拟。

所以这是一种权衡。是的，严格来说，声明显式依赖项是不必要的，使用它们会使我们的应用程序代码稍微复杂一些。但作为回报，我们会得到更易于编写和管理的测试。

最重要的是，声明显式依赖关系是依赖反转原则的一个例子 - 我们不是对特定细节有（隐式）依赖，而是对抽象有（显式）依赖：

!!! quote
    清晰比晦涩好。

    <p align='right'>Python 之禅</p>

```py title='显式依赖更加抽象（src/allocation/service_layer/handlers.py）'
def send_out_of_stock_notification(
    event: events.OutOfStock,
    send_mail: Callable,
):
    send_mail(
        "stock@made.com",
        f"Out of stock for {event.sku}",
    )
```

但是，如果我们确实改为明确声明所有这些依赖项，谁来注入它们，又如何注入？到目前为止，我们实际上只处理传递 UoW：我们的测试使用`FakeUnitOfWork`，而 Flask 和 Redis 事件消费者入口点使用真正的 UoW，消息总线将它们传递到我们的命令处理程序。如果我们添加真实和虚假的电子邮件类，谁来创建它们并传递它们？

它需要在流程生命周期中尽早发生，因此最明显的地方就是我们的入口点。这意味着 Flask 和 Redis 以及我们的测试中会出现额外的（重复的）垃圾。我们还必须将传递依赖项的责任添加到消息总线，而消息总线已经有工作要做；这感觉像是违反了 SRP。

相反，我们将采用一种称为Composition Root 的模式（对你我来说，这是一个引导脚本），并且我们将进行一些“手动 DI”（无需框架的依赖注入）。请参阅图3.入口点和消息总线之间的引导程序。

<figure markdown='span'>
    ![](https://www.cosmicpython.com/book/images/apwp_1303.png){ align=center }
    <figcaption><font size=2>图3.入口点和消息总线之间的引导程序</font></figcaption>
</figure>


## <font color='red'>准备处理程序：使用闭包和部分函数进行手动 DI</font>

将具有依赖项的函数转变为稍后可以调用的函数（这些依赖项已经注入）的一种方法是使用闭包或部分函数将该函数与其依赖项组合在一起：

```py title='使用闭包或部分函数的 DI 示例'
# existing allocate function, with abstract uow dependency
def allocate(
    cmd: commands.Allocate,
    uow: unit_of_work.AbstractUnitOfWork,
):
    line = OrderLine(cmd.orderid, cmd.sku, cmd.qty)
    with uow:
        ...

# bootstrap script prepares actual UoW

def bootstrap(..):
    uow = unit_of_work.SqlAlchemyUnitOfWork()

    # prepare a version of the allocate fn with UoW dependency captured in a closure
    allocate_composed = lambda cmd: allocate(cmd, uow)

    # or, equivalently (this gets you a nicer stack trace)
    def allocate_composed(cmd):
        return allocate(cmd, uow)

    # alternatively with a partial
    import functools
    allocate_composed = functools.partial(allocate, uow=uow)  #(1)

# later at runtime, we can call the partial function, and it will have
# the UoW already bound
allocate_composed(cmd)
```
1. 闭包（lambda 或命名函数）和`functools.partial`之间的区别在于前者使用变量的后期绑定，如果任何依赖项是可变的，这可能会造成混淆。

对于`send_out_of_stock_notification()`处理程序来说，这又是相同的模式，它具有不同的依赖关系：

```py title='另一个闭包和部分函数示例'
def send_out_of_stock_notification(
    event: events.OutOfStock,
    send_mail: Callable,
):
    send_mail(
        "stock@made.com",
        ...

# prepare a version of the send_out_of_stock_notification with dependencies
sosn_composed  = lambda event: send_out_of_stock_notification(event, email.send_mail)

...
# later, at runtime:
sosn_composed(event)  # will have email.send_mail already injected in
```

## <font color='red'>使用类的替代方法</font>

对于做过一些函数式编程的人来说，闭包和部分函数会很熟悉。下面是使用类的替代方法，可能对其他人有吸引力。不过，它需要将所有处理程序函数重写为类：

```py title='使用类的 DI'
# we replace the old `def allocate(cmd, uow)` with:

class AllocateHandler:
    def __init__(self, uow: unit_of_work.AbstractUnitOfWork):  #(2)
        self.uow = uow

    def __call__(self, cmd: commands.Allocate):  #(1)
        line = OrderLine(cmd.orderid, cmd.sku, cmd.qty)
        with self.uow:
            # rest of handler method as before
            ...

# bootstrap script prepares actual UoW
uow = unit_of_work.SqlAlchemyUnitOfWork()

# then prepares a version of the allocate fn with dependencies already injected
allocate = AllocateHandler(uow)

...
# later at runtime, we can call the handler instance, and it will have
# the UoW already injected
allocate(cmd)
```

1. 该类旨在生成可调用函数，因此它具有`__call__`方法。
2. 但是我们使用`init`来声明它所需的依赖项。如果您曾经创建过基于类的描述符或基于类的接受参数的上下文管理器，那么这种事情会让您感觉很熟悉。

使用您和您的团队感觉更舒服的任何一个。


## <font color='red'>引导脚本</font>

我们希望引导脚本执行以下操作：

1. 声明默认依赖项但允许我们覆盖它们
2. 执行启动应用程序所需的“init”操作
3. 将所有依赖项注入到我们的处理程序中
4. 将应用程序的核心对象，即消息总线归还给我们

以下是第一部分：

```py title='引导函数（src/allocation/bootstrap.py）'
def bootstrap(
    start_orm: bool = True,  #(1)
    uow: unit_of_work.AbstractUnitOfWork = unit_of_work.SqlAlchemyUnitOfWork(),  #(2)
    send_mail: Callable = email.send,
    publish: Callable = redis_eventpublisher.publish,
) -> messagebus.MessageBus:

    if start_orm:
        orm.start_mappers()  #(1)

    dependencies = {"uow": uow, "send_mail": send_mail, "publish": publish}
    injected_event_handlers = {  #(3)
        event_type: [
            inject_dependencies(handler, dependencies)
            for handler in event_handlers
        ]
        for event_type, event_handlers in handlers.EVENT_HANDLERS.items()
    }
    injected_command_handlers = {  #(3)
        command_type: inject_dependencies(handler, dependencies)
        for command_type, handler in handlers.COMMAND_HANDLERS.items()
    }

    return messagebus.MessageBus(  #(4)
        uow=uow,
        event_handlers=injected_event_handlers,
        command_handlers=injected_command_handlers,
    )
```

1. `orm.start_mappers()`这是我们的一个示例，该示例需要在应用程序开始时进行一次初始化工作。另一个常见示例是设置模块`logging`。
2. 我们可以使用参数 `defaults` 来定义正常/生产默认值。将它们放在一个地方固然很好，但有时依赖项在构造时会产生一些副作用，在这种情况下，您可能更愿意将它们设置为默认值`None`。
3. 我们使用一个名为`inject_dependencies()`的函数来构建处理程序映射的注入版本，我们将在接下来展示它。
4. 我们返回一个已配置且可供使用的消息总线。

以下是我们如何通过检查将依赖项注入处理程序函数：

```py title='通过检查函数签名进行 DI（src/allocation/bootstrap.py）'
def inject_dependencies(handler, dependencies):
    params = inspect.signature(handler).parameters  #(1)
    deps = {
        name: dependency
        for name, dependency in dependencies.items()  #(2)
        if name in params
    }
    return lambda message: handler(message, **deps)  #(3)
```

1. 我们检查我们的命令/事件处理程序的参数。
2. 我们通过名称将它们与我们的依赖项进行匹配。
3. 我们将它们作为 kwargs 注入以产生partial。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>更少魔法，更多手动 DI</strong></p>
<p>如果您发现上面的inspect代码有点难以理解，那么这个更简单的版本可能会吸引您。</p>

<p>Harry为<code>inject_dependencies()</code>编写了代码，作为进行“手动”依赖注入的初步尝试，当 Bob 看到它时，他指责他过度设计并编写了自己的 DI 框架。</p>

老实说，Harry根本没想过可以做得更简单，但你可以像这样：

<pre>
<code style='border: 1px solid;'>
# 手动创建内联部分函数（src/allocation/bootstrap.py）
injected_event_handlers = {
    events.Allocated: [
        lambda e: handlers.publish_allocated_event(e, publish),
        lambda e: handlers.add_allocation_to_read_model(e, uow),
    ],
    events.Deallocated: [
        lambda e: handlers.remove_allocation_from_read_model(e, uow),
        lambda e: handlers.reallocate(e, uow),
    ],
    events.OutOfStock: [
        lambda e: handlers.send_out_of_stock_notification(e, send_mail)
    ],
}
injected_command_handlers = {
    commands.Allocate: lambda c: handlers.allocate(c, uow),
    commands.CreateBatch: lambda c: handlers.add_batch(c, uow),
    commands.ChangeBatchQuantity: \
        lambda c: handlers.change_batch_quantity(c, uow),
}
</code>
</pre>
<p>Harry 说他甚至无法想象写出那么多行代码并手动查找那么多函数参数。不过，这将是一个完全可行的解决方案，因为每个添加的处理程序只有一行代码左右。即使您有几十个处理程序，也不会带来太多的维护负担。</p>
<p>我们的应用程序的结构使得我们始终只想在一个地方进行依赖注入，即处理程序函数，因此这个手动解决方案和Harry inspect()的解决方案都可以正常工作。</p>
<p>如果您发现自己想要在更多事物和不同时间进行 DI，或者您陷入依赖链（其中您的依赖项有它们自己的依赖项，等等），您可能会从“真正的” DI 框架中获得一些好处。</p>

<p>在 MADE 中，我们在几个地方使用了<a src='https://pypi.org/project/inject/'>Inject</a>，效果很好（尽管它会让 Pylint 不高兴）。您也可以查看 由 Bob 自己编写的Punq或 DRY-Python 团队的<a src='https://github.com/proofit404/dependencies'>Dependencies</a>。</p>
</div>

## <font color='red'>消息总线在运行时被赋予处理程序</font>

我们的消息总线将不再是静态的；它需要已注入的处理程序。因此，我们将其从模块转变为可配置的类：

```py title='MessageBus 作为一个类（src/allocation/service_layer/messagebus.py）'
class MessageBus:  #(1)
    def __init__(
        self,
        uow: unit_of_work.AbstractUnitOfWork,
        event_handlers: Dict[Type[events.Event], List[Callable]],  #(2)
        command_handlers: Dict[Type[commands.Command], Callable],  #(2)
    ):
        self.uow = uow
        self.event_handlers = event_handlers
        self.command_handlers = command_handlers

    def handle(self, message: Message):  #(3)
        self.queue = [message]  #(4)
        while self.queue:
            message = self.queue.pop(0)
            if isinstance(message, events.Event):
                self.handle_event(message)
            elif isinstance(message, commands.Command):
                self.handle_command(message)
            else:
                raise Exception(f"{message} was not an Event or Command")
```

1. 消息总线变成一个类…​
2. …​它被赋予了已经依赖注入的处理程序。
3. 主要的`handle()`功能基本相同，只是将一些属性和方法移到了`self`上。
4. 像这样使用`self.queue`不是线程安全的，如果您使用线程，这可能是一个问题，因为总线实例在 Flask 应用上下文中是全局的，就像我们编写的那样。需要注意这一点。

总线上还有哪些变化？

```py title='事件和命令处理程序逻辑保持不变（src/allocation/service_layer/messagebus.py）'
    def handle_event(self, event: events.Event):
        for handler in self.event_handlers[type(event)]:  #(1)
            try:
                logger.debug("handling event %s with handler %s", event, handler)
                handler(event)  #(2)
                self.queue.extend(self.uow.collect_new_events())
            except Exception:
                logger.exception("Exception handling event %s", event)
                continue

    def handle_command(self, command: commands.Command):
        logger.debug("handling command %s", command)
        try:
            handler = self.command_handlers[type(command)]  #(1)
            handler(command)  #(2)
            self.queue.extend(self.uow.collect_new_events())
        except Exception:
            logger.exception("Exception handling command %s", command)
            raise
```
1. `handle_event`和`handle_command`实质上是相同的，但是它们不是索引到静态`EVENT_HANDLERS`或`COMMAND_HANDLERS`字典中，而是使用`self`上的版本。
2. 我们不需要将 UoW 传递到处理程序，而是希望处理程序已经具有所有依赖项，因此它们所需要的只是一个参数，即特定的事件或命令。


## <font color='red'>在入口点使用 Bootstrap</font>

在我们的应用程序的入口点，我们现在只需调用bootstrap.bootstrap() 并获取一个已准备好的消息总线，而不是配置 UoW 和其余部分：

```py title='Flask 调用引导程序（src/allocation/entrypoints/flask_app.py）'
-from allocation import views
+from allocation import bootstrap, views

 app = Flask(__name__)
-orm.start_mappers()  #(1)
+bus = bootstrap.bootstrap()


 @app.route("/add_batch", methods=["POST"])
@@ -19,8 +16,7 @@ def add_batch():
     cmd = commands.CreateBatch(
         request.json["ref"], request.json["sku"], request.json["qty"], eta
     )
-    uow = unit_of_work.SqlAlchemyUnitOfWork()  #(2)
-    messagebus.handle(cmd, uow)
+    bus.handle(cmd)  #(3)
     return "OK", 201
```

1. 我们不再需要调用`start_orm()`；引导脚本的初始化阶段将会执行此操作。
2. 我们不再需要明确构建特定类型的 UoW；引导脚本默认会处理它。
3. 我们的消息总线现在是一个特定的实例，而不是全局模块。


## <font color='red'>在测试中初始化 DI</font>

在测试中，我们可以使用bootstrap.bootstrap()覆盖的默认值来获取自定义消息总线。以下是集成测试中的一个示例：

```py title='覆盖引导默认值（tests/integration/test_views.py）'
@pytest.fixture
def sqlite_bus(sqlite_session_factory):
    bus = bootstrap.bootstrap(
        start_orm=True,  #(1)
        uow=unit_of_work.SqlAlchemyUnitOfWork(sqlite_session_factory),  #(2)
        send_mail=lambda *args: None,  #(3)
        publish=lambda *args: None,  #(3)
    )
    yield bus
    clear_mappers()


def test_allocations_view(sqlite_bus):
    sqlite_bus.handle(commands.CreateBatch("sku1batch", "sku1", 50, None))
    sqlite_bus.handle(commands.CreateBatch("sku2batch", "sku2", 50, today))
    ...
    assert views.allocations("order1", sqlite_bus.uow) == [
        {"sku": "sku1", "batchref": "sku1batch"},
        {"sku": "sku2", "batchref": "sku2batch"},
    ]
```

1. 我们仍然想启动 ORM……​
2. ...​因为我们将使用真正的 UoW，尽管带有内存数据库。
3. 但我们不需要发送电子邮件或发布，所以我们将这些都设为无用操作。

相反，在我们的单元测试中，我们可以重用我们的`FakeUnitOfWork`：

```py title='单元测试中的引导（tests/unit/test_handlers.py）'
def bootstrap_test_app():
    return bootstrap.bootstrap(
        start_orm=False,  #(1)
        uow=FakeUnitOfWork(),  #(2)
        send_mail=lambda *args: None,  #(3)
        publish=lambda *args: None,  #(3)
    )
```

1. 无需启动 ORM…​
2. …​因为假的 UoW 不使用这个。
3. 我们也想伪造我们的电子邮件和 Redis 适配器。

这样就消除了一些重复，并且我们将一堆设置和合理的默认值移到了一个地方。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px">
    <p align='center'><strong style='font-size:1.6em;color:#186d7a'>读者练习 1</strong></p>
    <p>按照【使用类的替代方法】示例将所有处理程序更改为类，并根据需要修改引导程序的 DI 代码。这将让您知道在自己的项目中您更喜欢函数式方法还是基于类的方法。</p>
</div>


## <font color='red'>“正确”构建适配器：一个实际示例</font>

为了真正了解它的工作原理，让我们通过一个例子来了解如何“正确地”构建适配器并对其进行依赖注入。

目前，我们有两种类型的依赖关系：

```py title='两种类型的依赖项（src/allocation/service_layer/messagebus.py）'
    uow: unit_of_work.AbstractUnitOfWork,  #(1)
    send_mail: Callable,  #(2)
    publish: Callable,  #(2)
```

1. UoW 有一个抽象基类。这是声明和管理外部依赖关系的重量级选项。当依赖关系相对复杂时，我们会使用它。
2. 我们的电子邮件发送者和发布/订阅发布者被定义为函数。这对于简单的依赖关系来说非常有效。

以下是我们在工作中注入的一些东西：

- S3 文件系统客户端
- 键/值存储客户端
- requests会话对象

其中大多数都具有更复杂的 API，您无法将其捕获为单个函数：读取和写入、GET 和 POST 等等。

尽管很简单，但让我们用`send_mail`作为示例来讨论如何定义更复杂的依赖关系。


### <font color='red'>定义抽象和具体实现</font>

我们会设想一个更通用的通知 API。可能是电子邮件，可能是短信，将来也可能是 Slack 帖子。

```py title='ABC 和具体实现 (src/allocation/adapters/notifications.py）'
class AbstractNotifications(abc.ABC):
    @abc.abstractmethod
    def send(self, destination, message):
        raise NotImplementedError

...

class EmailNotifications(AbstractNotifications):
    def __init__(self, smtp_host=DEFAULT_HOST, port=DEFAULT_PORT):
        self.server = smtplib.SMTP(smtp_host, port=port)
        self.server.noop()

    def send(self, destination, message):
        msg = f"Subject: allocation service notification\n{message}"
        self.server.sendmail(
            from_addr="allocations@example.com",
            to_addrs=[destination],
            msg=msg,
        )
```

我们改变引导脚本中的依赖关系：

```py title='消息总线中的通知（src/allocation/bootstrap.py）'
def bootstrap(
    start_orm: bool = True,
    uow: unit_of_work.AbstractUnitOfWork = unit_of_work.SqlAlchemyUnitOfWork(),
-    send_mail: Callable = email.send,
+    notifications: AbstractNotifications = EmailNotifications(),
    publish: Callable = redis_eventpublisher.publish,
) -> messagebus.MessageBus:
```

### <font color='red'>为你的测试制作一个假版本</font>

我们研究并定义了一个用于单元测试的假版本：

```py title='假通知（tests/unit/test_handlers.py）'
class FakeNotifications(notifications.AbstractNotifications):
    def __init__(self):
        self.sent = defaultdict(list)  # type: Dict[str, List[str]]

    def send(self, destination, message):
        self.sent[destination].append(message)
...
```

我们在测试中使用它：

```py title='测试略有变化（tests/unit/test_handlers.py）'
    def test_sends_email_on_out_of_stock_error(self):
        fake_notifs = FakeNotifications()
        bus = bootstrap.bootstrap(
            start_orm=False,
            uow=FakeUnitOfWork(),
            notifications=fake_notifs,
            publish=lambda *args: None,
        )
        bus.handle(commands.CreateBatch("b1", "POPULAR-CURTAINS", 9, None))
        bus.handle(commands.Allocate("o1", "POPULAR-CURTAINS", 10))
        assert fake_notifs.sent["stock@made.com"] == [
            f"Out of stock for POPULAR-CURTAINS",
        ]
```

### <font color='red'>弄清楚如何对真实事物进行集成测试</font>

现在我们测试真实的东西，通常使用端到端或集成测试。我们已经将用[MailHog](https://github.com/mailhog/MailHog)作为 Docker 开发环境中的真实电子邮件服务器：

``` yml title='使用真实虚假电子邮件服务器的 Docker-compose 配置（docker-compose.yml）'
version: "3"

services:

  redis_pubsub:
    build:
      context: .
      dockerfile: Dockerfile
    image: allocation-image
    ...

  api:
    image: allocation-image
    ...

  postgres:
    image: postgres:9.6
    ...

  redis:
    image: redis:alpine
    ...

  mailhog:
    image: mailhog/mailhog
    ports:
      - "11025:1025"
      - "18025:8025"
```

在我们的集成测试中，我们使用真实的`EmailNotifications`类，与 Docker 集群中的 MailHog 服务器进行通信：

```py title='电子邮件集成测试（tests/integration/test_email.py）'
@pytest.fixture
def bus(sqlite_session_factory):
    bus = bootstrap.bootstrap(
        start_orm=True,
        uow=unit_of_work.SqlAlchemyUnitOfWork(sqlite_session_factory),
        notifications=notifications.EmailNotifications(),  #(1)
        publish=lambda *args: None,
    )
    yield bus
    clear_mappers()


def get_email_from_mailhog(sku):  #(2)
    host, port = map(config.get_email_host_and_port().get, ["host", "http_port"])
    all_emails = requests.get(f"http://{host}:{port}/api/v2/messages").json()
    return next(m for m in all_emails["items"] if sku in str(m))


def test_out_of_stock_email(bus):
    sku = random_sku()
    bus.handle(commands.CreateBatch("batch1", sku, 9, None))  #(3)
    bus.handle(commands.Allocate("order1", sku, 10))
    email = get_email_from_mailhog(sku)
    assert email["Raw"]["From"] == "allocations@example.com"  #(4)
    assert email["Raw"]["To"] == ["stock@made.com"]
    assert f"Out of stock for {sku}" in email["Raw"]["Data"]
```

1. 我们使用引导程序来构建与真实通知类对话的消息总线。
2. 我们弄清楚了如何从我们的“真实”电子邮件服务器中获取电子邮件。
3. 我们使用总线来进行测试设置。
4. 尽管困难重重，但这个方法实际上几乎在第一次尝试时就成功了！

事实就是这样。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px">
    <p align='center'><strong style='font-size:1.6em;color:#186d7a'>读者练习 2</strong></p>
    <p>对于适配器，你可以做两件事来练习：</p>
    <ul>
    <li>尝试使用 Twilio 或 Slack 通知将我们的通知从电子邮件替换为短信通知。你能找到与 MailHog 相当的集成测试工具吗？</li>
    <li>与我们从send_mail移动到Notifications类的方式类似，尝试重构我们的 redis_eventpublisher，它目前只是某种更正式的适配器/基类/协议的可调用。</li>
    </ul>
</div>


## <font color='red'>总结</font>

- 一旦您拥有多个适配器，您就会开始感受到手动传递依赖项的痛苦，除非您进行某种 依赖项注入。
- 设置依赖注入只是启动应用时只需执行一次的众多典型设置/初始化活动之一。将所有这些放入引导脚本中通常是一个好主意。
- 引导脚本还可以为您的适配器提供合理的默认配置，并作为为您的测试使用伪造内容覆盖这些适配器的单一位置。
- 如果您发现自己需要在多个级别执行 DI，那么依赖注入框架会很有用 - 例如，如果您拥有所有需要 DI 的组件的链式依赖关系。
- 本章还提供了一个实用示例，将隐式/简单依赖关系更改为“适当”的适配器，分解出 ABC，定义其真实和虚假实现，并进行集成测试思考。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px">
    <p align='center'><strong style='font-size:1.6em;color:#186d7a'>DI 和 Bootstrap 回顾</strong></p>
    <p>总之：</p>
    <ol>
        <li>使用 ABC 定义您的 API。</li>
        <li>落实实事。</li>
        <li>构建一个虚假的并且使用它来进行单元/服务层/处理程序测试。</li>
        <li>找到一个不太虚假的版本，并将其放入你的 Docker 环境中。</li>
        <li>测试不那么假的“真”东西。</li>
        <li>收益！</li>
    </ol>
</div>

这些是我们想要介绍的最后几种模式，这也将带我们进入[part2](./k.Part2.md)的结尾。在[结尾](./r.Epilogue.md)部分，我们将尝试为您提供一些在 Real World<sup>TM</sup>中应用这些技术的指示。