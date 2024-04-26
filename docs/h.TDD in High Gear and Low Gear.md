# 5.TDD高速和低速

我们引入了服务层，以捕获运行应用程序所需的一些额外编排职责。服务层可帮助我们明确定义用例和每个用例的工作流程：我们需要从存储库中获取什么、我们应该进行哪些预检查和当前状态验证，以及我们最终要保存什么。

但目前，我们的许多单元测试都在较低级别运行，直接作用于模型。在本章中，我们将讨论将这些测试移至服务层级别的权衡，以及一些更通用的测试指南。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>Harry说：看到测试金字塔的实际运作让我恍然大悟</strong></p>
<p>以下是 Harry 亲自说的几句话：</p>
<p>我最初对Bob的所有架构模式都持怀疑态度，但看到实际的测试金字塔后我改变了看法。</p>
<p>一旦实施了域建模和服务层，您实际上就可以进入单元测试数量比集成和端到端测试多一个数量级的阶段。我曾在一些地方工作过，那里的端到端测试构建需要几个小时（本质上是“等到明天”），我无法告诉您能够在几分钟或几秒钟内运行所有测试会带来多大的不同。</p>
<p>继续阅读，了解如何决定编写哪种测试以及编写到哪个级别的一些指导原则。高速与低速的思维方式确实改变了我的测试生活。</p>
</div>


## <font color='red'>我们的测试金字塔看起来怎么样？</font>

让我们看看使用服务层及其自己的服务层测试对我们的测试金字塔有什么影响：

``` py title='计算测试类型'
$ grep -c test_ */*/test_*.py
tests/unit/test_allocate.py:4
tests/unit/test_batches.py:8
tests/unit/test_services.py:3

tests/integration/test_orm.py:6
tests/integration/test_repository.py:2

tests/e2e/test_api.py:2
```

还不错！我们有 15 个单元测试、8 个集成测试和 2 个端到端测试。这已经是一个看起来很健康的测试金字塔了。


## <font color='red'>领域层测试应该移至服务层吗？</font>

让我们看看如果我们更进一步会发生什么。由于我们可以针对服务层测试我们的软件，因此我们实际上不再需要针对域模型的测试。相反，我们可以从服务层的角度重写[1.领域模型](./d.Domain%20Modeling.md)中的所有域级测试：

``` py title='在服务层重写领域测试（tests/unit/test_services.py）'
# domain-layer test:
def test_prefers_current_stock_batches_to_shipments():
    in_stock_batch = Batch("in-stock-batch", "RETRO-CLOCK", 100, eta=None)
    shipment_batch = Batch("shipment-batch", "RETRO-CLOCK", 100, eta=tomorrow)
    line = OrderLine("oref", "RETRO-CLOCK", 10)

    allocate(line, [in_stock_batch, shipment_batch])

    assert in_stock_batch.available_quantity == 90
    assert shipment_batch.available_quantity == 100


# service-layer test:
def test_prefers_warehouse_batches_to_shipments():
    in_stock_batch = Batch("in-stock-batch", "RETRO-CLOCK", 100, eta=None)
    shipment_batch = Batch("shipment-batch", "RETRO-CLOCK", 100, eta=tomorrow)
    repo = FakeRepository([in_stock_batch, shipment_batch])
    session = FakeSession()

    line = OrderLine('oref', "RETRO-CLOCK", 10)

    services.allocate(line, repo, session)

    assert in_stock_batch.available_quantity == 90
    assert shipment_batch.available_quantity == 100
```

我们为什么要这么做？

测试本应帮助我们无所畏惧地更改系统，但我们经常看到团队针对其领域模型编写了过多的测试。当他们更改代码库并发现需要更新数十甚至数百个单元测试时，这会带来问题。

如果您停下来思考一下自动化测试的目的，就会明白这一点。我们使用测试来确保系统的属性在我们工作时不会发生变化。我们使用测试来检查 API 是否继续返回 200、数据库会话是否继续提交以及订单是否仍在分配。

如果我们不小心改变了其中一种行为，我们的测试就会失败。但另一方面，如果我们想改变代码的设计，任何直接依赖该代码的测试也会失败。

随着我们进一步了解本书，您将看到服务层如何为我们的系统形成一个 API，我们可以用多种方式来驱动它。针对此 API 进行测试可以减少重构域模型时需要更改的代码量。如果我们将自己限制为仅针对服务层进行测试，我们将不会进行任何直接与模型对象上的“私有”方法或属性交互的测试，这让我们可以更自由地重构它们。

!!! tip
    我们放入测试的每一行代码都像一团胶水，将系统保持在特定的形状。我们进行的低级测试越多，改变事物就越困难。


## <font color='red'>决定编写哪种测试</font>

您可能会问自己：“那么，我应该重写所有单元测试吗？针对领域模型编写测试是错误的吗？”要回答这些问题，重要的是要了解耦合和设计反馈之间的权衡（请参阅 图1）。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0501.png){ align=center }
    <figcaption><font size=2>图1.测试频谱</font></figcaption>
</figure>

极限编程 (XP) 告诫我们要“倾听代码”。编写测试时，我们可能会发现代码难以使用或注意到代码异味。这会促使我们进行重构，并重新考虑我们的设计。

然而，只有在密切关注目标代码时，我们才能获得反馈。HTTP API 测试无法告诉我们对象的细粒度设计，因为它处于更高的抽象级别。

另一方面，我们可以重写整个应用程序，只要我们不更改 URL 或请求格式，我们的 HTTP 测试将继续通过。这让我们有信心，大规模更改（如更改数据库架构）不会破坏我们的代码。

另一方面，我们在[1.领域模型](./d.Domain%20Modeling.md)中编写的测试帮助我们充实了对所需对象的理解。这些测试引导我们进行合理且可读的领域语言设计。当我们的测试可读的领域语言时，我们会感到很放心，因为我们的代码符合我们对要解决的问题的直觉。

由于测试是用领域语言编写的，因此它们充当了我们模型的动态文档。新团队成员可以阅读这些测试，以快速了解系统的工作原理以及核心概念之间的相互关系。

我们经常通过在此级别编写测试来“勾勒”新行为，以了解代码可能是什么样子。但是，当我们想要改进代码设计时，我们需要替换或删除这些测试，因为它们与特定实现紧密相关。


## <font color='red'>高低速</font>

大多数情况下，当我们添加新功能或修复错误时，我们不需要对领域模型进行大量更改。在这些情况下，我们更喜欢针对服务编写测试，因为其耦合度较低且覆盖率较高。

例如，在编写`add_stock`函数或`cancel_order`功能时，我们可以通过针对服务层编写测试来更快地工作并减少耦合。

当开始一个新项目或遇到特别棘手的问题时，我们将重新针对领域模型编写测试，以便获得更好的反馈和可执行的意图文档。

我们用换挡来打比方。开始一段旅程时，自行车需要换到低挡，这样才能克服惯性。一旦我们开始骑行，换到高挡，我们就能骑得更快、更高效；但如果我们突然遇到陡坡，或者因为危险而被迫减速，我们就会再次降到低挡，直到能够再次加速。


## <font color='red'>将服务层测试与域完全分离</font>

我们在服务层测试中仍然直接依赖领域，因为我们使用领域对象来设置测试数据并调用服务层函数。

为了拥有一个与域完全分离的服务层，我们需要重写它的 API 以便按照原语来工作。

我们的服务层目前采用一个`OrderLine`域对象：

``` py title='之前：allocate 接受一个域对象 (service_layer/services.py）'
def allocate(line: OrderLine, repo: AbstractRepository, session) -> str:
```

如果它的参数都是原始类型，它会是什么样子？

``` py title='之后：分配采用字符串和整数（service_layer/services.py）'
def allocate(
    orderid: str, sku: str, qty: int,
    repo: AbstractRepository, session
) -> str:
```

我们也按照这些条款重写了测试：

``` py title='测试现在在函数调用中使用原语（tests/unit/test_services.py'
def test_returns_allocation():
    batch = model.Batch("batch1", "COMPLICATED-LAMP", 100, eta=None)
    repo = FakeRepository([batch])

    result = services.allocate("o1", "COMPLICATED-LAMP", 10, repo, FakeSession())
    assert result == "batch1"
```

但我们的测试仍然依赖于领域，因为我们仍然手动实例化`Batch`对象。因此，如果有一天我们决定大规模重构`Batch`模型的工作方式，我们将不得不更改一堆测试。


### <font color='red'>缓解措施：将所有域依赖项保留在 Fixture 函数中</font>

我们至少可以在测试中将其抽象为辅助函数或fixture。以下是您可以实现此目的的一种方式，在`FakeRepository`上添加工厂函数：

``` py title='fixture的工厂函数是一种可能（tests/unit/test_services.py）'
class FakeRepository(set):

    @staticmethod
    def for_batch(ref, sku, qty, eta=None):
        return FakeRepository([
            model.Batch(ref, sku, qty, eta),
        ])

    ...


def test_returns_allocation():
    repo = FakeRepository.for_batch("batch1", "COMPLICATED-LAMP", 100, eta=None)
    result = services.allocate("o1", "COMPLICATED-LAMP", 10, repo, FakeSession())
    assert result == "batch1"
```

至少这会将我们测试对域的所有依赖关系移动到一个地方。


### <font color='red'>添加缺失的服务</font>

不过，我们可以更进一步。如果我们有一个添加库存的服务，我们可以使用它，并根据服务层的官方用例完全表达我们的服务层测试，删除对域的所有依赖：

``` py title='测试新的 add_batch 服务 (tests/unit/test_services.py）'
def test_add_batch():
    repo, session = FakeRepository([]), FakeSession()
    services.add_batch("b1", "CRUNCHY-ARMCHAIR", 100, None, repo, session)
    assert repo.get("b1") is not None
    assert session.committed
```

!!! tip
    一般来说，如果您发现自己需要在服务层测试中直接执行领域层的工作，则可能表明您的服务层不完整。

实现只需要两行：

``` py title='用于 add_batch 的新服务（service_layer/services.py）'
def add_batch(
    ref: str, sku: str, qty: int, eta: Optional[date],
    repo: AbstractRepository, session,
) -> None:
    repo.add(model.Batch(ref, sku, qty, eta))
    session.commit()


def allocate(
    orderid: str, sku: str, qty: int,
    repo: AbstractRepository, session
) -> str:
```

!!! note
    您是否应该编写一项新服务，仅仅因为它有助于消除测试中的依赖关系？可能不会。但在这种情况下，我们几乎肯定有一天会需要一项`add_batch`的服务。

现在，我们可以纯粹根据服务本身重写所有服务层测试，只使用原语，而不依赖于模型：

``` py title='服务测试现在仅使用服务（tests/unit/test_services.py）'
def test_allocate_returns_allocation():
    repo, session = FakeRepository([]), FakeSession()
    services.add_batch("batch1", "COMPLICATED-LAMP", 100, None, repo, session)
    result = services.allocate("o1", "COMPLICATED-LAMP", 10, repo, session)
    assert result == "batch1"


def test_allocate_errors_for_invalid_sku():
    repo, session = FakeRepository([]), FakeSession()
    services.add_batch("b1", "AREALSKU", 100, None, repo, session)

    with pytest.raises(services.InvalidSku, match="Invalid sku NONEXISTENTSKU"):
        services.allocate("o1", "NONEXISTENTSKU", 10, repo, FakeSession())
```

这是一个非常好的处境。我们的服务层测试仅依赖于服务层本身，让我们完全自由地根据需要重构模型。


## <font color='red'>将改进贯彻到端到端测试</font>

就像添加`add_batch`有助于将我们的服务层测试与模型分离一样，添加API端点来添加批次将消除对丑陋的`add_stock`fixture的需要，并且我们的 E2E 测试可以摆脱那些硬编码的 SQL 查询和对数据库的直接依赖。

得益于我们的服务函数，添加端点非常容易，只需进行一点JSON处理和一个函数调用即可：

``` py title='添加批次的 API（entrypoints/flask_app.py）'
@app.route("/add_batch", methods=["POST"])
def add_batch():
    session = get_session()
    repo = repository.SqlAlchemyRepository(session)
    eta = request.json["eta"]
    if eta is not None:
        eta = datetime.fromisoformat(eta).date()
    services.add_batch(
        request.json["ref"],
        request.json["sku"],
        request.json["qty"],
        eta,
        repo,
        session,
    )
    return "OK", 201
```

!!! note 
    您是否在想，POST到/add_batch？这不太符合 REST 风格！您说得对。我们很随意，但如果您想让一切更符合 REST 风格，也许 POST 到/batches，那就尽情发挥吧！因为 Flask 是一个薄适配器，所以这很容易。

并且我们在conftest.py中的硬编码 SQL 查询被一些 API 调用所取代，这意味着 API 测试除了 API 之外没有其他依赖项，这也很好：

``` py title='API 测试现在可以添加自己的批次（tests/e2e/test_api.py）'
def post_to_add_batch(ref, sku, qty, eta):
    url = config.get_api_url()
    r = requests.post(
        f"{url}/add_batch", json={"ref": ref, "sku": sku, "qty": qty, "eta": eta}
    )
    assert r.status_code == 201


@pytest.mark.usefixtures("postgres_db")
@pytest.mark.usefixtures("restart_api")
def test_happy_path_returns_201_and_allocated_batch():
    sku, othersku = random_sku(), random_sku("other")
    earlybatch = random_batchref(1)
    laterbatch = random_batchref(2)
    otherbatch = random_batchref(3)
    post_to_add_batch(laterbatch, sku, 100, "2011-01-02")
    post_to_add_batch(earlybatch, sku, 100, "2011-01-01")
    post_to_add_batch(otherbatch, othersku, 100, None)
    data = {"orderid": random_orderid(), "sku": sku, "qty": 3}

    url = config.get_api_url()
    r = requests.post(f"{url}/allocate", json=data)

    assert r.status_code == 201
    assert r.json()["batchref"] == earlybatch
```


## <font color='red'>总结</font>

一旦您有了服务层，您实际上就可以将大部分测试覆盖转移到单元测试并开发健康的测试金字塔。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>回顾：不同类型测试的经验法则</strong></p>
<p><strong>针对每个功能进行一次端到端测试</strong></p>
<p>例如，这可能是针对 HTTP API 编写的。目标是证明该功能有效，并且所有移动部件都正确粘合在一起。</p>
<p><strong>针对服务层编写大部分测试</strong></p>
<p>这些边到边的测试在覆盖率、运行时间和效率之间提供了良好的平衡。每个测试往往覆盖一个功能的一个代码路径，并使用假冒 I/O。这是全面覆盖所有边缘情况以及业务逻辑的来龙去脉的地方。</p>
<p><strong>维护针对领域模型编写的一小部分核心测试</strong></p>
<p>这些测试覆盖范围高度集中，并且比较脆弱，但它们的反馈最多。如果功能稍后被服务层的测试覆盖，不要害怕删除这些测试。</p>
<p><strong>错误处理算是一项功能</strong></p>
<p>理想情况下，您的应用程序的结构应使所有冒泡到入口点（例如 Flask）的错误都以相同的方式处理。这意味着您只需测试每个功能的顺利路径，并为所有不顺利路径保留一个端到端测试（当然还有许多不顺利路径单元测试）。</p>
</div>

以下几件事会有所帮助：

- 用原语而不是领域对象来表达您的服务层。
- 在理想情况下，您将拥有所需的所有服务，能够完全针对服务层进行测试，而不是通过存储库或数据库来破解状态。这对您的端到端测试也有好处。

进入[下一章](./i.Unit%20of%20Work%20Pattern.md)！