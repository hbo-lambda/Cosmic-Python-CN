# 7.聚合和一致性边界


在本章中，我们将重新回顾我们的领域模型，讨论不变量和约束，并了解我们的领域对象如何在概念上和持久存储中保持其自身的内部一致性。我们将讨论一致性边界的概念，并展示如何将其明确化，以帮助我们构建高性能软件而不损害可维护性。

图1 显示了我们前进的方向：我们将引入一个新的模型对象来`Product`包装多个批次，我们将把旧的 allocate() 域服务作为 `Product` 上的方法提供。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0701.png){ align=center }
    <figcaption><font size=2>图1.添加产品聚合</font></figcaption>
</figure>

为什么？让我们来一探究竟。

!!! tip
    本章的代码位于 [GitHub](https://github.com/cosmicpython/code/tree/chapter_07_aggregate)上的chapter_07_aggregate 分支中：
    ```
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout chapter_07_aggregate
    # or to code along, checkout the previous chapter:
    git checkout chapter_06_uow
    ```


## <font color='red'>为什么不直接在电子表格中运行所有内容？</font>

那么，领域模型的意义何在？我们要解决的根本问题是什么？

我们不能只在电子表格中运行所有内容吗？我们的许多用户会对此感到高兴。业务用户喜欢电子表格，因为它们简单、熟悉，而且功能强大。

事实上，大量的业务流程确实需要通过电子邮件来回手动发送电子表格。这种“CSV over SMTP”架构的初始复杂度较低，但往往无法很好地扩展，因为很难应用逻辑并保持一致性。

谁有权查看此特定字段？谁有权更新它？当我们尝试订购 350 把椅子或 10,000,000 张桌子时会发生什么？员工的工资可以是负数吗？

这些是系统的约束。我们编写的大部分域逻辑都是为了强制执行这些约束，以维护系统的不变量。不变量是每次我们完成操作时都必须为真的事物。


## <font color='red'>不变量、约束和一致性</font>

这两个词在某种程度上可以互换，但是约束是一种限制我们的模型可能进入的状态的规则，而不变量的 定义更精确一些，是一种始终为真的条件。

如果我们编写酒店预订系统，我们可能会有不允许重复预订的约束。这支持了同一间客房同一晚不能有多个预订的不变性。

当然，有时我们可能需要暂时改变规则。也许我们需要因为 VIP 预订而调整房间。当我们在内存中移动预订时，我们可能会被重复预订，但我们的领域模型应该确保，当我们完成后，我们最终处于最终一致的状态，即满足不变量。如果我们找不到办法容纳所有客人，我们应该抛出错误并拒绝完成操作。

让我们看一下业务需求中的几个具体示例；我们从这个开始：

!!! quote
    一个订单行每次只能分配给一个批次。
    <p align='right'>—商业</p>

这是一条强制执行不变量的业务规则。不变量是，一条订单行分配给零个或一个批次，但不会超过一个。我们需要确保我们的代码永远不会意外地为同一条订单行在两个不同的批次调用`Batch.allocate()`，目前，没有什么可以明确阻止我们这样做。


### <font color='red'>不变量、并发性和锁</font>

让我们看看另一条业务规则：

!!! quote
    如果可用数量少于订单行的数量，我们就无法分配给该批次。
    <p align='right'>—商业</p>

这里的约束是我们不能分配比批次可用库存更多的库存，因此我们绝不会通过将两个客户分配到相同的物理缓冲来超卖库存。每次我们更新系统状态时，我们的代码都需要确保我们不会破坏不变性，即可用数量必须大于或等于零。

在单线程、单用户应用程序中，我们相对容易维护这个不变量。我们可以一次分配一行库存，如果没有库存，则引发错误。

当我们引入并发概念时，这变得更加困难。突然间，我们可能同时为多个订单行分配库存。我们甚至可能在处理批次本身的更改的同时分配订单行。

我们通常通过对数据库表应用锁来解决此问题。这可以防止两个操作同时发生在同一行或同一个表上。

当我们开始考虑扩展我们的应用程序时，我们意识到根据所有可用批次分配订单行的模型可能无法扩展。如果我们每小时处理数万个订单，处理数十万个订单行，我们就无法对每个订单行都锁定整`个batches`表——我们至少会遇到死锁或性能问题。


## <font color='red'>什么是聚合？</font>

好的，那么如果我们每次想要分配订单行时都无法锁定整个数据库，我们应该怎么做呢？我们希望保护系统的不变量，但允许最大程度的并发性。维护我们的不变量不可避免地意味着防止并​​发写入；如果多个用户可以同时分配`DEADLY-SPOON`，我们就有过度分配的风险。

另一方面，我们没有理由不能同时分配 `DEADLY-SPOON` 和 `FLIMSY-DESK`。同时分配两个产品是安全的，因为没有不变量可以覆盖它们。我们不需要它们彼此一致。

修改聚合内的对象的唯一方法是加载整个对象，然后调用聚合本身的方法。

随着模型变得越来越复杂，实体和值对象越来越多，在错综复杂的图中相互引用，很难跟踪谁可以修改什么。尤其是当我们在模型中有集合时（我们的批次是一个集合），指定一些实体作为修改其相关对象的单一入口点是一个好主意。如果您指定一些对象负责其他对象的一致性，它会使系统在概念上更简单，也更容易推理。

例如，如果我们要构建一个购物网站，购物车可能是一个很好的聚合：它是我们可以视为一个单元的商品集合。重要的是，我们希望将整个购物篮作为数据存储中的单个 blob 加载。我们不希望两个请求同时修改购物篮，否则我们会面临奇怪的并发错误的风险。相反，我们希望对购物篮的每次更改都在单个数据库事务中运行。

我们不想在一次交易中修改多个购物篮，因为没有同时更改多个客户的购物篮的用例。每个购物篮都是一个单一的一致性边界，负责维护自己的不变量。

!!! quote
    “AGGREGATE 是一组关联对象的集群，我们将其视为数据更改的一个单位。”
    <p align='right'>— Eric Evans</p>
    <p align='right'>—领域驱动蓝皮书</p>

根据 Evans 的说法，我们的聚合有一个根实体（购物车），用于封装对商品的访问。每个商品都有自己的身份，但系统的其他部分始终将购物车视为一个不可分割的整体。

!!! tip
    正如我们有时将`_leading_underscores`方法或函数标记为“私有”一样，您可以将聚合视为我们模型的“公共”类，而将其余实体和值对象视为“私有”。


## <font color='red'>选择聚合</font>

我们应该为系统使用哪种聚合？选择有点随意，但很重要。聚合将成为我们确保每个操作都以一致状态结束的边界。这有助于我们推理我们的软件并防止奇怪的竞争问题。我们希望围绕少数对象（越小，性能越好）划定边界，这些对象必须彼此一致，我们需要给这个边界起一个好名字。

我们在幕后操作的对象是`Batch`。我们把批次集合称为什么？我们应该如何将系统中的所有批次划分为离散的一致性孤岛吗？

我们可以使用`Shipment`作为边界。每批货物包含多个批次，并且它们同时运往我们的仓库。或者我们可以使用`Warehouse`作为边界：每个仓库包含多个批次，同时计算所有库存是有意义的。

但这两个概念都不能真正满足我们的需求。我们应该能够一次性分配`DEADLY-SPOONs`，`FLIMSY-DESKs`，即使它们不在同一个仓库或同一批货物中。这些概念的粒度是错误的。

当我们分配订单行时，我们只对与订单行具有相同 SKU 的批次感兴趣。类似`GlobalSkuStock`这样的概念可以发挥作用：给定 SKU 的所有批次的集合。

不过，这个名字不太好记，所以在`SkuStock`、`Stock`、`ProductStock`等一系列改装后，我们决定简单地称呼它为`Product`——毕竟，这是我们在[1.领域建模]Product中探索领域语言时遇到的第一个概念。

因此计划是这样的：当我们想要分配订单行时，而不是图2的样子，我们查找所有的`Batch`对象并将它们传递给`allocate()`域服务……

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0702.png){ align=center }
    <figcaption><font size=2>图2.之前：使用域服务针对所有批次进行分配</font></figcaption>
</figure>

…​我们将进入图3的样子，其中针对我们特定 SKU 的订单行有一个`Product`新对象，它将负责该 SKU 的所有批次，我们可以调用`.allocate()`方法来替换。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0703.png){ align=center }
    <figcaption><font size=2>图3.之后：要求产品根据其批次进行分配</font></figcaption>
</figure>

让我们看看其代码形式是怎样的：

```py title='我们选择的聚合，产品（src/allocation/domain/model.py）'
class Product:
    def __init__(self, sku: str, batches: List[Batch]):
        self.sku = sku  #(1)
        self.batches = batches  #(2)

    def allocate(self, line: OrderLine) -> str:  #(3)
        try:
            batch = next(b for b in sorted(self.batches) if b.can_allocate(line))
            batch.allocate(line)
            return batch.reference
        except StopIteration:
            raise OutOfStock(f"Out of stock for sku {line.sku}")
```

1. `Product`的主要标识符是`sku`。
2. 我们的`Product`保存了对该 SKU `batches` 集合的引用。
3. 最后，我们可以将`allocate()`领域服务移为`Product`聚合上的方法。

!!! note
    这`Product`可能看起来不像您期望的`Product`模型。没有价格、没有描述、没有尺寸。我们的分配服务不关心任何这些事情。这就是有界上下文的力量；一个应用程序中的产品概念可能与另一个应用程序中的产品概念截然不同。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>聚合、有界上下文和微服务</strong></p>
<p><a src='https://martinfowler.com/bliki/BoundedContext.html'>Evans和DDD社区最重要的贡献之一是“有界上下文” </a>的概念 </p>
<p>本质上，这是对试图将整个业务纳入单一模型的回应。客户这个词对销售、客户服务、物流、支持等行业的人来说意味着不同的东西。在一个上下文中需要的属性在另一个上下文中是无关紧要的；更有害的是，同名的概念在不同的上下文中可能具有完全不同的含义。与其试图构建一个单一的模型（或类或数据库）来捕获所有用例，不如拥有多个模型，在每个上下文周围划定界限，并明确处理不同上下文之间的转换。</p>
<p>这个概念可以很好地转化为微服务的世界，其中每个微服务都可以自由地拥有自己的“客户”概念以及与其集成的其他微服务之间进行转换的规则。</p>
<p>在我们的示例中，分配服务有<code>Product(sku, batches)</code>，而电子商务将有<code>Product(sku, description, price, image_url, dimensions, etc…​)</code>。根据经验法则，您的域模型应仅包含执行计算所需的数据。</p>
<p>无论您是否拥有微服务架构，选择聚合时的一个关键考虑因素也是选择它们将在其中运行的有界上下文。通过限制上下文，您可以保持聚合数量较少且其大小易于管理。</p>
<p>再次，我们不得不说，我们无法在这里对这个问题给予应有的处理，我们只能鼓励你在其他地方阅读它。本侧边栏开头的 Fowler 链接是一个很好的起点，而且任何一本（或者实际上任何一本） DDD 书籍都会有一章或更多关于有界上下文的内容。</p>
</div>


## <font color='red'>一个聚合 = 一个存储库</font>

一旦将某些实体定义为聚合，我们就需要应用规则，即它们是唯一可供外界公开访问的实体。换句话说，我们被允许的唯一存储库应该是返回聚合的存储库。

!!! note
    存储库应仅返回聚合的规则是我们强制执行惯例的主要地方，即聚合是进入域模型的唯一途径。小心不要打破它！

在我们的例子中，我们将从`BatchRepository`切换到`ProductRepository`：

```py title='我们的新 UoW 和存储库（unit_of_work.py 和 storage.py）'
class AbstractUnitOfWork(abc.ABC):
    products: repository.AbstractProductRepository

...

class AbstractProductRepository(abc.ABC):

    @abc.abstractmethod
    def add(self, product):
        ...

    @abc.abstractmethod
    def get(self, sku) -> model.Product:
        ...
```

ORM 层需要进行一些调整，以便自动加载正确的批次并将其与`Product`对象关联。好消息是，存储库模式意味着我们暂时不必担心这一点。我们只需使用我们的`FakeRepository`，然后将新模型输入到我们的服务层，看看它`Product`作为其主要入口点的样子：

```py title='服务层（src/allocation/service_layer/services.py）'
def add_batch(
    ref: str, sku: str, qty: int, eta: Optional[date],
    uow: unit_of_work.AbstractUnitOfWork,
):
    with uow:
        product = uow.products.get(sku=sku)
        if product is None:
            product = model.Product(sku, batches=[])
            uow.products.add(product)
        product.batches.append(model.Batch(ref, sku, qty, eta))
        uow.commit()


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


## <font color='red'>性能如何？</font>

我们曾多次提到，我们使用聚合建模是因为我们想要拥有高性能软件，但在这里我们加载了所有批次，而我们只需要一个批次。您可能认为这样做效率不高，但有几个原因让我们对此感到满意。

首先，我们特意对数据进行建模，以便我们只需对数据库进行一次查询即可读取数据，只需进行一次更新即可保存更改。这往往比发出大量临时查询的系统性能要好得多。在没有以这种方式建模的系统中，我们经常发现，随着软件的发展，事务会逐渐变得更长、更复杂。

其次，我们的数据结构非常精简，每行仅包含几个字符串和整数。我们可以在几毫秒内轻松加载数十甚至数百个批次。

第三，我们预计每次每种产品只有 20 批左右。一旦一批产品用完，我们就可以从计算中扣除。这意味着我们获取的数据量不会随着时间的推移而失控。

如果我们确实预计某个产品会有数千个活动批次，那么我们有几个选择。首先，我们可以对产品中的批次使用延迟加载。从我们的代码的角度来看，不会发生任何变化，但在后台，SQLAlchemy 会为我们分页浏览数据。这将导致更多请求，每个请求获取的行数较少。因为我们只需要找到一个具有足够容量来处理我们的订单的批次，所以这可能非常有效。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>读者练习</strong></p>
<p>您刚刚看到了代码的主要顶层，所以这应该不太难，但我们希望您从Batch开始实现Product聚合，就像我们一样。</p>
<p>当然，你可以从前面的清单中作弊并复制/粘贴，但即使你这样做，你仍然需要自己解决一些挑战，比如将模型添加到 ORM 并确保所有移动部件可以相互通信，我们希望这会有所帮助。</p>
<p><a src='https://github.com/cosmicpython/code/tree/chapter_07_aggregate_exercise'>您可以在 GitHub 上</a>找到代码。我们在现有 allocate() 函数的委托中加入了“作弊”实现，因此您应该能够将其发展为真正的东西。</p>
<p>我们用 标记了几个测试@pytest.skip()。阅读完本章的其余部分后，请返回这些测试，尝试实现版本号。如果您能让 SQLAlchemy 神奇地为您完成这些操作，您将获得加分！</p>
</div>

如果其他方法都失败了，我们只需寻找不同的聚合。也许我们可以按地区或仓库分批。也许我们可以围绕装运概念重新设计数据访问策略。聚合模式旨在帮助管理一致性和性能方面的一些技术限制。没有一个正确的聚合，如果我们发现我们的边界导致性能问题，我们应该放心地改变主意。


## <font color='red'>使用版本号的乐观并发</font>

我们有新的聚合，所以我们解决了选择负责一致性边界的对象的概念问题。现在让我们花点时间讨论如何在数据库级别强制执行数据完整性。

!!! note
    本节包含许多实现细节；例如，其中一些是 Postgres 特有的。但更一般地说，我们展示了一种管理并发问题的方法，但这只是一种方法。该领域的实际要求因项目而异。您不应期望能够将此处的代码复制并粘贴到生产中。

我们不想锁定整个`batches`表，但我们如何实现仅锁定特定 SKU 的行呢？

一个答案是，在`Product`模型上设置一个属性，作为整个状态更改完成的标记，并将其用作并发工作者可以争夺的单一资源。如果两个事务同时读取`batches`状态，并且都想更新`allocations`表，我们会强制两者也尝试更新`products`表`version_number`的状态，这样只有其中一个可以获胜，数据保持一致。

图4说明两个并发事务同时执行读取操作，因此它们看到一个`Product`，例如version=3。它们都调用`Product.allocate()`以修改状态。但我们设置了数据库完整性规则，使得只有其中一个被允许对`Product`进行`version=4`的`commit`，而另一个更新被拒绝。

!!! tip
    版本号只是实现乐观锁定的一种方式。您可以通过将 Postgres 事务隔离级别设置为来实现相同的目的SERIALIZABLE，但这通常会带来严重的性能损失。版本号还可以使隐含的概念变得明确。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0704.png){ align=center }
    <figcaption><font size=2>图4.序列图：两个事务尝试对 进行并发更新Product</font></figcaption>
</figure>

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>乐观并发控制和重试</strong></p>
<p>我们在这里实现的称为乐观并发控制，因为我们的默认假设是，当两个用户想要更改数据库时，一切都会顺利。我们认为他们之间不太可能发生冲突，所以我们让他们继续，只是确保我们有办法注意到是否存在问题。</p>
<p>悲观并发控制的工作原理是假设两个用户将引发冲突，而我们希望在所有情况下都防止冲突，因此我们锁定所有内容以确保安全。在我们的示例中，这意味着锁定整个batches表，或使用 SELECT FOR UPDATE — 我们假装出于性能原因排除了这些，但在现实生活中，您需要自己进行一些评估和测量。</p>
<p>使用悲观锁定，您无需考虑处理故障，因为数据库会为您防止故障（尽管您需要考虑死锁）。使用乐观锁定，您需要明确处理冲突情况下（希望不太可能）发生故障的可能性。</p>
<p>处理失败的常用方法是从头重试失败的操作。假设我们有两个客户，Harry 和 Bob，每个人都提交了一份订单SHINY-TABLE。两个线程都以版本 1 加载产品并分配库存。数据库阻止并发更新，Bob 的订单失败并出现错误。当我们重试操作时，Bob 的订单以版本 2 加载产品并尝试再次分配。如果剩余库存足够，则一切正常；否则，他将收到OutOfStock。在出现并发问题的情况下，大多数操作都可以通过这种方式重试。</p>
</div>

### <font color='red'>版本号的实现选项</font>
实现版本号基本上有三种选项：

1. `version_number`存在于域中；我们将其添加到`Product`构造函数中，并Product.allocate()负责增加它。
2. 服务层可以做到！版本号并不是严格意义上的领域关注点，因此我们的服务层可以假设当前版本号由存储库附加到`Product`上，并且服务层将在执行`commit()`之前增加它。
3. 由于这可以说是一个基础设施问题，因此 UoW 和存储库可以神奇地做到这一点。存储库可以访问它检索到的任何产品的版本号，并且当 UoW 进行提交时，它可以增加它所知道的任何产品的版本号，假设它们已经更改。

选项 3 并不理想，因为没有真正的方法可以做到这一点，而不必假设所有产品都已发生更改，因此，当不需要时，我们会增加版本号。

选项 2 涉及混合服务层和领域层之间改变状态的责任，因此也有点混乱。

因此，最后，即使版本号不必成为领域关注的问题，您也可能认为最干净的权衡是将它们放在域中：

```py title='我们选择的聚合，产品（src/allocation/domain/model.py）'
class Product:
    def __init__(self, sku: str, batches: List[Batch], version_number: int = 0):  #(1)
        self.sku = sku
        self.batches = batches
        self.version_number = version_number  #(1)

    def allocate(self, line: OrderLine) -> str:
        try:
            batch = next(b for b in sorted(self.batches) if b.can_allocate(line))
            batch.allocate(line)
            self.version_number += 1  #(1)
            return batch.reference
        except StopIteration:
            raise OutOfStock(f"Out of stock for sku {line.sku}")
```

1. 就在那儿！

!!! tip
    如果您对版本号感到困惑，那么记住版本号并不重要可能会有所帮助。重要的是，每当我们对`Product`聚合进行更改时，数据库行都会被修改。版本号是一种简单、人性化的方式，用于对每次写入时都会发生变化的事物进行建模，但它也可以每次都是一个随机的 UUID。


## <font color='red'>测试我们的数据完整性规则</font>

现在要确保我们可以得到我们想要的行为：如果我们有两个并发尝试对同一个`Product`进行分配，其中一个应该失败，因为它们不能同时更新版本号。

首先，让我们使用一个先分配内存然后显式休眠的函数来模拟一个“慢速”事务：

```py title='time.sleep 可以重现并发行为（tests/integration/test_uow.py）'
def try_to_allocate(orderid, sku, exceptions):
    line = model.OrderLine(orderid, sku, 10)
    try:
        with unit_of_work.SqlAlchemyUnitOfWork() as uow:
            product = uow.products.get(sku=sku)
            product.allocate(line)
            time.sleep(0.2)
            uow.commit()
    except Exception as e:
        print(traceback.format_exc())
        exceptions.append(e)
```

然后我们让测试使用线程同时调用这个缓慢的分配两次：

```py title='并发行为的集成测试（tests/integration/test_uow.py）'
def test_concurrent_updates_to_version_are_not_allowed(postgres_session_factory):
    sku, batch = random_sku(), random_batchref()
    session = postgres_session_factory()
    insert_batch(session, batch, sku, 100, eta=None, product_version=1)
    session.commit()

    order1, order2 = random_orderid(1), random_orderid(2)
    exceptions = []  # type: List[Exception]
    try_to_allocate_order1 = lambda: try_to_allocate(order1, sku, exceptions)
    try_to_allocate_order2 = lambda: try_to_allocate(order2, sku, exceptions)
    thread1 = threading.Thread(target=try_to_allocate_order1)  #(1)
    thread2 = threading.Thread(target=try_to_allocate_order2)  #(1)
    thread1.start()
    thread2.start()
    thread1.join()
    thread2.join()

    [[version]] = session.execute(
        "SELECT version_number FROM products WHERE sku=:sku",
        dict(sku=sku),
    )
    assert version == 2  #(2)
    [exception] = exceptions
    assert "could not serialize access due to concurrent update" in str(exception)  #(3)

    orders = session.execute(
        "SELECT orderid FROM allocations"
        " JOIN batches ON allocations.batch_id = batches.id"
        " JOIN order_lines ON allocations.orderline_id = order_lines.id"
        " WHERE order_lines.sku=:sku",
        dict(sku=sku),
    )
    assert orders.rowcount == 1  #(4)
    with unit_of_work.SqlAlchemyUnitOfWork() as uow:
        uow.session.execute("select 1")
```

1. 我们启动两个线程，它们将可靠地产生我们想要的并发行为：`read1`, `read2`, `write1`, `write2`。
2. 我们断言版本号仅增加了一次。
3. 如果愿意的话，我们还可以检查具体的异常。
4. 我们再次检查确保只有一项分配通过。


### <font color='red'>使用数据库事务隔离级别强制执行并发规则</font>

为了使测试通过，我们可以在会话上设置事务隔离级别：

```py title='设置会话的隔离级别（src/allocation/service_layer/unit_of_work.py）'
DEFAULT_SESSION_FACTORY = sessionmaker(
    bind=create_engine(
        config.get_postgres_uri(),
        isolation_level="REPEATABLE READ",
    )
)
```

!!! tip
    事务隔离级别是一个比较棘手的问题，因此值得花时间去理解[Postgres 文档](https://www.postgresql.org/docs/12/transaction-iso.html)。


### <font color='red'>悲观并发控制示例：SELECT FOR UPDATE</font>

有多种方法可以解决这个问题，但我们将展示其中一种。`SELECT FOR UPDATE` 产生不同的行为；两个并发事务将不允许同时对同一行进行读取：

`SELECT FOR UPDATE`是一种选择一行或多行用作锁的方法（尽管这些行不必是您更新的行）。如果两个事务同时尝试访问`SELECT FOR UPDATE`一行，则其中一个事务将获胜，而另一个事务将等待直到锁被释放。所以这是悲观并发控制的一个例子。

以下是使用 SQLAlchemy DSLFOR UPDATE在查询时指定的方法：

```py title='SQLAlchemy with_for_update（src/allocation/adapters/repository.py）'
    def get(self, sku):
        return (
            self.session.query(model.Product)
            .filter_by(sku=sku)
            .with_for_update()
            .first()
        )
```

这将改变并发模式

```
read1, read2, write1, write2(失败)
```

到

```
read1, write1, read2, write2(成功)
```

有些人将此称为“读取-修改-写入”故障模式。阅读“[PostgreSQL 反模式：读取-修改-写入循环](https://www.2ndquadrant.com/en/blog/postgresql-anti-patterns-read-modify-write-cycles/)”可获得全面概述。

 我们确实没有时间讨论`REPEATABLE READ`和`SELECT FOR UPDATE`之间的所有权衡，或者一般而言乐观锁定与悲观锁定之间的权衡。但是，如果您有一个像我们展示的那样的测试，您可以指定所需的行为并查看它如何变化。您还可以使用测试作为执行一些性能实验的基础。


## 总结

根据业务情况和存储技术选择，并发控制的具体选择会有很大差异，但我们希望将本章带回到聚合的概念思想：我们明确地将一个对象建模为我们模型某些子集的主要入口点，并负责执行适用于所有这些对象的不变量和业务规则。

选择正确的聚合是关键，而且你可能会随着时间的推移重新考虑这个决定。你可以在多本 DDD 书籍中阅读更多相关内容。我们还推荐 Vaughn Vernon（“红皮书”作者）撰写的有关有效聚合设计的这三篇在线论文。

表1对于实现聚合模式的权衡有一些想法

<font color='#186d7a'>表1.聚合：权衡</font>

|优点|缺点|
|---|---|
|Python 可能没有“官方”的公共和私有方法，但我们有下划线约定，因为尝试指示哪些是供“内部”使用以及哪些是供“外部代码”使用通常很有用。选择聚合只是下一个级别：它让您决定哪些域模型类是公共的，哪些不是。|对于新开发人员来说，这又是一个需要学习的新概念。解释实体与值对象已经是一个心理负担；现在还有第三种类型的域模型对象？|
|围绕明确的一致性边界对我们的操作进行建模有助于我们避免 ORM 的性能问题。|严格遵守每次只修改一个总量的规则，是一种巨大的思维转变。|
|让聚合独自负责其子模型的状态变化使得系统更容易推理，并且更容易控制不变量。|处理聚合之间的最终一致性可能很复杂。|

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>聚合和一致性边界回顾</strong></p>
<p><strong>聚合是进入领域模型的入口点</strong></p>
<p>通过限制改变事物的方式数量，我们使得系统更容易推理。</p>
<p><strong>聚合负责一致性边界</strong></p>
<p>聚合的作用是能够管理适用于一组相关对象的不变量业务规则。聚合的作用是检查其职责范围内的对象是否彼此一致以及是否符合我们的规则，并拒绝违反规则的更改。</p>
<p><strong>聚合和并发问题相伴而生</strong></p>
<p>在考虑实施这些一致性检查时，我们最终会考虑事务和锁。选择正确的聚合不仅关系到性能，还关系到域的概念组织。</p>
</div>


## <font color='red'>第一部分回顾</font>

还记得第一部分末尾的应用程序组件图吗（图5）？我们在[第一部分](./c.Part1.md)开头展示的该图表预览了我们的前进方向？

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0705.png){ align=center }
    <figcaption><font size=2>图5.第一部分末尾的应用程序组件图</font></figcaption>
</figure>

这就是第一部分的内容。我们取得了什么成就？我们已经了解了如何构建一个由一组高级单元测试执行的领域模型。我们的测试是活文档：它们以易读的代码描述我们系统的行为（我们与业务利益相关者商定的规则）。当我们的业务需求发生变化时，我们相信我们的测试将帮助我们证明新功能，当新开发人员加入项目时，他们可以阅读我们的测试以了解事情的工作原理。

我们已经将系统的基础设施部分（如数据库和 API 处理程序）分离，以便我们可以将它们插入到应用程序外部。这有助于我们保持代码库井然有序，并防止我们构建一个大泥球。

通过应用依赖倒置原则，并使用受端口和适配器启发的模式（如存储库和工作单元），我们既可以在高速档位，也可以在低速档位进行 TDD，并保持健康的测试金字塔。我们可以对系统进行端到端的测试，并将集成和端到端测试的需求降至最低。

最后，我们讨论了一致性边界的概念。我们不想在进行更改时锁定整个系统，因此我们必须选择哪些部分是彼此一致的。

对于小型系统，这就是您尝试领域驱动设计理念所需的一切。现在，您拥有了构建与数据库无关的领域模型的工具，这些模型代表了业务专家的共同语言。太棒了！

!!! note
    冒着过度强调的风险——我们一直在努力指出每个模式都是有代价的。每一层间接性都会增加代码的复杂性和重复性，并且会让从未见过这些模式的程序员感到困惑。如果您的应用程序本质上是数据库的简单 CRUD 包装器，并且在可预见的未来不太可能有更多功能，那么您就不需要这些模式。继续使用 Django，省去很多麻烦。

在第二部分中，我们将缩小范围并讨论一个更大的话题：如果聚合是我们的边界，并且我们一次只能更新一个，那么我们如何对跨越一致性边界的过程进行建模？
