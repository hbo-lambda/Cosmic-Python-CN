# 4.我们的第一个用例：Flask API和服务层


回到我们的分配项目！图1显示了我们在 [2.存储库模式](./e.Repository%20Pattern.md) 结束时达到的点，该章节介绍了存储库模式。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0401.png){ align=center }
    <figcaption><font size=2>图1.之前：我们通过与资源库和领域模型对话来驱动应用程序</font></figcaption>
</figure>

在本章中，我们讨论了编排逻辑、业务逻辑和接口代码之间的区别，并引入了**服务层模式**来负责编排我们的工作流程并定义系统的用例。

我们还将讨论测试：通过将服务层与我们对数据库的存储库抽象结合起来，我们能够编写快速测试，不仅仅是领域模型的用例，还有整个工作流程的用例。

如图2，我们的目标是添加一个`Flask API`，该API将与服务层通信，服务层将作为我们领域模型的入口。由于我们的服务层依赖于`AbstractRepository`，因此我们可以使用`FakeRepository`进行单元测试，使用`SqlAlchemyRepository`运行我们的生产代码。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0402.png){ align=center }
    <figcaption><font size=2>图2.服务层将成为我们应用程序的主要入口</font></figcaption>
</figure>

在我们的图表中，我们使用的惯例是用粗体文本/线条突出显示新组件(如果您正在阅读数字版本，则使用黄色/橙色)。

!!! tip
    本章的代码位于 [GitHub](https://github.com/cosmicpython/code/tree/chapter_04_service_layer) 上的 Chapter_04_service_layer 分支中
    ```
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout chapter_04_service_layer
    # or to code along, checkout Chapter 2:
    git checkout chapter_02_repository
    ```


## <font color='red'>将我们的应用程序与现实世界联系起来</font>

和任何优秀的敏捷团队一样，我们正在努力将 MVP 推向用户，开始收集反馈。我们有领域模型的核心和分配订单所需的领域服务，还有用于永久存储的存储库接口。

让我们尽快将所有活动部件整合在一起，然后重构为更清晰的架构。我们的计划如下：

1. 使用Flask在我们的`allocate`领域服务前放置一个API端点。连接数据库会话和我们的存储库。通过端到端测试和一些快速而粗略的SQL来准备测试数据。
2. 重构出一个服务层，它可以作为一个抽象层来捕获用例，并位于Flask和我们的领域模型之间。构建一些服务层测试并展示它们如何使用`FakeRepository`。
3. 对我们的服务层功能尝试不同类型的参数；表明使用原始数据类型可以使服务层的客户端（我们的测试和我们的 Flask API）与模型层分离。


## <font color='red'>首次端到端测试</font>

没人有兴趣卷入一场关于端到端 (E2E) 测试、功能测试、验收测试、集成测试和单元测试的长期术语争论。不同的项目需要不同的测试组合，我们已经看到非常成功的项目只是将测试分为“快速测试”和“慢速测试”。

现在，我们想编写一个或两个测试，用于测试“真实”API 端点（使用 HTTP）并与真实数据库通信。我们称它们为端到端测试，因为它是最不言自明的名称之一。

以下显示的是第一部分：

``` py title='第一个 API 测试 (test_api.py）'
@pytest.mark.usefixtures("restart_api")
def test_api_returns_allocation(add_stock):
    sku, othersku = random_sku(), random_sku("other")  #(1)
    earlybatch = random_batchref(1)
    laterbatch = random_batchref(2)
    otherbatch = random_batchref(3)
    add_stock(  #(2)
        [
            (laterbatch, sku, 100, "2011-01-02"),
            (earlybatch, sku, 100, "2011-01-01"),
            (otherbatch, othersku, 100, None),
        ]
    )
    data = {"orderid": random_orderid(), "sku": sku, "qty": 3}
    url = config.get_api_url()  #(3)

    r = requests.post(f"{url}/allocate", json=data)

    assert r.status_code == 201
    assert r.json()["batchref"] == earlybatch
```

1. `random_sku()`,`random_batchref()`, 等等，都是使用`uuid`模块生成随机字符的小辅助函数。因为我们现在针对的是实际数据库，所以这是防止各种测试和运行相互干扰的一种方法。

2. `add_stock` 是一个辅助的 fixture，它隐藏了使用 SQL 手动将行插入数据库的细节。我们将在本章后面展示一种更好的方法。

3. `config.py` 是我们保存配置信息的模块。

每个人解决这些问题的方式都不同，但你需要一种方法来启动 Flask，可能是在一个容器中，并与 Postgres 数据库进行通信。如果你想看看我们是如何做到的，可以查看[附录B 项目结构](./t.Appendix%20B.md)。


## <font color='red'>简单的实现</font>

以最明显的方式实现，你可能会得到这样的结果：

```py title='Flask 应用程序的初版（flask_app.py）'
from flask import Flask, request
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

import config
import model
import orm
import repository


orm.start_mappers()
get_session = sessionmaker(bind=create_engine(config.get_postgres_uri()))
app = Flask(__name__)


@app.route("/allocate", methods=["POST"])
def allocate_endpoint():
    session = get_session()
    batches = repository.SqlAlchemyRepository(session).list()
    line = model.OrderLine(
        request.json["orderid"], request.json["sku"], request.json["qty"],
    )

    batchref = model.allocate(line, batches)

    return {"batchref": batchref}, 201
```

这么远，看起来还不错。Bob和Harry，你们可能会想，没必要再说你们的“架构宇航员”的废话了。

但是稍等一下——没有提交。我们实际上没有将我们的分配保存到数据库中。现在我们需要进行第二个测试，要么检查数据库状态（不是非常黑盒），要么是检查如果第一行已经耗尽批次，我们是否不能分配第二行：

``` py title='测试分配的持久化（test_api.py）'
@pytest.mark.usefixtures("restart_api")
def test_allocations_are_persisted(add_stock):
    sku = random_sku()
    batch1, batch2 = random_batchref(1), random_batchref(2)
    order1, order2 = random_orderid(1), random_orderid(2)
    add_stock(
        [(batch1, sku, 10, "2011-01-01"), (batch2, sku, 10, "2011-01-02"),]
    )
    line1 = {"orderid": order1, "sku": sku, "qty": 10}
    line2 = {"orderid": order2, "sku": sku, "qty": 10}
    url = config.get_api_url()

    # first order uses up all stock in batch 1
    r = requests.post(f"{url}/allocate", json=line1)
    assert r.status_code == 201
    assert r.json()["batchref"] == batch1

    # second order should go to batch 2
    r = requests.post(f"{url}/allocate", json=line2)
    assert r.status_code == 201
    assert r.json()["batchref"] == batch2
```

虽然不太可爱，但这将迫使我们添加提交


## <font color='red'>需要数据库检查的错误情况</font>

如果我们继续这样下去，情况会变得越来越糟。

假设我们想添加一些错误处理。如果域引发错误，例如 SKU 缺货怎么办？或者 SKU 根本不存在怎么办？域甚至不知道，也不应该知道。这更像是一种健全性检查，我们应该在数据库层实现，甚至在调用域服务之前。

现在我们来看看另外两个端到端测试：

``` py title='在E2E层进行更多的测试（test_api.py）'
@pytest.mark.usefixtures("restart_api")
def test_400_message_for_out_of_stock(add_stock):  #(1)
    sku, small_batch, large_order = random_sku(), random_batchref(), random_orderid()
    add_stock(
        [(small_batch, sku, 10, "2011-01-01"),]
    )
    data = {"orderid": large_order, "sku": sku, "qty": 20}
    url = config.get_api_url()
    r = requests.post(f"{url}/allocate", json=data)
    assert r.status_code == 400
    assert r.json()["message"] == f"Out of stock for sku {sku}"


@pytest.mark.usefixtures("restart_api")
def test_400_message_for_invalid_sku():  #(2)
    unknown_sku, orderid = random_sku(), random_orderid()
    data = {"orderid": orderid, "sku": unknown_sku, "qty": 20}
    url = config.get_api_url()
    r = requests.post(f"{url}/allocate", json=data)
    assert r.status_code == 400
    assert r.json()["message"] == f"Invalid sku {unknown_sku}"
```

1. 在第一个测试中，我们尝试分配比库存更多的单位。
2. 在第二个测试中，SKU 不存在（因为我们从未调用过`add_stock`），因此就我们的应用程序而言，它是无效的。

当然，我们也可以在 Flask 应用程序中实现它：

``` py title='Flask 应用程序开始变得复杂（flask_app.py）'
def is_valid_sku(sku, batches):
    return sku in {b.sku for b in batches}


@app.route("/allocate", methods=["POST"])
def allocate_endpoint():
    session = get_session()
    batches = repository.SqlAlchemyRepository(session).list()
    line = model.OrderLine(
        request.json["orderid"], request.json["sku"], request.json["qty"],
    )

    if not is_valid_sku(line.sku, batches):
        return {"message": f"Invalid sku {line.sku}"}, 400

    try:
        batchref = model.allocate(line, batches)
    except model.OutOfStock as e:
        return {"message": str(e)}, 400

    session.commit()
    return {"batchref": batchref}, 201
```

但我们的 Flask 应用开始显得有点笨重。而且我们的 E2E 测试数量开始失控，很快我们就会得到一个倒置的测试金字塔（或者 Bob 喜欢称之为“冰淇淋锥模型”）


## <font color='red'>引入服务层并使用FakeRepository进行单元测试。</font>

当我们审视我们的 Flask 应用程序时，我们会发现有相当多的编排工作——从存储库中获取数据，根据数据库状态验证输入，处理错误，并在正常情况下进行提交。大多数这些操作与创建 Web API 端点无关（例如，如果您正在构建 CLI，您将需要它们；请参阅 [附录C csvs](./u.Appendix%20C.md)），而且它们实际上不需要通过端到端测试来进行测试。

分离出一个服务层（有时称为 业务流程层或用例层）通常是有意义的。

您还记得我们在 [3.解耦和抽象](./f.Coupling%20and%20Abstractions.md) 中准备的`FakeRepository`吗？

``` py title='我们的假存储库，内存中的批次集合（test_services.py）'
class FakeRepository(repository.AbstractRepository):
    def __init__(self, batches):
        self._batches = set(batches)

    def add(self, batch):
        self._batches.add(batch)

    def get(self, reference):
        return next(b for b in self._batches if b.reference == reference)

    def list(self):
        return list(self._batches)
```

这就是它的用处所在；它让我们用优秀、快速的单元测试来测试我们的服务层：

``` py title='在服务层用 fakes 进行单元测试（test_services.py）'
def test_returns_allocation():
    line = model.OrderLine("o1", "COMPLICATED-LAMP", 10)
    batch = model.Batch("b1", "COMPLICATED-LAMP", 100, eta=None)
    repo = FakeRepository([batch])  #(1)

    result = services.allocate(line, repo, FakeSession())  #(2) (3)
    assert result == "b1"


def test_error_for_invalid_sku():
    line = model.OrderLine("o1", "NONEXISTENTSKU", 10)
    batch = model.Batch("b1", "AREALSKU", 100, eta=None)
    repo = FakeRepository([batch])  #(1)

    with pytest.raises(services.InvalidSku, match="Invalid sku NONEXISTENTSKU"):
        services.allocate(line, repo, FakeSession())  #(2) (3)
```

1. `FakeRepository` 持有将在我们的测试中使用的`Batch`对象。

2. 我们的服务模块（services.py）将定义一个`allocate()`服务层函数。它将位于我们 API 层中的`allocate_endpoint()`函数和我们领域模型中的`allocate()`领域服务函数之间。

3. 我们还需要一个`FakeSession`来伪造数据库会话，如下面的代码片段所示。

``` py title='假的数据库会话（test_services.py）'
class FakeSession:
    committed = False

    def commit(self):
        self.committed = True
```

这个假的会话只是一个临时的解决方案。我们在[6.工作单元模式](./i.Unit%20of%20Work%20Pattern.md)中摆脱它，让事情变得更好。但与此同时，假的`.commit()`可以让我们从 E2E 层迁移第三个测试。

``` py title='服务层的第二次测试（test_services.py）'
def test_commits():
    line = model.OrderLine("o1", "OMINOUS-MIRROR", 10)
    batch = model.Batch("b1", "OMINOUS-MIRROR", 100, eta=None)
    repo = FakeRepository([batch])
    session = FakeSession()

    services.allocate(line, repo, session)
    assert session.committed is True
```


### <font color='red'>典型的服务函数</font>

我们将编写一个如下所示的服务函数：

``` py title='基本分配服务（services.py）'
class InvalidSku(Exception):
    pass


def is_valid_sku(sku, batches):
    return sku in {b.sku for b in batches}


def allocate(line: OrderLine, repo: AbstractRepository, session) -> str:
    batches = repo.list()  #(1)
    if not is_valid_sku(line.sku, batches):  #(2)
        raise InvalidSku(f"Invalid sku {line.sku}")
    batchref = model.allocate(line, batches)  #(3)
    session.commit()  #(4)
    return batchref
```

典型的服务层函数通常包含以下步骤：

1. 从存储库中获取一些对象。
2. 根据当前的状态对请求进行一些检查或断言。
3. 调用领域服务。
4. 如果一切顺利，我们会保存/更新已更改的任何状态。

目前，最后一步还有点不尽如人意，因为我们的服务层与数据库层紧密耦合。我们将在[6.工作单元模式](https://example.com/chapter_06_uow)中使用工作单元对此进行改进。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>依赖抽象</strong></p>
<p>关于我们的服务层函数，还要注意一件事：</p>
<code style='border: 1px solid;'>
def allocate(line: OrderLine, repo: AbstractRepository, session) -> str:
</code>
<p>它依赖于存储库。我们选择明确依赖关系，并使用类型提示表示我们依赖于 <code>AbstractRepository</code>。这意味着测试提供的<code>FakeRepository</code>和Flask应用程序提供的<code>SqlAlchemyRepository</code>都将起作用</p>
<p>如果你还记得 [dip]，这就是我们所说的“应该依赖抽象”的意思。我们的高级模块，即服务层，依赖于存储库抽象。而我们特定选择的持久存储的实现细节也依赖于相同的抽象。请参阅图3和图4。</p>

<p>另请参阅 [附录C csvs] 中一个实用示例，该示例在保持抽象不变的情况下交换 要使用的持久存储系统的细节。</p>
</div>

但是服务层的基本内容已经存在，我们的 Flask 应用现在看起来更加简洁了：
 
```py title='Flask 应用程序委托给服务层（flask_app.py）'
@app.route("/allocate", methods=["POST"])
def allocate_endpoint():
    session = get_session()  #(1)
    repo = repository.SqlAlchemyRepository(session)  #(1)
    line = model.OrderLine(
        request.json["orderid"], request.json["sku"], request.json["qty"],  #(2)
    )

    try:
        batchref = services.allocate(line, repo, session)  #(2)
    except (model.OutOfStock, services.InvalidSku) as e:
        return {"message": str(e)}, 400  #(3)

    return {"batchref": batchref}, 201  #(3)
```

1. 我们实例化了一个数据库会话和一些存储库对象。
2. 我们从网络请求中提取用户的命令并传递给服务层。
3. 我们返回一些带有适当状态代码的 JSON 响应。

Flask应用程序的职责仅涉及标准的Web任务：每个请求的会话管理、从POST参数中提取信息、响应状态代码和JSON。所有的编排逻辑都在用例/服务层中，而领域逻辑保持在领域中。

最后，我们可以自信地将E2E测试简化为两个，一个用于正常状态的，一个用于异常状态的：

``` py title='E2E仅测试正常状态的和异常状态的（test_api.py）'
@pytest.mark.usefixtures("restart_api")
def test_happy_path_returns_201_and_allocated_batch(add_stock):
    sku, othersku = random_sku(), random_sku("other")
    earlybatch = random_batchref(1)
    laterbatch = random_batchref(2)
    otherbatch = random_batchref(3)
    add_stock(
        [
            (laterbatch, sku, 100, "2011-01-02"),
            (earlybatch, sku, 100, "2011-01-01"),
            (otherbatch, othersku, 100, None),
        ]
    )
    data = {"orderid": random_orderid(), "sku": sku, "qty": 3}
    url = config.get_api_url()

    r = requests.post(f"{url}/allocate", json=data)

    assert r.status_code == 201
    assert r.json()["batchref"] == earlybatch


@pytest.mark.usefixtures("restart_api")
def test_unhappy_path_returns_400_and_error_message():
    unknown_sku, orderid = random_sku(), random_orderid()
    data = {"orderid": orderid, "sku": unknown_sku, "qty": 20}
    url = config.get_api_url()
    r = requests.post(f"{url}/allocate", json=data)
    assert r.status_code == 400
    assert r.json()["message"] == f"Invalid sku {unknown_sku}"
```

我们已成功地将测试分为两大类：关于 Web 内容的测试（我们以端到端的方式实施）；关于编排内容的测试（我们可以针对内存中的服务层进行测试）。。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>读者练习</strong></p>
<p>既然我们已经有了一个分配服务，为什么不构建一个用于取消分配的服务呢？我们已经在<a src='https://github.com/cosmicpython/code/tree/chapter_04_service_layer_exercise'>GitHub</a>上添加了一个端到端测试和一些服务层测试，供您使用。</p>

<p>如果这还不够，您可以继续查看E2E测试和 flask_app.py，并对 Flask 适配器进行重构以使其更加 RESTful。请注意，这样做不需要对我们的服务层或领域层进行任何更改！</p>
<div class='admonition tip'>
    <p class="admonition-title">Tip</p>
    <p>如果您决定要构建一个只读端点来检索分配信息，只需执行“可能最简单的操作”，也就是在 Flask 处理程序中直接调用<code>repo.get()</code>。我们将在 [12.命令查询职责分离] 中更详细地讨论读取与写入。</p>
</div>
</div>


## <font color='red'>为什么一切都称为服务？</font>

你们中的一些人此时可能正在绞尽脑汁试图弄清楚领域服务和服务层之间到底有什么区别。

很抱歉——这些名字不是我们选择的，否则我们会有更酷、更友好的方式来谈论这些事情。

在本章中， 我们使用两种称为服务的东西。第一种是应用服务（我们的服务层）。它的工作是处理来自外部世界的请求并协调操作。我们的意思是，服务层通过遵循一系列简单的步骤来驱动应用程序：

- 从数据库获取一些数据
- 更新领域模型
- 保留所有更改

这是系统中每个操作都必须发生的无聊工作，将其与业务逻辑分开有助于保持事情整洁。

第二种类型的服务是领域服务。这是属于领域模型但并不自然存在于有状态实体或值对象中的逻辑部分的名称。例如，如果你正在构建一个购物车应用程序，则可能选择将税收规则构建为领域服务。计算税款是一项与更新购物车不同的工作，它是模型的重要组成部分，但为这项工作保留一个持久实体似乎不太合适。相反，一个无状态的`TaxCalculator`类或一个`calculate_tax`函数可以完成这项工作。


## <font color='red'>将文件放入文件夹以查看其所属于位置</font>

随着我们的应用程序变得越来越大，我们需要不断整理目录结构。我们项目的布局为我们提供了有用的提示，让我们知道在每个文件中找到什么类型的对象。

以下是我们可以组织事物的一种方法：

``` shell title='一些子文件夹'
.
├── config.py
├── domain  #(1)
│   ├── __init__.py
│   └── model.py
├── service_layer  #(2)
│   ├── __init__.py
│   └── services.py
├── adapters  #(3)
│   ├── __init__.py
│   ├── orm.py
│   └── repository.py
├── entrypoints  (4)
│   ├── __init__.py
│   └── flask_app.py
└── tests
    ├── __init__.py
    ├── conftest.py
    ├── unit
    │   ├── test_allocate.py
    │   ├── test_batches.py
    │   └── test_services.py
    ├── integration
    │   ├── test_orm.py
    │   └── test_repository.py
    └── e2e
        └── test_api.py
```

1. 让我们为域模型创建一个文件夹。目前只有一个文件，但对于更复杂的应用程序，每个类可能都有一个文件；您可能会为`Entity`、`ValueObject`和`Aggregate`创建辅助父类，并且可能会添加一个用于领域层异常的exceptions.py，以及在[第二部分](./k.Part2.md)中将看到的commands.py和events.py。
2. 我们将区分服务层。目前只有一个名为services.py的文件，用于我们的服务层。您可以在此处添加服务层异常，如[5.TDD高速和低速](./h.TDD%20in%20High%20Gear%20and%20Low%20Gear.md)中的，我们还将在此添加unit_of_work.py。
3. 适配器是对端口和适配器术语的致敬。这将填充有关外部 I/O 的任何其他抽象（例如 redis_client.py ）。严格来说，您可以称这些为辅助适配器或驱动 适配器，有时也称其为面向内的适配器。
4. 入口点是我们驱动应用程序的地方。在官方端口和适配器术语中，这些也是适配器，被称为主适配器、驱动适配器或面向外的适配器。

那么端口呢？您可能还记得，它们是适配器实现的抽象接口。我们倾向于将它们与实现它们的适配器放在同一个文件中。


## <font color='red'>总结</font>

添加了服务层确实给我们带来了很多好处：

- 我们的 Flask API 端点变得非常薄且易于编写：它们唯一的职责是执行“Web 操作”，例如解析 JSON 并为顺利或不顺利的情况生成正确的 HTTP 代码。
- 我们为我们的领域定义了一个清晰的API，一组用例或入口点，任何适配器都可以使用它们，而无需了解领域模型类——无论是API、CLI（见[附录C csvs](./u.Appendix%20C.md)）还是测试！它们也是我们领域的适配器。
- 我们可以使用服务层“高速”地编写测试，这样我们就可以自由地以任何我们认为合适的方式重构领域模型。只要我们仍然可以提供相同的用例，我们就可以尝试新的设计，而无需重写大量测试。
- 我们的测试金字塔看起来不错——我们的大部分测试都是快速单元测试，仅包含最低限度的 E2E 和集成测试。


### <font color='red'>DIP在起作用</font>

图3 展示了服务层的依赖关系：领域模型和`AbstractRepository`（端口，端口和适配器术语）

当我们运行测试时，图4 展示了我们如何使用`FakeRepository`（适配器）来实现抽象依赖（适配器）

当我们实际运行应用时，图5 展示了“真正的”依赖项。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0403.png){ align=center }
    <figcaption><font size=2>图3.服务层的抽象依赖</font></figcaption>
</figure>

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0404.png){ align=center }
    <figcaption><font size=2>图4.测试提供抽象依赖关系的实现</font></figcaption>
</figure>

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0405.png){ align=center }
    <figcaption><font size=2>图5.运行时的依赖关系</font></figcaption>
</figure>


让我们暂停一下服务层：权衡，我们考虑拥有服务层的利弊。

<font color='#186d7a'>表1.服务层：权衡</font>

| 优点                                                         | 缺点                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
|我们可以在一个地方捕获应用程序的所有用例。|如果您的应用程序纯粹是一个网络应用程序，那么您的控制器/视图函数可以是捕获所有用例的唯一地方。
|我们将巧妙的域逻辑置于 API 后面，这样我们就可以自由地重构。。|这是又一层抽象。|
|我们已经将“谈论 HTTP 的东西”与“谈论分配的东西”明确地分开了。|在服务层中放入过多逻辑可能会导致贫血域 反模式。最好在发现编排逻辑潜入控制器后再引入此层。|
|当`Repository`模式和`FakeRepository`结合使用时，我们就有了一种在比领域层更高级别编写测试的好方法；我们可以测试更多的工作流程，而不需要使用集成测试（请继续阅读 [5.TDD高速和低速](./h.TDD%20in%20High%20Gear%20and%20Low%20Gear.md) 以了解更多细节）。|您只需将逻辑从控制器推送到模型层，即可获得丰富域模型带来的诸多好处，而无需在其间添加额外的层（又称“胖模型，瘦控制器”）。|

但仍有几处尴尬之处需要解决：

- 服务层仍然与领域紧密耦合，因为其 API 是以`OrderLine`对象表示的。在 [5.TDD高速和低速](./h.TDD%20in%20High%20Gear%20and%20Low%20Gear.md) 中，我们将解决这个问题，并讨论服务层如何实现更高效的TDD。
- 服务层与`session`紧密耦合。在[6.工作单元模式](./i.Unit%20of%20Work%20Pattern.md)中，我们将介绍另一种与存储库和服务层模式紧密配合的模式，即工作单元模式，一切都将变得非常美好。您会看到！
